# Guide d'architecture minimale — Runtime Artefactory

Ce document décrit **exactement** ce qu'il faut reproduire dans un `app.html` minimal pour exécuter les templates existants, sans l'éditeur Ace, sans l'assistant IA, sans le mode Edit.

---

## 1. Vue d'ensemble du flux d'exécution

```
app.html (parent Grist widget)
  │
  ├─ 1. grist.ready({ requiredAccess: 'full' })
  ├─ 2. Charge la table "Artefacts" via fetchTable()
  ├─ 3. Identifie les artefacts de type "app" → parse leur Code comme JSON manifest
  ├─ 4. Sélectionne une app → lit manifest.routes
  ├─ 5. Construit la sidebar (icônes + labels depuis les routes)
  ├─ 6. Au clic sur une route :
  │     a. Trouve l'artefact correspondant (route.artefact → id)
  │     b. Prend son Code (HTML brut)
  │     c. Injecte le GRIST_BRIDGE_SCRIPT (remplace le tag grist-plugin-api.js)
  │     d. Injecte le APP_RUNTIME_SCRIPT (window.app dans le <head>)
  │     e. Écrit le HTML dans iframe.srcdoc
  │     f. Connecte le GristBridgeParent à cet iframe
  └─ 7. Écoute les postMessage de l'iframe (navigation, événements)
```

---

## 2. Les 5 briques essentielles

### BRIQUE 1 — Chargement des données depuis Grist

**Table requise** : `Artefacts` avec les colonnes :
- `Nom` (Text), `Type` (Choice), `Code` (Text), `Dependencies` (Text)

**Conversion colonaire → objets** (identique au pattern tasks_app) :
```javascript
const TABLE_NAME = 'Artefacts';

async function loadArtefacts() {
    const data = await grist.docApi.fetchTable(TABLE_NAME);
    const items = [];
    const apps = [];
    const n = data.id ? data.id.length : 0;

    for (let i = 0; i < n; i++) {
        const art = {
            id: data.id[i],
            Nom: data.Nom?.[i] || '',
            Type: data.Type?.[i] || '',
            Code: data.Code?.[i] || '',
        };
        items.push(art);
        if (art.Type === 'app') apps.push(art);
    }

    state.artefacts = items;
    state.apps = apps;
}
```

Les artefacts de `Type === 'app'` contiennent un manifest JSON dans leur colonne `Code`.

---

### BRIQUE 2 — GristBridgeParent (côté parent)

**Problème résolu** : Les iframes sandboxées ne peuvent pas charger `grist-plugin-api.js` depuis le parent. Le bridge intercepte les appels et les relaie.

**Classe côté parent** (dans app.html) — **à copier intégralement** :

```javascript
class GristBridgeParent {
    constructor() {
        this.currentIframe = null;
        this.callbacks = new Map();
        this.nextId = 1;
        window.addEventListener('message', this._handleMessage.bind(this));
    }

    setCurrentIframe(iframe) {
        this.callbacks.clear();
        this.currentIframe = iframe;
    }

    async _handleMessage(event) {
        const data = event.data;
        if (!data || data.type !== 'grist-bridge') return;
        if (!this.currentIframe || event.source !== this.currentIframe.contentWindow) return;

        try {
            let result;
            switch (data.action) {
                case 'ready': result = { success: true }; break;
                case 'apiCall': result = await this._handleApiCall(data); break;
                case 'registerCallback': result = await this._handleRegisterCallback(data); break;
                default: throw new Error('Unknown action');
            }
            this._sendResponse(data.callId, result, null);
        } catch (error) {
            this._sendResponse(data.callId, null, error.message);
        }
    }

    _sendResponse(callId, result, error) {
        if (!this.currentIframe) return;
        this.currentIframe.contentWindow.postMessage(
            { type: 'grist-bridge-response', callId, result, error }, '*'
        );
    }

    _sendCallback(callbackId, args) {
        if (!this.currentIframe) return;
        this.currentIframe.contentWindow.postMessage(
            { type: 'grist-bridge-callback', callbackId, args }, '*'
        );
    }

    async _handleApiCall(data) {
        const { path, args } = data;
        const parts = path.split('.');
        let target = grist;
        for (let i = 0; i < parts.length - 1; i++) {
            target = target[parts[i]];
            if (!target) throw new Error(`Invalid path: ${path}`);
        }
        const method = target[parts[parts.length - 1]];
        if (typeof method !== 'function') throw new Error(`${path} is not a function`);
        return await method.apply(target, args || []);
    }

    async _handleRegisterCallback(data) {
        const { callbackType } = data;
        const callbackId = 'cb_' + (this.nextId++);
        const self = this;

        if (callbackType === 'onRecords') {
            grist.onRecords((records, mappings) => self._sendCallback(callbackId, [records, mappings]));
        } else if (callbackType === 'onRecord') {
            grist.onRecord((record, mappings) => self._sendCallback(callbackId, [record, mappings]));
        }

        return { callbackId };
    }
}
```

