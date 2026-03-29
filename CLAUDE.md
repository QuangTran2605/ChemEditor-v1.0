# CLAUDE.md — SMILES to Formula

## Overview

Client-side molecular formula converter powered by RDKit.js (WASM) with an interactive 3D molecular viewer. Single-file HTML application — no build tools, no bundler, no package manager.

## Project Structure

```
SMILES_to_formula/
├── smiles_to_formula.html      # Entire application (HTML + CSS + JS, ~39 KB)
├── RDKit_minimal.js            # Local RDKit JS wrapper (127 KB)
├── RDKit_minimal.wasm          # Local RDKit WASM binary (6.1 MB)
├── CLAUDE.md                   # This file
└── .claude/
    ├── launch.json             # Dev server config (Python http.server, autoPort)
    └── settings.local.json     # Permission allowlist
```

## Running the Project

Start the dev server via Claude Preview:

```
preview_start with name: "smiles-app"
```

This runs `python -m http.server` on an auto-assigned port and serves the app at `http://localhost:<port>/smiles_to_formula.html`.

**No install step.** All dependencies load from CDNs at runtime.

## External Dependencies (CDN)

| Library | Version | CDN | Purpose |
|---------|---------|-----|---------|
| RDKit.js | 2025.3.4-1.0.0 | unpkg.com/@rdkit/rdkit | SMILES parsing, formula calc, 2D SVG |
| OpenChemLib | 8.18.1 | cdn.jsdelivr.net/npm/openchemlib | 3D conformer generation (ETKDG) |
| 3Dmol.js | 2.5.3 | cdnjs.cloudflare.com/ajax/libs/3Dmol | Interactive 3D molecular viewer |
| Google Fonts | — | fonts.googleapis.com | IBM Plex Mono, DM Sans |

**No runtime API dependencies.** 3D conformers are generated client-side via OpenChemLib's ConformerGenerator. Falls back to RDKit 2D molblock if OpenChemLib fails.

**Local RDKit copies** (`RDKit_minimal.js`, `RDKit_minimal.wasm`) exist for offline/development use. Production loads from CDN — swap the `<script src>` and `locateFile` URL to switch between local and CDN.

## Architecture

Everything lives in `smiles_to_formula.html`:

### CSS (~700 lines, inline `<style>`)

- **Theme system**: CSS custom properties on `:root` / `[data-theme="light"]` / `[data-theme="dark"]`
- **Key variables**: `--bg`, `--surface`, `--surface-2`, `--border`, `--text`, `--text-dim`, `--accent`, `--accent-glow`, `--svg-bond`
- **Fonts**: `--font-sans` (DM Sans), `--font-mono` (IBM Plex Mono)
- **Card pattern**: `background: var(--surface); border: 1px solid var(--border); border-radius: 12px;`
- **Single breakpoint**: `@media (max-width: 520px)` — grid collapses, font shrinks

### HTML Structure

1. **Header** — title + theme toggle button (sun/moon icons)
2. **Status banner** — RDKit loading state (spinner → green dot)
3. **Input group** — SMILES text input + Convert button + example chips
4. **Results container** (`#results`) — hidden until conversion, fades in
   - Formula display (subscripted) + LaTeX copy button
   - Meta grid: input SMILES, canonical SMILES (with copy), MW, total atoms
   - Composition bar + legend + mass percentage table
   - 2D structure card (RDKit SVG)
   - 3D viewer card (3Dmol.js canvas + controls)
5. **History section** — last 12 conversions, clickable to re-convert

### JavaScript (~500 lines, inline `<script>`)

**Globals:**
- `RDKit` — RDKit module instance (set after WASM loads)
- `viewer3d` — current 3Dmol.js viewer instance
- `viewer3dStyle` / `viewer3dHideH` — current 3D display state
- `viewer3dConversionId` — counter to prevent stale async results
- `conversionHistory` — array of `{smiles, formula}` (max 12)

**Lookup tables:**
- `ELEM_COLORS` — element → hex color for composition bar
- `Z_TO_ELEM` — atomic number → symbol (Z=1–53)
- `ATOMIC_WEIGHT` — IUPAC 2021 standard weights for mass % calc

**Key functions:**

