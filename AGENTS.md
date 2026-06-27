# AGENTS.md

## Project overview

`midee` is a browser-native MIDI studio built as a static Vite SPA. It loads and plays MIDI files locally in the browser, renders the piano roll with PixiJS, supports live MIDI/QWERTY input, looping, practice flows, and exports MP4 video with WebCodecs.

- Package name: `midee`
- Package manager: `npm` (`package-lock.json` is canonical)
- Runtime model: browser-only for core features — no app server, no SSR, no API routes
- Deploy target: static hosting from `dist/`

## Stack and architecture

- UI: SolidJS + TypeScript
- Bundler/dev server: Vite + `vite-plugin-solid`
- Rendering: PixiJS 8 + `pixi-filters`
- Audio/MIDI: `tone`, `@tonejs/piano`, `@tonejs/midi`, `@tonaljs/*`
- Export: WebCodecs + `mediabunny`
- Validation/env: `@t3-oss/env-core` + Zod
- Lint/format: Biome
- Unit tests: Vitest + jsdom + `@solidjs/testing-library`
- E2E tests: Playwright

Important architectural boundaries:

- `src/core/` holds pure logic and should stay UI-independent.
- `src/audio/` owns playback, scheduling, and offline audio rendering.
- `src/renderer/` owns Pixi canvas rendering.
- `src/midi/` owns MIDI input/recording flows.
- `src/export/` owns MP4 export.
- `src/ui/` contains the UI shell; parts of the app are still orchestrated by the imperative `App` class.
- State lives in `src/store/` and uses local reactive primitives, not React-style state management.

Do not assume React or Next.js patterns here.

## Setup commands

Run all commands from the `midee/` repository root.

```bash
npm install
```

Requirements:

- Node 18+ locally; CI uses Node 20
- Modern browser with Web MIDI and WebCodecs support

Before running Playwright locally on a fresh machine, install the browser binary:

```bash
npx playwright install chromium
```

Optional client env vars live in `src/env.ts` and must use the `VITE_` prefix:

- `VITE_POSTHOG_KEY`
- `VITE_POSTHOG_HOST`
- `VITE_ENABLE_BENCH`
- `VITE_SHOW_FPS`

Playwright also reads:

- `E2E_PORT` (defaults to `4173`)
- `E2E_HEAVY=1` to enable heavy export specs

## Development workflow

- Start dev server: `npm run dev`
- Default dev URL: `http://localhost:5173`
- Preview production build: `npm run preview`
- Preferred verification gate before calling work done: `npm run check`

Primary scripts from `package.json`:

```bash
npm run dev
npm run build
npm run preview
npm run typecheck
npm run lint
npm run lint:fix
npm run format
npm run test
npm run test:watch
npm run test:coverage
npm run test:e2e
npm run test:e2e:heavy
npm run test:e2e:report
npm run check
npm run icons
npm run preview:serp
npm run bench
npm run bench:update
```

## Testing instructions

### Unit tests

- Command: `npm run test`
- Watch mode: `npm run test:watch`
- Coverage: `npm run test:coverage`
- Framework: Vitest with `jsdom`
- Test files: `src/**/*.test.ts` and `src/**/*.test.tsx`
- Test setup: `vitest.setup.ts`
- Shared helpers: `src/test/`
- MIDI fixtures: `fixtures/*.mid`

### E2E tests

- Command: `npm run test:e2e`
- Heavy/local codec-sensitive suite: `npm run test:e2e:heavy`
- Report viewer: `npm run test:e2e:report`
- Fresh-machine prerequisite: `npx playwright install chromium`
- Config: `playwright.config.ts`
- Test directory: `e2e/`
- The default suite is intentionally serial (`workers: 1`) because timing-sensitive specs get flaky under parallel CPU contention.

Playwright uses:

```bash
npx vite build && npx vite preview --port <PORT> --strictPort
```

It deliberately does **not** call `npm run build`, because E2E only needs a static preview bundle and should skip `tsc` plus postbuild scripts.

### CI expectations

GitHub Actions runs two jobs in `.github/workflows/ci.yml`:

