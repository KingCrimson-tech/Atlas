# Building an Obsidian-like Markdown Editor in React — Full Technical Breakdown

> Project: `markdown/` — a live Markdown editor with split preview, GFM support, syntax highlighting, dark mode, and localStorage persistence.
> Stack: React 19 + Vite 7 + `react-markdown` + `remark-gfm` + `react-syntax-highlighter` + hand-rolled utility CSS.
> Use this article to **relearn the project from zero in ~15 minutes**.

---

## 1. TL;DR — What Did We Build?

A single-page app with two panes:

```
┌─────────────────────────────────────────────────┐
│ Header: Title | [Write|Split|Preview] Export MD │
│         Toolbar: Bold Italic H1 H2 H3 Code ...  │
├──────────────────────┬──────────────────────────┤
│                      │                          │
│   <textarea>         │   <ReactMarkdown>        │
│   raw markdown       │   rendered HTML          │
│   (controlled)       │   + Prism highlighting   │
│                      │                          │
├──────────────────────┴──────────────────────────┤
│ Footer: Words: N | Chars: M | Mode | autosaved  │
└─────────────────────────────────────────────────┘
```

Core behaviors:

1. **Type → see rendered HTML instantly** (controlled `textarea` → state → `ReactMarkdown`).
2. **GitHub Flavored Markdown** — tables, task lists, strikethrough, autolinks via `remark-gfm`.
3. **Code blocks get VS Code Dark+ highlighting** via `Prism`.
4. **Toolbar + `Ctrl+B` / `Ctrl+I`** insert formatting at the cursor.
5. **Theme: Auto / Dark / Light** — follows `prefers-color-scheme`, persisted.
6. **Content + theme persist** in `localStorage` — reload-safe.
7. **Export `.md`** via `Blob` + `URL.createObjectURL`.
8. **Live stats** — word / character count in footer.

If you remember only one sentence: **`markdown` state is the single source of truth; everything else is derived.**

---

## 2. Project Map — Where Is What?

```
markdown/
├── index.html                  # #root mount + /src/main.jsx entry
├── vite.config.js              # vite + @vitejs/plugin-react
├── package.json                # react, react-dom, react-markdown, remark-gfm,
│                               # react-syntax-highlighter, dompurify, tailwindcss
├── eslint.config.js
├── public/favicon.png
└── src/
    ├── main.jsx                # StrictMode + createRoot(<App/>)
    ├── App.jsx                 # thin wrapper → <MarkdownEditor/>
    ├── App.css                 # ALL layout styling (see §3)
    ├── index.css               # base fonts, button, link, .dark overrides
    └── components/
        └── MarkdownEditor.jsx  # ~360 lines — 95% of the logic lives here
```

There is no router, no backend, no store library. `App.jsx` exists only to keep `main.jsx` clean. All learning effort goes into `MarkdownEditor.jsx`.

Run it:

```bash
npm install
npm run dev      # vite dev server
npm run build    # production build
npm run preview  # preview dist/
npm run lint     # eslint .
```

---

## 3. The Styling Decision You Must Remember

`package.json` lists `tailwindcss@4`, but **there is no `@import "tailwindcss"` and no `tailwind.config.js`**.

Instead `src/App.css` hand-defines Tailwind-like utilities:

```css
.flex { display: flex; }
.w-1\/2 { width: 50%; }
.bg-gray-900 { background-color: #111827; }
.dark .dark\:bg-gray-900 { background-color: #111827; }
/* ... + .markdown-content h1/h2/p/code/pre/blockquote/a ... */
```

Plus custom `.markdown-content` styles for rendered HTML, custom scrollbars, and `.dark` variants.

Why this matters for relearning:

- You don't need to learn Tailwind to edit this project — just edit `App.css`.
- Class names *look* like Tailwind (`flex flex-col h-screen w-1/2 dark:bg-gray-900`) but they're local definitions.
- If you ever migrate to real Tailwind v4, delete most of `App.css` and add `@import "tailwindcss";` to `index.css` — the JSX class names will mostly keep working.

