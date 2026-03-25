# Communication inter-widgets

## Vue d'ensemble

Grist permet à plusieurs widgets de communiquer via :
1. **Sélection partagée** — `setSelectedRows` / `onRecord`
2. **Curseur partagé** — `setCursorPos` / `onRecord`
3. **Options widget** — `setOption` / `onOptions`
4. **Données partagées** — La table Grist est le "bus" de communication

---

## Sélection partagée

### Émettre une sélection
```javascript
let selectedTaskId = null;

function selectTask(taskId) {
    selectedTaskId = taskId;
    grist.setSelectedRows([taskId]);
    highlightTask(taskId);
}

// Au clic sur une tâche
document.addEventListener('click', (e) => {
    const taskEl = e.target.closest('[data-task-id]');
    if (taskEl) {
        selectTask(parseInt(taskEl.dataset.taskId));
    }
});
```

### Recevoir une sélection (depuis un autre widget)
```javascript
grist.onRecord((record) => {
    if (!record) return;

    // Éviter la boucle infinie
    if (record.id === selectedTaskId) return;

    selectedTaskId = record.id;
    highlightTask(record.id);
    scrollToTask(record.id);
});
```

### Pattern complet (TaskFlow)
```javascript
// État global
let selectedTaskId = null;

// Initialisation
grist.ready({ requiredAccess: 'full' });

// Réception de sélection externe
grist.onRecord((record) => {
    if (record?.id && record.id !== selectedTaskId) {
        selectedTaskId = record.id;

        // Highlight visuel
        document.querySelectorAll('.task').forEach(el => {
            el.classList.toggle('selected',
                parseInt(el.dataset.taskId) === selectedTaskId
            );
        });
    }
});

// Émission au clic
function onTaskClick(taskId) {
    selectedTaskId = taskId;
    grist.setSelectedRows([taskId]);
}
```

---

## Curseur partagé (navigation)

### Configuration "Select By"
Dans Grist, configurer le widget avec "Select By" pour recevoir le curseur d'un autre widget.

### Pattern navigation (grist-navigation-widgets)
```javascript
// Widget Navigation - Émettre la page sélectionnée
async function selectPage(pageId) {
    await grist.setCursorPos({ rowId: pageId });
}

// Widget Content - Recevoir et filtrer
grist.onRecord((record) => {
    if (record) {
        currentPageId = record.id;
        filterContentByPage(currentPageId);
    }
});
```

---

## Options widget

### Stocker des options persistantes
```javascript
// Sauvegarder une option (persiste avec le widget)
await grist.setOption('theme', 'dark');
await grist.setOption('filters', { project: 1, priority: 2 });

// Récupérer les options
grist.onOptions((options) => {
    if (options?.theme) applyTheme(options.theme);
    if (options?.filters) applyFilters(options.filters);
});
```

### Pattern complet
```javascript
let currentOptions = {};

grist.onOptions((options) => {
    currentOptions = options || {};
    applyOptions(currentOptions);
});

async function saveOption(key, value) {
    currentOptions[key] = value;
    await grist.setOption(key, value);
}
```

---

## Pattern : Multi-widgets synchronisés

### Architecture (3 widgets TaskFlow)
```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Kanban    │     │    Gantt    │     │  Calendar   │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       ▼                   ▼                   ▼
    setSelectedRows     onRecord           onRecord
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                    ┌──────▼──────┐
                    │    Grist    │
                    │  (Tasks)    │
                    └─────────────┘
```

### Code identique dans chaque widget
```javascript
// Chaque widget utilise ce même pattern
let selectedTaskId = null;

// Émission (au clic)
function selectTask(taskId) {
    if (taskId === selectedTaskId) return;
    selectedTaskId = taskId;
    grist.setSelectedRows([taskId]);
    highlightInDOM(taskId);
}

// Réception (depuis autre widget)
grist.onRecord((record) => {
    if (record?.id && record.id !== selectedTaskId) {
        selectedTaskId = record.id;
        highlightInDOM(record.id);
    }
});
```

---

## Pattern : Navigation + Contenu (grist-navigation-widgets)

### Widget Navigation
```javascript
// Structure table Pages
// { id, nom, icone, ordre }

let pages = [];
let currentPageId = null;

grist.onRecords((data) => {
    pages = convertToRows(data);
    renderNavigation();
});

async function navigateTo(pageId) {
    currentPageId = pageId;
    await grist.setCursorPos({ rowId: pageId });
    highlightCurrentPage();
}
```

### Widget Contenu
```javascript
// Structure table Contenu
// { id, page (Ref:Pages), titre, contenu }

let allContent = [];
let filteredContent = [];

grist.onRecords((data) => {
    allContent = convertToRows(data);
    filterByCurrentPage();
});

grist.onRecord((record) => {
    // record = la page sélectionnée dans Navigation
    if (record) {
        currentPageId = record.id;
        filterByCurrentPage();
    }
});

function filterByCurrentPage() {
    filteredContent = allContent.filter(c => c.page === currentPageId);
    render();
}
```

---

## Pattern : App Runtime (Artefactory)

### Objet window.app
```javascript
window.app = {
    // Navigation entre composants
    navigate(path) {
        window.parent.postMessage({
            type: 'app-navigate',
            path
        }, '*');
    },

    // Événements inter-composants
    emit(eventName, data) {
        window.parent.postMessage({
            type: 'app-event',
            event: eventName,
            data
        }, '*');
    },

    on(eventName, callback) {
        window.addEventListener('message', (e) => {
            if (e.data?.type === 'app-event' && e.data.event === eventName) {
                callback(e.data.data);
            }
        });
    },

    // État partagé
    state: {},

    // Manifest de l'app
    manifest: null
};
```

### Usage dans les composants
```javascript
// Composant A : émettre un événement
function onBatimentSelect(batId) {
    appContext.app.emit('batiment-selected', { id: batId });
}

// Composant B : recevoir l'événement
appContext.app.on('batiment-selected', (data) => {
    loadLocaux(data.id);
});

// Navigation
document.getElementById('btnLocaux').onclick = () => {
    appContext.app.navigate('/locaux');
};
```

---

## Détection du contexte

### Widget standalone vs intégré
```javascript
const appContext = {
    // Détecte si on est dans Artefactory
    isArtefactory: typeof window.app !== 'undefined' &&
                   typeof window.app.navigate === 'function',

    // API avec fallback
    app: window.app || {
        navigate: () => console.log('Navigation non disponible'),
        emit: () => {},
        on: () => {},
        state: {}
    }
};

// Usage
if (appContext.isArtefactory) {
    appContext.app.emit('ready', { component: 'Dashboard' });
}
```
