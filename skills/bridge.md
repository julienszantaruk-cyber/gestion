# GristBridge - Communication iframe sandboxée

## Problème

Les widgets Grist s'exécutent dans des iframes sandboxées. Un widget qui charge du contenu dans une **sous-iframe** (ex: Artefactory) ne peut pas accéder directement à l'API Grist depuis cette sous-iframe.

```
┌─────────────────────────────────────────────┐
│ Grist (parent)                              │
│  └── Widget iframe (accès API Grist)        │
│       └── Sous-iframe (PAS d'accès API)     │ ← Problème !
└─────────────────────────────────────────────┘
```

## Solution : GristBridge

Le bridge intercepte les appels API depuis les sous-iframes et les route vers le parent qui a accès à Grist.

```
Sous-iframe              Widget (parent)           Grist
    │                         │                      │
    ├─ postMessage ──────────►│                      │
    │  {grist-bridge, fetch}  │── grist.docApi ─────►│
    │                         │◄── data ─────────────┤
    │◄── postMessage ─────────┤                      │
    │   {result: data}        │                      │
```

---

## Implémentation côté parent

### Classe GristBridgeParent
```javascript
class GristBridgeParent {
    constructor() {
        this.callbackSources = new Map();
        this.nextId = 1;
        window.addEventListener('message', this._handleMessage.bind(this));
    }

    async _handleMessage(event) {
        const data = event.data;
        if (!data || data.type !== 'grist-bridge') return;

        const source = event.source;
        if (!source) return;

        const { action, args, callbackId } = data;

        try {
            let result;
            switch (action) {
                case 'fetchTable':
                    result = await grist.docApi.fetchTable(args.tableName);
                    break;
                case 'applyUserActions':
                    result = await grist.docApi.applyUserActions(args.actions);
                    break;
                case 'listTables':
                    result = await grist.docApi.listTables();
                    break;
                default:
                    throw new Error(`Action inconnue: ${action}`);
            }

            source.postMessage({
                type: 'grist-bridge-response',
                callbackId,
                result
            }, '*');
        } catch (error) {
            source.postMessage({
                type: 'grist-bridge-response',
                callbackId,
                error: error.message
            }, '*');
        }
    }
}

// Initialiser le bridge
const gristBridge = new GristBridgeParent();
```

---

## Script injecté dans les sous-iframes

### GRIST_BRIDGE_SCRIPT
```javascript
const GRIST_BRIDGE_SCRIPT = `
(function() {
    const callbacks = new Map();
    let nextId = 1;

    window.addEventListener('message', function(event) {
        const data = event.data;
        if (!data || data.type !== 'grist-bridge-response') return;

        const cb = callbacks.get(data.callbackId);
        if (cb) {
            callbacks.delete(data.callbackId);
            if (data.error) {
                cb.reject(new Error(data.error));
            } else {
                cb.resolve(data.result);
            }
        }
    });

    function bridgeCall(action, args) {
        return new Promise(function(resolve, reject) {
            const callbackId = 'cb_' + (nextId++);
            callbacks.set(callbackId, { resolve, reject });

            window.parent.postMessage({
                type: 'grist-bridge',
                action: action,
                args: args,
                callbackId: callbackId
            }, '*');

            // Timeout
            setTimeout(function() {
                if (callbacks.has(callbackId)) {
                    callbacks.delete(callbackId);
                    reject(new Error('Bridge timeout'));
                }
            }, 30000);
        });
    }

    // API compatible avec grist.docApi
    window.grist = {
        ready: function() {},
        docApi: {
            fetchTable: function(tableName) {
                return bridgeCall('fetchTable', { tableName: tableName });
            },
            applyUserActions: function(actions) {
                return bridgeCall('applyUserActions', { actions: actions });
            },
            listTables: function() {
                return bridgeCall('listTables', {});
            }
        }
    };
})();
`;
```

---

## Injection dans l'iframe

