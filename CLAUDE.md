# CLAUDE.md — ChemEditor v1.0

## Overview

Client-side molecular chemistry toolkit: draw structures, generate SMILES, calculate formulas, predict NMR/IR spectra, and visualize molecules in 2D/3D. Multi-page HTML application — no build tools, no bundler, no package manager.

## Project Structure

```
SMILES_to_formula/
├── index.html                  # Landing page / app hub (8 KB, ~240 lines)
├── smiles_to_formula.html      # Formula converter + NMR/IR prediction (132 KB, ~3582 lines)
├── molecule_drawing.html       # Molecule structure editor (48 KB, ~1329 lines)
├── favicon.svg                 # App icon — pen-nib-as-atom Bohr model (4 KB)
├── README.md                   # Project README (8 KB)
├── ketcher/                    # Ketcher 3.2.0 standalone build (gitignored, ~97 MB)
├── ketcher-build/              # Ketcher source repo for rebuilding (not tracked)
├── RDKit_minimal.js            # Local RDKit JS wrapper (gitignored)
├── RDKit_minimal.wasm          # Local RDKit WASM binary (gitignored)
├── CLAUDE.md                   # This file
├── .gitignore                  # Ignores RDKit copies and ketcher/ build
└── .claude/
    ├── launch.json             # Dev server config (Python http.server, autoPort)
    └── settings.local.json     # Permission allowlist (local only)
```

## Running the Project

Start the dev server via Claude Preview:

```
preview_start with name: "smiles-app"
```

This runs `python -m http.server` on an auto-assigned port. Entry point is `http://localhost:<port>/index.html` (the hub), which links to both tools.

**No install step.** CDN dependencies load at runtime. Ketcher must be built locally into `ketcher/` (gitignored due to size).

## External Dependencies (CDN)

| Library | Version | CDN | Used By | Purpose |
|---------|---------|-----|---------|---------|
| RDKit.js | 2025.3.4-1.0.0 | unpkg.com/@rdkit/rdkit | Both pages | SMILES parsing, formula calc, 2D SVG, export conversions |
| OpenChemLib (core) | 8.18.1 | cdn.jsdelivr.net/npm/openchemlib/core.js | smiles_to_formula | 3D conformer generation |
| 3Dmol.js | 2.5.3 | cdnjs.cloudflare.com/ajax/libs/3Dmol | smiles_to_formula | Interactive 3D molecular viewer |
| Google Fonts | — | fonts.googleapis.com | All pages | IBM Plex Mono, DM Sans |

### Local Dependencies

| Library | Source | Used By | Purpose |
|---------|--------|---------|---------|
| Ketcher (EPAM) | ketcher/index.html (built from ketcher-build/) | molecule_drawing | Full-featured structure editor via iframe |

**No runtime API dependencies.** 3D conformers are generated client-side via OpenChemLib's ConformerGenerator. Falls back to RDKit 2D molblock if OpenChemLib fails.

## Architecture

### index.html — Landing Page (~240 lines)

Simple hub page with two cards linking to the tools:
- **Molecule Editor** → `molecule_drawing.html`
- **Formula Converter** → `smiles_to_formula.html`

Footer links to GitHub repo. Shares the same theme system and CSS variables as the tool pages (minus `--border-focus`, `--green`, `--red`, `--orange`, `--svg-bond` which are only in the tool pages).

### molecule_drawing.html — Molecule Editor (~1329 lines)

Interactive 2D structure editor using **Ketcher** (EPAM, Apache 2.0) embedded via same-origin iframe.

**CSS (~576 lines):** Theme variables, loading screen, two-column grid layout, export toolbar, source badge, user guide accordion, responsive breakpoints at 800px and 520px.

**HTML (~174 lines):** Loading screen → header → status banner → two-column grid (SMILES I/O + export toolbar + templates on left, Ketcher iframe on right) → collapsible toolbar user guide.

**JavaScript (~560 lines):**

