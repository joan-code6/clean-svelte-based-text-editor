<script lang="ts">
import { onMount, onDestroy } from 'svelte';
import MarkdownEditor from './components/MarkdownEditor.svelte';
import Settings from './components/Settings.svelte';

let files: { id: string; name: string; content: string }[] = [];
let activeId: string | null = null; // null means Home
let showBar = true;
let theme = localStorage.getItem('ia:theme') || 'light';
// preference: skip in-app confirmations (when true, delete/rename happen immediately)
let skipDialogs = localStorage.getItem('ia:skipDialogs') === 'true';

// modal state
let showModal = false;
let modalType: 'rename' | 'delete' | null = null;
let modalTargetId: string | null = null;
let modalInput = '';
let modalDontAskAgain = false;
let modalTargetExt = '';
// undo state for deletes
let lastDeleted: { file: { id: string; name: string; content: string }; index: number } | null = null;
let showUndo = false;
let undoTimer: any = null;

function saveFiles() {
  localStorage.setItem('ia:files', JSON.stringify(files));
}

function loadFiles() {
  const raw = localStorage.getItem('ia:files');
  if (raw) files = JSON.parse(raw);
  if (!files.length) {
    files = [{ id: crypto.randomUUID(), name: 'Welcome.md', content: '# Welcome\n\nStart typing...' }];
  }
  const last = localStorage.getItem('ia:last');
  if (last && files.find(x => x.id === last)) activeId = last;
  else activeId = files[0]?.id || null;
}

function newNote() {
  const id = crypto.randomUUID();
  const name = makeUniqueName('Untitled.md');
  const content = defaultContentForName(name);
  files.unshift({ id, name, content });
  files = [...files];
  activeId = id;
  saveFiles();
}

function openFile(id: string) {
  activeId = id;
  localStorage.setItem('ia:last', id);
}

function goHome() {
  activeId = null;
  // don't remove files, just clear last-open pointer
  localStorage.removeItem('ia:last');
}

function openRenameModal(id: string) {
  const f = files.find(x => x.id === id);
  if (!f) return;
  modalType = 'rename';
  modalTargetId = id;
  // show name without extension in input
  const m = f.name.match(/^(.*?)(\.[^.]*)?$/);
  modalInput = m ? m[1] : f.name;
  modalTargetExt = m && m[2] ? m[2] : '';
  modalDontAskAgain = false;
  showModal = true;
}

function openDeleteModal(id: string) {
  const f = files.find(x => x.id === id);
  if (!f) return;
  // if user chose to skip dialogs, perform delete immediately
  if (skipDialogs) {
    performDelete(id);
    return;
  }
  modalType = 'delete';
  modalTargetId = id;
  modalInput = files.find(x => x.id === id)?.name || '';
  modalDontAskAgain = false;
  showModal = true;
}

function performRename(id: string, newName: string, dontAskAgain = false) {
  const f = files.find(x => x.id === id);
  if (!f) return;
  if (newName && newName.trim()) {
  f.name = makeUniqueName(newName.trim(), id);
  files = [...files];
  saveFiles();
  }
  if (dontAskAgain) {
    skipDialogs = true;
    localStorage.setItem('ia:skipDialogs', 'true');
  }
}

function performDelete(id: string, dontAskAgain = false) {
  const idx = files.findIndex(x => x.id === id);
  if (idx === -1) return;
  const f = files[idx];
  // remove and keep for undo
  files.splice(idx, 1);
  files = [...files];
  saveFiles();
  lastDeleted = { file: { ...f }, index: idx };
  showUndo = true;
  clearTimeout(undoTimer);
  undoTimer = setTimeout(() => { showUndo = false; lastDeleted = null; undoTimer = null; }, 5000);
  if (activeId === id) {
    activeId = files[0]?.id || null;
    if (activeId) localStorage.setItem('ia:last', activeId);
    else localStorage.removeItem('ia:last');
  }
  if (dontAskAgain) {
    skipDialogs = true;
    localStorage.setItem('ia:skipDialogs', 'true');
  }
}

