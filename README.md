<div align="center">
  <img src="public/icon/128.png" alt="Readsight logo" width="80" />
  <h1>Readsight · 见微</h1>
  <p><strong>Bilingual reading, contextual lookup, and AI explanations — without leaving the page.</strong></p>
  <p><strong>English</strong> · <a href="README.zh-CN.md">简体中文</a></p>
</div>

Readsight (见微) is a Chrome MV3 extension plus a self-hosted Next.js backend for reading English articles deeply. It is built for readers who can follow most of an English page but get stuck on individual words, sentences, or grammar — the current primary workflow is **English → Simplified Chinese**.

Run your own backend and bring your own model API keys. Model requests go through the backend, so provider keys stay server-side.

## Features

- **In-page lookup** — select a word, sentence, or paragraph and click the dot / "Look up" indicator next to the selection to open a draggable popup: IPA, definitions, and inflections for words; inline Chinese translation for longer selections. A local ECDICT SQLite (optional, not bundled) adds Collins stars, Oxford 3000, and frequency ranks.
- **Contextual reading actions** — from the same popup, ask for meaning/usage, sentence translation, grammar analysis, rewriting, or an explanation of the difficult parts — and continue the thread in the side panel.
- **Context-aware AI chat** — a streaming assistant in the Chrome side panel that knows what you selected, powered by DeepSeek.
- **Bilingual page translation** — injects Chinese below English paragraphs. Article mode filters out navigation clutter; page mode covers broader page text. Lazy-loaded and SPA content is picked up by rescans, paragraphs are cached, and failed blocks retry individually.
- **Reading layouts** — a full-screen immersive reader with single-column or parallel two-column bilingual views.
- **Learning records** — the backend stores your selections with source and context, tracks lookups and re-encounters, and links you back to the original page.
- **Quiet controls & pronunciation** — manual by default, with opt-in auto-translation per language or per domain. Popup, sentence, and phrase audio use the browser's `speechSynthesis` (`en-US`); word cards may play dictionary audio first with synthesis as fallback. Voice availability varies by OS and browser.

## Usage

1. Open an English article in Chrome.
2. Select a word or a passage, then click the dot / "Look up" indicator that appears next to the selection.
3. In the popup, read the dictionary card or the Chinese translation, and use actions like grammar, rewriting, or difficult-part explanations — open the side panel when you want to dig deeper.
4. Toggle bilingual translation with the floating button, the right-click context menu, or `Alt+T` (`Option+T` on macOS).
5. Open the immersive reader for a focused single- or two-column layout.
6. Review your learning records from the side panel.

## Quick start

### Prerequisites

- Node.js **>= 22.19.0**
- pnpm **10.32.1**
- Chrome with side panel support
- A dedicated local Postgres database (e.g. named `readsight`) — create it before migrating
- A DeepSeek API key

### Install

The extension and the server are separate packages (not a workspace); install both:

```bash
git clone https://github.com/hifizz/relata.git
cd relata
pnpm install
pnpm --dir server install
```

### Configure

Create `.env.local` in the repo root — only the public backend URL the extension should call (never put provider secrets in the extension):

```ini
WXT_BACKEND_URL=http://localhost:4010
```

Create `server/.env.local` — replace every placeholder with real values:

```ini
DATABASE_URL=postgresql://readsight:change-me@localhost:5432/readsight
DEEPSEEK_API_KEY=your-deepseek-api-key
```

