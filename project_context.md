# Project Context — NetOps Cheat Sheets

## 1. Technical Stack

- **Languages:** HTML5, CSS3, vanilla JavaScript (ES6+)
- **Frameworks/Libraries:** None — zero runtime dependencies
- **Fonts:** Google Fonts (Inter, homepage only); system monospace stack everywhere else
- **Hosting:** GitHub Pages (`https://Likkyh.github.io/cheat-sheets/`, custom domain `cheatsheets.ochosts.com`)
- **Build tools:** None — static files served directly

---

## 2. Architecture and File Structure

```
cheat-sheets/
├── index.html                  # Homepage: card grid, search, theme toggle
├── ios-reference-sheet.html    # Cisco IOS cheat sheet (~2 500 lines)
├── linux-reference-sheet.html  # Linux cheat sheet (~3 100 lines)
└── CNAME                       # Custom domain for GitHub Pages
```

Every file is fully self-contained (CSS + data + rendering engine inline). There is no build step, no bundler, no external JS file.

**UI components** — all inline `<style>` blocks inside each HTML file.
**Cheat-sheet data** — inline `<script>` blocks at the bottom of each HTML file, populating the global `SECTIONS` array.

---

## 3. Data Model

### Shared concepts

All sheets use a global `const SECTIONS = []` (IOS) or `SECTIONS.push(…)` calls (Linux). Each element is a **section object**.

---

### Section types

| `type` value | Used in | Rendered as |
|---|---|---|
| `'switching'` | IOS | command table (green header) |
| `'routing'` | IOS | command table (orange header) |
| `'commands'` | Linux | command table (blue header) |
| `'filesystem'` | Linux | pre tree block + directory cards |
| `'oneliners'` | Linux | card list with copy buttons |
| `'procedure'` | Both | numbered step cards |

---

### `commands` / `switching` / `routing` section

```js
{
  id: 'section-id',           // unique string, used as HTML anchor
  title: 'SECTION TITLE',     // displayed in TOC and section header
  type: 'commands',           // 'commands' | 'switching' | 'routing'
  subsections: [
    {
      id: 'sub-id',           // unique string, HTML anchor
      title: '1. Subsection Title',
      commands: [
        {
          cmd: 'command name',              // displayed in Command column
          syntax: 'cmd <req> [opt] {a|b}', // tokenized for syntax highlighting
          distros: ['all'],                 // Linux: 'all'|'ubuntu'|'arch'|'fedora'
          // IOS uses `devices` instead:
          devices: ['all'],                 // IOS: 'all'|'l2'|'l3'|'router'
          desc: 'One-sentence description.',
          args: [
            {
              name: 'param-name',   // argument name
              type: 'integer',      // string|integer|ip|path|flag|keyword|…
              range: '1–65535',     // valid range or '-'
              req: true,            // boolean — REQ vs opt badge
              def: '-',             // default value or '-'
              desc: 'What it does'
            }
          ]
          // args: [] is valid (no arguments)
        }
      ]
    }
  ]
}
```

**Syntax notation** (tokenized by `tokenizeSyntax()`):
- `<param>` → orange (required variable)
- `[param]` → light blue (optional)
- `{a|b}` → purple (choice)
- bare text → white (keyword)

---

### `procedure` section

```js
{
  id: 'procedures',
  title: 'PROCEDURES',
  type: 'procedure',
  subsections: [
    {
      id: 'proc-example',
      title: '1. Procedure Title',
      desc: 'Optional short description shown as italic subtitle.',
      steps: [
        {
          title: 'Step title',
          desc: 'Optional detail paragraph.',   // omit if not needed
          code: '# comment line\ncommand here\nanother command'
          // Lines starting with # → muted italic (Linux)
          // Lines starting with ! → muted italic (IOS)
          // Other lines → blue (var(--cmd-color))
          // omit `code` if the step has no code block
        }
      ]
    }
  ]
}
```

---

### `oneliners` section (Linux only)

```js
{
  id: 'oneliners',
  title: 'ONE-LINERS',
  type: 'oneliners',
  subsections: [
    {
      id: 'ol-category',
      title: 'Category Name',
      items: [
        {
          title: 'Short descriptive title',
          tag: 'tool-name',        // optional badge (e.g. 'grep', 'awk')
          cmd: 'the shell command',
          desc: 'What it does and why.'
        }
      ]
    }
  ]
}
```

---

### `filesystem` section (Linux only)

