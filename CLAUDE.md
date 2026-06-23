# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

NextChat (package name `nextchat`, formerly ChatGPT-Next-Web) is a cross-platform LLM chat UI built with Next.js 14 (App Router). It runs as a web app (standalone server or static export) and as a desktop app via Tauri (`src-tauri/`). The same codebase talks to ~17 different LLM providers (OpenAI, Azure, Anthropic, Google Gemini, Baidu, ByteDance, Alibaba, Tencent, Moonshot, iFlytek, DeepSeek, XAI, ChatGLM, SiliconFlow, 302.AI, Stability) through a uniform client abstraction, and supports MCP (Model Context Protocol) tooling.

## Commands

- `yarn dev` — run the dev server (also rebuilds masks on change via `concurrently`).
- `yarn build` — production build in `standalone` mode (runs `yarn mask` first).
- `yarn export` — static export build (`BUILD_MODE=export`); used for the Tauri app and static hosting.
- `yarn lint` — `next lint` (ESLint).
- `yarn test` — Jest in watch mode. `yarn test:ci` — single non-watch run for CI.
- Run a single test: `node --no-warnings --experimental-vm-modules $(yarn bin jest) test/model-available.test.ts`
- `yarn mask` — regenerate `public/masks.json` from `app/masks/{cn,tw,en}.ts` (see Masks below).
- `yarn app:dev` / `yarn app:build` — Tauri desktop app dev/build.

Package manager is **yarn 1.x** (`packageManager: yarn@1.22.19`). Node 18+ assumed.

Pre-commit runs `lint-staged` (husky) which applies `eslint --fix` + `prettier --write` to `app/**`. Prettier config: 2-space indent, double quotes, trailing commas (`all`), 80 col width.

## Architecture

### Provider abstraction (the core pattern)

The defining feature of this codebase is that adding/modifying an LLM provider touches a fixed set of files. The flow is **browser → Next.js API route (proxy) → upstream provider**, so API keys can be kept server-side.

Client side (`app/client/`):
- `app/client/api.ts` defines the abstract `LLMApi` class (`chat`, `speech`, `usage`, `models`) and the `ClientApi` wrapper. `getClientApi(provider)` maps a `ServiceProvider` to a concrete implementation. `getHeaders()` here centralizes per-provider auth header construction (each provider has its own header name and key source).
- `app/client/platforms/*.ts` — one file per provider implementing `LLMApi` (e.g. `openai.ts`, `anthropic.ts`, `google.ts`). These build the request body in the provider's wire format, stream responses, and handle tool calls.

Server side (`app/api/`):
- `app/api/[provider]/[...path]/route.ts` — the single catch-all edge route. It switches on `ApiPath` and dispatches to a per-provider `handle()` function. **`runtime = "edge"`.** (Tencent uses its own `/api/tencent` route, not the catch-all.)
- `app/api/<provider>.ts` — per-provider server handler that authenticates, injects the server-side API key, rewrites the path, and proxies to the upstream. `app/api/common.ts` holds the shared OpenAI proxy logic (`requestOpenai`) and model-availability filtering.
- `app/api/auth.ts` — `auth()` validates the access code (`CODE` env, md5-hashed) or a user-supplied API key, and injects the system key when the user didn't provide one.

Enums tying it together live in `app/constant.ts`: `ServiceProvider` (UI/config-level), `ModelProvider` (client implementation selector), `ApiPath` (route paths), and per-provider base URLs + path constants. **When adding a provider, you generally update: `constant.ts` (enums, base URL, paths), a new `client/platforms/*.ts`, a new `api/*.ts` handler wired into `[provider]/[...path]/route.ts`, `client/api.ts` (`ClientApi` switch + `getClientApi` + `getHeaders`), `config/server.ts` (env key), and the access store.**

### State management (Zustand + persistence)

Stores live in `app/store/` and are re-exported from `app/store/index.ts`. Key stores:
- `chat.ts` — sessions, messages, and the orchestration of `api.llm.chat(...)` calls (summarization, tool calls, retries). This is where the request lifecycle is driven.
- `access.ts` — provider API keys/endpoints and access control; fetches server config.
- `config.ts` — app/model config. `mask.ts`, `prompt.ts`, `plugin.ts`, `sync.ts`, `sd.ts`, `update.ts` cover the rest.

Persistence uses a custom store helper (`app/utils/store.ts`) with IndexedDB (`app/utils/indexedDB-storage.ts`). Stores are versioned with migrations — when changing persisted state shape, bump the version and add a migration.

### Masks (prompt templates / personas)

Masks are pre-built prompt presets. Source is TypeScript in `app/masks/{cn,tw,en}.ts`; `app/masks/build.ts` (run via `yarn mask`) compiles them into `public/masks.json`, which the app fetches at runtime. **Edit the `.ts` sources, not the generated JSON.** `yarn dev` watches and rebuilds automatically.

### MCP (Model Context Protocol)

`app/mcp/` implements MCP client support, gated behind the `ENABLE_MCP` env var. `actions.ts` are Next.js server actions (`"use server"`) managing MCP server processes; runtime config is written to `app/mcp/mcp_config.json` (seeded from `mcp_config.default.json`). Because it spawns processes and reads/writes files, MCP only works in the server runtime, not the static export/desktop builds.

### UI

`app/components/` holds the React UI; each component pairs a `.tsx` with a `.module.scss`. `home.tsx` is the shell and sets up routing (react-router-dom, paths in the `Path` enum in `constant.ts`). `chat.tsx`, `settings.tsx`, `mask.tsx`, `sd.tsx` (Stable Diffusion), and `mcp-market.tsx` are the major screens. SVG icons in `app/icons/` are imported as React components via `@svgr/webpack`.

### Build modes

`next.config.mjs` reads `BUILD_MODE` (`standalone` default, or `export`). Export mode forces a single chunk, unoptimized images, and disables the API rewrites/CORS headers (no server). The `app/config/client.ts` `getClientConfig()` exposes `isApp` (Tauri) which changes proxy behavior (apps call upstreams directly rather than via `/api/proxy/...`).

## Conventions

- Path alias `@/*` maps to the repo root (see `tsconfig.json` and `jest.config.ts`).
- Configuration is read on the server via `app/config/server.ts` (`getServerSideConfig()`, env vars) and on the client via `app/config/client.ts`. Add new provider env vars to `getServerSideConfig` and document them in `.env.template` + README.
- i18n: locale files in `app/locales/`; the app supports many languages — add keys to all locales (or at least `en.ts` and `cn.ts`).
- Tests live in `test/` and use Jest + Testing Library (jsdom). Existing tests focus on model availability/provider parsing logic (`app/utils/model.ts`).
- There is a deliberately preserved share-message string in `app/client/api.ts` (`share()`); a code comment asks not to modify it.