`src/index.css` handles globals: system font stack, `button`, `a`, and their `.dark` overrides.

---

## 4. Data Flow — The Mental Model

```
                    ┌──────────────┐
                    │ localStorage │
                    │  -content    │
                    │  -theme      │
                    └──────┬───────┘
                           │ load on mount (useEffect [])
                           ▼
user types ──► textarea (controlled, value={markdown})
                           │ onChange → setMarkdown
                           ├──► useEffect → localStorage.setItem(content)
                           ├──► footer word/char count (derived, no state)
                           └──► <ReactMarkdown> re-renders preview
                                     │
                                     ├── remarkGfm plugin
                                     └── code() override → Prism or <code>
```

State inventory (all in `MarkdownEditor`):

| State | Type | Default | Persisted as |
|---|---|---|---|
| `markdown` | `string` | welcome template | `markdown-editor-content` |
| `viewMode` | `"write" \| "split" \| "preview"` | `"split"` | no (session only) |
| `theme` | `"auto" \| true \| false` | `"auto"` | `markdown-editor-theme` |
| `textareaRef` | `useRef` | `null` | n/a — DOM handle for cursor ops |

Note the quirk: `theme` is **overloaded** — string `"auto"` vs boolean `true` (dark) / `false` (light). It works but is easy to misread. When relearning, read `true` = dark, `false` = light.

---

## 5. File-by-File Deep Dive

### 5.1 `src/main.jsx` + `src/App.jsx` + `index.html`

Standard Vite React bootstrap, nothing custom:

```jsx
// main.jsx
createRoot(document.getElementById('root')).render(
  <StrictMode><App /></StrictMode>,
)

// App.jsx
function App() {
  return <MarkdownEditor />
}
```

### 5.2 `src/components/MarkdownEditor.jsx` — the whole app

#### a) Mount: restore persisted state

```jsx
useEffect(() => {
  const savedTheme = localStorage.getItem("markdown-editor-theme");
  if (savedTheme === "true") setTheme(true);
  else if (savedTheme === "false") setTheme(false);
  else setTheme("auto");

  const savedMarkdown = localStorage.getItem("markdown-editor-content");
  if (savedMarkdown) setMarkdown(savedMarkdown);
}, []);
```

Runs once. `localStorage` stores only strings, hence the `"true"`/`"false"` dance.

#### b) Theme: apply + persist + follow OS

Three effects work together:

```jsx
// 1. apply theme to <html> + <body> whenever `theme` changes
useEffect(() => {
  let dark;
  if (theme === "auto") {
    dark = window.matchMedia("(prefers-color-scheme: dark)").matches;
  } else {
    dark = theme === true;
  }
  if (dark) {
    document.documentElement.classList.add("dark");
    document.body.style.backgroundColor = "#111827";
    // ...
  } else {
    document.documentElement.classList.remove("dark");
    // ...
  }
  localStorage.setItem("markdown-editor-theme", theme);
}, [theme]);

// 2. if in "auto", re-render when OS theme flips
useEffect(() => {
  if (theme !== "auto") return;
  const mq = window.matchMedia("(prefers-color-scheme: dark)");
  const handleChange = () => setTheme("auto"); // force re-eval
  mq.addEventListener("change", handleChange);
  return () => mq.removeEventListener("change", handleChange);
}, [theme]);
```

The toggle cycles `auto → dark (true) → light (false) → auto`:

```jsx
onClick={() => {
  if (theme === "auto") setTheme(true);
  else if (theme === true) setTheme(false);
  else setTheme("auto");
}}
// label: {theme === "auto" ? "Auto" : theme === true ? "Dark" : "Light"}
```

Gotcha for relearning: the root `div` *also* computes dark/light inline with `window.matchMedia(...)` during render (lines ~177-185). That duplicates the effect logic. It works, but a cleaner relearn refactor is `const isDark = theme === true || (theme === "auto" && matchMedia(...).matches)`.

