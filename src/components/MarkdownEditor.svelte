<script lang="ts">
import { onMount, onDestroy, createEventDispatcher } from 'svelte';
import { EditorState, Extension } from '@codemirror/state';
import { EditorView, keymap, highlightSpecialChars, drawSelection } from '@codemirror/view';
import { defaultKeymap, history, historyKeymap } from '@codemirror/commands';
import { markdown, markdownLanguage } from '@codemirror/lang-markdown';
import { syntaxHighlighting, HighlightStyle, bracketMatching } from '@codemirror/language';
import { tags as t } from '@lezer/highlight';
import { closeBrackets, closeBracketsKeymap } from '@codemirror/autocomplete';
import { StateEffect, StateField } from '@codemirror/state';
import { Decoration, DecorationSet, WidgetType } from '@codemirror/view';
import { RangeSet } from '@codemirror/state';

export let value: string = '';
export let readOnly = false;
export let font: string = localStorage.getItem('ia:font') || "Inter, system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', Arial";
export let spellcheck = localStorage.getItem('ia:spellcheck') === 'true';
export let fontSize: number = parseInt(localStorage.getItem('ia:fontSize') || '20', 10);

const dispatch = createEventDispatcher();
let container: HTMLDivElement;
let view: EditorView;

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
      
      resolve(`data:${file.type};base64,${imageData.split(',')[1]}`);
    };
    reader.readAsDataURL(file);
  });
}

function insertImageMarkdown(imagePath: string, altText: string = 'Pasted Image'): void {
  if (!view) return;
  
  const imageMarkdown = `![${altText}](${imagePath})`;
  const state = view.state;
  const cursor = state.selection.main.head;
  
  // Insert image markdown at cursor position
  view.dispatch({
    changes: {
      from: cursor,
      insert: imageMarkdown
    },
    selection: {
      anchor: cursor + imageMarkdown.length
    }
  });
}

// custom highlight style that keeps punctuation (markers) visible but styles text
const highlightStyle = HighlightStyle.define([
  { tag: t.heading, color: '#b83280', fontWeight: '600' },
  { tag: t.heading1, fontSize: '2.3rem', fontWeight: '600', color: '#b83280' },
  { tag: t.heading2, fontSize: '1.9rem', fontWeight: '600', color: '#b83280' },
  { tag: t.heading3, fontSize: '1.55rem', fontWeight: '600', color: '#b83280' },
  { tag: t.strong, fontWeight: '600' },
  { tag: t.emphasis, fontStyle: 'italic' }
  ,{ tag: t.punctuation, color: 'var(--muted-color, #9aa)' }
  ,{ tag: t.meta, color: 'var(--muted-color, #9aa)' }
]);

const minimalistTheme = EditorView.theme({
  '&': {
    height: '100%',
    fontFamily: "Inter, system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', Arial",
    background: 'var(--bg-color)'
  },
  '.cm-editor': { outline: 'none !important', background: 'transparent', boxShadow: 'none' },
  '.cm-scroller': { lineHeight: '1.6', padding: '32px 0 120px', outline: 'none !important', boxShadow: 'none' },
  '.cm-content': { maxWidth: '820px', margin: '0 auto', padding: '0 56px', color: 'var(--text-color)' },
  '.cm-line': { padding: '0 0' },
  '.cm-activeLine': { backgroundColor: 'transparent' },
  '.cm-gutters': { display: 'none' },
  '.cm-selectionBackground, &.cm-focused .cm-selectionBackground': { backgroundColor: 'rgba(100,150,250,0.30)' },
  '.cm-cursor': { borderLeft: '1px solid currentColor' },
  '.cm-listMark': { color: 'var(--list-color)' }
}, { dark: false });

// Folding: click a markdown heading to collapse the content below it.
const toggleFold = StateEffect.define<{ from: number; to: number }>();

class FoldWidget extends WidgetType {
  count: number;
  constructor(count: number) { super(); this.count = count; }
  toDOM() {
    const wrap = document.createElement('span');
    wrap.className = 'cm-fold-widget';
    wrap.textContent = `⯈ ${this.count} lines`;
    wrap.style.opacity = '0.8';
    wrap.style.padding = '2px 8px';
    wrap.style.borderRadius = '6px';
    wrap.style.background = 'var(--muted-overlay)';
    wrap.style.color = 'var(--muted-color, #9aa)';
    wrap.style.cursor = 'pointer';
    return wrap;
  }
  ignoreEvent() { return false; }
}

