# Grist API - Patterns de base

## Initialisation

### Widget simple (lecture seule)
```javascript
grist.ready({ requiredAccess: 'read table' });
```

### Widget avec écriture
```javascript
grist.ready({ requiredAccess: 'full' });
```

### Niveaux d'accès
| Niveau | Capacités |
|--------|-----------|
| `none` | Aucun accès aux données |
| `read table` | Lecture de la table liée uniquement |
| `full` | Lecture/écriture sur tout le document |

---

## Lecture de données

### onRecords — Données de la table liée
```javascript
grist.onRecords((data, mappings) => {
    // data = format colonaire { id: [...], titre: [...], ... }
    // mappings = correspondance colonnes configurées
    const records = convertToRows(data);
    render(records);
});
```

### fetchTable — Lecture directe d'une table
```javascript
async function loadTable(tableName) {
    const data = await grist.docApi.fetchTable(tableName);
    return convertToRows(data);
}

// Exemples
const tasks = await loadTable('Tasks');
const team = await loadTable('Team');
```

### listTables — Liste des tables du document
```javascript
const tables = await grist.docApi.listTables();
// ['Tasks', 'Team', 'Projects', ...]
```

---

## Écriture de données

### AddRecord — Créer un enregistrement
```javascript
await grist.docApi.applyUserActions([
    ['AddRecord', 'Tasks', null, {
        titre: 'Nouvelle tâche',
        statut: 'todo',
        priorite: 3,
        dateDebut: Math.floor(Date.now() / 1000)
    }]
]);
```

### UpdateRecord — Modifier un enregistrement
```javascript
await grist.docApi.applyUserActions([
    ['UpdateRecord', 'Tasks', taskId, {
        statut: 'done',
        progression: 100
    }]
]);
```

### RemoveRecord — Supprimer un enregistrement
```javascript
await grist.docApi.applyUserActions([
    ['RemoveRecord', 'Tasks', taskId]
]);
```

### Actions multiples
```javascript
await grist.docApi.applyUserActions([
    ['UpdateRecord', 'Tasks', 1, { statut: 'done' }],
    ['UpdateRecord', 'Tasks', 2, { statut: 'done' }],
    ['AddRecord', 'Tasks', null, { titre: 'Nouvelle' }]
]);
```

---

## Création de schéma

**Voir [schema.md](schema.md)** pour la documentation complète :
- Ordre de création des tables (refs)
- Labels, widgetOptions, choices
- Configuration des Ref/RefList
- Colonnes calculées (formules)
- Pattern `ensureSchema()` complet

---

## Mode démo (fallback sans Grist)

```javascript
let isDemo = false;

async function init() {
    try {
        grist.ready({ requiredAccess: 'full' });
        await loadData();
    } catch (e) {
        console.log('Mode démo activé');
        isDemo = true;
        loadDemoData();
    }
}

function loadDemoData() {
    return [
        { id: 1, titre: 'Tâche exemple 1', statut: 'todo' },
        { id: 2, titre: 'Tâche exemple 2', statut: 'done' }
    ];
}
```
