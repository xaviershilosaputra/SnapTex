# SnapTex

SnapTex is a simple LaTeX editor built for students, educators, researchers, and anyone who works with mathematical notation. Users type LaTeX source on the left, see a rendered preview update in real time on the right. It is equipped with autocomplete, snippet libraries, version history, and one-click export. No account, no server, no complicated setup needed.

---

## Features

- **Live preview** — Every keystroke re-renders your equations instantly using KaTeX.
- **Autocomplete** — Type a shorthand like `\fr` and press Tab to expand it into `\frac{}{}`. The cursor lands inside the first `{}` automatically.
- **Toolbar** — One-click insertion of the most common symbols: fractions, integrals, Greek letters, set notation, arrows, and more.
- **Snippet sidebar** — Categorised library of full expressions (quadratic formula, matrices, probability distributions) that insert with a single click. Includes a live search filter.
- **Recently used** — The sidebar surfaces the last 5 symbols you inserted so you never have to hunt twice.
- **All functions overlay** — Browse every supported shorthand in one searchable grid, grouped by category.
- **Version history** — Autosaves a snapshot every 2 seconds. Manually save with Ctrl+S. Browse, restore, copy, or delete any past version from the history panel.
- **Settings panel** — Adjust editor font size, toggle line numbers, toggle word wrap, toggle autosave — all persisted between sessions.
- **Focus mode** — Hides the toolbar and sidebar for a distraction-free full-screen editing experience.
- **Multiple themes** — System (follows OS), Light, Dark, and Sepia.
- **Export options** — Print/PDF, download as `.tex` file, export rendered SVG, or copy raw LaTeX to clipboard.
- **LocalStorage persistence** — Your latex source, title, settings, theme, zoom level, and history all survive a page refresh without any account.
- **Docs panel** — Built-in reference covering fractions, exponents, integrals, matrices, Greek letters, formatting, and common pitfalls.

---

## Tech stack

| Layer | Technology |
|---|---|
| UI framework | React 18 (via Vite) |
| Math rendering | KaTeX |
| Styling | Plain CSS with CSS custom properties (no Tailwind) |
| State persistence | Browser `localStorage` |
| Build & dev server | Vite 5 |
| Deployment | GitHub Pages via GitHub Actions |

---

## Project structure

```
snaptex/
├── index.html                        Entry point HTML
├── package.json
├── vite.config.js                    Build config + GitHub Pages base path
├── .github/
│   └── workflows/
│       └── deploy.yml                Auto-deploy to GitHub Pages on push
└── src/
    ├── main.jsx                      React root, mounts <App />
    ├── App.jsx                       All global state, wires every component together
    ├── index.css                     Imports all CSS modules in order
    │
    ├── constants/                    Pure data — no logic, no side effects
    │   ├── shortcuts.js              Every autocomplete shorthand + category
    │   ├── toolbar.js                Toolbar button definitions
    │   ├── sidebar.js                Sidebar snippet groups and expressions
    │   ├── docs.js                   Built-in documentation page content
    │   └── themes.js                 Theme key → CSS class mapping
    │
    ├── hooks/                        Reusable React hooks
    │   ├── useLocalStorage.js        useState that syncs to localStorage
    │   ├── useAutosave.js            Debounced autosave trigger
    │   └── useKeyboardShortcuts.js   Global Ctrl+S / Ctrl+Shift+F / Escape handler
    │
    ├── utils/                        Plain functions, no React
    │   ├── render.js                 KaTeX block renderer, cursor helpers
    │   └── storage.js                localStorage read/write wrapper + constants
    │
    ├── components/
    │   ├── Icons.jsx                 All SVG icon components in one file
    │   ├── Topbar.jsx                Top navigation bar with title, view toggle, menus
    │   ├── ToolbarStrip.jsx          Quick-insert symbol toolbar below the topbar
    │   ├── Sidebar.jsx               Snippet library with search and recently used
    │   ├── Editor.jsx                Textarea, line numbers, autocomplete dropdown
    │   ├── Preview.jsx               KaTeX output panel with zoom controls
    │   └── overlays/
    │       ├── AllFunctionsOverlay.jsx   Searchable grid of every shorthand
    │       ├── DocsOverlay.jsx           Built-in docs with sidebar navigation
    │       ├── HistoryOverlay.jsx        Version history browser
    │       └── SettingsOverlay.jsx       Editor preferences panel
    │
    └── styles/
        ├── base.css                  CSS reset, variables, all four themes
        ├── layout.css                App shell, workspace, split view, resize handle
        ├── topbar.css                Top navigation bar and dropdown menus
        ├── toolbar.css               Quick-insert toolbar strip
        ├── sidebar.css               Sidebar panel, search input, snippet items
        ├── editor.css                Code textarea, line numbers, autocomplete, status bar
        ├── preview.css               Preview panel and KaTeX output area
        └── overlays.css              All overlay panels: backdrop, header, docs, history, settings
```

