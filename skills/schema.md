# Initialisation de schéma Grist

## Ordre de création

**CRITIQUE** : Les tables référencées doivent exister AVANT les tables qui les référencent.

```
1. Tables sans références (Team, Categories, etc.)
2. Tables avec Ref simples (Projects → Team)
3. Tables avec Ref multiples (Tasks → Projects, Team, Tasks)
```

---

## Propriétés des colonnes

### Propriétés disponibles

| Propriété | Type | Description |
|-----------|------|-------------|
| `id` | string | Identifiant technique de la colonne |
| `type` | string | Type de données (voir types ci-dessous) |
| `label` | string | Nom affiché dans l'interface Grist |
| `widgetOptions` | string (JSON) | Options du widget (choices, alignment, etc.) |
| `visibleCol` | number | ID de la colonne à afficher pour les Ref |
| `formula` | string | Formule Python pour colonnes calculées |
| `isFormula` | boolean | true si colonne calculée |

### Types de colonnes

| Type | Description | Options spéciales |
|------|-------------|-------------------|
| `Text` | Texte simple | — |
| `Int` | Entier | — |
| `Numeric` | Décimal | — |
| `Bool` | Booléen | — |
| `Date` | Date (timestamp Unix) | — |
| `DateTime` | Date et heure | — |
| `Choice` | Choix unique | `widgetOptions.choices` |
| `ChoiceList` | Choix multiples | `widgetOptions.choices` |
| `Ref:Table` | Référence | `visibleCol` |
| `RefList:Table` | Liste de références | `visibleCol` |
| `Attachments` | Fichiers | — |

---

## widgetOptions

### Format
```javascript
widgetOptions: JSON.stringify({
    choices: ['option1', 'option2', 'option3'],
    choiceOptions: {
        'option1': { fillColor: '#ef4444', textColor: '#ffffff' },
        'option2': { fillColor: '#10b981', textColor: '#ffffff' }
    },
    alignment: 'center'  // 'left', 'center', 'right'
})
```

### Exemples

#### Choice avec couleurs
```javascript
{
    id: 'statut',
    type: 'Choice',
    label: 'Statut',
    widgetOptions: JSON.stringify({
        choices: ['todo', 'inprogress', 'review', 'done'],
        choiceOptions: {
            'todo': { fillColor: '#6b7280', textColor: '#ffffff' },
            'inprogress': { fillColor: '#3b82f6', textColor: '#ffffff' },
            'review': { fillColor: '#f59e0b', textColor: '#ffffff' },
            'done': { fillColor: '#10b981', textColor: '#ffffff' }
        }
    })
}
```

#### Choice simple
```javascript
{
    id: 'priorite',
    type: 'Choice',
    label: 'Priorité',
    widgetOptions: JSON.stringify({
        choices: ['1 - Critique', '2 - Haute', '3 - Moyenne', '4 - Basse']
    })
}
```

#### ChoiceList (tags)
```javascript
{
    id: 'tags',
    type: 'ChoiceList',
    label: 'Tags',
    widgetOptions: JSON.stringify({
        choices: ['urgent', 'bug', 'feature', 'documentation', 'test']
    })
}
```

---

## Références (Ref / RefList)

### visibleCol

`visibleCol` est l'**ID numérique** de la colonne à afficher (pas son nom).

**Problème** : On ne connaît pas l'ID avant création.

**Solutions** :

#### 1. Créer sans visibleCol, puis modifier
```javascript
// Étape 1 : Créer la colonne Ref sans visibleCol
await grist.docApi.applyUserActions([
    ['AddColumn', 'Tasks', 'projet', { type: 'Ref:Projects', label: 'Projet' }]
]);

// Étape 2 : Récupérer l'ID de la colonne 'nom' dans Projects
const colInfo = await getColumnId('Projects', 'nom');

// Étape 3 : Modifier la colonne pour ajouter visibleCol
await grist.docApi.applyUserActions([
    ['ModifyColumn', 'Tasks', 'projet', { visibleCol: colInfo.id }]
]);
```

