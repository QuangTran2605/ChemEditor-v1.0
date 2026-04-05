# ChemEditor v1.0 - link: https://quangtran2605.github.io/ChemEditor-v1.0/

A client-side molecular chemistry toolkit with two interconnected tools:

1. **Molecule Editor** — Draw molecular structures and reactions, export in multiple formats (SMILES, Molfile, InChI, SMARTS, RXN, CDXML)
2. **SMILES to Formula Converter** — Convert SMILES to molecular formulas with 2D/3D visualization, composition analysis, NMR prediction, and IR absorption

Everything runs in the browser. No server, no build tools, no package manager.

## Quick Start

Serve the directory with any static HTTP server:

```bash
# Python
python -m http.server 8000

# Node.js
npx serve .
```

Then open `http://localhost:8000/index.html` — the landing page links to both tools.

## Molecule Editor (`molecule_drawing.html`)

Interactive molecular structure editor powered by [Ketcher](https://github.com/epam/ketcher) (EPAM, Apache 2.0), embedded as a same-origin iframe.

### Features

- **Full-featured structure editor** — Draw atoms, bonds, rings, functional groups, reactions, and stereochemistry using Ketcher's professional toolbar
- **Bidirectional Convert** — Push SMILES from textarea into the editor, or pull the current structure out
- **Multi-format export** — Export as SMILES, Molfile, InChI, InChIKey (via RDKit.js), SMARTS, RXN, or CDXML
- **Selection export** — Export SMILES for only the selected fragment
- **Template molecules** — Quick-insert from 25 common molecule presets
- **Copy to clipboard** — One-click copy of any export format
- **Cross-link** — Send drawn molecules directly to the Formula Converter for analysis
- **Toolbar user guide** — Expandable quick-reference for Ketcher's tools and shortcuts
- **URL parameters** — Load molecules via `?smiles=CCO`
- **Dark/light theme** — Synced across all pages via localStorage

### Layout

Side-by-side on desktop (SMILES I/O + export toolbar + templates on left, Ketcher editor on right). Stacks vertically on mobile.

## Formula Converter (`smiles_to_formula.html`)

Full-featured SMILES analysis tool powered by [RDKit.js](https://github.com/rdkit/rdkit-js).

### Features

- **Molecular formula** — Hill notation with subscripts + LaTeX copy
- **Molecular weight** — IUPAC 2021 standard atomic weights (92 elements, H through U)
- **InChIKey** — IUPAC International Chemical Identifier Key generation
- **Composition analysis** — Color-coded element bar + mass percentage table
- **2D structure** — RDKit SVG rendering with theme-aware bond colors
- **3D viewer** — Interactive 3Dmol.js viewer with stick/ball-and-stick/sphere/line styles, auto-rotate, and H visibility toggle
- **1H NMR prediction** — SMARTS-based chemical shift prediction with interactive spectrum, splitting patterns, coupling constants, and integration curve
- **13C NMR prediction** — Broadband decoupled spectrum (0-250 ppm) with interactive viewer and peak table
- **Interactive NMR assignment** — Click peaks in the table to highlight corresponding atoms on the 2D structure
- **IR absorption prediction** — SMARTS-based functional group identification with wavenumber ranges, intensity, and group labels
- **Collapsible panels** — NMR and IR sections presented as expandable dropdowns
- **Conversion history** — Last 10 conversions stored in localStorage
- **100+ example molecules** — Random selection of common chemicals as quick-try chips
- **URL parameters** — Auto-convert via `?smiles=CCO`

## Dependencies

### CDN

| Library | Version | Purpose |
|---------|---------|---------|
| [RDKit.js](https://www.rdkit.org/) | 2025.3.4 | SMILES parsing, formula calculation, 2D SVG, export conversions |
| [OpenChemLib](https://github.com/cheminfo/openchemlib-js) | 8.18.1 | 3D conformer generation (formula page) |
| [3Dmol.js](https://3dmol.csb.pitt.edu/) | 2.5.3 | Interactive 3D molecular viewer |
| [Google Fonts](https://fonts.google.com/) | - | IBM Plex Mono, DM Sans |

### Local

| Library | Version | Purpose |
|---------|---------|---------|
| [Ketcher](https://github.com/epam/ketcher) (EPAM, Apache 2.0) | 3.2.0 | Structure/reaction editor (served from `ketcher/` via iframe) |

No npm packages. No build step. Zero runtime API dependencies.

## Project Structure

```
ChemEditor-v1.0/
├── index.html                # Landing page / app hub
├── molecule_drawing.html     # Molecule Editor (single-file app)
├── smiles_to_formula.html    # Formula Converter (single-file app)
├── favicon.svg               # App icon
├── ketcher/                  # Ketcher 3.2.0 standalone build (gitignored, ~97 MB)
├── CLAUDE.md                 # Development instructions for Claude Code
├── .gitignore
└── README.md
```

Each `.html` file is a complete, self-contained application with inline CSS and JavaScript.

## Browser Support

Modern browsers with ES module and WebAssembly support:
- Chrome/Edge 123+
- Firefox 120+
- Safari 16+

## Theme

All pages share a dark/light theme system via CSS custom properties and `localStorage`. Toggle with the sun/moon button in the header. Respects system `prefers-color-scheme` on first visit.

## License

MIT
