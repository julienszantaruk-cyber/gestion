# Widgets Grist

Collection de **widgets personnalisés** pour [Grist](https://www.getgrist.com/), la plateforme open-source de bases de données collaboratives.

Ce repo sert de **catalogue de widgets** hébergé sur GitHub Pages. Les widgets publiés sont directement utilisables dans Grist.

---

## Widgets publiés

Ces widgets sont stables et prêts à l'emploi.

### TaskFlow

Suite de 3 widgets de gestion de projet, utilisables ensemble ou séparément.

| Widget | Description | Lien direct |
|--------|-------------|-------------|
| **Kanban** | Tableau de tâches avec drag & drop, filtres, modales | [Utiliser](https://nic01asfr.github.io/Widgets-Grist/taskflow/kanban/) |
| **Gantt** | Diagramme de planification interactif | [Utiliser](https://nic01asfr.github.io/Widgets-Grist/taskflow/gantt/) |
| **Calendar** | Vue calendrier mois/semaine/jour | [Utiliser](https://nic01asfr.github.io/Widgets-Grist/taskflow/calendar/) |

Les 3 widgets se synchronisent automatiquement quand ils sont dans le même document Grist (sélection partagée).

---

## Utilisation

### Option 1 : URL directe

Dans Grist : **Add Widget** → **Custom** → **Enter Custom URL**

Collez l'URL du widget souhaité (voir tableau ci-dessus).

### Option 2 : Catalogue de widgets (Grist self-hosted)

Configurez votre instance Grist pour utiliser ce repo comme source de widgets :

```bash
GRIST_WIDGET_LIST_URL=https://nic01asfr.github.io/Widgets-Grist/manifest.json
```

Les widgets apparaîtront automatiquement dans le sélecteur "Custom Widget".

### Option 3 : Fork et personnalisation

1. Forkez ce repo
2. Modifiez les widgets dans `projects/`
3. Publiez vers `published/`
4. Activez GitHub Pages sur votre fork

---

## Structure du repo

```
published/         ← Widgets en production (déployés sur GitHub Pages)
projects/          ← Projets en développement
skills/            ← Documentation technique et patterns
scripts/           ← Outils de build et publication
```

Pour contribuer ou comprendre l'architecture : voir [CLAUDE.md](CLAUDE.md)

---

## Contribuer

- **Signaler un bug** : [Ouvrir une issue](../../issues/new)
- **Proposer une amélioration** : [Discussions](../../discussions)
- **Voter pour un projet** : Ajoutez une reaction sur l'issue du projet

---

## Ressources

- [Documentation Grist](https://support.getgrist.com/)
- [Grist Custom Widgets](https://support.getgrist.com/widget-custom/)
- [Grist Plugin API](https://support.getgrist.com/code/modules/grist_plugin_api/)

---

## Licence

MIT — Libre d'utilisation, modification et distribution.