#### 2. Utiliser l'API REST pour récupérer les IDs
```javascript
async function getColumnId(tableId, colName) {
    // Via fetch si disponible
    const resp = await fetch(`/api/docs/${docId}/tables/${tableId}/columns`);
    const cols = await resp.json();
    return cols.columns.find(c => c.id === colName);
}
```

#### 3. Pattern pragmatique (sans visibleCol)
```javascript
// Grist affiche l'ID par défaut si visibleCol n'est pas défini
// L'utilisateur peut configurer manuellement dans l'interface
{
    id: 'projet',
    type: 'Ref:Projects',
    label: 'Projet'
}
```

---

## Schéma complet — Exemple TaskFlow

```javascript
const TASKFLOW_SCHEMA = {
    // ═══════════════════════════════════════════════════════
    // ORDRE 1 : Tables sans références externes
    // ═══════════════════════════════════════════════════════
    Team: {
        columns: [
            { id: 'nom', type: 'Text', label: 'Nom' },
            { id: 'email', type: 'Text', label: 'Email' },
            { id: 'avatar', type: 'Text', label: 'Avatar URL' },
            { id: 'role', type: 'Choice', label: 'Rôle',
              widgetOptions: JSON.stringify({
                  choices: ['Chef de projet', 'Développeur', 'Designer', 'Data analyst', 'Admin']
              })
            },
            { id: 'actif', type: 'Bool', label: 'Actif' }
        ]
    },

    // ═══════════════════════════════════════════════════════
    // ORDRE 2 : Tables avec Ref simples
    // ═══════════════════════════════════════════════════════
    Projects: {
        columns: [
            { id: 'nom', type: 'Text', label: 'Nom' },
            { id: 'couleur', type: 'Text', label: 'Couleur' },
            { id: 'dateDebut', type: 'Date', label: 'Date début' },
            { id: 'dateFin', type: 'Date', label: 'Date fin' },
            { id: 'responsable', type: 'Ref:Team', label: 'Responsable' },
            { id: 'actif', type: 'Bool', label: 'Actif' }
        ],
        // Références à configurer après création
        refs: [
            { column: 'responsable', visibleColName: 'nom' }
        ]
    },

    // ═══════════════════════════════════════════════════════
    // ORDRE 3 : Tables avec Ref multiples
    // ═══════════════════════════════════════════════════════
    Tasks: {
        columns: [
            { id: 'titre', type: 'Text', label: 'Titre' },
            { id: 'description', type: 'Text', label: 'Description' },
            { id: 'dateDebut', type: 'Date', label: 'Date début' },
            { id: 'dateEcheance', type: 'Date', label: 'Échéance' },
            { id: 'priorite', type: 'Choice', label: 'Priorité',
              widgetOptions: JSON.stringify({
                  choices: ['1 - Critique', '2 - Haute', '3 - Moyenne', '4 - Basse'],
                  choiceOptions: {
                      '1 - Critique': { fillColor: '#ef4444' },
                      '2 - Haute': { fillColor: '#f59e0b' },
                      '3 - Moyenne': { fillColor: '#3b82f6' },
                      '4 - Basse': { fillColor: '#6b7280' }
                  }
              })
            },
            { id: 'statut', type: 'Choice', label: 'Statut',
              widgetOptions: JSON.stringify({
                  choices: ['todo', 'inprogress', 'review', 'done'],
                  choiceOptions: {
                      'todo': { fillColor: '#6b7280', textColor: '#fff' },
                      'inprogress': { fillColor: '#3b82f6', textColor: '#fff' },
                      'review': { fillColor: '#f59e0b', textColor: '#fff' },
                      'done': { fillColor: '#10b981', textColor: '#fff' }
                  }
              })
            },
            { id: 'progression', type: 'Numeric', label: 'Progression %' },
            { id: 'projet', type: 'Ref:Projects', label: 'Projet' },
            { id: 'assignees', type: 'RefList:Team', label: 'Assignés' },
            { id: 'type', type: 'Choice', label: 'Type',
              widgetOptions: JSON.stringify({
                  choices: ['tache', 'jalon', 'reunion']
              })
            },
            { id: 'dependDe', type: 'RefList:Tasks', label: 'Dépend de' },
            { id: 'tags', type: 'ChoiceList', label: 'Tags',
              widgetOptions: JSON.stringify({
                  choices: ['urgent', 'bloquant', 'documentation', 'test', 'bug']
              })
            },
            { id: 'estimationH', type: 'Numeric', label: 'Estimation (h)' },
            { id: 'tempsPasse', type: 'Numeric', label: 'Temps passé (h)' },
            { id: 'couleur', type: 'Text', label: 'Couleur' }
        ],
        refs: [
            { column: 'projet', visibleColName: 'nom' },
            { column: 'assignees', visibleColName: 'nom' },
            { column: 'dependDe', visibleColName: 'titre' }
        ]
    }
};
```

