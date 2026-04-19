# md-reader — Architecture Plan

A simple, lightweight macOS markdown reader/editor with live preview.

---

## 1. Tech Stack Decision

**Choice: SwiftUI + Swift Package Manager + WKWebView**

### Reasoning

The requirements pull in four directions: native feel, lightweight, live preview, minimal deps, one-command launch. SwiftUI hits all five cleanly:

| Requirement | How SwiftUI delivers |
|---|---|
| macOS native feel | Real AppKit under the hood — system fonts, traffic-light window chrome, native file dialogs, dark mode, keyboard shortcuts all free |
| Lightweight | Single binary, ~2–5 MB, no runtime to install (Swift stdlib ships with macOS 13+) |
| Live preview | Two-pane `HSplitView` with `TextEditor` + `WKWebView` — textbinding publishes changes, WebView re-renders on every keystroke |
| Minimal external deps | **Zero** Swift packages required. Markdown→HTML done via a single bundled `marked.min.js` (~30 KB, MIT) loaded into the WebView. No Homebrew, no pip, no npm |
| Simple launch command | `swift run` from source, or build once and drop a `md-reader` shim in `~/bin` — opens files via `md-reader path/to/file.md` |

### Alternatives considered and rejected

- **Electron** — violates "lightweight" hard. 150 MB+ per app, ships a whole Chromium.
- **Tauri (Rust + WebView)** — lightweight and cross-platform, but requires Rust toolchain and the native feel is weaker (webview-only, no AppKit integration). Overkill for macOS-only.
- **Python + pywebview** — fast to build, but requires Python runtime, pip deps, and feels non-native (generic webview window, no AppKit polish). Fails "macOS native feel".
- **Pure WKWebView wrapped in minimal AppKit** — equivalent weight to SwiftUI but more boilerplate. SwiftUI wins on maintainability.
- **Apple `AttributedString(markdown:)`** — zero-dep markdown but no live HTML rendering, no GFM tables, no code blocks with syntax. Too limited.

### Trade-offs accepted

- macOS 13+ only. Fine — it's a personal/modern-Mac tool.
- Requires Xcode command-line tools (`xcode-select --install`) to build. One-time cost.
- Bundled `marked.min.js` is a dep in spirit — but it's a single committed file, offline, no package manager involvement. Acceptable.

---

## 2. File / Component Breakdown

```
md-reader/
├── Package.swift                    # SPM manifest — executable target, macOS 13+
├── README.md                        # How to build/run
├── PLAN.md                          # This file
├── Sources/
│   └── MdReader/
│       ├── MdReaderApp.swift        # @main — App scene, window config, menu commands
│       ├── ContentView.swift        # HSplitView: editor left, preview right
│       ├── EditorView.swift         # SwiftUI TextEditor with monospace font + binding
│       ├── PreviewView.swift        # NSViewRepresentable wrapping WKWebView
│       ├── MarkdownRenderer.swift   # Injects markdown into WebView's JS context
│       ├── DocumentModel.swift      # ObservableObject — text, fileURL, isDirty
│       ├── FileCommands.swift       # Open/Save/SaveAs via NSOpenPanel/NSSavePanel
│       └── Resources/
│           ├── marked.min.js        # Bundled markdown parser (MIT, vendored)
│           ├── preview.html         # Scaffold HTML loaded once; JS patches innerHTML
│           └── preview.css          # GitHub-ish stylesheet, respects prefers-color-scheme
└── bin/
    └── md-reader                    # Tiny shell shim: exec the built binary with $@
```

### Component responsibilities

