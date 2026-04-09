# SnapTex — Technical Documentation

This document explains how SnapTex works in full detail. It is written for readers who may have little to no coding experience. Every concept is explained from first principles before the technical specifics. If you want to make a change to the project, this is the right place to start.

---

## Table of contents

1. [How the app works at a high level](#1-how-the-app-works-at-a-high-level)
2. [Technology primer](#2-technology-primer)
3. [File-by-file reference](#3-file-by-file-reference)
   - [Entry points](#entry-points)
   - [App.jsx — the brain](#appjsx--the-brain)
   - [Constants](#constants)
   - [Hooks](#hooks)
   - [Utils](#utils)
   - [Components](#components)
   - [Overlays](#overlays)
   - [Styles](#styles)
4. [Data flow walkthrough](#4-data-flow-walkthrough)
5. [How autocomplete works](#5-how-autocomplete-works)
6. [How localStorage persistence works](#6-how-localstorage-persistence-works)
7. [How the render pipeline works](#7-how-the-render-pipeline-works)
8. [How themes work](#8-how-themes-work)
9. [How the split-view drag handle works](#9-how-the-split-view-drag-handle-works)
10. [How history and autosave work](#10-how-history-and-autosave-work)
11. [How export works](#11-how-export-works)
12. [How ConfirmButton works](#12-how-confirmbutton-works)
13. [Adding a new snippet to the sidebar](#13-adding-a-new-snippet-to-the-sidebar)
14. [Adding a new autocomplete shorthand](#14-adding-a-new-autocomplete-shorthand)
15. [Adding a new toolbar button](#15-adding-a-new-toolbar-button)
16. [Adding a new theme](#16-adding-a-new-theme)
17. [Adding a new docs page](#17-adding-a-new-docs-page)
18. [Adding a new settings toggle](#18-adding-a-new-settings-toggle)
19. [Changing the GitHub Pages base path](#19-changing-the-github-pages-base-path)
20. [Common errors and fixes](#20-common-errors-and-fixes)

---

## 1. How the app works at a high level

SnapTex is a single-page web application. When you open it in a browser:

1. The browser loads `index.html`, which is a nearly empty HTML file. It has one `<div id="root">` and a `<script>` tag pointing to the JavaScript.
2. React (a JavaScript library) takes over and builds the entire visible interface in that `<div>` using JavaScript.
3. The user types LaTeX into a `<textarea>`. React sees every keystroke, updates its internal state, and re-renders the relevant parts of the screen.
4. The preview panel calls KaTeX (a math typesetting library) to convert the raw LaTeX string into rendered HTML with proper mathematical symbols.
5. Everything the user has typed, their settings, and their version history are saved to the browser's `localStorage` — a small built-in database inside every browser. No server is involved at any point.

When the user leaves and comes back, the app reads from `localStorage` and restores exactly where they left off.

---

## 2. Technology primer

You do not need to fully understand these technologies to make simple changes, but knowing what each one does helps you know which file to look in.

### React

React is a JavaScript library for building user interfaces. The core idea is that the UI is a function of state — you describe what the screen should look like for a given state, and React figures out the minimal set of DOM changes needed to get there. Components are the building blocks: each one is a JavaScript function that returns HTML-like syntax (called JSX).

```jsx
// A simple React component
function Greeting({ name }) {
  return <p>Hello, {name}!</p>
}
```

When `name` changes, React re-renders just this component.

### JSX

JSX looks like HTML but it is actually JavaScript. It gets compiled by Vite into regular function calls before it reaches the browser. A JSX file has the extension `.jsx`.

```jsx
// JSX
const el = <button onClick={() => alert('clicked')}>Click me</button>

// What it compiles to (you never write this yourself)
const el = React.createElement('button', { onClick: () => alert('clicked') }, 'Click me')
```

### State and props

- **State** is data that belongs to a component and can change over time. When state changes, React re-renders the component. State is created with `useState`.
- **Props** (properties) are values passed from a parent component to a child component. They are read-only from the child's perspective.

```jsx
function Counter() {
  const [count, setCount] = useState(0)  // count is state
  return <button onClick={() => setCount(count + 1)}>Count: {count}</button>
}
```

### Hooks

Hooks are special React functions whose names start with `use`. They let you tap into React features from inside a function component. The ones used in this project:

- `useState` — holds a piece of state
- `useEffect` — runs a side effect (like saving to localStorage) after a render
- `useRef` — holds a mutable value that does not trigger re-renders (used for the textarea DOM element)
- `useCallback` — memoises a function so it is not recreated on every render
- `useMemo` — memoises a computed value

### KaTeX

KaTeX is a fast, open-source library that renders LaTeX math strings into HTML and CSS. It is the same library used by Khan Academy and Wikipedia. SnapTex calls `katex.render(latexString, domElement, options)` every time the source changes.

### Vite

Vite is the build tool and development server. During development, it serves files directly and reloads the browser instantly on save. For production, it bundles and minifies everything into the `dist/` folder.

### localStorage

`localStorage` is a key-value store built into every browser. It persists data across page refreshes and browser restarts until the user manually clears it. Values must be strings, so SnapTex uses `JSON.stringify` when writing and `JSON.parse` when reading.

```js
localStorage.setItem('theme', JSON.stringify('dark'))
const theme = JSON.parse(localStorage.getItem('theme'))  // 'dark'
```

---

## 3. File-by-file reference

### Entry points

#### `index.html`

The only HTML file in the project. Nearly empty — it exists just to load the JavaScript bundle and provide the `<div id="root">` mounting point. You should rarely need to touch this file unless you want to change the page `<title>` or add a `<meta>` tag.

#### `src/main.jsx`

The JavaScript entry point. It imports React, the global CSS, and the root `App` component, then mounts the app into `<div id="root">`:

```jsx
createRoot(document.getElementById('root')).render(
  <StrictMode>
    <App />
  </StrictMode>
)
```

`StrictMode` is a development-only wrapper that helps catch bugs by intentionally rendering components twice. It has no effect in production.

#### `vite.config.js`

Tells Vite how to build the project. The most important setting is `base`, which must match your GitHub repository name for GitHub Pages to serve assets from the correct path.

```js
base: '/snaptex/',  // change this to match your repo name
```

The `manualChunks` option splits the output into separate JS files for React (`vendor`) and KaTeX (`katex`), so browsers can cache them independently and avoid downloading them again when only your app code changes.

---

### `App.jsx` — the brain

`App.jsx` is the largest and most central file. It owns all the global state (the latex string, theme, zoom, history, settings) and passes pieces of it down to every component as props. It also holds the logic for autosave, export, history management, and keyboard shortcuts.

**State declared in App.jsx:**

| Variable | Type | Default | What it controls |
|---|---|---|---|
| `latex` | string | quadratic formula | The raw LaTeX source in the editor |
| `docTitle` | string | `'Untitled equation'` | The title shown and editable in the topbar |
| `theme` | string | `'system'` | Active theme key |
| `zoom` | number | `1` | Preview zoom multiplier |
| `editorFlex` | number | `50` | Split-view left panel width as a percentage |
| `recentlyUsed` | array | `[]` | Last 8 inserted snippets |
| `history` | array | `[]` | Autosave/manual snapshots |
| `fontSize` | number | `13` | Editor textarea font size in px |
| `showLineNums` | boolean | `true` | Whether the line number column is visible |
| `autoSave` | boolean | `true` | Whether the 2-second autosave is active |
| `wordWrap` | boolean | `false` | Whether long lines wrap in the editor |
| `viewMode` | string | `'split'` | One of `'split'`, `'edit'`, `'preview'` |
| `sidebarOpen` | boolean | `true` | Whether the sidebar is visible |
| `focusMode` | boolean | `false` | Whether focus mode is active |
| `confirmClear` | boolean | `false` | Whether the Clear button is in confirm state |
| `renderError` | string or null | `null` | Current KaTeX error message, if any |
| `toast` | string | `''` | Current toast notification message |
| `acItems`, `acSel`, `acTrigger`, `acPos` | various | empty | Autocomplete dropdown state |
| overlay booleans | boolean | `false` | Whether each overlay panel is open |

All state marked as "persisted" uses the `useLocalStorage` hook instead of plain `useState`, which means changes are automatically written to `localStorage`.

**Key functions in App.jsx:**

- `saveSnapshot(src, manual)` — creates a history entry. Called by autosave and by Ctrl+S.
- `trackRecent(item)` — prepends an item to the `recentlyUsed` array, capped at `MAX_RECENT` (8).
- `insertAtCursor(str, item)` — calls `insertRef.current(str)` to inject a string at the current cursor position inside the textarea, then tracks the item as recently used.
- `handleClear()` — implements the two-step clear: first click sets `confirmClear = true` with a 3-second timeout; second click actually clears the editor and saves a snapshot first.
- `handleExportPDF/Tex/SVG()` — the three export handlers.
- `handleSplitDrag()` — the mouse drag handler for resizing the split view.
- `closeAll()` — closes every overlay, used by the Escape key handler.

**The `insertRef` pattern:**

`insertRef` is a `useRef` that holds a function. `Editor.jsx` writes its cursor-aware insert function into `insertRef.current` via a `useEffect`. Every other component (`Sidebar`, `ToolbarStrip`, overlays) calls `insertAtCursor` in `App.jsx`, which in turn calls `insertRef.current`. This is how toolbar clicks and sidebar clicks still correctly place text at the cursor position inside the textarea — even though the textarea lives in a different component.

---

### Constants

Constants files contain only pure data — arrays and objects with no logic. If you want to add or change a snippet, shorthand, or theme, these are almost always the only files you need to edit.

#### `src/constants/shortcuts.js`

Exports `SHORTCUTS` — an array of objects used by the autocomplete system. Each object has:

```js
{
  label:   '\\frac{}{}',   // the LaTeX string that gets inserted
  trigger: '\\fr',         // the typed prefix that activates this suggestion
  sym:     'a/b',          // a short display symbol shown in the dropdown
  desc:    'Fraction',     // a human-readable description
  cat:     'Structure',    // the category shown in the All Functions overlay
}
```

Also exports `SHORTCUT_CATS` — a deduplicated list of all category names, used to render the grouped sections in the All Functions overlay.

#### `src/constants/toolbar.js`

Exports `TOOLBAR` — an array of groups, each containing an array of button definitions:

```js
{
  sym:   'a/b',           // what the button displays
  latex: '\\frac{}{}',    // what gets inserted when clicked
  tip:   'Fraction',      // tooltip text
}
```

#### `src/constants/sidebar.js`

Exports `SIDEBAR_GROUPS` — an array of groups, each with a `label` and an `items` array:

```js
{
  sym:   'qf',
  label: '\\frac{-b \\pm \\sqrt{b^2 - 4ac}}{2a}',  // the full expression inserted
  desc:  'Quadratic formula',
}
```

Unlike toolbar buttons, sidebar snippets typically insert complete, ready-to-use expressions rather than templates with empty `{}` slots.

#### `src/constants/docs.js`

Exports `DOCS` — an array of documentation pages. Each page has a `title` string and a `content` string (plain text with manual formatting). To add a new docs page, append an object to this array.

#### `src/constants/themes.js`

Exports two objects:
- `THEMES` — maps theme keys (`'light'`, `'dark'`, etc.) to the CSS class applied to `document.body`.
- `THEME_LABELS` — maps those same keys to human-readable display strings for the UI.

---

### Hooks

Hooks are reusable pieces of stateful logic that any component can use.

#### `src/hooks/useLocalStorage.js`

A drop-in replacement for `useState` that also syncs to `localStorage`. Usage is identical:

```js
// Instead of:
const [theme, setTheme] = useState('system')

// Use:
const [theme, setTheme] = useLocalStorage('theme', 'system')
//                                          ^key     ^default
```

The first argument is the `localStorage` key. The default value is only used if no value has been stored yet. Every time `setTheme` is called, the new value is also written to `localStorage` via a `useEffect`.

#### `src/hooks/useAutosave.js`

Takes three arguments: `latex` (the current source), `enabled` (boolean), and `onSave` (the function to call). It uses `useEffect` to watch for changes to `latex`. When `latex` changes and differs from the last saved version, it sets a 2-second debounce timer. If the user keeps typing, the timer resets. If they pause for 2 seconds, `onSave` is called.

```js
useAutosave(latex, autoSave, saveSnapshot)
```

#### `src/hooks/useKeyboardShortcuts.js`

Attaches a single global `keydown` event listener on `window`. Handles three shortcuts:
- `Escape` → calls `onCloseAll()`
- `Ctrl+S` / `Cmd+S` → calls `onSave()` and prevents the browser's save dialog
- `Ctrl+Shift+F` / `Cmd+Shift+F` → calls `onToggleFocus()`

The listener is cleaned up (removed) when the component unmounts, preventing memory leaks.

---

### Utils

Utils are plain JavaScript functions with no React inside them.

#### `src/utils/storage.js`

Exports:

- `LS` — an object with `get(key, default)` and `set(key, value)` methods. Wraps `localStorage` in a try/catch so a storage quota error or a private-browsing restriction never crashes the app. Values are automatically serialised/deserialised with `JSON.parse` / `JSON.stringify`.
- `MAX_HISTORY` — `20`. The maximum number of history snapshots kept.
- `MAX_RECENT` — `8`. The maximum number of recently used items shown.

#### `src/utils/render.js`

Exports:

- `renderBlocks(src, container, zoom, fontSize)` — Splits the LaTeX source string at every newline. Each non-empty line is treated as a separate equation and rendered into its own `<div>` using `katex.render()`. If KaTeX throws an error for a line, that line renders as a red error message instead of crashing the whole preview. Returns the first error message encountered (or `null` if all lines rendered successfully). The `zoom` and `fontSize` arguments control the output `font-size` CSS property.

- `getCursorWord(text, cursor)` — Takes the full text and the current cursor position. Returns the word immediately before the cursor, split on whitespace, `{`, `}`, and `$`. This is used by the autocomplete system to extract the current trigger word (e.g. `\fr` from `\sin(\fr`).

- `pluralize(n, word)` — Returns `"1 char"` or `"5 chars"`. Used in the status bar.

---

### Components

#### `src/components/Icons.jsx`

All SVG icons used across the app in one file. Each is a small React component that returns an `<svg>` element. Keeping them here means you only need to look in one place to find or change an icon.

Current icons: `IconSidebar`, `IconHistory`, `IconSettings`, `IconFocus`, `IconCopy`, `IconExport`, `IconClose`, `IconChevronDown`.

To add a new icon, add a new exported function here:

```jsx
export const IconMyIcon = () => (
  <svg viewBox="0 0 14 14" fill="none" stroke="currentColor" strokeLinecap="round" strokeWidth="1.6">
    {/* your SVG path data here */}
  </svg>
)
```

#### `src/components/Topbar.jsx`

The horizontal navigation bar at the top. Manages its own local state for whether the title is being edited (`editingTitle`), and whether the theme/export dropdown menus are open.

Receives everything it needs as props from `App.jsx` — it holds no global state itself. When the user changes the theme or clicks Export, it calls the handler functions passed in from `App.jsx`.

The theme picker and export options are both built as custom dropdown menus (`.dropdown-menu`). They close when you click outside them via a `mousedown` listener on `document`, which is cleaned up in the effect's return function.

#### `src/components/ToolbarStrip.jsx`

A thin component. Renders the `TOOLBAR` constant as rows of buttons. When a button is clicked, it calls `onInsert(item.latex)` — a function passed from `App.jsx` that handles cursor-aware insertion.

#### `src/components/Sidebar.jsx`

Manages its own local search query state (`query`). Filters `SIDEBAR_GROUPS` using `useMemo` — the filtered result is only recomputed when `query` changes, not on every render. Renders the recently used section (only when `query` is empty and `recentlyUsed` is non-empty) and the filtered groups. Each snippet calls `onInsert(item.label, item)`.

#### `src/components/Editor.jsx`

The source code panel. Contains the textarea, optional line numbers, the autocomplete dropdown, the error bar, and the status bar.

The most complex part is the `insertRef` mechanism. `App.jsx` passes down a `insertRef` object (created with `useRef(null)`). Inside a `useEffect`, `Editor.jsx` writes a function into `insertRef.current`:

```js
insertRef.current = (str, replaceTrigger = '') => {
  // reads cursor position from the textarea DOM element
  // builds the new string
  // calls onChange(newString) to update state in App.jsx
  // uses requestAnimationFrame to restore focus and move cursor into the first {}
}
```

This function is then callable from anywhere in the app by going through `App.jsx`'s `insertAtCursor`, which calls `insertRef.current(str)`.

The `useEffect` that writes `insertRef.current` has `[latex, onChange, insertRef]` as its dependency array, which means it is re-written every time `latex` changes. This is intentional — the function closes over `latex`, so it needs to be refreshed to always operate on the latest version of the string.

#### `src/components/Preview.jsx`

A simple presentational component. Renders the panel header with zoom controls and a `<div>` whose ref is passed in from `App.jsx`. The actual KaTeX rendering happens in `App.jsx`'s `useEffect` (which calls `renderBlocks` from `utils/render.js`) and writes directly to the DOM node held by `previewRef`.

---

### Overlays

All four overlay components share the same structure: a `.overlay-backdrop` (the dark semi-transparent background) that closes the overlay when clicked, and an `.overlay-panel` that stops click propagation so clicking inside the panel does not close it.

#### `src/components/overlays/AllFunctionsOverlay.jsx`

Manages its own `query` state. When the query is empty, renders all shortcuts grouped by `cat` using `SHORTCUT_CATS`. When the query is non-empty, renders a flat filtered list. Each item calls `onInsert(item.label, item)` and then `onClose()`.

#### `src/components/overlays/DocsOverlay.jsx`

Manages its own `activeDoc` index state. Renders a two-column layout: the left column lists all doc page titles as buttons, the right column shows the content of the active page as preformatted text.

#### `src/components/overlays/HistoryOverlay.jsx`

Receives `history`, `onRestore`, `onDelete`, `onClearAll`, and `onClose` as props. When the history array is empty, shows an explanatory message. Otherwise renders each snapshot in a card with Restore, Copy, and Delete actions.

#### `src/components/overlays/SettingsOverlay.jsx`

Receives every setting value and its corresponding setter as a prop. Structured with two helper components defined in the same file: `SettingsGroup` (a labelled section) and `SettingsRow` (a label + control row with a bottom border). This keeps the JSX in the main return statement clean and readable.

At the bottom is the keyboard shortcuts reference table, which is a plain array constant defined at the top of the file.

---

### Styles

All CSS is split into purpose-specific files and imported in order via `src/index.css`. Later files can override earlier ones. The cascade is:

`base` (variables, reset) → `layout` (app shell) → `topbar` → `toolbar` → `sidebar` → `editor` → `preview` → `overlays`

Every colour and spacing decision relies on CSS custom properties (`var(--name)`). These properties are defined on `:root` for the default (light) theme and overridden on `.theme-dark`, `.theme-sepia`, etc. This is how the entire colour scheme switches when a theme class is added to `document.body`.

#### `src/styles/base.css`

Contains:
- The Google Fonts `@import` for IBM Plex Mono and IBM Plex Sans.
- The CSS reset (`*, *::before, *::after { box-sizing: border-box; ... }`).
- `:root` — the full set of CSS custom properties for the default light theme.
- `.theme-light`, `.theme-dark`, `.theme-sepia` — overrides for each named theme.
- `@media (prefers-color-scheme: dark)` — applies the dark overrides automatically when no explicit theme class is set (i.e. when theme is `'system'`).
- Global `html`, `body`, `#root` styles.
- Custom scrollbar styles.

**To change any colour**, find the relevant `--variable-name` in this file and update its value.

#### `src/styles/overlays.css`

The largest CSS file. Contains styles for every overlay component: the shared backdrop and panel shell, the docs layout, history cards, and the settings rows. All overlay-specific classes live here so they do not pollute the component-level stylesheets.

---

## 4. Data flow walkthrough

Here is what happens step by step when the user types `\fr` in the editor:

1. The `<textarea>` in `Editor.jsx` fires `onChange`, which calls `handleChange`.
2. `handleChange` calls `onChange(val)` — a prop passed from `App.jsx`. This calls `setLatex(val)`, updating the root state.
3. `getCursorWord` extracts `\fr` from the text before the cursor.
4. `\fr` starts with `\` and has length > 1, so the autocomplete filter runs.
5. `SHORTCUTS.filter(s => s.trigger.startsWith('\fr'))` matches the `\\frac{}{}` entry.
6. `onSetAcItems`, `onSetAcSel`, `onSetAcTrigger`, `onSetAcPos` are called to update the autocomplete state in `App.jsx`.
7. `App.jsx` passes the updated `acItems`, `acSel`, `acPos` back down to `Editor.jsx` as props.
8. `Editor.jsx` renders the `.ac-dropdown` because `acItems.length > 0`.

Meanwhile:

9. The `useEffect` in `App.jsx` that watches `[latex, zoom, fontSize]` fires.
10. It calls `renderBlocks(latex, previewRef.current, zoom, fontSize)`.
11. KaTeX renders `\fr` — which is not valid LaTeX — and returns an error.
12. `setError(err)` updates the `renderError` state, causing the red error bar to appear in `Editor.jsx`.

When the user presses Tab:

13. `handleKeyDown` in `Editor.jsx` detects Tab with `acItems.length > 0`.
14. It calls `onAcApply(acItems[acSel])`, a prop from `App.jsx`.
15. `App.jsx`'s `applyAC` calls `insertRef.current(item.label, acTrigger)`.
16. The function stored in `insertRef.current` (written by `Editor.jsx`) reads the textarea's cursor position, replaces `\fr` with `\frac{}{}`, calls `onChange(newSrc)`, and positions the cursor inside the first `{}`.
17. `trackRecent(item)` is called, adding the frac entry to `recentlyUsed`.
18. `hideAC()` clears the autocomplete dropdown.
19. The `useEffect` fires again with the new `latex` value and KaTeX renders a valid (if empty) fraction.

---

## 5. How autocomplete works

The autocomplete system works by watching what is being typed to the left of the cursor. The trigger detection lives in `handleChange` inside `Editor.jsx`:

```js
const word = getCursorWord(val, cursor)
// word = the characters since the last whitespace/brace/dollar sign
// e.g. if you typed "x + \fr", word = "\fr"

if (word.startsWith('\\') && word.length > 1) {
  const filtered = SHORTCUTS.filter(
    s => s.trigger.startsWith(word) || s.label.startsWith(word)
  ).slice(0, 8)
  // filtered = all shortcuts whose trigger begins with "\fr"
  // = [{ label: '\\frac{}{}', trigger: '\\fr', ... }]
}
```

Filtering checks both `trigger` (the shorthand like `\fr`) and `label` (the full command like `\frac`), so users can also type out the start of the actual LaTeX command.

The position of the dropdown (`acPos`) is calculated by estimating where the cursor is in the textarea using the line height and line number. The dropdown is rendered as a `position: fixed` element in `Editor.jsx` so it can escape the overflow-hidden editor pane.

When the user accepts a suggestion with Tab or Enter, `applyAC` replaces the trigger word with the full label. It does this by slicing the text up to the cursor, removing the trigger from the end, appending the label, then appending the rest of the text after the cursor.

---

## 6. How localStorage persistence works

The `useLocalStorage` hook is the core of persistence:

```js
export function useLocalStorage(key, defaultValue) {
  const [value, setValue] = useState(() => LS.get(key, defaultValue))
  // ^ reads from localStorage on first render only (lazy initialiser)

  useEffect(() => {
    LS.set(key, value)
  }, [key, value])
  // ^ writes to localStorage whenever value changes

  return [value, setValue]
}
```

The lazy initialiser (`() => LS.get(...)`) means `localStorage.getItem` is only called once — during the first render — not on every render. After that, the value lives in React state, and any change syncs to `localStorage` in the background via `useEffect`.

The `LS` utility wraps everything in try/catch because `localStorage` can throw in private browsing modes on some browsers, or when storage is full. Failing silently means the app still works — it just will not persist data.

---

## 7. How the render pipeline works

The preview re-renders every time `latex`, `zoom`, or `fontSize` changes. This is controlled by a `useEffect` in `App.jsx`:

```js
useEffect(() => {
  if (!previewRef.current) return
  const src = latex.trim()
  if (!src) {
    setError(null)
    previewRef.current.innerHTML = ''
    return
  }
  const err = renderBlocks(src, previewRef.current, zoom, fontSize)
  setError(err)
}, [latex, zoom, fontSize])
```

`renderBlocks` (in `src/utils/render.js`) splits the source on newlines, then calls `katex.render()` for each non-empty line. Each equation goes into its own `<div>` so they stack vertically. The font size of the container is set proportionally to both the zoom level and the editor font size setting.

KaTeX is called with `throwOnError: true` so syntax errors produce a thrown exception that can be caught per-line. This means one broken equation does not prevent others on different lines from rendering. The error div is styled in red with `var(--red)`.

---

## 8. How themes work

Themes work by adding a class to `document.body` and overriding every CSS custom property at that class level. The effect hook is:

```js
useEffect(() => {
  document.body.className = THEMES[theme] || ''
}, [theme])
```

When `theme` is `'dark'`, `THEMES['dark']` is `'theme-dark'`, so `document.body.className` becomes `'theme-dark'`. The CSS in `base.css` defines all properties under `.theme-dark`, which overrides the `:root` defaults for every element in the page.

When `theme` is `'system'`, `THEMES['system']` is `''`, so no class is added. The `@media (prefers-color-scheme: dark)` block in `base.css` then applies the dark overrides automatically based on the operating system setting.

---

## 9. How the split-view drag handle works

The drag handle is the thin vertical strip between the editor and preview in split mode. Dragging it adjusts `editorFlex` (a percentage stored in state), which is applied as a `flex-basis` to the editor column:

```jsx
<div style={{ flex: `0 0 ${editorFlex}%` }}>
  {/* editor */}
</div>
<div style={{ flex: `0 0 ${100 - editorFlex}%` }}>
  {/* preview */}
</div>
```

The drag handler in `App.jsx` attaches `mousemove` and `mouseup` listeners directly to `window` (not to the drag handle element) so the drag continues even if the mouse moves off the handle. It calculates the new percentage by comparing the current mouse X position to where the drag started and the total width of the container. On `mouseup`, the listeners are removed.

---

## 10. How history and autosave work

`saveSnapshot(src, manual)` creates a history entry:

```js
const snap = {
  id:    Date.now(),  // unique id, also sortable by time
  latex: src,
  title: docTitle,
  ts:    new Date().toLocaleString(),  // human-readable timestamp
}
setHistory(prev => {
  const deduped = prev.filter(s => s.latex !== src)  // remove exact duplicates
  return [snap, ...deduped].slice(0, MAX_HISTORY)    // newest first, capped at 20
})
```

Autosave is handled by `useAutosave`. It debounces saves — the timer resets every time `latex` changes, so a save only happens after the user has stopped typing for 2 full seconds. It also compares the current value to a `lastSaved` ref to avoid saving identical content twice.

Manual save via Ctrl+S calls `saveSnapshot(latex, true)`, which also shows a toast notification. Clear always saves a snapshot before wiping the content so the user can recover it from history.

---

## 11. How export works

**PDF / Print:** Opens a new browser window, writes a complete self-contained HTML document into it (including the already-rendered KaTeX HTML from the preview panel and the KaTeX stylesheet), then calls `window.print()` on that window. The browser's print dialog handles page formatting and PDF generation.

**Download .tex:** Creates a `Blob` with `text/plain` mime type containing the raw LaTeX source, creates a temporary `<a>` element with a `download` attribute, programmatically clicks it, then revokes the object URL. No server involved.

**Export SVG:** Finds the first `<svg>` element inside the preview panel (KaTeX renders math as SVG), creates a Blob from its `outerHTML`, and downloads it the same way as the `.tex` file.

---

## 12. How ConfirmButton works

`ConfirmButton` (in `src/components/ConfirmButton.jsx`) is a controlled component that wraps a button with a two-step confirmation pattern. On the first click it enters a "pending" state, changing its label to something like "Confirm?" and starting a timeout. On the second click it fires the actual action. If the user does not confirm within the timeout, it resets to its default state.

It is used for destructive actions: the **Clear** button in the editor and the **Reset** and **Clear** buttons in the Settings overlay. Without it, a mis-click could wipe content or reset settings immediately.

To use it in a new context, pass it the button label, the confirm label, the timeout duration, and the action to run on confirmation:

```jsx
<ConfirmButton
  label="Delete"
  confirmLabel="Are you sure?"
  timeout={3000}
  onConfirm={() => handleDelete(item.id)}
/>
```

The component manages only its own `pending` boolean state internally, so the parent does not need to track the confirmation phase.

---

## 13. Adding a new snippet to the sidebar

Open `src/constants/sidebar.js`. Find the group you want to add the snippet to (or create a new group). Add an object to the `items` array:

```js
{ sym: '∇²', label: '\\nabla^2 f', desc: 'Laplacian' }
```

- `sym` — a short display character shown as the icon on the left of the row. One or two characters works best.
- `label` — the full LaTeX string that will be inserted into the editor when clicked.
- `desc` — a short description shown below the label.

Save the file. The development server will reload and your snippet will appear immediately.

---

## 14. Adding a new autocomplete shorthand

Open `src/constants/shortcuts.js`. Add an object to the `SHORTCUTS` array:

```js
{ label: '\\nabla^2 {}', trigger: '\\lap', sym: '∇²', desc: 'Laplacian', cat: 'Operators' }
```

- `trigger` must start with `\`. It is the prefix the user types to see this suggestion.
- `label` is what gets inserted. Place `{}` where you want the cursor to land after insertion.
- `cat` must be one of the existing category strings (Structure, Operators, Accents, Greek, Functions, Relations, Logic, Formatting) or a new one — new categories will appear automatically in the All Functions overlay.

---

## 15. Adding a new toolbar button

Open `src/constants/toolbar.js`. Find the group where the button belongs (or add a new group object to the `TOOLBAR` array). Add a button definition:

```js
{ sym: '∇²', latex: '\\nabla^2 {}', tip: 'Laplacian' }
```

- `sym` — the character displayed on the button face. Keep it very short (one or two characters).
- `latex` — the string inserted when clicked.
- `tip` — tooltip shown on hover and used as the `aria-label`.

---

## 16. Adding a new theme

**Step 1:** Open `src/constants/themes.js` and add the theme to both objects:

```js
export const THEMES = {
  // ...existing themes...
  ocean: 'theme-ocean',
}

export const THEME_LABELS = {
  // ...existing labels...
  ocean: 'Ocean',
}
```

**Step 2:** Open `src/styles/base.css` and add a new rule block defining all the required custom properties for your theme:

```css
.theme-ocean {
  --bg:         #0a1628;
  --bg-subtle:  #0d1f3c;
  --bg-hover:   #122347;
  --bg-active:  #172a52;
  --border:     rgba(100, 160, 255, 0.12);
  --border-med: rgba(100, 160, 255, 0.22);
  --text:       rgba(200, 225, 255, 0.90);
  --text-2:     rgba(200, 225, 255, 0.60);
  --text-3:     rgba(200, 225, 255, 0.35);
  --text-ph:    rgba(200, 225, 255, 0.20);
  --red:        #ff6b6b;
  --red-bg:     rgba(255, 107, 107, 0.12);
  color-scheme: dark;
}
```

You must define all of the above properties. Any property you omit will fall back to the `:root` defaults, which may look wrong.

The new theme will appear automatically in the gear icon dropdown in the topbar.

---

## 17. Adding a new docs page

Open `src/constants/docs.js`. Append a new object to the `DOCS` array:

```js
{
  title: 'Physics notation',
  content: `COMMON SYMBOLS
  \\hbar        ℏ  (reduced Planck constant)
  \\nabla^2     ∇² (Laplacian)
  ...`
}
```

The `content` string is rendered as preformatted text (inside a `<pre>` tag), so line breaks and spaces are preserved exactly as you write them. There is no Markdown — just plain text.

The new page will appear in the docs navigation sidebar automatically.

---

## 18. Adding a new settings toggle

**Step 1:** Declare the state in `App.jsx`:

```js
const [myOption, setMyOption] = useLocalStorage('myOption', false)
```

**Step 2:** Pass it to `SettingsOverlay` in `App.jsx`'s JSX:

```jsx
<SettingsOverlay
  // ...existing props...
  myOption={myOption}
  onMyOption={setMyOption}
/>
```

**Step 3:** Add a prop to the `SettingsOverlay` component signature in `src/components/overlays/SettingsOverlay.jsx` and add a `SettingsRow` for it:

```jsx
export default function SettingsOverlay({
  // ...existing props...
  myOption, onMyOption,
}) {
  return (
    // ...
    <SettingsRow label="My option" desc="What this option does">
      <input
        type="checkbox"
        className="settings-toggle"
        checked={myOption}
        onChange={e => onMyOption(e.target.checked)}
      />
    </SettingsRow>
  )
}
```

**Step 4:** Use the value wherever it is needed. Pass it as a prop to whatever component should behave differently based on it.

---

## 19. Changing the GitHub Pages base path

If you rename your repository or transfer it, update the `base` field in `vite.config.js`:

```js
base: '/new-repo-name/',
```

If you add a custom domain (e.g. `snaptex.example.com`), set `base: '/'`. The base path tells Vite where to prefix asset URLs in the built HTML. Getting it wrong results in blank pages because the browser cannot find the JS and CSS files.

After changing it, run `npm run build` locally and check that `dist/index.html` has the correct paths before pushing.

---

## 20. Common errors and fixes

### The preview is blank and shows no error

The most common cause is an empty editor. Check that the `latex` state is non-empty. If the editor has content but the preview is blank, open the browser console (F12) and look for JavaScript errors. A KaTeX version mismatch or a missing `katex/dist/katex.min.css` import can cause silent failures.

### "Cannot find module" on npm run dev

Run `npm install` again. A missing `node_modules` folder is the most common cause. If the error names a specific package, install it: `npm install package-name`.

### GitHub Pages shows a 404

Check that `base` in `vite.config.js` matches your repository name exactly, including capitalisation. Also confirm that GitHub Pages is set to deploy from GitHub Actions (Settings → Pages → Source → GitHub Actions), not from a branch.

### Autocomplete triggers the wrong shorthand

Open `src/constants/shortcuts.js` and check for duplicate `trigger` values. The filter returns all matches, so duplicates will show multiple entries in the dropdown for the same trigger. Remove or rename the duplicate.

### Adding a snippet but it does not appear

Make sure the object has all three required fields: `sym`, `label`, and `desc`. A missing field will not throw an error but the item will render oddly or not at all. Also confirm you saved the file — the dev server only reloads after a save.

### A theme looks wrong in dark mode

The `@media (prefers-color-scheme: dark)` block in `base.css` only applies when the theme is set to `'system'` (because the selector is `:root:not(.theme-light):not(.theme-sepia)`). If you add a new named theme and want it to apply its own dark-mode overrides, add a similar `:not(.theme-yourtheme)` exclusion to that media query so the system dark mode does not override it.

### The "Clear" button does nothing on first click

This is intentional — `ConfirmButton` requires two clicks within 3 seconds. The first click changes the button label to "Confirm?". Click again to confirm. This prevents accidental data loss.

### Autosave is not saving

Check that `autoSave` is `true` in the settings panel. Also note that autosave only fires if the content has changed since the last save — editing and then immediately undoing to the same state will not create a new snapshot.
