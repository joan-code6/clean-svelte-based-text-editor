<script lang="ts">
import { createEventDispatcher, onMount, tick } from 'svelte';
const dispatch = createEventDispatcher();
export let active: { id: string; name: string; content: string };
let text = active.content;
let textarea: HTMLTextAreaElement;

$: if (text !== undefined) dispatch('change', text);

// Image handling functions
function saveImageLocally(file: File): Promise<string> {
  return new Promise((resolve) => {
    const reader = new FileReader();
    reader.onload = (e) => {
      const imageData = e.target?.result as string;
      const imageId = crypto.randomUUID();
      const imageKey = `ia:image:${imageId}`;
      
      // Store image data in localStorage
      localStorage.setItem(imageKey, imageData);
      
      // Store image metadata
      const metadata = {
        id: imageId,
        name: file.name,
        type: file.type,
        size: file.size,
        timestamp: Date.now()
      };
      
      const existingImages = JSON.parse(localStorage.getItem('ia:images') || '[]');
      existingImages.push(metadata);
      localStorage.setItem('ia:images', JSON.stringify(existingImages));
      
      resolve(imageData);
    };
    reader.readAsDataURL(file);
  });
}

function handlePaste(event: ClipboardEvent) {
  const clipboardData = event.clipboardData;
  if (!clipboardData) return;
  
  // Check if clipboard contains image files
  const items = Array.from(clipboardData.items);
  const imageItem = items.find(item => item.type.startsWith('image/'));
  
  if (imageItem) {
    event.preventDefault();
    const file = imageItem.getAsFile();
    if (file) {
      saveImageLocally(file).then(imagePath => {
        const imageMarkdown = `![${file.name || 'Pasted Image'}](${imagePath})`;
        insertAtCursor(imageMarkdown);
      }).catch(err => {
        console.error('Failed to save image:', err);
        // Fallback: insert placeholder markdown
        insertAtCursor('![Failed to save image](path/to/image.png)');
      });
    }
  }
}

function onInput() {
  text = textarea.value;
}

function tokenize(s: string) {
  // escape HTML
  const esc = (t: string) => t.split('&').join('&amp;').split('<').join('&lt;').split('>').join('&gt;');
  return s.split('\n').map(line => {
    // heading: keep '#' visible but style the text after it
    const h = line.match(/^(\s*)(#\s)(.*)$/);
    if (h) {
      const [, pre, marker, rest] = h;
      return `${esc(pre)}<span class="heading"><span class="marker">${esc(marker)}</span><span class="heading-text">${esc(rest)}</span></span>`;
    }
    // list item: keep '- ' visible but style content
    const li = line.match(/^(\s*)(-\s)(.*)$/);
    if (li) {
      const [, pre, marker, rest] = li;
      return `${esc(pre)}<span class="list"><span class="marker">${esc(marker)}</span><span class="list-text">${esc(rest)}</span></span>`;
    }

    // image: render with visible markdown syntax
    const img = line.match(/^(\s*)!\[([^\]]*)\]\(([^)]+)\)(.*)$/);
    if (img) {
      const [, pre, alt, src, rest] = img;
      return `${esc(pre)}<span class="image"><span class="marker">![</span><span class="alt-text">${esc(alt)}</span><span class="marker">](</span><span class="image-src">${esc(src)}</span><span class="marker">)</span></span>${esc(rest)}`;
    }

    // bold markers: keep ** visible but style inner text
    const withBold = esc(line).replace(/\*\*(.+?)\*\*/g, '<span class="bold"><span class="marker">**</span><span class="bold-text">$1</span><span class="marker">**</span></span>');
    return withBold || '&nbsp;';
  }).join('\n');
}

onMount(() => {
  // sync when active changes
  text = active.content;
});

// keyboard helpers
function setSelection(pos: number) {
  // wait for DOM update
  tick().then(() => {
    if (textarea) {
      textarea.selectionStart = textarea.selectionEnd = pos;
      textarea.focus();
    }
  });
}