- **`MdReaderApp`** — defines the `WindowGroup`, sets min size (800×500), wires `CommandGroup` replacements for File menu (⌘O, ⌘S, ⌘⇧S) and "New" (⌘N).
- **`DocumentModel`** — `@Published var text: String`, `@Published var fileURL: URL?`, `@Published var isDirty: Bool`. Single source of truth.
- **`ContentView`** — owns the `DocumentModel` as `@StateObject`, lays out editor and preview side-by-side with a draggable divider.
- **`EditorView`** — `TextEditor($document.text)` with `.font(.system(.body, design: .monospaced))` and line padding. Sets `isDirty = true` on change.
- **`PreviewView`** — `NSViewRepresentable` wrapping a `WKWebView`. On `makeNSView`, loads `preview.html` from bundle. On `updateNSView`, calls `webView.evaluateJavaScript("render(\(escaped))")` with the current markdown text. Debounced ~50 ms to avoid jitter during fast typing.
- **`preview.html`** — `<div id="content"></div>` + `<script src="marked.min.js">` + inline `function render(md) { document.getElementById('content').innerHTML = marked.parse(md); }`.
- **`FileCommands`** — `openDocument()` shows `NSOpenPanel` filtered to `.md/.markdown/.txt`; `saveDocument()` writes UTF-8 atomically, falls back to Save As when no `fileURL`.

### Data flow

```
user types in TextEditor
        ↓
DocumentModel.text updated (SwiftUI binding)
        ↓
ContentView re-renders → PreviewView.updateNSView called
        ↓
evaluateJavaScript("render(<escaped markdown>)")
        ↓
marked.js parses → innerHTML updated → WebView repaints
```

---

## 3. Key Design Decisions and Trade-offs

1. **WebView for preview, not native `AttributedString`.**
   WKWebView + marked.js gives real GFM: tables, fenced code, task lists, autolinks. Cost: ~30 KB JS file in bundle. Worth it — users expect GitHub-style rendering.

2. **Debounce render at 50 ms, not per-keystroke.**
   Typing latency matters more than preview latency. 50 ms feels instant but collapses burst updates. Implemented with a `DispatchWorkItem` cancel/reschedule.

3. **One window = one document.**
   No tabs, no document browser. Matches TextEdit's model and the "simple" requirement. Multiple files = multiple windows (`⌘N` + `⌘O` in new window).

4. **No autosave in v1.** Dirty indicator in title bar (`● filename.md`), standard Cocoa unsaved-changes sheet on close. Autosave is an easy v2 addition via `NSDocument` but adds complexity now.

5. **No custom markdown extensions in v1.** Math, mermaid, wiki-links deferred. Keeps the bundled JS small and the scope tight.

6. **Preview CSS respects system appearance.**
   `@media (prefers-color-scheme: dark)` inside `preview.css` — automatic dark mode without any Swift code.

7. **Security: WebView isolated.** `WKWebView` loads from bundle `file://` with JavaScript enabled but navigation delegate blocks external URL loads (opens them in default browser via `NSWorkspace.shared.open`). Prevents a markdown link from hijacking the preview pane.

8. **Text encoding: UTF-8 only.** Simplifies file I/O. 99.9% of markdown in the wild is UTF-8.

---

## 4. How to Run / Launch

### First-time build

```bash
cd /Users/ych/workspace/md-reader
swift build -c release
ln -sf "$PWD/.build/release/MdReader" /usr/local/bin/md-reader
```

### Daily use

```bash
md-reader                      # opens empty window
md-reader README.md            # opens file in new window
md-reader ~/notes/*.md         # opens each file in its own window
```

### Dev loop

```bash
swift run                      # build + run from source
```

### Packaging (optional, future)

```bash
swift build -c release
# Wrap binary into MdReader.app bundle via a small Makefile target
# Drag into /Applications → Spotlight-launchable, Dock icon, file-association for .md
```

---

## 5. Scope Summary

**v1 (this plan):** open, edit, live preview, save, save-as, new window. Dark mode follows system. That's it.

**Explicitly out of scope:** syntax highlighting in editor, outline sidebar, export to PDF/HTML, custom themes, plugin system, file browser, autosave, iCloud sync.

The guiding principle: **every feature deferred is a feature that can't break on ship day.**
