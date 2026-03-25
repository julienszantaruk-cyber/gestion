# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Artefactory AI v12** is a no-code/low-code IDE that runs as a Grist custom widget. It allows creating, editing, and composing web applications ("artefacts") directly inside a Grist document. The entire UI is in **French**.

The `templates/` directory contains a complete example application: **Gestion Patrimoine** — a property management system with 6 modules (dashboard, buildings, interventions, energy, compliance, admin).

## Development

### No Build System
Everything runs in-browser. No package manager, no bundler, no transpilation step.

### Running Locally
Open `app.html` in a browser — it will fail to connect to Grist but the IDE UI loads. For full functionality, deploy as a Grist Custom Widget with `full` access level.

### Deploying to Grist
Add `app.html` as a Custom Widget linked to an `Artefacts` table. Artefactory reads/writes code to that table. Templates in `templates/` are stored as rows in this table (each file = one artefact record).

### CDN Dependencies
- **Grist Plugin API** — `grist-plugin-api.js`
- **Tailwind CSS** — all templates use Tailwind utilities
- **Ace Editor v1.32.6** — code editing (JS, JSX, HTML, JSON, SVG, Markdown modes)
- **Babel Standalone** — in-browser JSX/React compilation
- **Chart.js** — dashboard charts in templates
- **marked.js** — Markdown rendering
- **html2canvas** — screenshot capture for AI context

## Architecture

### Two-Layer System

**app.html (~2700 lines)** = The IDE/Factory layer
- Ace editor for code editing with hot reload (~800ms debounce)
- Preview iframe for real-time rendering
- Grist Bridge for iframe→Grist API access
- AI panel with webhook integration
- Dual mode: **Edit** (IDE) and **Run** (app runtime with sidebar nav)

**templates/** = Example artefacts (the output)
- Each file is a standalone HTML widget (~1000-1500 lines)
- Stored as rows in the Grist `Artefacts` table
- Linked together via `manifest.json` which maps routes to artefact IDs

### Artefact Types (TYPES object)
| Type | Editor Mode | Rendering |
|------|-------------|-----------|
| `app` | json | Parsed as manifest (routes, theme, layout) |
| `grist` | html | HTML in iframe with Grist Bridge injected |
| `react` | jsx | Compiled with Babel, rendered in iframe |
| `html` | html | Rendered in iframe |
| `component` | jsx | Reusable React component, exported |
| `markdown` | markdown | Rendered with marked.js |
| `svg` | svg | Rendered inline |

### Grist Bridge (GristBridgeParent class)
Sandboxed iframes cannot access Grist API directly. The bridge intercepts `postMessage` calls from child iframes and routes them to the parent's Grist API:

```
Child iframe                    app.html (parent)              Grist
  grist.docApi.fetchTable() → postMessage('grist-bridge') → grist.docApi.fetchTable()
                             ← postMessage(result)          ← data
```

Bridge script is injected into iframe `srcdoc` before rendering. Supports: `fetchTable`, `applyUserActions`, `listTables`, `onRecords`, `onRecord`.

### App Runtime Script (window.app)
Injected into all running artefacts for inter-widget communication:
```javascript
window.app = {
    navigate(path),      // Trigger route change in parent
    emit(event, data),   // Broadcast event to all iframes
    on(event, callback), // Listen for events from other iframes
    state,               // Shared state object
    manifest             // Current app manifest
}
```

### Dual Context Detection
Every template detects whether it runs standalone or inside Artefactory:
```javascript
const appContext = {
    isArtefactory: typeof window.app !== 'undefined' && window.app.navigate,
    app: window.app || { navigate: () => {}, emit: () => {}, on: () => {} }
};
```

## Data Model

### Artefacts table (Grist)
| Column | Type | Purpose |
|--------|------|---------|
| `Nom` | Text | Display name |
| `Type` | Choice | app/grist/react/html/component/svg/markdown |
| `Code` | Text | Full source code |
| `Description` | Text | User description |
| `Dependencies` | Text | JSON array of artefact IDs |
| `IsDoc` | Bool | Is documentation? |
| `Icon` | Text | Emoji icon |
| `CreatedAt` | DateTime | Creation timestamp |

### Template App Tables (manifest.json → tables)
The Gestion Patrimoine app uses these Grist table groups:
- **patrimoine**: Services, Agents, Adresses, Batiments, Locaux
- **interventions**: Interventions, INT_Categories, INT_Commentaires
- **energie**: ENE_Fournisseurs, ENE_PDL, ENE_Consommations, ENE_DPE
- **controles**: CTR_Types, CTR_Controles

## Shared Patterns in Templates

### Data Loading (identical across all templates)
```javascript
async function safeLoad(tableName) {
    const data = await grist.docApi.fetchTable(tableName);
    const n = data.id?.length || 0;
    const records = [];
    for (let i = 0; i < n; i++) {
        const record = { id: data.id[i] };
        Object.keys(data).forEach(key => {
            if (key !== 'id') record[key] = data[key][i];
        });
        records.push(record);
    }
    return records;
}
```

### Rendering Pattern
Templates use vanilla JS with template literals — no React despite Artefactory supporting it:
```javascript
function render() {
    document.getElementById('app').innerHTML = `...`;
}
```

### Inter-Widget Communication
```javascript
appContext.app.emit('batiment-selected', { id: batId });   // Send
appContext.app.on('batiment-selected', (data) => { ... }); // Receive
```

### Grist Timestamps
All Grist dates are Unix seconds (not milliseconds): `Date.now() / 1000` to write, `new Date(ts * 1000)` to read.

## AI Integration

The AI panel sends webhook requests with context including: current code, selection, dependencies, manifest, error logs (last 5), and optional screenshot (base64). Responses support partial operations: `full` (replace all), `replace` (line range), `insert` (after line), `delete` (line range). Configuration stored in `localStorage`.

## Key Keyboard Shortcuts (in app.html)
- **Ctrl+S** — Save artefact to Grist
- **Ctrl+K** — Quick AI prompt