See [Configuration](#configuration) for optional fallback providers and other variables.

### Migrate the database

`server/scripts/migrate.mjs` reads `process.env` directly and does **not** auto-load `.env.local`, so pass the env file explicitly from the repo root:

```bash
pnpm --dir server exec node --env-file=.env.local scripts/migrate.mjs
```

This applies the committed Drizzle migrations into the `readsight` schema — no manual SQL needed.

### Run

Terminal 1 — backend on `localhost:4010`:

```bash
pnpm --dir server dev
```

Terminal 2 — extension. For a production bundle that points at your local backend:

```bash
WXT_BACKEND_URL=http://localhost:4010 pnpm build
```

Then open `chrome://extensions`, enable **Developer mode**, **Load unpacked**, and select `.output/chrome-mv3`.

For development, `pnpm dev` runs WXT dev mode and outputs to `.output/chrome-mv3-dev` instead. When using the production unpacked output, rebuild and reload the extension after source changes.

## Architecture

Strict context isolation: the content script is the only code that touches page DOM; all cross-context traffic flows through the background service worker; the side panel never reads the page directly.

| Path | Role |
| --- | --- |
| `entrypoints/content/` | Content script: selection popup, bilingual injection, immersive reader |
| `entrypoints/background.ts` | Message broker and network gateway between contexts |
| `entrypoints/sidepanel/` | Side panel UI: chat, settings, learning records |
| `components/`, `lib/` | Shared UI cards, settings, messaging helpers |
| `server/src/app/api/` | Next.js API routes: translate, lookup, chat, records |
| `server/src/db/`, `server/drizzle/` | Drizzle schema and committed Postgres migrations |

**Stack:** WXT 0.20 + React 19 + TypeScript + Tailwind CSS 4 + assistant-ui (extension); Next.js 16 + Drizzle + Postgres (server).

**Models:** DeepSeek `deepseek-v4-pro` for contextual chat; `deepseek-v4-flash` for translation and model-assisted lookup (thinking disabled). Translation falls back DeepSeek → Zhipu GLM-4-Flash → OpenRouter.

## Commands

| Command | Description |
| --- | --- |
| `pnpm dev` | WXT dev mode → `.output/chrome-mv3-dev` |
| `pnpm build` | Production extension build → `.output/chrome-mv3` |
| `pnpm zip` | Zip the built extension |
| `pnpm compile` | Extension typecheck (`tsc --noEmit`) |
| `pnpm --dir server dev` | Backend dev server on port 4010 |
| `pnpm --dir server build` / `pnpm --dir server start` | Next.js production build / serve |
| `pnpm --dir server lint` | ESLint |
| `pnpm --dir server exec tsc --noEmit` | Backend typecheck |
| `pnpm --dir server db:generate` | Generate a new Drizzle migration (schema development only) |
| `pnpm --dir server db:migrate` | Apply migrations — requires `DATABASE_URL` in the environment; see the `--env-file` command in [Quick start](#quick-start) |

## Configuration

**Extension (build-time, public):**

| Variable | Description |
| --- | --- |
| `WXT_BACKEND_URL` | Backend base URL baked into the bundle. Defaults to `http://localhost:4010`. |

**Server (`server/.env.local`):**

| Variable | Required | Description |
| --- | --- | --- |
| `DATABASE_URL` | Yes | Postgres connection string for the dedicated database. |
| `DEEPSEEK_API_KEY` | Yes | Chat, model-assisted lookup, and primary translation. |
| `ZHIPU_API_KEY` | No | First translation fallback (GLM-4-Flash). |
| `OPENROUTER_API_KEY` | No | Second translation fallback. |
| `OPENROUTER_MODEL` | No | OpenRouter model override (default `meta-llama/llama-3.3-70b-instruct:free`). |
| `TRANSLATE_PRIMARY` | No | `deepseek` (default), `zhipu`, or `openrouter` — reorders the translation chain. |
| `ECDICT_PATH` | No | Absolute path to a compatible ECDICT SQLite file. |
| `CORS_ALLOW_ORIGIN` | No | CORS origin for the API routes (default `*`). Not an auth mechanism. |

**Optional local dictionary (ECDICT):** not bundled. To enable local lookups, place a compatible ECDICT-derived SQLite database with a `dict` table at `server/data/ecdict.sqlite`, or point `ECDICT_PATH` at it — see [`server/src/lib/ecdict.ts`](server/src/lib/ecdict.ts) for the adapter format. The file is opened lazily on first lookup; without it, word lookup falls back to an online dictionary plus the model. Dictionary data attribution: skywind3000/ECDICT, CC BY 4.0.

## Status & caveats

- **Self-hosted, single-user development baseline.** API routes currently have no authentication or rate limiting — add auth, restrict `CORS_ALLOW_ORIGIN`, and secure any private deployment before exposing it to the Internet. Never expose provider keys inside the extension.
- Selected text/context and paragraphs to translate can be sent through your backend to model or dictionary services (a cache hit may avoid a provider call). Word-pronunciation audio may be fetched directly from external audio URLs. Learning records and the translation cache are stored in Postgres; Chrome sync is used only for extension preferences.
- Chrome only (requires the side panel API). Dynamic or restricted pages may have limitations.
- Pronunciation uses browser speech synthesis; voice availability and quality depend on the OS/browser voices installed.

## Documentation

- [PRD.md](PRD.md) — product requirements (中文)
- [CLAUDE.md](CLAUDE.md) — developer and architecture notes
- [docs/PROJECT_STATUS.md](docs/PROJECT_STATUS.md) — historical milestone snapshot (the top addendum reflects current model routing)
- [docs/notes-translate-injection.md](docs/notes-translate-injection.md) — translation injection internals
