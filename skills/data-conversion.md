# Conversion de données Grist

## Format colonaire → Objets

Grist envoie les données en **format colonaire** :
```javascript
// Format Grist (colonaire)
{
    id: [1, 2, 3],
    titre: ['Tâche A', 'Tâche B', 'Tâche C'],
    statut: ['todo', 'done', 'todo']
}
```

Les widgets travaillent avec des **objets** :
```javascript
// Format widget (objets)
[
    { id: 1, titre: 'Tâche A', statut: 'todo' },
    { id: 2, titre: 'Tâche B', statut: 'done' },
    { id: 3, titre: 'Tâche C', statut: 'todo' }
]
```

---

## Fonctions de conversion

### Version simple
```javascript
function convertToRows(data) {
    const n = data.id?.length || 0;
    const records = [];
    for (let i = 0; i < n; i++) {
        const record = {};
        Object.keys(data).forEach(key => {
            record[key] = data[key][i];
        });
        records.push(record);
    }
    return records;
}
```

### Version avec gestion d'erreurs
```javascript
function safeConvert(data) {
    if (!data || !data.id) return [];
    const n = data.id.length;
    const records = [];
    for (let i = 0; i < n; i++) {
        const record = { id: data.id[i] };
        Object.keys(data).forEach(key => {
            if (key !== 'id') {
                record[key] = data[key]?.[i] ?? null;
            }
        });
        records.push(record);
    }
    return records;
}
```

### Version avec mapping de colonnes
```javascript
function convertWithMappings(data, mappings) {
    const n = data.id?.length || 0;
    const records = [];
    for (let i = 0; i < n; i++) {
        const record = { id: data.id[i] };
        // Appliquer les mappings configurés dans Grist
        Object.entries(mappings).forEach(([widgetCol, gristCol]) => {
            record[widgetCol] = data[gristCol]?.[i];
        });
        records.push(record);
    }
    return records;
}
```

---

## Dates

### Grist → JavaScript
Grist stocke les dates en **timestamps Unix (secondes)**, JavaScript utilise des **millisecondes**.

```javascript
function gristToDate(timestamp) {
    if (!timestamp) return null;
    return new Date(timestamp * 1000);
}

function gristToISOString(timestamp) {
    if (!timestamp) return '';
    return new Date(timestamp * 1000).toISOString().split('T')[0];
}
```

### JavaScript → Grist
```javascript
function dateToGrist(date) {
    if (!date) return null;
    if (typeof date === 'string') date = new Date(date);
    return Math.floor(date.getTime() / 1000);
}

// Exemples
dateToGrist(new Date())           // Maintenant
dateToGrist('2024-03-15')         // Date ISO
dateToGrist(Date.now())           // Timestamp JS (ms) → Grist (s)
```

### Formatage pour affichage
```javascript
function formatDate(timestamp, locale = 'fr-FR') {
    if (!timestamp) return '';
    return new Date(timestamp * 1000).toLocaleDateString(locale);
}

function formatDateTime(timestamp, locale = 'fr-FR') {
    if (!timestamp) return '';
    return new Date(timestamp * 1000).toLocaleString(locale);
}
```

---

## RefList (références multiples)

### Format Grist
```javascript
// RefList en Grist = ['L', id1, id2, id3]
// Le 'L' indique une liste
{ assignees: ['L', 1, 3, 5] }
```

### Extraction des IDs
```javascript
function getRefListIds(refList) {
    if (!refList) return [];
    if (!Array.isArray(refList)) return [];
    if (refList[0] === 'L') {
        return refList.slice(1);
    }
    return refList;
}

// Usage
const assigneeIds = getRefListIds(task.assignees);
// [1, 3, 5]
```

### Création pour écriture
```javascript
function createRefList(ids) {
    if (!ids || ids.length === 0) return null;
    return ['L', ...ids.map(id => parseInt(id))];
}

// Usage
const refList = createRefList([1, 3, 5]);
// ['L', 1, 3, 5]
```

---

## Ref (référence simple)

```javascript
// Référence simple = juste l'ID (number)
{ projet: 42 }

// Lecture
const projectId = task.projet; // 42

// Écriture
await grist.docApi.applyUserActions([
    ['UpdateRecord', 'Tasks', taskId, { projet: 42 }]
]);
```

---

## Résolution des références

### Joindre les données liées
```javascript
async function loadTasksWithTeam() {
    const [tasksData, teamData] = await Promise.all([
        grist.docApi.fetchTable('Tasks'),
        grist.docApi.fetchTable('Team')
    ]);

    const tasks = convertToRows(tasksData);
    const team = convertToRows(teamData);
    const teamById = Object.fromEntries(team.map(m => [m.id, m]));

    // Enrichir les tâches avec les données team
    return tasks.map(task => ({
        ...task,
        assigneesData: getRefListIds(task.assignees)
            .map(id => teamById[id])
            .filter(Boolean)
    }));
}
```

---

## Escape HTML (sécurité)

```javascript
function escapeHtml(text) {
    if (!text) return '';
    const div = document.createElement('div');
    div.textContent = text;
    return div.innerHTML;
}

// Usage dans les templates
`<div class="title">${escapeHtml(task.titre)}</div>`
```