---

## Getting started

### Prerequisites

- [Node.js](https://nodejs.org/) version 18 or higher
- npm (comes with Node)

You do not need any API keys. SnapTex has no backend and no external data dependencies.

### Running locally

```bash
# 1. Clone the repository
git clone https://github.com/xaviershilosaputra/snaptex.git
cd snaptex

# 2. Install dependencies (only needed once)
npm install

# 3. Start the development server
npm run dev
```

Open [http://localhost:5173](http://localhost:5173) in your browser. The page reloads automatically whenever you save a file.

### Building for production

```bash
npm run build
```

This outputs a fully static site into the `dist/` folder — plain HTML, CSS, and JS files that you can host anywhere: GitHub Pages, Netlify, Vercel, or a plain web server.

---

## Deploying to GitHub Pages

### One-time setup

1. Open `vite.config.js` and set the `base` field to match your repository name:

   ```js
   base: '/snaptex/',
   ```

   If you are deploying to a custom domain or a user/org page (`yourname.github.io`), set `base: '/'` instead.

2. Push the repository to GitHub.

3. Go to your repo on GitHub → **Settings** → **Pages** → **Source** → select **GitHub Actions**.

### Automatic deployment

The file `.github/workflows/deploy.yml` is already included. Every time you push to the `main` branch, GitHub Actions will automatically run `npm run build` and publish the `dist/` folder to GitHub Pages. No further configuration is needed.

You can also trigger a deploy manually from the **Actions** tab → **Deploy to GitHub Pages** → **Run workflow**.

---

## Keyboard shortcuts

| Shortcut | Action |
|---|---|
| `\fr` → Tab | Insert `\frac{}{}` |
| `\sq` → Tab | Insert `\sqrt{}` |
| `\al` → Tab | Insert `\alpha` |
| Tab (in autocomplete dropdown) | Accept selected suggestion |
| ↑ / ↓ | Navigate autocomplete suggestions |
| Ctrl+Enter | Insert a new equation line |
| Ctrl+S | Save manual snapshot to history |
| Ctrl+Shift+F | Toggle focus mode |
| Esc | Close any open panel or overlay |
| Click the zoom percentage | Reset zoom to 100% |

The full list of autocomplete shorthands is available inside the app under **All functions**.

---

## Themes

| Name | Description |
|---|---|
| System | Follows your operating system light/dark preference automatically |
| Light | White background, dark text |
| Dark | Near-black background, light text |
| Sepia | Warm cream background, low-contrast reading mode |

Themes are applied by toggling a class on `document.body`. The full palette for each theme is defined in `src/styles/base.css` using CSS custom properties.

---

## LocalStorage keys

All data is stored in the browser's `localStorage` under these keys. You can clear them individually from the browser's DevTools or use the Settings panel inside the app.

| Key | What it stores |
|---|---|
| `latex` | Current editor content |
| `docTitle` | Document title shown in the topbar |
| `theme` | Active theme name |
| `zoom` | Preview zoom level |
| `editorFlex` | Split-view divider position (as a percentage) |
| `recentlyUsed` | Last 8 inserted snippets |
| `history` | Up to 20 autosave/manual snapshots |
| `fontSize` | Editor font size in px |
| `showLineNums` | Whether line numbers are visible |
| `autoSave` | Whether autosave is enabled |
| `wordWrap` | Whether word wrap is enabled |

---

## Contributing

All contributions are welcome. If you want to fix a bug, add a snippet, improve accessibility, or build a new feature, please open an issue first to discuss the change, then submit a pull request against `main`.

---

## Future updates

- **Improved mobile support** — Better touch targets, responsive layout adjustments for small screens, and swipe-to-open sidebar.
- **Expanded snippet library** — More prebuilt expressions covering physics, chemistry notation, number theory, and signal processing.
- **Advanced text formatting** — Rich label support, colour annotations, and multi-line expression grouping.
- **Drag-and-drop block editor** — A visual, block-based interface for building equations without typing LaTeX — similar to a coding block tool but for math.

---

## License

MIT License — see [LICENSE](https://raw.githubusercontent.com/xaviershilosaputra/SnapTex/main/LICENSE) for the full text.

You are free to use, modify, and redistribute this project for any purpose, including commercially. Attribution to the original author must be maintained.