function insertAtCursor(insertText: string, moveCursorBack = 0) {
  const start = textarea.selectionStart;
  const end = textarea.selectionEnd;
  const before = text.slice(0, start);
  const after = text.slice(end);
  text = before + insertText + after;
  const newPos = start + insertText.length - moveCursorBack;
  dispatch('change', text);
  setSelection(newPos);
}

function handleKeydown(e: KeyboardEvent) {
  if (!textarea) return;
  const pairMap: Record<string,string> = { '(': ')', '[': ']', '{': '}', '"': '"', "'": "'" };
  const k = e.key;
  // bracket auto-pair
  if (k in pairMap && !e.ctrlKey && !e.metaKey && !e.altKey) {
    e.preventDefault();
    const closing = pairMap[k];
    insertAtCursor(k + closing, 1);
    return;
  }

  // list auto-continuation
  if (k === 'Enter' && !e.shiftKey) {
    const start = textarea.selectionStart;
    const lineStart = text.lastIndexOf('\n', start - 1) + 1;
    const line = text.slice(lineStart, start);
    const m = line.match(/^(\s*)(-\s)/);
    if (m) {
      e.preventDefault();
      const leading = m[1] || '';
      insertAtCursor('\n' + leading + '- ');
    }
  }
}

</script>

<style>
.editor {
  position: relative;
  min-height: 300px;
}
.highlighter {
  position: absolute;
  inset: 0;
  padding: 28px;
  white-space: pre-wrap;
  word-wrap: break-word;
  color: inherit;
  pointer-events: none;
  z-index: 1; /* placed under textarea */
}
/* selection color for visible highlighted layer */
.highlighter ::selection, .textarea ::selection { background: var(--selection-bg); }
:global(.highlighter .marker) { color: rgba(100,100,100,0.8); }
:global(.highlighter .heading-text) { color: var(--heading-color, #b83280); }
:global(.highlighter .list-text) { color: var(--list-color, #6b7280); }
:global(.highlighter .bold-text) { color: var(--bold-color, #111827); }
:global(.highlighter .image .alt-text) { color: var(--image-alt-color, #059669); }
:global(.highlighter .image .image-src) { color: var(--image-src-color, #0284c7); }
.textarea {
  position: relative;
  width: 100%;
  min-height: 300px;
  padding: 28px;
  background: transparent;
  border: none;
  outline: none;
  resize: vertical;
  font-size: 18px;
  line-height: 1.6;
  /* hide textarea text so highlighted layer shows, keep caret visible */
  color: transparent;
  caret-color: currentColor;
  /* ensure selection background still visible */
  background-clip: padding-box;
  z-index: 2;
  /* make sure font and spacing match the highlighter */
  font-family: inherit;
}
.container {
  background: var(--editor-bg, #fff);
  border-radius: 8px;
}

/* remove any dotted/focus outline around the editor container */
.container, .editor, .textarea, .highlighter {
  outline: none !important;
  border: none !important;
}

.container:focus-within {
  outline: none !important;
}

/* aggressively clear any UA/browser focus rings or borders inside the editor */
.container *, .container *::before, .container *::after {
  outline: none !important;
  box-shadow: none !important;
  border: none !important;
}

.textarea:focus, .container:focus, .container:focus-within, .highlighter:focus {
  outline: none !important;
  box-shadow: none !important;
  border: none !important;
}

/* ensure no decorative background/border-image or dotted patterns remain */
.container, .highlighter, .editor, .textarea {
  background-image: none !important;
  -webkit-background-clip: padding-box !important;
  border-image: none !important;
  border-style: none !important;
  outline-color: transparent !important;
}

.highlighter { background: transparent !important; }
</style>

<div class="container editor">
  <pre class="highlighter" aria-hidden="true" on:mousedown|preventDefault>{@html tokenize(text)}</pre>
  <textarea class="textarea" bind:this={textarea} bind:value={text} on:input={onInput} on:keydown={handleKeydown} on:paste={handlePaste}></textarea>
</div>
 
