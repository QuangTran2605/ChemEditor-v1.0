# ChemEditor v1.0

A client-side molecular chemistry toolkit with two interconnected tools:

1. **Molecule Editor** — Draw molecular structures visually and generate SMILES strings
2. **SMILES to Formula Converter** — Convert SMILES strings to molecular formulas with 2D/3D visualization, composition analysis, and NMR prediction

Everything runs in the browser. No server, no build tools, no package manager. All dependencies load from CDNs at runtime.

## Quick Start

Serve the directory with any static HTTP server:

```bash
# Python
python -m http.server 8000

# Node.js
npx serve .
```

Then open:
- `http://localhost:8000/molecule_drawing.html` — Molecule Editor
- `http://localhost:8000/smiles_to_formula.html` — Formula Converter

## Molecule Editor (`molecule_drawing.html`)

Interactive molecular structure editor powered by [OpenChemLib](https://github.com/cheminfo/openchemlib-js).

### Features

- **Visual structure editor** — Draw atoms, bonds, rings, and functional groups using the built-in toolbar
- **Two-way SMILES binding** — Draw a structure to generate SMILES, or type/paste SMILES to render a structure
- **Template molecules** — Quick-insert common molecules (benzene, caffeine, aspirin, etc.)
- **2D preview** — RDKit-powered canonical SVG rendering for verification
- **Copy to clipboard** — One-click SMILES copy
- **Cross-link** — Send drawn molecules directly to the Formula Converter
- **URL parameters** — Load molecules via `?smiles=CCO`
- **Dark/light theme** — Synced with Formula Converter via localStorage

### Layout

Side-by-side on desktop (SMILES output on left, editor canvas on right). Stacks vertically on mobile.

## Formula Converter (`smiles_to_formula.html`)

Full-featured SMILES analysis tool powered by [RDKit.js](https://github.com/rdkit/rdkit-js).

### Features

- **Molecular formula** — Hill notation with subscripts + LaTeX copy
- **Molecular weight** — IUPAC 2021 standard atomic weights
- **Composition analysis** — Color-coded element bar + mass percentage table
- **2D structure** — RDKit SVG rendering with theme-aware bond colors
- **3D viewer** — Interactive 3Dmol.js viewer with stick/ball-and-stick/sphere/line styles, auto-rotate, and H visibility toggle
- **NMR prediction** — SMARTS-based 1H NMR chemical shift prediction with interactive spectrum
- **Conversion history** — Last 10 conversions stored in localStorage
- **100+ example molecules** — Random selection of common chemicals as quick-try chips
- **URL parameters** — Auto-convert via `?smiles=CCO`

## Dependencies (CDN)

| Library | Version | Purpose |
|---------|---------|---------|
| [RDKit.js](https://www.rdkit.org/) | 2025.3.4 | SMILES parsing, formula calculation, 2D SVG |
| [OpenChemLib](https://github.com/cheminfo/openchemlib-js) | 8.18.1 | Structure editor (CanvasEditor), 3D conformer generation |
| [3Dmol.js](https://3dmol.csb.pitt.edu/) | 2.5.3 | Interactive 3D molecular viewer |
| [Google Fonts](https://fonts.google.com/) | — | IBM Plex Mono, DM Sans |

No npm packages. No build step. Zero runtime API dependencies.

## Project Structure

```
ChemEditor-v1.0/
├── molecule_drawing.html    # Molecule Editor (single-file app)
├── smiles_to_formula.html   # Formula Converter (single-file app)
├── CLAUDE.md                # Development instructions for Claude Code
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

Both pages share a dark/light theme system via CSS custom properties and `localStorage`. Toggle with the sun/moon button in the header. Respects system `prefers-color-scheme` on first visit.

## License

MIT