- `check`: `npm run typecheck`, `npm run lint`, `npm run test`, `npm run build`
- `e2e`: `npm run test:e2e`

CI installs Chromium first with:

```bash
npx playwright install --with-deps chromium
```

Blocking checks are:

- `typecheck`
- `test`
- `build`
- `test:e2e`

`lint` is visible in CI but intentionally non-blocking.

For most local changes, run at least:

```bash
npm run check
```

Run `npm run test:e2e` when your change affects browser behavior, playback flows, export flows, or other integration-heavy paths.

## Code style and conventions

Biome is the source of truth.

- Indentation: 2 spaces
- Line width: 100
- Quotes: single quotes
- Semicolons: `asNeeded`
- Trailing commas: always
- Arrow params: always include parentheses
- Import organization: enabled

Lint/format scope is `src/**/*.ts`, `src/**/*.tsx`, and `src/**/*.css`.

TypeScript rules worth preserving:

- `strict: true`
- `noUncheckedIndexedAccess: true`
- `exactOptionalPropertyTypes: true`
- `jsxImportSource: solid-js`
- `include: ["src"]`

Practical conventions:

- Match existing `src/ui/` patterns and `src/styles/main.css` variables for UI work.
- Read `src/core/clock/MasterClock.ts`, `src/audio/AudioEngine.ts`, and export call sites before changing timing, playback, or offline render behavior.
- Keep heavy export code lazy-loaded where the app already does so.

## Build and deployment

- Production build: `npm run build`
- Output directory: `dist/`
- Build command: `tsc && vite build`
- Postbuild runs automatically and executes:

```bash
node scripts/build-content.mjs
node scripts/build-og.mjs
node scripts/stamp-sitemap.mjs
node scripts/check-links.mjs
node scripts/upload-sourcemaps.mjs
```

What the postbuild chain does:

- builds static content pages
- generates OG assets
- stamps sitemap output
- checks links
- uploads hidden sourcemaps to PostHog

Notes:

- Production sourcemaps are `hidden` in Vite.
- `npm run build` may involve network-dependent postbuild steps.
- For app-only verification, prefer `npm run check` unless you specifically need full build coverage.

## Known gotchas

- This is not a monorepo package inside a running app server; it is a standalone static app.
- Do not remove the `events` alias in `vite.config.ts`; `@tonejs/piano` depends on it for browser builds.
- Do not remove the Pixi manual chunking in `vite.config.ts`; it prevents a real TDZ/circular-chunk crash.
- `mediabunny` is pre-bundled on purpose to avoid Vite dep-optimizer breakage on dynamic import.
- E2E Chromium launch flags exist as a workaround for codec/export behavior; do not simplify them casually.
- Hidden sourcemaps are uploaded separately; production stack traces rely on that workflow.
- The repo may appear on disk as `pianoroll`, but the product/package name is `midee`.

## Security and privacy

- Core product behavior is local-first: no MIDI upload flow, no account system, no backend dependency for playback/export.
- Keep client env vars prefixed with `VITE_`.
- Treat analytics keys and sourcemap-upload credentials as secrets; never hardcode them.

## Pull request and change guidance

- Prefer minimal, scoped changes.
- Preserve existing architecture boundaries instead of introducing cross-layer shortcuts.
- Add or update tests when behavior changes.
- Before calling work complete, prefer `npm run check` and any additional targeted test coverage your change needs.
- Do not create commits from the agent unless the user explicitly asks. If the user only wants a commit message, provide the message text instead of running `git commit`.

## Useful paths

- Entry: `index.html`, `src/main.tsx`
- Bootstrap/orchestration: `src/createApp.ts`, `src/app.ts`, `src/AppRoot.tsx`
- Store: `src/store/`
- Core logic: `src/core/`
- Audio: `src/audio/`
- Renderer: `src/renderer/`
- MIDI: `src/midi/`
- Export: `src/export/`
- UI: `src/ui/`
- Styles: `src/styles/`
- Tests/helpers: `src/test/`, `fixtures/`, `e2e/`
- Build/SEO scripts: `scripts/`