function undoDelete() {
  if (!lastDeleted) return;
  files.splice(lastDeleted.index, 0, lastDeleted.file);
  files = [...files];
  saveFiles();
  showUndo = false;
  clearTimeout(undoTimer);
  lastDeleted = null;
  undoTimer = null;
}

function makeUniqueName(base: string, skipId?: string) {
  const names = new Set(files.filter(f => f.id !== skipId).map(f => f.name));
  if (!names.has(base)) return base;
  // if base has extension, insert suffix before extension
  const m = base.match(/^(.*?)(\.[^.]*)?$/);
  const stem = m ? m[1] : base;
  const ext = m && m[2] ? m[2] : '';
  let i = 2;
  while (true) {
    const candidate = `${stem}-${i}${ext}`;
    if (!names.has(candidate)) return candidate;
    i++;
  }
}

function stripExtension(name: string) {
  return name.replace(/\.[^/.]+$/, '');
}

function defaultContentForName(name: string) {
  const title = stripExtension(name) || 'New Note';
  return `# ${title}\n\nStart typing...`;
}

function extractHeaderTitle(content: string) {
  if (!content) return '';
  // Find the first non-empty line that starts with one or more # followed by space
  const lines = content.split(/\r?\n/);
  for (const line of lines) {
    const m = line.match(/^\s*(#{1,6})\s+(.*)$/);
    if (m) {
      return m[2].trim();
    }
    // also allow plain text first-line title (no #) if it's a single-line and followed by blank or underline? Keep simple: require #
    if (line.trim() === '') continue;
    // if first non-empty line is not a header, stop searching
    break;
  }
  return '';
}

function exportFile(id: string) {
  const f = files.find(x => x.id === id);
  if (!f) return;
  const blob = new Blob([f.content], { type: 'text/markdown;charset=utf-8' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = f.name || 'note.md';
  document.body.appendChild(a);
  a.click();
  a.remove();
  URL.revokeObjectURL(url);
}

function updateContent(id: string, content: string) {
  const f = files.find(x => x.id === id);
  if (f) f.content = content;
  saveFiles();
}

function toggleTheme() {
  theme = theme === 'light' ? 'dark' : 'light';
  localStorage.setItem('ia:theme', theme);
  document.documentElement.setAttribute('data-theme', theme);
}
let showSettings = false;
let editorFont = localStorage.getItem('ia:font') || "Inter, system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', Arial";
let editorSpellcheck = localStorage.getItem('ia:spellcheck') === 'true';
let editorFontSize = parseInt(localStorage.getItem('ia:fontSize') || '20', 10);
let editingTitle = false;
let editTitle = '';
let focusMode = false;

function onEditorMouseEnter() {
  // when entering editor area, ensure focus mode shows focused editor
  // (if focusMode is toggled on, keep it; otherwise nothing)
}

function onEditorMouseLeave() {
  // no-op: don't auto-exit focus mode on mouse leave; only exit via top hover or leaving fullscreen
}

async function enableFocusMode() {
  try {
    focusMode = true;
    // try to enter browser fullscreen (must be called from user gesture when possible)
    if (!document.fullscreenElement && document.documentElement.requestFullscreen) {
      await document.documentElement.requestFullscreen();
    }
  } catch (err) {
    // ignore failures (browser may block fullscreen if not a user gesture)
  }
}

async function disableFocusMode() {
  try {
    focusMode = false;
    if (document.fullscreenElement && document.exitFullscreen) {
      await document.exitFullscreen();
    }
  } catch (err) {
    // ignore
  }
}

function toggleFocusMode() {
  if (focusMode) disableFocusMode(); else enableFocusMode();
}

$: editorKey = `${editorFont}|${editorFontSize}|${String(editorSpellcheck)}`;

$: activeFile = files.find(f => f.id === activeId) || null;
$: themeKey = theme;

function openSettings() { showSettings = true; }
function applySettings(e: CustomEvent<{ font: string; spellcheck: boolean; lang: string; fontSize?: number }>) {
  const { font, spellcheck, lang, fontSize } = e.detail;
  editorFont = font;
  editorSpellcheck = spellcheck;
  if (fontSize) editorFontSize = fontSize;
  localStorage.setItem('ia:font', font);
  localStorage.setItem('ia:spellcheck', spellcheck ? 'true' : 'false');
  if (fontSize) localStorage.setItem('ia:fontSize', String(fontSize));
  // remount editor and other affected UI by updating editorKey (reactive above)
  // force a microtask so Svelte updates bindings cleanly
  setTimeout(() => { /* noop - reactive editorKey is enough */ }, 0);
}

function startInlineRename() {
  if (!activeFile) return;
  editingTitle = true;
  editTitle = stripExtension(activeFile.name);
  // focus will be handled by bind:this on input
}

function cancelInlineRename() {
  editingTitle = false;
}

function saveInlineRename() {
  if (!activeFile) return;
  const newName = (editTitle || stripExtension(activeFile.name)).trim() + (activeFile.name.match(/(\.[^.]*)$/)?.[1] || '.md');
  performRename(activeFile.id, newName);
  editingTitle = false;
}

onMount(() => {
  loadFiles();
  document.documentElement.setAttribute('data-theme', theme);

  window.addEventListener('keydown', (e) => {
    if ((e.ctrlKey || e.metaKey) && e.key.toLowerCase() === 's') {
      e.preventDefault();
      // simple save (already persisted on change)
      saveFiles();
    }
    if ((e.ctrlKey || e.metaKey) && e.key.toLowerCase() === 'n') {
      e.preventDefault();
      newNote();
    }
    if (e.key === 'Escape') {
      // If in fullscreen, let Escape exit fullscreen (which will sync focus mode off);
      // otherwise reveal taskbar without toggling focus mode.
      if (document.fullscreenElement && document.exitFullscreen) {
        document.exitFullscreen();
      } else {
        showBar = true;
      }
    }
    if (e.key === 'F11') {
      // intercept F11 to toggle in-app focus mode and try to fullscreen the page
      e.preventDefault();
      toggleFocusMode();
    }
  });

  // when focusMode is active, moving the cursor near the top (edge) should reveal the topbar
  const mouseMoveForTopbar = (ev: MouseEvent) => {
    try {
      if (focusMode && ev.clientY <= 80) {
        // exit visual focus mode when user hovers near where the navbar lives
        focusMode = false;
      }
    } catch (err) {}
  };
  window.addEventListener('mousemove', mouseMoveForTopbar);

  // Keep focusMode synced with native fullscreen state
  const onFsChange = () => {
    try {
      if (!document.fullscreenElement && focusMode) {
        // left fullscreen via browser -> end focus mode visuals
        focusMode = false;
      }
    } catch (err) {}
  };
  window.addEventListener('fullscreenchange', onFsChange);

  onDestroy(() => {
    window.removeEventListener('mousemove', mouseMoveForTopbar);
    window.removeEventListener('fullscreenchange', onFsChange);
  });
});
</script>

<style>
:global(body) {
  margin: 0;
  font-family: Inter, system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', Arial;
}
.app {
  height: 100vh;
  display: flex;
  flex-direction: column;
}
.topbar {
  position: fixed;
  top: 8px;
  left: 8px;
  right: 8px;
  height: 44px;
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 0 12px;
  background: var(--topbar-bg);
  backdrop-filter: blur(6px);
  border-radius: 10px;
  box-shadow: 0 6px 20px rgba(0,0,0,0.06);
  z-index: 60;
}
/* .files rule removed (unused) */
.main { flex: 1; display: flex; }
.center { flex: 1; display: flex; align-items: flex-start; justify-content: center; padding-top: 64px; }
.editor-wrap { width: min(900px, 86%); height: calc(100vh - 96px); box-sizing: border-box; }

.tabs { display:flex; gap:6px; align-items:center; }
.tab { background:transparent; border:none; padding:6px 10px; border-radius:6px; color:inherit; }

.tab.add { font-weight:700; }
.card-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
  gap: 12px;
}
.card {
  display:flex;
  flex-direction:column;
  justify-content:space-between;
  background: var(--card-bg, rgba(255,255,255,0.03));
  border-radius: 10px;
  padding: 12px;
  box-shadow: 0 6px 18px rgba(0,0,0,0.04);
  cursor: pointer;
  transition: transform 160ms ease, box-shadow 160ms ease;
}
.card:hover { transform: translateY(-4px); box-shadow: 0 12px 30px rgba(0,0,0,0.06); }
.card-body { padding-bottom:8px; }
.card-title { font-weight:600; margin-bottom:6px; }
.card-excerpt { font-size:13px; color:var(--muted-color, #9aa); max-height:40px; overflow:hidden; text-overflow:ellipsis }
.card-actions { display:flex; gap:6px; justify-content:flex-end; }

.icon-btn { background: transparent; border: none; padding:6px 8px; border-radius:6px; cursor:pointer; display:inline-flex; align-items:center; justify-content:center; width:36px; height:36px; }
.icon-btn:hover { background: var(--icon-hover-bg); }

/* toast */
.undo-toast { position: fixed; left: 20px; bottom: 20px; background: var(--topbar-bg); color: var(--topbar-ink); padding: 10px 12px; border-radius: 8px; display:flex; gap:12px; align-items:center; box-shadow: 0 8px 30px rgba(0,0,0,0.12); z-index:400; animation: toast-in 220ms cubic-bezier(.2,.9,.2,1); }
.undo-toast button { background:transparent; border:1px solid var(--border-muted); padding:6px 8px; border-radius:6px; cursor:pointer; }

@keyframes toast-in {
  from { transform: translateY(8px) scale(0.98); opacity: 0; }
  to { transform: translateY(0) scale(1); opacity: 1; }
}

.modal-overlay {
  position:fixed; inset:0; background:rgba(0,0,0,0.35); display:flex; align-items:center; justify-content:center; z-index:200;
}
.modal { background:var(--topbar-bg); padding:18px; border-radius:10px; width:min(560px,92%); box-shadow:0 20px 60px rgba(0,0,0,0.4); }
/* inline title input in topbar (themed) */
.title-input {
  font-weight: 600;
  max-width: 60%;
  text-align: center;
  padding: 6px 8px;
  border-radius: 6px;
  border: 1px solid var(--border-muted);
  background: var(--editor-bg);
  color: var(--text-color);
  box-sizing: border-box;
}
.title-input::placeholder { color: var(--muted-color); }
</style>

<div class="app" class:focus-mode={focusMode}>
  <div class="topbar {showBar ? 'show' : ''}" role="toolbar" tabindex="0" on:mouseenter={() => showBar = true} on:mouseleave={() => showBar = false}>
    <div style="display:flex;align-items:center;gap:12px">
      <div class="tabs">
        <button class="tab" on:click={goHome} aria-label="Home">Home</button>
        <button class="tab add" on:click={newNote}>＋</button>
      </div>
    </div>
    <div style="flex:1;display:flex;align-items:center;justify-content:center;">
      {#if activeFile}
        {#if !editingTitle}
          <div role="button" tabindex="0" on:click={startInlineRename} on:keydown={(e) => { if (e.key === 'Enter') startInlineRename(); }} style="font-weight:600;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;max-width:60%;cursor:text">{activeFile.name.replace(/\.[^/.]+$/, '')}</div>
          {:else}
          <input class="title-input" bind:value={editTitle} on:blur={saveInlineRename} on:keydown={(e) => { if (e.key === 'Enter') saveInlineRename(); if (e.key === 'Escape') cancelInlineRename(); }} />
        {/if}
      {/if}
    </div>
    <div style="display:flex;align-items:center;gap:12px">
      <button class="icon-btn" on:click={openSettings} aria-label="Settings">
        <!-- gear -->
        <svg width="18" height="18" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg"><path d="M12 15.5A3.5 3.5 0 1112 8.5a3.5 3.5 0 010 7zM19.4 13a7.6 7.6 0 000-2l2.1-1.6-2-3.4-2.5.7a7.7 7.7 0 00-1.7-1L14 2h-4l-.3 3.6a7.7 7.7 0 00-1.7 1L5.5 6.9l-2 3.4L5.7 12a7.6 7.6 0 000 2l-2.1 1.6 2 3.4 2.5-.7c.5.4 1 .7 1.7 1L10 22h4l.3-3.6c.6-.3 1.2-.7 1.7-1l2.5.7 2-3.4L19.4 13z" stroke="currentColor" stroke-width="0" fill="currentColor"/></svg>
      </button>
      <!-- focus mode toggle -->
  <button class="icon-btn" on:click={() => toggleFocusMode()} aria-label="Toggle focus mode" title="Toggle focus mode (F11)">
        <!-- simple focus icon -->
        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg"><path d="M3 7v-2a2 2 0 0 1 2-2h2" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"/><path d="M21 17v2a2 2 0 0 1-2 2h-2" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"/><path d="M7 21H5a2 2 0 0 1-2-2v-2" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"/><path d="M17 3h2a2 2 0 0 1 2 2v2" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"/></svg>
      </button>
      <button class="icon-btn" on:click={toggleTheme} aria-label="Toggle theme">
        {#if theme === 'light'}
          <!-- sun -->
          <svg viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg"><g id="SVGRepo_bgCarrier" stroke-width="0"></g><g id="SVGRepo_tracerCarrier" stroke-linecap="round" stroke-linejoin="round"></g><g id="SVGRepo_iconCarrier"> <path d="M17 12C17 14.7614 14.7614 17 12 17C9.23858 17 7 14.7614 7 12C7 9.23858 9.23858 7 12 7C14.7614 7 17 9.23858 17 12Z" fill="#111827"></path> <path fill-rule="evenodd" clip-rule="evenodd" d="M12 1.25C12.4142 1.25 12.75 1.58579 12.75 2V4C12.75 4.41421 12.4142 4.75 12 4.75C11.5858 4.75 11.25 4.41421 11.25 4V2C11.25 1.58579 11.5858 1.25 12 1.25ZM3.66865 3.71609C3.94815 3.41039 4.42255 3.38915 4.72825 3.66865L6.95026 5.70024C7.25596 5.97974 7.2772 6.45413 6.9977 6.75983C6.7182 7.06553 6.2438 7.08677 5.9381 6.80727L3.71609 4.77569C3.41039 4.49619 3.38915 4.02179 3.66865 3.71609ZM20.3314 3.71609C20.6109 4.02179 20.5896 4.49619 20.2839 4.77569L18.0619 6.80727C17.7562 7.08677 17.2818 7.06553 17.0023 6.75983C16.7228 6.45413 16.744 5.97974 17.0497 5.70024L19.2718 3.66865C19.5775 3.38915 20.0518 3.41039 20.3314 3.71609ZM1.25 12C1.25 11.5858 1.58579 11.25 2 11.25H4C4.41421 11.25 4.75 11.5858 4.75 12C4.75 12.4142 4.41421 12.75 4 12.75H2C1.58579 12.75 1.25 12.4142 1.25 12ZM19.25 12C19.25 11.5858 19.5858 11.25 20 11.25H22C22.4142 11.25 22.75 11.5858 22.75 12C22.75 12.4142 22.4142 12.75 22 12.75H20C19.5858 12.75 19.25 12.4142 19.25 12ZM17.0255 17.0252C17.3184 16.7323 17.7933 16.7323 18.0862 17.0252L20.3082 19.2475C20.6011 19.5404 20.601 20.0153 20.3081 20.3082C20.0152 20.6011 19.5403 20.601 19.2475 20.3081L17.0255 18.0858C16.7326 17.7929 16.7326 17.3181 17.0255 17.0252ZM6.97467 17.0253C7.26756 17.3182 7.26756 17.7931 6.97467 18.086L4.75244 20.3082C4.45955 20.6011 3.98468 20.6011 3.69178 20.3082C3.39889 20.0153 3.39889 19.5404 3.69178 19.2476L5.91401 17.0253C6.2069 16.7324 6.68177 16.7324 6.97467 17.0253ZM12 19.25C12.4142 19.25 12.75 19.5858 12.75 20V22C12.75 22.4142 12.4142 22.75 12 22.75C11.5858 22.75 11.25 22.4142 11.25 22V20C11.25 19.5858 11.5858 19.25 12 19.25Z" fill="#111827"></path> </g></svg>
        {:else}
          <!-- moon -->
          <svg viewBox="-2.4 -2.4 28.80 28.80" fill="none" xmlns="http://www.w3.org/2000/svg" stroke="#E6EEF8"><g id="SVGRepo_bgCarrier" stroke-width="0"></g><g id="SVGRepo_tracerCarrier" stroke-linecap="round" stroke-linejoin="round" stroke="#CCCCCC" stroke-width="0.048"></g><g id="SVGRepo_iconCarrier"> <path d="M12 22C17.5228 22 22 17.5228 22 12C22 11.5373 21.3065 11.4608 21.0672 11.8568C19.9289 13.7406 17.8615 15 15.5 15C11.9101 15 9 12.0899 9 8.5C9 6.13845 10.2594 4.07105 12.1432 2.93276C12.5392 2.69347 12.4627 2 12 2C6.47715 2 2 6.47715 2 12C2 17.5228 6.47715 22 12 22Z" fill="#E6EEF8"></path> </g></svg>  
        {/if}
      </button>
      <div style="opacity:0.7;font-size:12px;padding-left:12px">Made by Redstonekey</div>
    </div>
  </div>

  <div class="main">
    <div class="center">
  <div class="editor-wrap" role="region" aria-label="Editor area" on:mouseenter={onEditorMouseEnter} on:mouseleave={onEditorMouseLeave}>
        {#if activeId === null}
          <!-- Home screen: card grid -->
          <div style="padding:28px;">
            <div style="display:flex;align-items:center;justify-content:space-between;margin-bottom:12px;">
              <h2 style="margin:0">Files</h2>
              <div>
                <button class="tab add" on:click={newNote}>＋ New note</button>
              </div>
            </div>
            <div class="card-grid">
              {#each files as f}
                <div class="card" role="button" tabindex="0" on:click={() => openFile(f.id)} on:keydown={(e) => { if (e.key === 'Enter' || e.key === ' ') { openFile(f.id); e.preventDefault(); } }}>
                  <div class="card-body">
                    <div class="card-title">{f.name.replace(/\.[^/.]+$/, '')}</div>
                    <div class="card-excerpt">{f.content.split('\n')[0] || 'Empty'}</div>
                  </div>
                  <div class="card-actions">
                    <button title="Rename" on:click|stopPropagation={() => openRenameModal(f.id)} aria-label="Rename" class="icon-btn">
                      <svg width="16" height="16" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg"><path d="M3 17.25V21h3.75L17.81 9.94l-3.75-3.75L3 17.25zM20.71 7.04a1 1 0 0 0 0-1.41l-2.34-2.34a1 1 0 0 0-1.41 0l-1.83 1.83 3.75 3.75 1.83-1.83z" stroke="currentColor" stroke-width="0" fill="currentColor"/></svg>
                    </button>
                    <button title="Delete" on:click|stopPropagation={() => openDeleteModal(f.id)} aria-label="Delete" class="icon-btn">
                      <svg width="16" height="16" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg"><path d="M3 6h18M8 6v12a2 2 0 0 0 2 2h4a2 2 0 0 0 2-2V6M10 6V4a2 2 0 0 1 2-2h0a2 2 0 0 1 2 2v2" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></svg>
                    </button>
                    <button title="Export" on:click|stopPropagation={() => exportFile(f.id)} aria-label="Export" class="icon-btn">
                      <svg width="16" height="16" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg"><path d="M12 3v12M8 7l4-4 4 4M21 21H3" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></svg>
                    </button>
                  </div>
                </div>
              {/each}
            </div>
            <!-- skip-dialogs preference is only shown inside modal now -->
          </div>
        {:else}
          {#each files.filter(x => x.id === activeId) as active}
            {#key active.id}
              {#key editorKey}
                <MarkdownEditor value={active.content} on:change={(e) => updateContent(active.id, e.detail)} on:save={saveFiles} on:new={newNote} on:escape={() => showBar = true} font={editorFont} spellcheck={editorSpellcheck} fontSize={editorFontSize} />
              {/key}
            {/key}
          {/each}
        {/if}
      </div>
    </div>
  </div>

  {#if showModal}
    {#key themeKey}
      <div class="modal-overlay" role="button" tabindex="0" on:click={(e) => { if (e.target === e.currentTarget) { showModal = false; modalType = null; modalTargetId = null; } }} on:keydown={(e) => { if (e.key === 'Enter' || e.key === 'Escape') { showModal = false; modalType = null; modalTargetId = null; } }}>
        <div class="modal" role="dialog" tabindex="-1">
        {#if modalType === 'rename'}
          <h3>Rename file</h3>
          <input bind:value={modalInput} />
          <div style="margin-top:12px;display:flex;gap:8px;justify-content:flex-end;">
            <button on:click={() => { showModal = false; modalType = null; modalTargetId = null; modalTargetExt = ''; }}>Cancel</button>
            <button on:click={() => { if (modalTargetId) performRename(modalTargetId, modalInput + modalTargetExt); showModal = false; modalType = null; modalTargetId = null; modalTargetExt = ''; }}>Save</button>
          </div>
        {:else if modalType === 'delete'}
          <h3>Delete file</h3>
          <p>Delete "{modalInput}"? This cannot be undone.</p>
          <div style="margin-top:8px;display:flex;align-items:center;gap:8px;">
            <label style="font-size:13px;display:flex;gap:6px;align-items:center;"><input type="checkbox" bind:checked={modalDontAskAgain} /> Always skip dialogs</label>
          </div>
          <div style="margin-top:12px;display:flex;gap:8px;justify-content:flex-end;">
            <button on:click={() => { showModal = false; modalType = null; modalTargetId = null; }}>Cancel</button>
            <button on:click={() => { if (modalTargetId) performDelete(modalTargetId, modalDontAskAgain); showModal = false; modalType = null; modalTargetId = null; }}>Delete</button>
          </div>
        {/if}
        </div>
      </div>
    {/key}
  {/if}

  {#if showSettings}
    {#key themeKey}
      <div class="modal-overlay" role="button" tabindex="0" on:click={(e) => { if (e.target === e.currentTarget) showSettings = false; }} on:keydown={(e) => { if (e.key === 'Enter' || e.key === 'Escape') showSettings = false; }}>
        <div class="modal" role="dialog" tabindex="-1">
          <Settings on:update={applySettings} on:close={() => showSettings = false} />
        </div>
      </div>
    {/key}
  {/if}
  {#if showUndo}
    <div class="undo-toast" style="border: 2px solid #e53935;">
      <div>File deleted</div>
      <button style="background: #e53935; color: white; border: none;" on:click={undoDelete}>Undo</button>
    </div>
  {/if}
</div>