**Protocole de messages** :
```
iframe → parent : { type: 'grist-bridge', action: 'apiCall', callId, path, args }
parent → iframe : { type: 'grist-bridge-response', callId, result, error }
parent → iframe : { type: 'grist-bridge-callback', callbackId, args }  (pour onRecords/onRecord)
```

---

### BRIQUE 3 — GRIST_BRIDGE_SCRIPT (côté iframe, injecté)

Ce script **remplace** le tag `<script src="grist-plugin-api.js">` dans le HTML de l'artefact. Il crée un faux objet `window.grist` qui communique avec le parent via postMessage.

**À copier intégralement comme string** :

```javascript
const GRIST_BRIDGE_SCRIPT = `
(function() {
    const pending = new Map();
    const callbacks = new Map();
    let nextId = 1;

    window.addEventListener('message', (event) => {
        const data = event.data;
        if (data?.type === 'grist-bridge-response') {
            const p = pending.get(data.callId);
            if (p) {
                pending.delete(data.callId);
                data.error ? p.reject(new Error(data.error)) : p.resolve(data.result);
            }
        }
        if (data?.type === 'grist-bridge-callback') {
            const cb = callbacks.get(data.callbackId);
            if (cb) cb.apply(null, data.args || []);
        }
    });

    function send(action, payload) {
        return new Promise((resolve, reject) => {
            const callId = 'call_' + (nextId++);
            pending.set(callId, { resolve, reject });
            window.parent.postMessage({ type: 'grist-bridge', action, callId, ...payload }, '*');
            setTimeout(() => {
                if (pending.has(callId)) { pending.delete(callId); reject(new Error('Timeout')); }
            }, 30000);
        });
    }

    async function registerCallback(type, fn) {
        const result = await send('registerCallback', { callbackType: type });
        callbacks.set(result.callbackId, fn);
    }

    window.grist = {
        ready: (options) => send('ready', { options }),
        docApi: {
            listTables: () => send('apiCall', { path: 'docApi.listTables', args: [] }),
            fetchTable: (t) => send('apiCall', { path: 'docApi.fetchTable', args: [t] }),
            applyUserActions: (a) => send('apiCall', { path: 'docApi.applyUserActions', args: [a] })
        },
        onRecords: (cb) => registerCallback('onRecords', cb),
        onRecord: (cb) => registerCallback('onRecord', cb)
    };

    console.log('🌉 Grist Bridge ready');
})();
`;
```

**Injection** — remplace le tag script Grist par le bridge :
```javascript
function injectGristBridge(html) {
    return html.replace(
        /<script\s+src=["']https:\/\/docs\.getgrist\.com\/grist-plugin-api\.js["']\s*><\/script>/gi,
        '<script>' + GRIST_BRIDGE_SCRIPT + '<\/script>'
    );
}
```

---

### BRIQUE 4 — APP_RUNTIME_SCRIPT (côté iframe, injecté)

Crée `window.app` dans l'iframe pour permettre :
- **Navigation** : `app.navigate('/batiments')` → postMessage vers le parent → `navigateToRoute()`
- **Événements** : `app.emit('refresh', data)` → parent peut broadcaster aux autres iframes
- **État partagé** : `app.state`, `app.manifest`
- **Détection de contexte** : les templates utilisent `typeof window.app !== 'undefined'` pour savoir s'ils sont dans Artefactory