| Function / Class | Purpose |
|----------|---------|
| `KetcherBridge` | Bridge class wrapping the Ketcher iframe API (postMessage init → direct API). Methods: `getSmiles()`, `setMolecule()`, `clear()`, `getSmarts()`, `getRxn()`, `getCDXml()`, `hasSelection()`, `getSelectedSmiles()`, `injectCSS()`, `destroy()` |
| `loadSmiles(smiles)` | Async: load SMILES into Ketcher editor via bridge |
| `convert()` | Bidirectional: push textarea SMILES into editor, or pull SMILES from editor into textarea |
| `getSelected()` | Async: export SMILES for only the selected fragment, show source badge |
| `refreshExport()` | Re-export current structure in the active format (SMILES, Molfile, InChI, InChIKey, SMARTS, RXN, CDXML) |
| `initExportToolbar()` | Wire up export format chip buttons |
| `updateExportLabel()` | Update the I/O section label to match active export format |
| `showSourceBadge(type)` / `hideSourceBadge()` | Show/hide badge indicating SMILES source (selection vs full canvas) |
| `withRDKitMol(smiles, fn)` | Helper: create RDKit mol, run callback, always clean up |
| `renderTemplateChips()` | Display 10 shuffled template molecules from 25 presets (Fisher-Yates shuffle) |
| `sendToConverter()` | Navigate to Formula Converter with current SMILES as URL param |
| `copyToClipboard(btn)` | Copy exported text with visual feedback |
| `clearInput()` | Clear SMILES textarea and copy data |
| `onKetcherReady()` | Post-init: inject CSS to hide 3D/fullscreen buttons, enable UI, wire events |
| `toggleGuide()` | Show/hide the expandable toolbar reference guide |
| `loadFromUrl()` | Load SMILES from `?smiles=` URL parameter |

**Globals:** `ketcherBridge` (KetcherBridge), `activeExportFormat` (string), `currentSmiles` (string), `RDKit`, `isUpdating`, `TEMPLATES` (25 molecules).

**Ketcher integration details:**
- Ketcher served from `ketcher/index.html` (same origin) via `<iframe>` with `sandbox="allow-scripts allow-same-origin"`.
- Ketcher signals readiness via `postMessage({ eventType: 'init' })`, then `iframe.contentWindow.ketcher` provides the full API.
- CSS injected into iframe to hide 3D Viewer and fullscreen buttons.
- Selection support: `hasSelection()` checks editor selection, `getSelectedSmiles()` uses Ketcher's `formatterFactory` to extract SMILES for selected atoms only.
- RDKit.js loaded on this page for Molfile/InChI/InChIKey export conversions (non-blocking).
- The editor does **not** auto-sync SMILES on every change — users click "Convert" to transfer.

### smiles_to_formula.html — Formula Converter (~3582 lines)

Full molecular analysis tool: formula, MW, composition, InChIKey, 2D/3D visualization, NMR prediction, and IR absorption prediction.

**CSS (~1051 lines):** Theme system, loading screen, NMR spectrum styles, NMR assignment structure with atom highlight animation (`.atom-hl-glow`), IR table styles, responsive breakpoint at 520px.

**HTML (~42 lines):** Loading screen → header → status banner → input group with example chips → results container → history section.

**JavaScript (~2460 lines):**

**Globals:**
- `RDKit` — RDKit module instance
- `viewer3d` / `viewer3dStyle` / `viewer3dHideH` / `viewer3dConversionId` — 3D viewer state
- `nmrConversionId` — counter for async NMR operations
- `currentCanonical` — current molecule's canonical SMILES
- `currentHNmrPeaks` / `selectedNmrPeakIdx` — NMR assignment interaction state
- `conversionHistory` — array of `{smiles, formula}` (persisted in localStorage)
- `MOLECULE_LIBRARY` — ~116 common chemicals (drugs, solvents, amino acids, steroids, etc.)

**Lookup tables:**
- `ELEM_COLORS` — 13 elements → hex color for composition bar
- `Z_TO_ELEM` — atomic number → symbol (Z=1–92, H through U)
- `ATOMIC_WEIGHT` — IUPAC 2021 standard weights (92 elements)
- `NMR_SHIFT_RULES` — 49 SMARTS-based rules for 1H NMR shift prediction
- `CNMR_SHIFT_RULES` — 40 SMARTS-based rules for 13C NMR prediction
- `IR_ABSORPTION_RULES` — 31 SMARTS-based rules for IR absorption band prediction
- `AROMATIC_SUB_CORRECTIONS` — 20 aromatic substituent shift corrections
- `SPLITTING_NAMES` — multiplicity labels (singlet → septet)
- `PASCAL_TRIANGLE` — Pascal's triangle for NMR splitting patterns (7 rows)
- `SPEC_FREQ` — spectrometer frequency constant (400 MHz)