There is a commented-out older `isDarkMode` boolean implementation — that's the evolutionary fossil. Current `auto/true/false` version supersedes it.

#### c) Autosave

```jsx
useEffect(() => {
  localStorage.setItem("markdown-editor-content", markdown);
}, [markdown]);
```

Every keystroke writes. Fine for KB-sized docs. If you extend to large docs, debounce this.

#### d) `insertFormatting(before, after)` — toolbar + shortcuts engine

This is the most "clever" function — learn it once:

```jsx
const insertFormatting = useCallback((before, after = "") => {
  const textarea = textareaRef.current;
  if (!textarea) return;
  const start = textarea.selectionStart;
  const end = textarea.selectionEnd;

  setMarkdown((prev) => {
    const selectedText = prev.substring(start, end);
    const newText =
      prev.substring(0, start) + before + selectedText + after + prev.substring(end);
    setTimeout(() => {
      textarea.focus();
      textarea.setSelectionRange(
        start + before.length,
        start + before.length + selectedText.length
      );
    }, 0);
    return newText;
  });
}, []);
```

Key ideas:

- Reads cursor/selection from the real DOM via `useRef` (earlier version manipulated DOM directly — git commit `455ecdb` fixed this to be "more React-y").
- Wraps selection: `**selection**`, `*selection*`, `[selection](https://example.com)`, or prefix-inserts `# `, `- `, `> ` when nothing selected.
- `setTimeout(..., 0)` restores caret *after* React re-renders — otherwise selection is lost.
- `useCallback` with `[]` keeps the reference stable for the keyboard-shortcut effect.

Toolbar is just config over this function:

```jsx
const toolbarButtons = [
  { label: "Bold",   title: "Bold (Ctrl+B)",   action: () => insertFormatting("**", "**") },
  { label: "Italic", title: "Italic (Ctrl+I)", action: () => insertFormatting("*", "*") },
  { label: "H1", action: () => insertFormatting("# ") },
  // H2, H3, Code (` `), Link ([ ](...)), List (- ), Quote (> )
];
```

Keyboard shortcuts delegate to the same engine:

```jsx
useEffect(() => {
  const handleKeyDown = (e) => {
    if (e.ctrlKey || e.metaKey) {
      switch (e.key) {
        case "b": e.preventDefault(); insertFormatting("**", "**"); break;
        case "i": e.preventDefault(); insertFormatting("*", "*"); break;
        case "s": e.preventDefault(); break; // prevent browser save dialog
      }
    }
  };
  window.addEventListener("keydown", handleKeyDown);
  return () => window.removeEventListener("keydown", handleKeyDown);
}, [insertFormatting]);
```

#### e) Rendering: `ReactMarkdown` + GFM + Prism

```jsx
<ReactMarkdown
  remarkPlugins={[remarkGfm]}
  components={{
    code({ node, inline, className, children, ...props }) {
      const match = /language-(\w+)/.exec(className || "");
      return !inline && match ? (
        <SyntaxHighlighter style={vscDarkPlus} language={match[1]} PreTag="div" {...props}>
          {String(children).replace(/\n$/, "")}
        </SyntaxHighlighter>
      ) : (
        <code className={className} {...props}>{children}</code>
      );
    },
  }}
>
  {markdown}
</ReactMarkdown>
```

What to remember:

- `remark-gfm` adds tables, task lists (`- [x]`), strikethrough, autolinks. Without it, those render as plain text.
- Fenced block language (` ```javascript `) arrives as `className="language-javascript"` — the regex extracts `javascript` for Prism.
- Inline code (`\`foo\``) falls through to plain `<code>`, styled by `.markdown-content code`.
- `react-markdown` escapes raw HTML by default — that's why the old `dompurify` sanitization step (see git history `5708057`, `0ea8808`) is no longer wired in. `dompurify` remains in `package.json` as an unused leftover. Safe to remove or re-add only if you enable `rehype-raw`.

#### f) View modes + export + stats