**À copier intégralement comme string** :

```javascript
const APP_RUNTIME_SCRIPT = `
(function() {
    const eventHandlers = new Map();

    window.app = {
        navigate: (path) => {
            window.parent.postMessage({ type: 'app-navigate', path }, '*');
        },
        getCurrentRoute: () => window.__APP_ROUTE__ || '/',

        emit: (event, data) => {
            window.parent.postMessage({ type: 'app-event', event, data }, '*');
        },
        on: (event, callback) => {
            if (!eventHandlers.has(event)) eventHandlers.set(event, []);
            eventHandlers.get(event).push(callback);
        },

        state: window.__APP_STATE__ || {},
        setState: (key, value) => {
            window.parent.postMessage({ type: 'app-setState', key, value }, '*');
        },

        manifest: window.__APP_MANIFEST__ || null,
        isEditMode: false
    };

    window.addEventListener('message', (event) => {
        const data = event.data;
        if (data?.type === 'app-event-broadcast') {
            const handlers = eventHandlers.get(data.event) || [];
            handlers.forEach(h => h(data.data));
        }
    });
})();
`;
```

**Injection** — insère le manifest + route + script avant `</head>` :
```javascript
function injectAppRuntime(html, manifest, route) {
    const runtimeInit = `
        <script>
            window.__APP_MANIFEST__ = ${JSON.stringify(manifest)};
            window.__APP_ROUTE__ = "${route}";
            window.__APP_STATE__ = {};
            ${APP_RUNTIME_SCRIPT}
        <\/script>
    `;

    if (html.includes('</head>')) {
        return html.replace('</head>', runtimeInit + '</head>');
    }
    return runtimeInit + html;
}
```

---

### BRIQUE 5 — Routing + Sidebar + Rendu

#### 5a. Parsing du manifest

Quand une app est sélectionnée, son `Code` est parsé comme JSON :
```javascript
app.manifest = JSON.parse(app.Code);
```

Structure attendue du manifest (voir `templates/manifest.json`) :
```json
{
    "name": "Nom de l'app",
    "icon": "🏛️",
    "layout": "sidebar",
    "theme": { "primary": "#16b378" },
    "routes": [
        { "path": "/", "label": "Accueil", "icon": "🏠", "artefact": 27 },
        { "path": "/batiments", "label": "Bâtiments", "icon": "🏢", "artefact": 28 }
    ]
}
```

Chaque `route.artefact` est un **ID d'enregistrement** dans la table Artefacts.

#### 5b. Construction de la sidebar

```javascript
function loadAppRuntime(app) {
    const manifest = app.manifest;

    // Logo + titre
    document.getElementById('runtimeLogo').textContent = manifest.icon || '📦';
    document.getElementById('runtimeTitle').textContent = manifest.name || 'Application';

    // Construire les items de navigation
    const nav = document.getElementById('runtimeNav');
    nav.innerHTML = '';
    (manifest.routes || []).forEach(route => {
        const item = document.createElement('div');
        item.className = 'nav-item' + (route.path === state.currentRoute ? ' active' : '');
        item.innerHTML = `
            <span class="nav-item-icon">${route.icon || '📄'}</span>
            <span class="nav-item-label">${route.label || route.path}</span>
        `;
        item.onclick = () => navigateToRoute(route.path);
        nav.appendChild(item);
    });

    // Charger la route initiale
    navigateToRoute(state.currentRoute);
}
```

#### 5c. Navigation vers une route

```javascript
function navigateToRoute(path) {
    state.currentRoute = path;
    if (!state.currentApp?.manifest) return;

    const route = state.currentApp.manifest.routes?.find(r => r.path === path);

    // Mettre à jour l'état actif dans la sidebar
    document.querySelectorAll('.nav-item').forEach((item, i) => {
        item.classList.toggle('active', state.currentApp.manifest.routes[i]?.path === path);
    });

    // Breadcrumb
    document.getElementById('runtimeBreadcrumb').textContent = route?.label || path;

    // Trouver et rendre l'artefact
    if (route?.artefact) {
        const artefact = state.artefacts.find(a => a.id === route.artefact);
        if (artefact) {
            renderInRuntime(artefact);
        }
    }
}
```