**Key functions:**

| Function | Purpose |
|----------|---------|
| `convert()` | Main pipeline: validate SMILES → extract data → `showResult()` |
| `showResult(data)` | Render formula, MW, InChIKey, composition, 2D SVG, kick off 3D + NMR + IR |
| `showNMRResult(hNmr, cNmr)` | Render NMR spectra, assignment structure, and IR table |
| `showError(smiles)` | Display invalid SMILES error |
| `generate3DAndInit(smiles)` | Async: OpenChemLib 3D conformer → fallback RDKit 2D → `init3DViewer()` |
| `init3DViewer(molBlock, source)` | Create 3Dmol viewer, attach all control handlers |
| `applyViewerStyle(styleName)` | Apply stick/ballstick/sphere/line style, respect H visibility |
| `predictHNMR(molJson, mol, rdkit)` | 1H NMR: rule-based shifts, J-coupling, multiplicity, diastereotopic detection |
| `predictCNMR(molJson, mol, rdkit)` | 13C NMR: broadband decoupled shifts |
| `predictIR(molJson, mol, rdkit)` | IR: SMARTS-based functional group absorption band prediction |
| `computeMorganClasses(...)` | Morgan algorithm for atom equivalence classes |
| `computeAromaticShift(...)` | Aromatic proton shift calculation with substituent corrections |
| `isDiastereotopic(...)` | Check for diastereotopic protons |
| `initNMRViewer(peaks, opts)` | Interactive canvas-based NMR spectrum viewer (zoom, pan, peak picking) |
| `renderAssignmentStructure(...)` | Render 2D SVG structure for interactive NMR atom assignment |
| `highlightAssignmentAtoms(...)` | Highlight atoms on assignment structure when peak is selected |
| `selectNmrPeak(idx)` | Handle interactive NMR peak selection (bidirectional table ↔ structure) |
| `buildNMRTable(peaks)` / `buildCNMRTable(peaks)` | HTML tables for NMR peak data |
| `buildIRTable(bands)` | HTML table for IR absorption bands (wavenumber, intensity, group) |
| `toggleNMRDropdown(header)` | Expandable/collapsible NMR and IR sections |
| `buildHillFormula(counts)` | Atom counts → Hill notation string (C first, H second, rest alpha) |
| `hillSort(counts)` | Sort `{element: count}` entries in Hill order |
| `fixSvgColors(svg)` | Replace RDKit black strokes with theme-aware `--svg-bond` color |
| `formulaToLatex(formula)` | `"C8H10N4O2"` → `"C_{8}H_{10}N_{4}O_{2}"` |
| `copyToClipboard(btn)` | Copy `data-copy` attribute to clipboard with visual feedback |
| `buildMassPctRows(sortedAtoms, mw)` | Generate HTML rows for mass percentage table |
| `addHistory()` / `renderHistory()` | Manage conversion history (dedup, cap, render) |
| `renderExampleChips()` | Populate example molecule chips from MOLECULE_LIBRARY |
| `escHtml(s)` / `escAttr(s)` | XSS-safe escaping |

## Conventions

### Naming
- **Functions**: camelCase with descriptive verbs (`buildHillFormula`, `generate3DAndInit`)
- **CSS classes**: kebab-case, BEM-inspired (`.result-card`, `.viewer-3d-card`, `.atom-bar-segment`)
- **State classes**: `.visible`, `.active`, `.ready`, `.error`, `.copied`, `.open`
- **IDs**: kebab-case (`#smiles-input`, `#viewer-3d-container`, `#spin-toggle-cb`)
- **Data attributes**: `data-smiles`, `data-copy`, `data-style`, `data-theme`, `data-format`

### Patterns
- **Cards**: Always `var(--surface)` bg + `var(--border)` border + `12px` radius
- **Section labels**: `.section-label` — 0.72rem uppercase, letter-spaced, `var(--text-dim)`
- **Buttons**: `.btn` (primary), `.chip` / `.style-btn` / `.export-chip` (tertiary), `.copy-btn` (inline utility)
- **Transitions**: 0.15s for interactive feedback, 0.35s for layout reveals
- **Error handling**: try/catch around all RDKit calls and 3D operations; graceful fallback messages
- **Memory**: Always call `mol.delete()` after using RDKit mol objects (prevents WASM memory leaks). Use `withRDKitMol()` helper in molecule_drawing.html.
- **Loading screens**: All pages show a full-screen spinner while WASM/libraries load, then fade out

