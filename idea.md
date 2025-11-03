# Ideen für den Editor (Free)

- Fokus: ultrasaubere, ablenkungsfreie Oberfläche
  - Text mittig, breite Ränder (Lesekomfort)
  - Taskbar minimal, blendet sich aus und erscheint beim Hovern (oben)
- Markup (sichtbare Trigger):
  - `**fett**` bleibt mit Sternchen sichtbar
  - `# Überschrift` zeigt das `#` weiterhin
  - `- Liste` zeigt das `-` weiterhin
- direkt in dem input markup rendern
- Dark-/Lightmode Umschaltung in Taskbar
- Startbildschirm: Liste aller Dateien im Projektordner (Home)
  - Klick öffnet Datei im Editor

- Tastaturfokus:
  - Strg+S (oder Cmd+S) speichert
  - Strg+N neue Notiz
  - Esc blendet Taskbar sofort ein
- Animationen dezent (max 200–300ms)
- Keine native App-Menüleiste (File/Edit/View) – alles in der Taskbar
- Persistenz: letzter Zustand (Light/Dark, letzte Datei) merken


## Later:
- Export
- Firebase acc / file sync
- Text reader (Better Readability)
- multi language support (German and English)
- Rechtschreibung: einfache Hervorhebung von Fehlern (später)

## Preferences:
- never use emojis always use icons!
- after adding a feature or fixing a bug ask me if i like it and if the task is done. If i say yes move it here from to do / bugs to done

## Done

- Svelte + Vite prototype scaffold (TypeScript + Tailwind-ready). 
- Single-surface editor using CodeMirror 6 (accurate caret/selection, markdown language).
- Visible markup triggers preserved (editor shows raw markers like `**`, `#`, `-`).
- Minimal top taskbar with tabs, theme toggle, and keyboard shortcuts wired.
- Keyboard shortcuts: Ctrl/Cmd+S (save), Ctrl/Cmd+N (new note), Esc (reveal taskbar).
- Persistent state in `localStorage`: files (`ia:files`), last-open file (`ia:last`), theme (`ia:theme`).
- Dark / Light mode implemented and wired to CSS variables; topbar/theme now apply globally.
- Tabs/topbar icons (icon button for theme) and accessible buttons for file actions.
- Replaced left sidebar with centered editor layout; editor starts below topbar.
- Bracket auto-close and list-continuation on Enter implemented in editor keymap.
- Basic styling and layout with CSS variables for theming and CodeMirror theme integration.
- Scrollbar theming added and an optional `.hide-scrollbar` utility.
- Light icon replaced with clearer theme SVG (theme icon updated).
- Image support added: Both markdown syntax `![alt](src)` and Ctrl+V paste functionality with local storage.
- File action buttons (rename/delete/export) moved to Home cards and hidden in file view.
- Topbar no longer shows file tabs; only a Home button remains on the topbar.
- Rename flow hides the `.md` extension in the input and the "always skip" prompt option is handled in-app.
- Selection color for dark mode updated for better visibility.

- Settings screen implemented (in-app modal, font and spellcheck prefs).
- Editor font preference added and applied; basic font presets available.
- Filename display above the editor (top area) when a file is open.
- Spellcheck toggle persisted and applied to the editor container (browser spellcheck enabled when selected).

## To-Do

- Closing headers (#) when clicking on them.
- Spellcheck integration (highlight misspellings; toggle in settings).
- Export functionality (export note to file/markdown/plaintext).
- Real filesystem persistence and OS integration (Tauri) for native app packaging.
- Fine-grained CodeMirror token theming per theme (further polishing of highlight colors).
- Multi-language support (German/English) and localization strings.
- Firebase or cloud sync / account support (optional, later).
- add AI features.
- Accessibility and additional keyboard navigation improvements.

## Bugs

- fix darkmode selection color (when you select text in darkmode it is way to bright and you cant see the text that is selected and white)