#### 5d. Rendu dans l'iframe

```javascript
function renderInRuntime(artefact) {
    const iframe = document.getElementById('runtimeIframe');
    let html = artefact.Code || '';

    // 1. Injecter window.app (navigation, events, state)
    if (state.currentApp?.manifest) {
        html = injectAppRuntime(html, state.currentApp.manifest, state.currentRoute);
    }

    // 2. Remplacer grist-plugin-api.js par le bridge
    if (html.includes('grist-plugin-api.js')) {
        html = injectGristBridge(html);
    }

    // 3. Connecter le bridge parent à cet iframe
    state.gristBridge.setCurrentIframe(iframe);

    // 4. Injecter le HTML
    iframe.srcdoc = html;
}
```

**Ordre d'injection critique** :
1. D'abord `injectAppRuntime` (ajoute `window.__APP_MANIFEST__` + `window.app` dans `<head>`)
2. Ensuite `injectGristBridge` (remplace le tag `grist-plugin-api.js` par le bridge)
3. Puis `setCurrentIframe` (route les messages bridge vers ce iframe)
4. Enfin `iframe.srcdoc = html` (lance l'exécution)

---

## 3. Écoute des messages du parent (onMessage)

Le parent doit écouter les `postMessage` venant des iframes :

```javascript
window.addEventListener('message', function(e) {
    const d = e.data;
    if (!d || !d.type) return;

    // Navigation demandée par un template (app.navigate('/path'))
    if (d.type === 'app-navigate') {
        navigateToRoute(d.path);
    }

    // Événement émis par un template (app.emit('event', data))
    if (d.type === 'app-event') {
        // Optionnel : broadcaster vers d'autres iframes si multi-iframe
        console.log('App event:', d.event, d.data);
    }
});
```

---

## 4. Structure HTML minimale du runtime

```html
<body>
    <!-- Sidebar + Content -->
    <div class="runtime-container" id="runtimeContainer">
        <div class="runtime-main">
            <!-- Sidebar -->
            <div class="runtime-sidebar" id="runtimeSidebar">
                <div class="sidebar-header">
                    <div class="sidebar-logo" id="runtimeLogo">📦</div>
                    <div class="sidebar-title" id="runtimeTitle">Application</div>
                </div>
                <nav class="sidebar-nav" id="runtimeNav">
                    <!-- Items générés dynamiquement -->
                </nav>
            </div>

            <!-- Zone de contenu -->
            <div class="runtime-content">
                <div class="runtime-toolbar">
                    <span id="runtimeBreadcrumb">Accueil</span>
                </div>
                <div class="runtime-frame">
                    <iframe class="runtime-iframe" id="runtimeIframe"
                            sandbox="allow-scripts allow-same-origin"></iframe>
                </div>
            </div>
        </div>
    </div>
</body>
```

---

## 5. CSS minimal du runtime

```css
* { box-sizing: border-box; margin: 0; padding: 0; }
body { font-family: system-ui, sans-serif; min-height: 100vh; display: flex; flex-direction: column; overflow: hidden; }

.runtime-container { flex: 1; display: flex; flex-direction: column; overflow: hidden; }
.runtime-main { flex: 1; display: flex; overflow: hidden; }

/* Sidebar */
.runtime-sidebar { width: 220px; background: #fff; border-right: 1px solid #e2e8f0; display: flex; flex-direction: column; flex-shrink: 0; }
.sidebar-header { padding: 16px; border-bottom: 1px solid #e2e8f0; display: flex; align-items: center; gap: 10px; }
.sidebar-logo { width: 36px; height: 36px; border-radius: 10px; background: linear-gradient(135deg, #059669, #10b981); display: flex; align-items: center; justify-content: center; color: #fff; font-size: 20px; }
.sidebar-title { font-size: 14px; font-weight: 700; }
.sidebar-nav { flex: 1; overflow-y: auto; padding: 8px; }

/* Items de navigation */
.nav-item { padding: 10px 12px; border-radius: 8px; cursor: pointer; display: flex; align-items: center; gap: 10px; font-size: 13px; transition: all 0.15s; margin-bottom: 4px; }
.nav-item:hover { background: #f1f5f9; }
.nav-item.active { background: #3b82f6; color: #fff; }
.nav-item-icon { font-size: 18px; flex-shrink: 0; }
.nav-item-label { white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }

/* Contenu */
.runtime-content { flex: 1; display: flex; flex-direction: column; overflow: hidden; background: #f8fafc; }
.runtime-toolbar { padding: 8px 16px; background: #fff; border-bottom: 1px solid #e2e8f0; font-size: 13px; color: #64748b; }
.runtime-frame { flex: 1; overflow: hidden; position: relative; }
.runtime-iframe { width: 100%; height: 100%; border: none; background: #fff; }
```

---

## 6. Séquence d'initialisation minimale

```javascript
const state = {
    artefacts: [],
    apps: [],
    currentApp: null,
    currentRoute: '/',
    gristBridge: null,
};

async function init() {
    // 1. Créer le bridge
    state.gristBridge = new GristBridgeParent();

    // 2. Écouter les messages des iframes
    window.addEventListener('message', onMessage);

    // 3. Se connecter à Grist
    grist.ready({ requiredAccess: 'full' });

    // 4. Charger les artefacts
    await loadArtefacts();

    // 5. Auto-sélectionner la première app
    if (state.apps.length > 0) {
        const app = state.apps[0];
        try { app.manifest = JSON.parse(app.Code); } catch { app.manifest = { routes: [] }; }
        state.currentApp = app;
        loadAppRuntime(app);
    }
}

init();
```

---

## 7. Ce que les templates attendent

Chaque template (Accueil, batiments, etc.) repose sur ces deux APIs injectées :

### API Grist (via bridge)
```javascript
// Ces appels passent par le bridge postMessage → parent → vrai grist API
await grist.docApi.fetchTable('Batiments');
await grist.docApi.applyUserActions([['UpdateRecord', 'Batiments', id, data]]);
```

### API App (via runtime script)
```javascript
// Détection de contexte
const appContext = {
    isArtefactory: typeof window.app !== 'undefined' && window.app.navigate,
    app: window.app || { navigate: () => {}, emit: () => {}, on: () => {} }
};

// Navigation (le parent interprète et change de route)
appContext.app.navigate('/batiments');

// Événements inter-widgets
appContext.app.emit('batiment-selected', { id: 42 });
appContext.app.on('batiment-selected', (data) => { ... });

// Accès au manifest
const manifest = window.app?.manifest;
```

---

## 8. Résumé : fichiers et dépendances pour le runtime minimal

### Fichier unique à créer : `app.html`
Contient :
1. **CSS** (~40 lignes) : sidebar + content layout
2. **HTML** (~20 lignes) : sidebar-header, sidebar-nav, iframe
3. **JS** (~200 lignes) :
   - `GristBridgeParent` (classe, ~70 lignes)
   - `GRIST_BRIDGE_SCRIPT` (string injectée, ~40 lignes)
   - `APP_RUNTIME_SCRIPT` (string injectée, ~30 lignes)
   - `injectGristBridge()` + `injectAppRuntime()` (~15 lignes)
   - `loadArtefacts()` (~20 lignes)
   - `loadAppRuntime()` + `navigateToRoute()` + `renderInRuntime()` (~40 lignes)
   - `onMessage()` (~10 lignes)
   - `init()` (~15 lignes)

### Dépendance CDN unique
```html
<script src="https://docs.getgrist.com/grist-plugin-api.js"></script>
```
(Tailwind, Chart.js etc. sont chargés par les templates eux-mêmes, pas par app.html)

### Table Grist requise
`Artefacts` avec au minimum : `Nom`, `Type`, `Code`

### Templates existants (inchangés)
Les fichiers `templates/` fonctionneront tels quels car le runtime minimal expose exactement les mêmes APIs (`window.grist` via bridge + `window.app` via runtime script).
