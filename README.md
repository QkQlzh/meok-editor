# meok-editor · 먹
A React AI document editor for people who write in Korean. Built on Tiptap 3.

```tsx
<MeokEditor ai={createFetchAdapter('/api/ai')} />
```

Select some text and hit **✦ AI** to improve it, fix spelling and spacing, summarize, translate, and so on. `Ctrl + J` continues writing from the cursor. Nothing goes straight into the document: results show up in a preview first, and you insert them only if you like them.

## Install

Clone and build from source.

```bash
git clone <repo-url> meok
cd meok
npm install
npm run build        # outputs dist/
```

In your app, install it by path, e.g. `npm install ../meok`. Requires React 18 or 19.

## Usage

### 1. Editor

```tsx
'use client';

import { MeokEditor, createFetchAdapter } from 'meok';
import 'meok/styles.css';

const ai = createFetchAdapter('http://localhost:8000/api/ai');

export default function Page() {
  return (
    <MeokEditor
      ai={ai}
      content="<h1>회의록</h1><p>오늘 회의에서 예산안을 논의했다.</p>"
      onChange={({ json }) => saveDraft(json)}
    />
  );
}
```

Without `ai`, it works as a plain editor with no AI buttons.

### 2. Server (`server/main.py`)

```bash
pip install fastapi uvicorn anthropic python-multipart
export ANTHROPIC_API_KEY=sk-ant-...
export MEOK_ORIGINS=http://localhost:5173      # where the editor runs (CORS)
uvicorn server.main:app --reload --port 8000
```

Two endpoints:

| Route | What it does |
|---|---|
| `POST /api/ai` | Takes an editor request and **streams the Claude response as plain text** |
| `POST /api/import` | Upload `.hwp` / `.hwpx` → HTML ready for the editor (requires hanji) |

Environment variables:

| Variable | Default |
|---|---|
| `MEOK_MODEL` | `claude-sonnet-5-5` |
| `MEOK_MAX_TOKENS` | `4096` |
| `MEOK_ORIGINS` | `http://localhost:5173` (comma-separated for several) |

The editor also sends fully built prompts (`system`, `prompt`) with each request, but **the server ignores them** and rebuilds the prompt from `action`, `text`, `context` and the other fields. Anything coming from the browser can be modified by the user. Input length limits are enforced on the server as well. Authentication and rate limiting are up to you; there is a placeholder for them in `server/main.py`.

### 3. Importing HWP files

Install [hanji](#sibling-project) on the server to enable `/api/import`.

```tsx
<MeokEditor onReady={(e) => (editorRef.current = e)} />

async function importHwp(file: File) {
  const form = new FormData();
  form.append('file', file);
  const res = await fetch('http://localhost:8000/api/import', { method: 'POST', body: form });
  const { html } = await res.json();
  editorRef.current?.commands.setContent(html);   // merged table cells come through intact
}
```

## Using another model or server

`ai` is just a function. Return an async iterable that yields text chunks, or a Promise of the full text.

```ts
import type { AiAdapter } from 'meok';

const ai: AiAdapter = async ({ system, prompt, signal }) => {
  const res = await fetch('/my/llm', { method: 'POST', body: JSON.stringify({ system, prompt }), signal });
  return (await res.json()).text;
};
```

Request fields: `action` (`improve` `fix` `shorten` `expand` `summarize` `translate` `tone` `custom` `continue` `write`), `text` (the selection), `context` (up to 2,000 characters of preceding document text), `instruction`, `language`, `tone`, `system`, `prompt`, `signal`.

## Props

| Prop | Default | Description |
|---|---|---|
| `content` | `''` | Initial content (HTML or Tiptap JSON) |
| `ai` | — | AI adapter |
| `locale` | `'ko'` | `'ko'` / `'en'` |
| `theme` | `'auto'` | `'auto'` / `'light'` / `'dark'` |
| `editable` | `true` | Read-only when `false` |
| `translateTo` | English, Korean, Japanese, Chinese | Languages in the translate menu |
| `onChange` | — | `{ html, json, text }`, 300 ms after typing stops |
| `onReady` | — | Tiptap `Editor` instance |
| `toolbarExtra` | — | Buttons rendered at the right of the toolbar |

| Keys | Action |
|---|---|
| `/` | Command menu (searchable in Korean too: `/표`, `/제목`, `/ai`) |
| `Ctrl/⌘ + J` | Continue writing with AI |
| `Esc` | Discard the AI result |
| `Ctrl/⌘ + Z` | Undo (an inserted AI result reverts in one step) |

Colors and fonts are CSS variables on `.meok` (`--meok-accent`, `--meok-ai`, `--meok-font`, …).