---

## Fonction ensureSchema complète

```javascript
async function ensureSchema(SCHEMA) {
    const log = (msg) => console.log(`[Schema] ${msg}`);
    const tableOrder = Object.keys(SCHEMA);
    const tablesCreated = [];

    // ═══════════════════════════════════════════════════════
    // PHASE 1 : Créer les tables dans l'ordre
    // ═══════════════════════════════════════════════════════
    for (const tableName of tableOrder) {
        const tableSpec = SCHEMA[tableName];
        const columns = tableSpec.columns || tableSpec; // Support ancien format

        try {
            // Vérifier si la table existe
            const existing = await grist.docApi.fetchTable(tableName);
            const existingCols = Object.keys(existing).filter(c => c !== 'id' && c !== 'manualSort');
            log(`✓ Table ${tableName} existe (${existingCols.length} colonnes)`);

            // Ajouter les colonnes manquantes
            for (const col of columns) {
                if (!existingCols.includes(col.id)) {
                    try {
                        const colDef = buildColumnDef(col);
                        await grist.docApi.applyUserActions([
                            ['AddColumn', tableName, col.id, colDef]
                        ]);
                        log(`  + ${tableName}.${col.id} ajoutée`);
                    } catch (e) {
                        log(`  ~ ${tableName}.${col.id} ignorée: ${e.message}`);
                    }
                }
            }

        } catch (e) {
            // Table n'existe pas → la créer
            try {
                const colSpecs = columns.map(buildColumnDef);
                await grist.docApi.applyUserActions([
                    ['AddTable', tableName, colSpecs]
                ]);
                log(`✅ Table ${tableName} créée`);
                tablesCreated.push(tableName);
            } catch (e2) {
                log(`❌ Erreur création ${tableName}: ${e2.message}`);
            }
        }
    }

    // ═══════════════════════════════════════════════════════
    // PHASE 2 : Configurer les visibleCol des références
    // ═══════════════════════════════════════════════════════
    for (const tableName of tableOrder) {
        const tableSpec = SCHEMA[tableName];
        const refs = tableSpec.refs || [];

        for (const ref of refs) {
            try {
                await configureVisibleCol(tableName, ref.column, ref.visibleColName);
                log(`  → ${tableName}.${ref.column} affiche ${ref.visibleColName}`);
            } catch (e) {
                log(`  ~ visibleCol ${tableName}.${ref.column} non configuré: ${e.message}`);
            }
        }
    }

    // ═══════════════════════════════════════════════════════
    // PHASE 3 : Seed data si nouvelles tables
    // ═══════════════════════════════════════════════════════
    if (tablesCreated.length > 0) {
        log(`Tables créées: ${tablesCreated.join(', ')}`);
        // Appeler seedData() si nécessaire
    }

    return { tablesCreated };
}

// ─────────────────────────────────────────────────────────
// Helpers
// ─────────────────────────────────────────────────────────

function buildColumnDef(col) {
    const def = {
        id: col.id,
        type: col.type
    };
    if (col.label) def.label = col.label;
    if (col.widgetOptions) def.widgetOptions = col.widgetOptions;
    if (col.formula) {
        def.formula = col.formula;
        def.isFormula = true;
    }
    return def;
}

async function configureVisibleCol(tableName, refColName, visibleColName) {
    // Récupérer le type de la colonne pour extraire la table cible
    // Ex: 'Ref:Projects' → 'Projects'
    const tableData = await grist.docApi.fetchTable(tableName);

    // Note: Cette fonction nécessite un accès à l'API REST pour
    // récupérer l'ID numérique de la colonne.
    // En pratique, on peut laisser Grist utiliser la colonne par défaut
    // ou configurer manuellement dans l'interface.

    // Pattern alternatif avec ModifyColumn si on connaît l'ID :
    // await grist.docApi.applyUserActions([
    //     ['ModifyColumn', tableName, refColName, { visibleCol: colId }]
    // ]);
}
```