### Theme
- Theme stored in `localStorage` key `'smiles-theme'`
- Applied via `data-theme` attribute on `<html>` element
- Falls back to `prefers-color-scheme` system preference on first visit
- Shared across all pages (index, formula converter, molecule editor)
- 3D viewer background: `#151a21` (dark) / `#ffffff` (light) — matches `--surface`
- Tool pages define extra CSS variables: `--border-focus`, `--green`, `--green-dim`, `--red`, `--red-dim`, `--orange`, `--svg-bond`

### Cross-Page Navigation
- `index.html` links to both tools via cards
- `molecule_drawing.html` has "Analyze in Formula Converter" button → navigates to `smiles_to_formula.html?smiles=<SMILES>`
- `smiles_to_formula.html` header links back to molecule editor
- Both tools accept SMILES via URL query parameter (`?smiles=...`)

## Testing

There are no automated tests. Verify manually via the preview server:

1. **Start server**: `preview_start` with `"smiles-app"`
2. **Navigate**: `/index.html` (hub) or directly to a tool page
3. **Wait**: ~10-15s for RDKit WASM to load (status banner turns green); Ketcher may take up to 30s

**Formula Converter checklist:**
- Click example chips (Ethanol, Benzene, Caffeine, Aspirin, etc.)
- Verify formula, MW, InChIKey, atom counts, composition bar, mass percentages
- Verify 2D SVG structure renders
- Verify 3D viewer loads with source label
- Test all 4 3D styles: Stick, Ball & Stick, Sphere, Line
- Test Hide H / Show H toggle
- Test auto-rotate checkbox + speed slider (0.1x-5.0x)
- Toggle light/dark theme — SVG recolors, 3D background updates
- Enter invalid SMILES — only error message, no structure cards
- Verify 1H NMR and 13C NMR dropdown panels render with spectrum + peak table
- Click NMR table rows — verify atoms highlight on the 2D assignment structure
- Verify IR absorption table renders with wavenumber, intensity, and group labels
- Test NMR spectrum zoom/pan and peak picking
- Convert multiple molecules — history populates, 3D viewer resets cleanly
- Test LaTeX copy button and canonical SMILES copy button

**Molecule Editor checklist:**
- Wait for Ketcher iframe to load (status banner turns green)
- Draw structures in Ketcher, click Convert to get SMILES
- Select atoms in Ketcher, click "Get Selected" to export selection SMILES
- Verify source badge shows "Selected fragment" vs "Full canvas"
- Click template chips, verify structure loads in Ketcher editor
- Shuffle templates button shows different set
- Type SMILES in textarea, click Convert to load into editor
- Test all export formats: SMILES, Molfile, InChI, InChIKey, SMARTS, RXN, CDXML
- Copy button works for each export format
- "Analyze in Formula Converter" button navigates with correct SMILES
- Clear button resets SMILES textarea
- Expand "Quick Guide — Toolbar Reference" accordion
- Theme toggle works and persists across pages

## Important Notes

- **Multi-page app**: Three HTML files (`index.html`, `smiles_to_formula.html`, `molecule_drawing.html`), each self-contained with inline CSS + JS.
- **No build step**: Changes take effect on page reload.
- **Local vs CDN**: Local RDKit copies are gitignored. Production loads from CDN. Ketcher build (`ketcher/`) is also gitignored (~97 MB).
- **RDKit MinimalLib limitations**: No `set_3d_coords()` in the JS WASM build — 3D coordinates are generated by OpenChemLib's ConformerGenerator instead.
- **OpenChemLib loading**: Loaded as ESM module (async). Uses `core.js` for conformer generation (formula page only). Not used by molecule_drawing.html.
- **Ketcher editor**: molecule_drawing.html uses Ketcher (EPAM, Apache 2.0) via a same-origin iframe (`ketcher/index.html`). Built from the `ketcher-build/` source repo. Communication uses `KetcherBridge` class (postMessage init → direct API). The editor does not auto-sync SMILES on every change — users click "Convert" to transfer between editor and textarea.
- **3Dmol.js + screenshots**: The WebGL canvas can interfere with automated screenshot tools. Use `eval` for programmatic verification when screenshots timeout.
