# CLAUDE.md - whisperly_client

> **Git policy — never auto-commit or auto-push.** Leave your work in the working tree.
> Run `git commit`, `git push`, `gh pr create`, or any push script **only when the user
> explicitly asks in that turn**. Approval for an earlier change does not carry forward, and
> finishing a task is not permission to commit it. Do not create feature branches; stay on
> the current branch.

## What it is

`@sudobility/whisperly_client` is the typed API client library for the Whisperly
localization platform. It provides a `WhisperlyClient` class wrapping every REST
endpoint and TanStack Query hooks for React data fetching.

- **Platform**: web + React Native. No DOM APIs in `src` — HTTP goes through an
  injected `NetworkClient`, so keep it DOM-free.
- **Output**: ESM-only (`"type": "module"`), `dist/index.js` + `dist/index.d.ts`.
- **Published to GitHub Packages** (not public npm). `.npmrc` scopes `@sudobility`
  to `https://npm.pkg.github.com`; CI publishes with `publish-target: github`.

## Stack & tooling

- **Bun** is the package manager and runner: `bun install`, `bun run <script>`, `bun add`.
- TypeScript (`tsc`), `moduleResolution: "bundler"`, builds via `tsconfig.build.json`.
- Vitest for tests; ESLint (flat config) + Prettier.

## Commands (verified against package.json scripts)

```bash
bun run build        # tsc -p tsconfig.build.json && node scripts/fix-esm-imports.mjs
bun run build:watch  # tsc --watch
bun run typecheck    # tsc --noEmit
bun run lint         # eslint src --ext .ts,.tsx
bun run lint:fix     # eslint --fix
bun run format       # prettier --write src/**/*.ts
bun run test:run     # vitest run   (test:unit is an alias)
bun run verify       # typecheck + lint + test:unit + build — run before publishing
```

## Architecture

```
src/
├── index.ts                        # Public barrel: re-exports network, hooks, utils,
│                                   #   QUERY_KEYS, and many types from @sudobility/whisperly_types
├── types.ts                        # WhisperlyClientConfig, QUERY_KEYS, QueryKeys
├── network/
│   ├── index.ts                    # exports WhisperlyClient
│   └── WhisperlyClient.ts          # the API client (all endpoint methods)
├── hooks/
│   ├── index.ts                    # hooks barrel
│   ├── useProjects.ts  useDictionary.ts  useLanguages.ts
│   ├── useSettings.ts  useAnalytics.ts   useTranslate.ts
│   └── hooks.test.ts               # single combined hook test suite
└── utils/
    ├── index.ts
    └── whisperly-helpers.ts        # buildUrl, handleNetworkResponse, formatQueryParams, WhisperlyApiError
```

Tests live beside source as `*.test.ts` (`types.test.ts`, `network/WhisperlyClient.test.ts`,
`utils/whisperly-helpers.test.ts`, `hooks/hooks.test.ts`).

### Request flow (NON-OBVIOUS — not fetch/auth-header based)

1. `new WhisperlyClient({ baseUrl, networkClient })` — config is exactly those two
   fields. `networkClient` is a `NetworkClient` from `@sudobility/types`; **this library
   does NOT build auth headers or call `fetch` directly.** Transport and auth are the
   injected client's job.
2. `WhisperlyClient` prefixes every path with `apiPrefix = '/api/v1'`, builds the URL
   via `buildUrl` + `formatQueryParams`, then calls `networkClient.get/post/put/delete`.
3. `handleNetworkResponse<T>(response)` unwraps the API envelope `{ success, data, timestamp }`,
   returning `response.data.data`; on `!response.ok` it throws `WhisperlyApiError(message, statusCode, details)`.

### Hooks

- Every hook's first arg is the `WhisperlyClient`. Queries use an `enabled` flag so they
  stay idle until scope params are truthy.
- `useProjects`, `useDictionaries`, `useProjectLanguages`, `useSettings` return the query
  spread plus nested mutation objects and an `update()` refetch helper.
- `useSearchDictionary` is a **query** (key `dictionarySearch`), not a mutation.
- `useAnalytics(client, entitySlug, options?)` takes an options object
  (`startDate`, `endDate`, `projectId`, `enabled`), not positional dates.
- `useGenerateApiKey` / `useDeleteApiKey` take `(client, entitySlug, projectId)`.
- `useTranslate(client, testMode?)` → mutate with `{ orgPath, projectName, request, apiKey? }`.
- Query keys are the `QUERY_KEYS` constants in `types.ts` (e.g. `whisperly-projects`).
  Note `QUERY_KEYS.subscription` exists but has no hook yet.

## Conventions

- Add a method to `WhisperlyClient`, add a `QUERY_KEYS` entry if needed, write the hook
  following the query/mutation + `update()` pattern, export from the relevant
  `index.ts`, and add a test.
- Re-export shared domain types from `@sudobility/whisperly_types` through `src/index.ts`
  rather than redefining them.
- Return types come from `whisperly_types`; e.g. `getDictionaries` returns
  `DictionaryListItem[]` (not `DictionarySearchResponse[]`).

## Gotchas

- **ESM import fix-up is required.** `tsc` (bundler resolution) emits extensionless
  relative imports that Node's native ESM loader rejects; `scripts/fix-esm-imports.mjs`
  rewrites them to explicit `.js` paths after every build. Do not drop it from `build`.
- **Duplicate config files exist** in the repo root: `eslint.config.js` (flat, active)
  and `eslint.config.mjs`; `.prettierrc` and `.prettierrc.json`. The active ESLint flat
  config is `eslint.config.js`, and its `ignores` list still names unrelated dirs
  (`lib`, `indexer`, `wildduck`) copied from a template — harmless but stale.
- Installing deps requires a GitHub Packages token with `read:packages` (the
  `@sudobility` scope is private).