class ImageWidget extends WidgetType {
  src: string;
  alt: string;
  constructor(src: string, alt: string) { 
    super(); 
    this.src = src; 
    this.alt = alt;
  }
  toDOM() {
    const wrap = document.createElement('div');
    wrap.className = 'cm-image-widget';
    wrap.style.margin = '12px 0';
    wrap.style.padding = '0';
    wrap.style.display = 'block';
    wrap.style.textAlign = 'center';
    
    const img = document.createElement('img');
    img.src = this.src;
    img.alt = this.alt;
    img.style.maxWidth = '100%';
    img.style.height = 'auto';
    img.style.display = 'block';
    img.style.borderRadius = '6px';
    img.style.boxShadow = '0 2px 8px rgba(0, 0, 0, 0.1)';
    img.style.cursor = 'pointer';
    img.style.margin = '0 auto';
    
    // Add click handler to show markdown on hover/click for editing
    img.addEventListener('mouseenter', () => {
      img.style.opacity = '0.8';
    });
    
    img.addEventListener('mouseleave', () => {
      img.style.opacity = '1';
    });
    
    wrap.appendChild(img);
    return wrap;
  }
  ignoreEvent() { return false; }
}

const foldField = StateField.define<DecorationSet>({
  create() { return Decoration.none; },
  update(deco, tr) {
    deco = deco.map(tr.changes);
    for (let e of tr.effects) {
      if (e.is(toggleFold)) {
        const { from, to } = e.value;
        let has = false;
        deco.between(from, to, () => { has = true; });
        if (has) {
          // remove decorations that intersect this range
          deco = deco.update({ filter: (fromA, toA) => !(fromA >= from && toA <= to) });
        } else {
          // compute number of lines
          const startLine = tr.state.doc.lineAt(from).number;
          const endLine = tr.state.doc.lineAt(to).number;
          const count = Math.max(1, endLine - startLine + 1);
          const widget = Decoration.replace({ widget: new FoldWidget(count) });
          deco = deco.update({ add: [widget.range(from, to)] });
        }
      }
    }
    return deco;
  },
  provide: f => EditorView.decorations.from(f)
});

const imageField = StateField.define<DecorationSet>({
  create(state) {
    return buildImageDecorations(state);
  },
  update(deco, tr) {
    if (tr.docChanged) {
      return buildImageDecorations(tr.state);
    }
    return deco.map(tr.changes);
  },
  provide: f => EditorView.decorations.from(f)
});

function buildImageDecorations(state: any): DecorationSet {
  const decorations: any[] = [];
  const doc = state.doc;
  
  for (let i = 1; i <= doc.lines; i++) {
    const line = doc.line(i);
    const text = line.text;
    const imageMatch = text.match(/!\[([^\]]*)\]\(([^)]+)\)/);
    
    if (imageMatch) {
      const [fullMatch, alt, src] = imageMatch;
      const start = line.from + text.indexOf(fullMatch);
      const end = start + fullMatch.length;
      
      // Replace the entire markdown image syntax with the image widget
      const widget = Decoration.replace({ 
        widget: new ImageWidget(src, alt)
      });
      decorations.push(widget.range(start, end));
    }
  }
  
  return RangeSet.of(decorations);
}

// reactive theme extension for font size (applied on mount)
let fontSizeTheme = EditorView.theme({ '.cm-scroller': { fontSize: `${fontSize}px`, lineHeight: '1.6', padding: '32px 0 120px' } });

$: fontSizeTheme = EditorView.theme({ '.cm-scroller': { fontSize: `${fontSize}px`, lineHeight: '1.6', padding: '32px 0 120px' } });