```jsx
// viewMode: "write" | "split" | "preview"
// write → textarea w-full, split → two w-1/2, preview → preview w-full

const exportAsMarkdown = () => {
  const blob = new Blob([markdown], { type: "text/markdown" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = "document.md";
  a.click();
  URL.revokeObjectURL(url);
};

// footer, derived on every render — no extra state needed:
Words: {markdown.split(/\s+/).filter((w) => w.length > 0).length}
Characters: {markdown.length}
```

---

## 6. Git History — How the Project Evolved (relearn in order)

```
d7ecfb5  MarkdownEditor component and rest of files      ← skeleton
56b743c  color changes
cd0f063  comments + markdown improvements
5708057  html sanitization via dompurify                 ← dangerouslySetInnerHTML era
0ea8808  switch to react-markdown                        ← safer parsing, dompurify orphaned
53b88ee  syntax highlighting (Prism)                     ← code() override born
1031725  follow OS color scheme
455ecdb  useRef instead of direct DOM manipulation       ← insertFormatting matured
3c736f9  final touches
```

Reading the log tells you *why* `dompurify` is still installed but unused, and why there's dead commented dark-mode code — both are fossils, not bugs you introduced.

---

## 7. Quick Relearn Checklist (5-minute refresh)

1. Single source of truth is `markdown` state in `MarkdownEditor.jsx`.
2. `textarea` is controlled; preview is `ReactMarkdown` + `remarkGfm`.
3. Toolbar/shortcuts all funnel through `insertFormatting(before, after)` + `textareaRef` + `setTimeout` caret restore.
4. Code highlighting = custom `code` component → `Prism` + `vscDarkPlus` when `language-*` matches, else plain `<code>`.
5. Theme = `"auto" | true | false`; effects sync `.dark` class + `body` styles + `localStorage`; OS changes re-trigger via `matchMedia` listener.
6. Persistence keys: `markdown-editor-content`, `markdown-editor-theme`.
7. Styling is hand-rolled utilities in `App.css`, not real Tailwind — edit there.
8. Export = `Blob` download; stats = derived split/length.

---

## 8. Known Rough Edges + Easy Next Steps

| Issue | Fix |
|---|---|
| `theme` as `string \| boolean` is confusing | Refactor to `"auto" \| "dark" \| "light"` |
| Dark check duplicated in render + effect | Extract `const isDark = ...` helper |
| Autosave on every keystroke | Debounce `localStorage.setItem` (~300ms) |
| Unused `dompurify`, `tailwindcss` deps | Remove or actually wire them |
| `/* eslint-disable no-unused-vars */` at top + unused `node`/`inline` destructure | Clean lint, rename to `_node` |
| No HTML export (comment says "as markdown and html") | Add `exportAsHtml` reusing rendered output |
| No debounced preview for huge docs | Memoize `ReactMarkdown` or debounce `markdown` for preview only |
| `inline` prop deprecated in newer `react-markdown` | Check for `inline` → use `node.position` alternative on upgrade |

Suggested starter extensions when relearning by doing:

1. Add `Ctrl+K` for link, `Ctrl+\`` for inline code.
2. Add word-goal progress bar in footer.
3. Add `Copy HTML` button.
4. Support drag-and-drop `.md` file open (mirror of export).
5. Replace boolean theme with string union + `useDarkMode()` custom hook — best refactor exercise.

---

## 9. One-Paragraph Recall

> The app is one component (`MarkdownEditor`) holding `markdown`, `viewMode`, and `theme`. Typing updates `markdown`, which autosaves to `localStorage` and re-renders through `ReactMarkdown` with GFM and Prism highlighting. All formatting goes through `insertFormatting` using cursor offsets from a `ref`. Theme resolves `auto` via `prefers-color-scheme`, toggles the `.dark` class, and persists. Layout is split/write/preview panes with a header toolbar and footer stats, styled by bespoke utilities in `App.css`.

Read this paragraph, then the checklist, then the code — you'll have the whole project back.