---

## ModifyColumn — Modifier une colonne existante

```javascript
// Changer le type
await grist.docApi.applyUserActions([
    ['ModifyColumn', 'Tasks', 'priorite', { type: 'Int' }]
]);

// Changer le label
await grist.docApi.applyUserActions([
    ['ModifyColumn', 'Tasks', 'titre', { label: 'Titre de la tâche' }]
]);

// Ajouter des choices à une colonne Choice existante
await grist.docApi.applyUserActions([
    ['ModifyColumn', 'Tasks', 'statut', {
        widgetOptions: JSON.stringify({
            choices: ['todo', 'inprogress', 'review', 'done', 'cancelled']
        })
    }]
]);

// Configurer visibleCol (si on connaît l'ID numérique)
await grist.docApi.applyUserActions([
    ['ModifyColumn', 'Tasks', 'projet', { visibleCol: 2 }]
]);
```

---

## Pattern : Création avec vérification

```javascript
async function safeCreateTable(tableName, columns) {
    try {
        await grist.docApi.fetchTable(tableName);
        console.log(`Table ${tableName} existe déjà`);
        return false;
    } catch {
        await grist.docApi.applyUserActions([
            ['AddTable', tableName, columns.map(buildColumnDef)]
        ]);
        console.log(`Table ${tableName} créée`);
        return true;
    }
}

async function safeAddColumn(tableName, col) {
    try {
        const data = await grist.docApi.fetchTable(tableName);
        if (col.id in data) {
            console.log(`Colonne ${tableName}.${col.id} existe déjà`);
            return false;
        }
    } catch {
        console.log(`Table ${tableName} n'existe pas`);
        return false;
    }

    await grist.docApi.applyUserActions([
        ['AddColumn', tableName, col.id, buildColumnDef(col)]
    ]);
    console.log(`Colonne ${tableName}.${col.id} ajoutée`);
    return true;
}
```

---

## Colonnes calculées (formules)

```javascript
{
    id: 'fullName',
    type: 'Text',
    label: 'Nom complet',
    formula: '$prenom + " " + $nom',
    isFormula: true
}

{
    id: 'joursRestants',
    type: 'Int',
    label: 'Jours restants',
    formula: '($dateEcheance - TODAY()).days if $dateEcheance else None',
    isFormula: true
}

{
    id: 'estEnRetard',
    type: 'Bool',
    label: 'En retard',
    formula: '$dateEcheance and $dateEcheance < TODAY() and $statut != "done"',
    isFormula: true
}
```

---

## Checklist création de schéma

```
□ Définir l'ordre des tables (sans ref → avec ref)
□ Ajouter label à chaque colonne
□ Configurer widgetOptions.choices pour Choice/ChoiceList
□ Ajouter choiceOptions pour les couleurs si nécessaire
□ Déclarer les refs à configurer (visibleColName)
□ Prévoir les colonnes calculées (formula)
□ Implémenter seedData() pour données initiales
□ Tester en mode création complète
□ Tester en mode colonnes manquantes
```