function baseExtensions(): Extension[] {
  return [
    highlightSpecialChars(),
    history(),
    drawSelection(),
  closeBrackets(),
  bracketMatching(),
    keymap.of([
  ...closeBracketsKeymap,
  ...historyKeymap,
  ...defaultKeymap,
      {
        key: 'Mod-s', preventDefault: true, run: () => {
          dispatch('save');
          return true;
        }
      },
      {
        key: 'Mod-n', preventDefault: true, run: () => {
          dispatch('new');
          return true;
        }
      },
      {
        key: 'Escape', preventDefault: false, run: () => {
          dispatch('escape');
          return false;
        }
      },
      // list continuation tweak
      {
        key: 'Enter', run: (view: EditorView) => {
          const { state } = view;
          const pos = state.selection.main.head;
          const line = state.doc.lineAt(pos);
          const text = line.text;
          const bulletMatch = text.match(/^(\s*- )/);
          if (bulletMatch) {
            const afterBullet = text.slice(bulletMatch[0].length);
            if (afterBullet.trim().length === 0) {
              // terminate list
              view.dispatch({ changes: { from: pos, to: pos, insert: '\n' } });
              return true;
            }
            view.dispatch({ changes: { from: pos, to: pos, insert: '\n' + bulletMatch[1] } });
            return true;
          }
          return false;
        }
      }
    ]),
  markdown({ base: markdownLanguage }),
  syntaxHighlighting(highlightStyle),
  minimalistTheme,
  foldField,
  imageField,
  EditorView.domEventHandlers({
    paste: (event: ClipboardEvent, view: EditorView) => {
      if (readOnly) return false;
      
      const clipboardData = event.clipboardData;
      if (!clipboardData) return false;
      
      // Check if clipboard contains image files
      const items = Array.from(clipboardData.items);
      const imageItem = items.find(item => item.type.startsWith('image/'));
      
      if (imageItem) {
        event.preventDefault();
        const file = imageItem.getAsFile();
        if (file) {
          saveImageLocally(file).then(imagePath => {
            insertImageMarkdown(imagePath, file.name || 'Pasted Image');
          }).catch(err => {
            console.error('Failed to save image:', err);
            // Fallback: insert placeholder markdown
            insertImageMarkdown('path/to/image.png', 'Failed to save image');
          });
        }
        return true;
      }
      
      return false;
    },
    click: (e: MouseEvent, view: EditorView) => {
      const pos = view.posAtCoords({ x: e.clientX, y: e.clientY });
      if (pos == null) return false;
      const line = view.state.doc.lineAt(pos);
      const text = line.text.trimStart();
      if (text.startsWith('#')) {
        // find next heading or end
        let start = line.to;
        let i = line.number + 1;
        while (i <= view.state.doc.lines) {
          const l = view.state.doc.line(i);
          if (l.text.trimStart().startsWith('#')) break;
          start = l.to;
          i++;
        }
        // range to fold: from end of heading line to start (start)
        const from = line.to + 1;
        const to = start;
        if (from <= to) {
          view.dispatch({ effects: toggleFold.of({ from, to }) });
          return true;
        }
      }
      return false;
    },
    mousemove: (e: MouseEvent, view: EditorView) => {
      try {
        const pos = view.posAtCoords({ x: e.clientX, y: e.clientY });
        if (pos == null) {
          if (view.dom) view.dom.style.cursor = 'text';
          return false;
        }
        const line = view.state.doc.lineAt(pos);
        const text = line.text.trimStart();
        if (text.startsWith('#')) {
          if (view.dom) view.dom.style.cursor = 'pointer';
        } else {
          if (view.dom) view.dom.style.cursor = 'text';
        }
      } catch (err) {
        // ignore
      }
      return false;
    }
  })
  ];
}

onMount(() => {
  const state = EditorState.create({
    doc: value,
    extensions: baseExtensions().concat([
  fontSizeTheme,
      EditorView.updateListener.of((v: { docChanged: boolean; state: EditorState }) => {
        if (v.docChanged) {
          value = v.state.doc.toString();
          dispatch('change', value);
        }
      }),
      EditorView.editable.of(!readOnly)
    ])
  });
  view = new EditorView({ state, parent: container });
  // apply spellcheck and font preferences to the editor container
  try {
    container.spellcheck = !!spellcheck;
  container.style.fontFamily = font;
  container.style.fontSize = fontSize + 'px';
  } catch (e) {}
});

// keep the editor in sync if the `value` prop changes from the parent
$: if (view && value !== undefined) {
  const docText = view.state.doc.toString();
  if (value !== docText) {
    // replace full document with new value
    view.dispatch({ changes: { from: 0, to: docText.length, insert: value } });
  }
}

onDestroy(() => {
  view?.destroy();
});

export function focus() { view?.focus(); }
</script>

<style>
.wrapper { height: 100%; }

/* Image widget styles */
:global(.cm-image-widget) {
  margin: 12px 0 !important;
  padding: 0 !important;
  display: block !important;
  text-align: center !important;
}

:global(.cm-image-widget img) {
  max-width: 100% !important;
  height: auto !important;
  display: block !important;
  border-radius: 6px !important;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1) !important;
  cursor: pointer !important;
  margin: 0 auto !important;
  transition: opacity 0.2s ease !important;
}

:global(.cm-image-widget img:hover) {
  opacity: 0.8 !important;
}

/* Dark mode support for shadow */
:global([data-theme="dark"] .cm-image-widget img) {
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3) !important;
}
</style>

<div class="wrapper" bind:this={container}></div>
