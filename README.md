# @sudobility/whisperly_client

Typed API client and TanStack Query hooks for the Whisperly localization platform.

It ships a `WhisperlyClient` class that wraps every Whisperly REST endpoint with
type-safe methods, plus a set of React hooks (TanStack Query) for data fetching.
HTTP is performed through an injected `NetworkClient` (from `@sudobility/types`),
so the library is DOM-free and runs on both web and React Native. Published as an
ESM-only package.

## Installation

Packages in the `@sudobility` scope are hosted on **GitHub Packages**, not the public
npm registry. The repo's `.npmrc` already points the scope at the GitHub registry:

```
@sudobility:registry=https://npm.pkg.github.com
```

Consumers need a GitHub token with `read:packages` configured for that registry, then:

```bash
bun add @sudobility/whisperly_client
```

### Peer dependencies

```jsonc
"@sudobility/types": "^1.9.78",
"@tanstack/react-query": ">=5.0.0",
"react": ">=18.0.0"
```

## Usage

```typescript
import { WhisperlyClient, useProjects, useTranslate } from "@sudobility/whisperly_client";

// networkClient implements the NetworkClient interface from @sudobility/types.
// It is responsible for transport and for attaching auth headers — this library
// does not build auth headers itself.
const client = new WhisperlyClient({
  baseUrl: "https://api.whisperly.io",
  networkClient,
});

// Direct client usage (all paths are prefixed with /api/v1 internally)
const projects = await client.getProjects(entitySlug);

// React hook usage — every hook takes the client as its first argument
const { data: projects } = useProjects(client, entitySlug);
```

## API

### WhisperlyClient methods

All methods return the unwrapped `data` payload and throw `WhisperlyApiError`
(exposing `statusCode` and `details`) on a non-OK response.

- **Projects** (`/entities/:entitySlug/projects`): `getProjects`, `getProject`,
  `createProject`, `updateProject`, `deleteProject`, `generateProjectApiKey`,
  `deleteProjectApiKey`
- **Dictionary** (`/.../dictionary`): `getDictionaries`, `searchDictionary`,
  `createDictionary`, `updateDictionary`, `deleteDictionary`
- **Project languages** (`/.../languages`): `getProjectLanguages`, `updateProjectLanguages`
- **Available languages** (`/available-languages`): `getAvailableLanguages`
- **Settings** (`/users/:userId/settings`): `getSettings`, `updateSettings`
- **Analytics** (`/entities/:entitySlug/analytics`): `getAnalytics`
- **Rate limits** (`/ratelimits/:entitySlug`): `getRateLimits`, `getRateLimitHistory`
- **Translation** (`/translate/:orgPath/:projectName`): `translate` — the primary
  public endpoint; auth is optional and can be supplied via an `apiKey` argument

### TanStack Query hooks

Every hook takes `(client, ...scopeParams)`. Queries are disabled until their
scope params are truthy.

| Hook | Signature | Kind |
|------|-----------|------|
| `useProjects` | `(client, entitySlug)` | query + create/update/delete mutations |
| `useProject` | `(client, entitySlug, projectId)` | query |
| `useGenerateApiKey` | `(client, entitySlug, projectId)` | mutation |
| `useDeleteApiKey` | `(client, entitySlug, projectId)` | mutation |
| `useDictionaries` | `(client, entitySlug, projectId)` | query + create/update/delete mutations |
| `useSearchDictionary` | `(client, entitySlug, projectId, languageCode, text)` | query |
| `useProjectLanguages` | `(client, entitySlug, projectId)` | query + update mutation |
| `useAvailableLanguages` | `(client)` | query |
| `useSettings` | `(client, userId)` | query + update mutation |
| `useAnalytics` | `(client, entitySlug, options?)` | query |
| `useTranslate` | `(client, testMode?)` | mutation |

## Development

Uses **Bun** as the package manager and task runner.

```bash
bun install          # Install dependencies
bun run build        # Compile to dist/ (tsc + ESM import fix-up)
bun run build:watch  # Compile in watch mode
bun run typecheck    # tsc --noEmit
bun run lint         # ESLint over src
bun run format       # Prettier write over src
bun run test:run     # Run the Vitest suite once (test:unit is an alias)
bun run verify       # typecheck + lint + test + build (run before publishing)
```

Tests use Vitest in a `node` environment by default; hook tests opt into `jsdom`
via a per-file `// @vitest-environment jsdom` pragma.

## Related packages

- `whisperly_types` — Shared type definitions
- `whisperly_lib` — Business logic with Zustand stores
- `whisperly_api` — Backend API server
- `whisperly_app` — Web application

## License

BUSL-1.1