| Function | Purpose |
|----------|---------|
| `convert()` | Main pipeline: validate SMILES → extract data → `showResult()` |
| `showResult(data)` | Render full results UI, kick off async 3D generation |
| `showError(smiles)` | Display invalid SMILES error |
| `generate3DAndInit(canonical)` | Async: OpenChemLib 3D conformer → fallback RDKit 2D → `init3DViewer()` |
| `init3DViewer(molBlock, source)` | Create 3Dmol viewer, attach all control handlers |
| `applyViewerStyle(styleName)` | Apply stick/ballstick/sphere/line style, respect H visibility |
| `buildHillFormula(counts)` | Atom counts → Hill notation string (C first, H second, rest alpha) |
| `hillSort(counts)` | Sort `{element: count}` entries in Hill order |
| `fixSvgColors(svg)` | Replace RDKit black strokes with theme-aware `--svg-bond` color |
| `formulaToLatex(formula)` | `"C8H10N4O2"` → `"C_{8}H_{10}N_{4}O_{2}"` |
| `copyToClipboard(btn)` | Copy `data-copy` attribute to clipboard with visual feedback |
| `buildMassPctRows(sortedAtoms, mw)` | Generate HTML rows for mass percentage table |
| `addHistory()` / `renderHistory()` | Manage conversion history (dedup, cap, render) |
| `escHtml(s)` / `escAttr(s)` | XSS-safe escaping |

## Conventions

### Naming
- **Functions**: camelCase with descriptive verbs (`buildHillFormula`, `generate3DAndInit`)
- **CSS classes**: kebab-case, BEM-inspired (`.result-card`, `.viewer-3d-card`, `.atom-bar-segment`)
- **State classes**: `.visible`, `.active`, `.ready`, `.error`, `.copied`
- **IDs**: kebab-case (`#smiles-input`, `#viewer-3d-container`, `#spin-toggle-cb`)
- **Data attributes**: `data-smiles`, `data-copy`, `data-style`, `data-theme`

### Patterns
- **Cards**: Always `var(--surface)` bg + `var(--border)` border + `12px` radius
- **Section labels**: `.section-label` — 0.72rem uppercase, letter-spaced, `var(--text-dim)`
- **Buttons**: `.btn` (primary), `.chip` / `.style-btn` (tertiary), `.copy-btn` (inline utility)
- **Transitions**: 0.15s for interactive feedback, 0.35s for layout reveals
- **Error handling**: try/catch around all RDKit calls and 3D operations; graceful fallback messages
- **Memory**: Always call `mol.delete()` after using RDKit mol objects (prevents WASM memory leaks)

### Theme
- Theme stored in `localStorage` key `'smiles-theme'`
- Applied via `data-theme` attribute on `<html>` element
- Falls back to `prefers-color-scheme` system preference on first visit
- 3D viewer background: `#151a21` (dark) / `#ffffff` (light) — matches `--surface`

## Testing

There are no automated tests. Verify manually via the preview server:

1. **Start server**: `preview_start` with `"smiles-app"`
2. **Navigate**: `/smiles_to_formula.html`
3. **Wait**: ~10–15s for RDKit WASM to load (status banner turns green)

**Test checklist:**
- Click example chips (Ethanol, Benzene, Caffeine, Aspirin, etc.)
- Verify formula, MW, atom counts, composition bar, mass percentages
- Verify 2D SVG structure renders
- Verify 3D viewer loads with "PubChem 3D Conformer" source label
- Test all 4 styles: Stick, Ball & Stick, Sphere, Line
- Test Hide H / Show H toggle
- Test auto-rotate checkbox + speed slider (0.1x–5.0x)
- Toggle light/dark theme — SVG recolors, 3D background updates
- Enter invalid SMILES — only error message, no structure cards
- Convert multiple molecules — history populates, 3D viewer resets cleanly
- Test LaTeX copy button and canonical SMILES copy button
- Test unusual molecule (e.g. `[Cu+2].[O-]S([O-])(=O)=O`) — should show "OpenChemLib 3D Conformer" (all 3D is now local, no PubChem dependency)

## Important Notes

- **Single-file app**: All edits go in `smiles_to_formula.html`. No other source files.
- **No build step**: Changes take effect on page reload.
- **Local vs CDN**: For development, swap `<script src>` and `locateFile` to use local `RDKit_minimal.js`/`.wasm`. Always restore CDN URLs before committing.
- **RDKit MinimalLib limitations**: No `set_3d_coords()` in the JS WASM build — 3D coordinates are generated by OpenChemLib's ConformerGenerator instead.
- **OpenChemLib loading**: Loaded as ESM module (async). The `generate3DAndInit()` function waits for the `ocl-ready` event with a 15s timeout before falling back to RDKit 2D.
- **3Dmol.js + screenshots**: The WebGL canvas can interfere with automated screenshot tools. Use `eval` for programmatic verification when screenshots timeout.