### Lors du rendu
```javascript
function renderInIframe(iframe, htmlContent) {
    // Injecter le bridge avant le contenu
    const bridgedHtml = htmlContent.replace(
        '<head>',
        `<head><script>${GRIST_BRIDGE_SCRIPT}</script>`
    );

    iframe.srcdoc = bridgedHtml;
}
```

### Pattern complet
```javascript
function createSandboxedWidget(container, html) {
    const iframe = document.createElement('iframe');
    iframe.sandbox = 'allow-scripts allow-same-origin';
    iframe.style.cssText = 'width:100%;border:none;';

    // Injecter le bridge
    const injectedHtml = `
        <!DOCTYPE html>
        <html>
        <head>
            <script>${GRIST_BRIDGE_SCRIPT}</script>
        </head>
        <body>
            ${html}
        </body>
        </html>
    `;

    iframe.srcdoc = injectedHtml;
    container.appendChild(iframe);

    return iframe;
}
```

---

## Iframes imbriquées (3 niveaux)

### Problème
Pour une iframe de niveau 3, le `window.parent` pointe vers le niveau 2, pas vers le widget principal.

```
Grist
└── Widget (niveau 1) ← a l'API Grist
    └── Composant (niveau 2)
        └── Sous-composant (niveau 3) ← doit atteindre niveau 1
```

### Solution : Retarget
Avant d'injecter le HTML, remplacer les appels `window.parent` :

```javascript
function prepareNestedHtml(html) {
    // Retargeter vers le grand-parent
    return html.replace(
        /window\.parent\.postMessage/g,
        'window.parent.parent.postMessage'
    );
}
```

### Pattern complet pour inclusion de composants
```javascript
const AUTORESIZE_SCRIPT = `
<script>
(function() {
    function notifyHeight() {
        var height = document.body.scrollHeight;
        window.parent.postMessage({ type: 'iframe-resize', height: height }, '*');
    }
    window.addEventListener('load', notifyHeight);
    new MutationObserver(notifyHeight).observe(document.body, {
        childList: true, subtree: true
    });
})();
</script>
`;

function prepareNestedHtml(html) {
    // 1. Retargeter le bridge vers le parent du parent
    let prepared = html.replace(
        /window\.parent\.postMessage/g,
        'window.parent.parent.postMessage'
    );

    // 2. Ajouter le script de resize (reste sur parent immédiat)
    prepared = prepared.replace('</body>', AUTORESIZE_SCRIPT + '</body>');

    return prepared;
}
```

---

## Gestion du resize

### Côté parent (écouter les messages de resize)
```javascript
window.addEventListener('message', (event) => {
    if (event.data?.type === 'iframe-resize') {
        const iframe = findIframeBySource(event.source);
        if (iframe) {
            iframe.style.height = event.data.height + 'px';
        }
    }
});

function findIframeBySource(source) {
    return Array.from(document.querySelectorAll('iframe'))
        .find(iframe => iframe.contentWindow === source);
}
```

### Côté iframe (notifier le resize)
```javascript
function notifyResize() {
    const height = document.documentElement.scrollHeight;
    window.parent.postMessage({
        type: 'iframe-resize',
        height: height
    }, '*');
}

// Observer les changements
new ResizeObserver(notifyResize).observe(document.body);
window.addEventListener('load', notifyResize);
```

---

## Sécurité

### Attribut sandbox recommandé
```html
<iframe sandbox="allow-scripts allow-same-origin"></iframe>
```

| Attribut | Effet |
|----------|-------|
| `allow-scripts` | Permet l'exécution de JavaScript |
| `allow-same-origin` | Nécessaire pour postMessage |

### Validation des messages
```javascript
window.addEventListener('message', (event) => {
    // Ignorer les messages sans type connu
    if (!event.data?.type) return;

    // Valider le type
    const validTypes = ['grist-bridge', 'grist-bridge-response', 'iframe-resize'];
    if (!validTypes.includes(event.data.type)) return;

    // Traiter...
});
```