```js
{
  id: 'filesystem',
  title: 'FILESYSTEM',
  type: 'filesystem',
  tree: [
    { indent: '├── ', name: '/bin', desc: 'Essential user binaries' }
    // indent is a raw string prefix; name is colored green; desc is muted
  ],
  dirs: [
    {
      path: '/etc',
      desc: 'System-wide configuration.',
      items: [
        { name: '/etc/fstab', desc: 'Mount table' }
      ]
    }
  ]
}
```

---

## 4. Conventions and Logic

### Naming

- **Section IDs:** kebab-case, prefixed by domain: `sw-vlan`, `rt-ospf`, `fm-find`, `proc-docker`
- **Procedure IDs:** always prefixed `proc-`: `proc-switch-init`, `proc-static-ip-ubuntu`
- **One-liner subsection IDs:** prefixed `ol-`: `ol-files`, `ol-net`
- **File names:** `<topic>-reference-sheet.html`

### Routing

No router. Navigation is plain `<a href="…">` links:
- `index.html` → sheet files
- Each sheet has `<a href="index.html" id="home-link">` in the topbar

### Filtering

**IOS sheet** — `devices` filter: `all | l2 | l3 | router`
**Linux sheet** — `distros` filter: `all | ubuntu | arch | fedora`

Filter logic (`applyFilter()`):
1. Each `<tr class="cmd-row">` carries `data-distros="ubuntu,all"` (or `data-devices`).
2. On filter click: rows whose distro list doesn't include `activeFilter` (and doesn't include `'all'`) get class `hidden` (`display:none`).
3. Empty subsections and empty sections are hidden automatically.
4. `procedure`, `filesystem`, and `oneliners` sections are never affected by the filter.

### Search

**IOS & Linux sheets** — real-time overlay search:
- `initSearch()` builds a flat `searchIndex[]` from all `commands` and `oneliners` subsections (procedures and filesystem sections are excluded).
- Each index entry: `{ cmd, syntax, desc, section, sectionId, sectionType }`.
- On input: filter index by substring match on `cmd | syntax | desc`; show top 30 in dropdown overlay.
- Selecting a result: expands the parent section/subsection, smooth-scrolls to it.
- Keyboard: `/` to focus, `↑↓` to navigate, `Enter` to select, `Esc` to close.

**Homepage** — searches `data-tags`, `.card-title`, `.card-desc` on each `.sheet-card`.

### Persistence (localStorage)

| Key | Scope | Content |
|---|---|---|
| `cisco-ref-theme` | IOS sheet | `'dark'` \| `'light'` |
| `cisco-ref-collapse` | IOS sheet | `{[sectionId]: boolean}` |
| `linux-ref-theme` | Linux sheet | `'dark'` \| `'light'` |
| `linux-ref-collapse` | Linux sheet | `{[sectionId]: boolean}` |
| `netops-theme` | Homepage | `'dark'` \| `'light'` |

### Keyboard shortcuts (both sheets)

`/` focus search · `Esc` close · `T` toggle theme · `E` expand all · `C` collapse all · `B` toggle sidebar · `?` show shortcut modal

---

## 5. Current State

### Implemented and functional

- **Homepage (`index.html`):** Card grid with 2 available sheets + 1 coming-soon (Docker). Live search, dark/light theme, keyboard shortcuts.
- **Cisco IOS sheet (`ios-reference-sheet.html`):** ~300 commands across 4 top-level sections (SWITCHING, ROUTING, SERVICES, PROCEDURES). 15 step-by-step procedures. Device filter (L2/L3/Router/All). Collapsible sections. TOC sidebar with IntersectionObserver active-link tracking. Sidebar toggle. Copy-to-clipboard on every command syntax. Search overlay. Syntax tokenizer. Print stylesheet.
- **Linux sheet (`linux-reference-sheet.html`):** ~400 commands across 11 sections (FILESYSTEM, FILE MANAGEMENT, TEXT PROCESSING, PROCESS MANAGEMENT, NETWORKING, USERS & PERMISSIONS, DISK & STORAGE, SYSTEM INFO, SYSTEMD, PACKAGE MANAGEMENT, ONE-LINERS). 13 procedures covering Ubuntu, Arch, and Fedora. Distro filter. Filesystem tree renderer. One-liner card renderer with copy buttons. All IOS sheet features also present.

### Not yet implemented

- Docker cheat sheet (card shows "Coming Soon")
- Any backend, authentication, or dynamic data source
