# Clean Svelte based Text Editor with simple markup
This is a minimal Svelte + Vite scaffold for the IA Editor UI prototype.

Quick start (PowerShell):

```powershell
npm install
npm run dev
```

Notes:
- This is a web dev scaffold. For a native desktop app wrap with Tauri later (recommended) or use Electron.
- File persistence currently uses localStorage. We'll swap to real filesystem when integrating Tauri.
- The editor uses a mirrored highlight layer so markup markers (like `**`, `#`, `-`) remain visible while styled.

## Image Support

The editor supports adding images in two ways:

1. **Markdown syntax**: Use standard markdown image syntax
   ```markdown
   ![Tux, the Linux mascot](/assets/images/tux.png)
   ```

2. **Paste images**: Simply use Ctrl+V (or Cmd+V) to paste images directly from your clipboard

**Note**: When pasting images, they are automatically stored locally for easy access and portability.
