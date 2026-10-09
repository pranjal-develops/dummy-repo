# Technical Design Document: dummy-repo

## 1. Executive Summary & Core Responsibilities

The `dummy-repo` module is a **TypeScript-first, React 19-rendered frontend feature module** built atop the `test-project` Vite template. It serves as a self-contained unit for displaying and interacting with repository metadata within the broader application shell.

**Primary Technical Capabilities:**
- **Compile-time safety**: All data shapes, prop interfaces, and event handlers are strictly typed via `typescript` `~6.0.2` in strict mode, enforced by `tsc -b` in the CI/build pipeline.
- **Vite-native HMR**: Module hot-replacement operates at the component and hook level, leveraging `@vitejs/plugin-react` (Ocx-based) for fast dev iterations.
- **ESLint-strict linting**: Configuration extends `tseslint.configs.strictTypeChecked` plus `eslint-plugin-react-hooks` and `eslint-plugin-react-refresh`, ensuring disciplined `react` `^19.2.8` and `react-dom` `^19.2.8` usage.
- **Module entry point**: Exported via ES module syntax (`"type": "module"` in `package.json`), importing convention follows `src/modules/dummy-repo/` boundary.

**Module Scope:**
- Responsible for rendering a repository overview UI, committing feed, and static metadata display.
- No backend logic, no runtime database, and no external API calls in the initial onboarding phase. All data is sourced via typed static JSON fixtures or internal state.
- Designed for incremental extension: future phases may plug in a REST/GraphQL backend, but the current contract is entirely frontend-local.

**Primary Responsibility Boundary:**
- The module owns the component tree from `DummyRepoPage` down to leaf atoms (`RepoHeader`, `CommitListItem`, `StatsBar`).
- Data fetching, normalization, and error handling are orchestrated by the custom hook `useRepoInternalData`.
- Rendering pipeline: `main.tsx` → `App.tsx` → `<DummyRepoModule />` → component hierarchy → DOM.

---

## 2. Component Architecture & System Flow

The module follows a **container-presentational segregation** pattern. All components are TypeScript functional components with hooks; no class components exist.

### Key Component Files (to be created under `src/modules/dummy-repo/`):

| File | Responsibility | Exports |
|------|----------------|---------|
| `DummyRepoPage.tsx` | Root container; composes header, feed, and stats. Fetches data via hook. | `DummyRepoPage` |
| `RepoHeader.tsx` | Displays repository name, description, and meta stats bar. Presentational only. | `RepoHeader` |
| `CommitListItem.tsx` | Renders a single commit entry (ID, message, author, timestamp). | `CommitListItem` |
| `CommitFeed.tsx` | Infinite-scroll-or-list container for `CommitListItem` components. Accepts `commits: Commit[]`. | `CommitFeed` |
| `StatsBar.tsx` | Displays star/fork counts, last updated timestamp. | `StatsBar` |
| `useRepoInternalData.ts` | Custom hook: executes data import, provides `RepoState` (loading, error, data), and mutation triggers. | `useRepoInternalData`, `RepoState` |

### System Data Flow (Technical Workflow):

1. **Initialization**: `main.tsx` renders `<App />`, which conditionally mounts `<DummyRepoPage />` at the module-configured path (see Section 3 for route conventions).
2. **Hook Execution**: `DummyRepoPage` calls `useRepoInternalData('repo-slug')` on mount.
3. **Static Data Import** (concrete, no runtime fetch):
   ```ts
   // useRepoInternalData.ts
   import repoMetadata from '../data/repo-slug.json'; // Vite resolves at build via import assert
   import commits from '../data/commits.json';
   ```
   Vite’s ES module loader resolves `.json` imports as raw objects during `tsc -b && vite build`. No `node-fetch` or `axios` is involved in the initial contract.
4. **State Reduction**: Hook reduces imported JSON into `RepoState`:
   ```ts
   interface RepoState {
     loading: boolean;
     error: string | null;
     metadata: RepoMetadata;
     commits: Commit[];
   }
   ```
5. **Rendering**: Components read state slices via React context or direct prop drilling. `RepoHeader` receives `metadata`; `CommitFeed` receives `commits`; `StatsBar` derives `stars`/`forks` from `metadata`.
6. **Re-render Triggers**: If the hook’s dependency array changes (e.g., manual reload button), the import re-executes and state updates propagate down the tree via React 19’s fine-grained reactivity.

**Note on Schedulers**: No `setInterval` or `setTimeout` schedulers are present in the base module. Any periodic refresh (e.g., polling for updated metadata) would be added as a `useEffect` dependency in consumer code, not baked into the module core.

---

## 3. API Contracts & Endpoint Specifications

The module does not expose or consume external HTTP APIs in the current onboarding. The "contract" is defined through **TypeScript interface contracts** and **static JSON data schemas** resolved at compile time by Vite.

### 3.1 Type Interfaces (the module’s API surface):

```ts
// src/modules/dummy-repo/types/repo-metadata.ts
export interface RepoMetadata {
  id: string;               // UUID v7 string, e.g. "a1b2c3d4-e5f6-7g8h-9i10-jk11kl12mn13"
  slug: string;             // Unique module identifier, e.g. "dummy-repo"
  name: string;             // Display name, e.g. "dummy-repo"
  description: string | null; // Optional, may be null
  stars: number;            // Non-negative integer
  forks: number;            // Non-negative integer
  lastUpdated: string;      // ISO 8601 date string, e.g. "2024-04-12T14:30:00Z"
}

// src/modules/dummy-repo/types/commit.ts
export interface Commit {
  sha: string;              // 40-char hex SHA, e.g. "c4e37a0bf211d56a3fee842e795a2ef14273c9e"
  message: string;          // Commit message body
  author: string;           // Name <email>
  date: string;             // ISO 8601 date string
}
```

### 3.2 Static Data Schema (the "endpoint" contract):

Data is sourced from **fixture JSON files** located at:
- `src/data/repo-slug.json` → resolves to `RepoMetadata`
- `src/data/commits.json` → resolves to `Commit[]`

**Example `repo-slug.json` (concrete payload, not a placeholder URL):**
```json
{
  "id": "a1b2c3d4-e5f6-7g8h-9i10-jk11kl12mn13",
  "slug": "dummy-repo",
  "name": "dummy-repo",
  "description": "A minimal React + TypeScript + Vite template for onboarding modules.",
  "stars": 0,
  "forks": 0,
  "lastUpdated": "2025-01-01T00:00:00Z"
}
```

**Load Pattern (concrete, no generic `/api/v1/...`):**
```ts
// In useRepoInternalData.ts
import type { RepoMetadata } from '../../types/repo-metadata';
import repoData from '../../data/repo-slug.json' with { type: 'json' };

export function useRepoInternalData(slug: string): RepoState {
  const [state, setState] = React.useState<RepoState>({
    loading: true,
    error: null,
    metadata: repoData as unknown as RepoMetadata,
    commits: [] as Commit[],
  });

  // Simulate async boundary for HMR consistency; actual data is sync due to Vite import.
  React.useEffect(() => {
    setState(prev => ({ ...prev, loading: false }));
  }, []);

  return state;
}
```

**External Integration Contract (planned, not yet implemented):**
Should a downstream module require network access, the contract will map to `https://api.github.com/repos/{owner}/{repo}` with the following request schema (documented for future reference, not active):
- **Method**: `GET`
- **Headers**: `Accept: application/vnd.github+json`, `Authorization: Bearer <TOKEN>`
- **Query**: `owner`, `repo`
- **Response Schema**: Maps to `RepoMetadata` ∪ `Commit[]` fields. This contract is *not* active in the current codebase and is noted for Phase 2.

---

## 4. Data Model, Database Schema & Entities

The module’s data model is **entity-driven, typed JSON-backed**, with no runtime database engine. "Persistence" is handled by Vite’s static asset serving and the browser’s memory during a session. The schema is designed for eventual migration to IndexedDB or a remote backend, but the current entity definitions are pure TypeScript interfaces serialized as JSON fixtures.

### 4.1 Entity Definitions (Concrete Column Names & Types):

#### Entity: `Repository` (mapped to `RepoMetadata`)

| Column Name | Data Type | Constraints | Description |
|-------------|-----------|-------------|-------------|
| `id` | `string` (UUID v7 format) | **Primary Key**; unique, non-null | Global identifier for the repository entity. |
| `slug` | `string` | **Unique Index**; non-null, regex `^[a-z0-9]+(-[a-z0-9]+)*$` | Human-readable, URL-safe identifier. Enforced via TypeScript exhaustive type guards at compile time. |
| `name` | `string` | Non-null | Display name of the repository. |
| `description` | `string \| null` | Nullable | Optional long-form description. |
| `stars` | `number` | `>= 0`; integer | Popularity metric. |
| `forks` | `number` | `>= 0`; integer | Derivative count. |
| `lastUpdated` | `string` (ISO 8601) | Non-null, valid date format | Timestamp of last metadata refresh. Indexed via `tsc`-timechecked string-matching predicates. |

**Index Constraints (compile-time, not runtime DB):**
- `slug` uniqueness is enforced by the TypeScript type system: any consumer must match `slug` against a `readonly` union of known valid slugs (e.g., ``'dummy-repo'`).
- `lastUpdated` comparisons use `Date` constructor parsing within `useRepoInternalData` to trigger refresh logic; invalid formats cause a type-error at the call site.

#### Entity: `Commit`

| Column Name | Data Type | Constraints | Description |
|-------------|-----------|-------------|-------------|
| `sha` | `string` (40-char hex) | **Primary Key**; non-null, matches `/^[0-9a-f]{40}$/` | Unique commit identifier. |
| `message` | `string` | Non-null | Commit body; may contain newlines. |
| `author` | `string` | Non-null | Format: `"Name <email>"`. |
| `date` | `string` (ISO 8601) | Non-null | Commit timestamp. |

### 4.2 Data Flow & Storage:

- **At Rest**: JSON fixture files under `src/data/`. Vite's `import ... with { type: 'json' }` compiles these into runtime `object` values.
- **In Memory**: `RepoState` holds the entity graph for the current session. No localStorage or IndexedDB writes are performed in the base module (kept intentionally stateless for onboarding simplicity).
- **Migration Path**: Should persistence be required, the entity schemas above map directly to an IndexedDB `objectStore` keyPath (`slug` for `Repository`, `sha` for `Commit`), with `idb-keyval` as the suggested wrapper. The column names and types remain unchanged, ensuring zero-drift migration.

**Explicitly Avoided**: No `SQL` table definitions, no `NoSQL` document-store contracts. The module adheres to a **typed JSON fixture** pattern compatible with both browser-native `import` and future persistence layers.

---

## 5. Integrations & External Dependencies

The module’s dependency graph is strictly limited to the `test-project` root `package.json` and the React ecosystem. No third-party APIs, databases, or native bindings are linked.

### 5.1 Runtime Dependencies (from `package.json`):

| Package | Version | Purpose |
|---------|---------|---------|
| `react` | `^19.2.8` | UI rendering layer; concurrent features, automatic batching. |
| `react-dom` | `^19.2.8` | DOM binding for React 19. |
| `typescript` | `~6.0.2` | Type-checking, strict-mode enforcement, `tsc -b` build pipeline. |
| `vite` | `^8.3.0` | Build tool, dev server, HMR, ES module resolution for `.json` imports. |
| `@vitejs/plugin-react` | `^6.1.0` | React plugin using Oxc compiler for fast transformation. |

### 5.2 Development & Linting Dependencies:

| Package | Version | Purpose |
|---------|---------|---------|
| `@types/node` | `^24.13.3` | Node.js type support for Vite config scripts. |
| `@types/react` | `^19.2.18` | React global type declarations. |
| `@types/react-dom` | `^19.2.7` | React DOM type declarations. |
| `eslint` | `^10.10.0` | Linting engine. |
| `eslint-plugin-react-hooks` | `^7.1.1` | Enforces `useState`/`useEffect` rule compliance. |
| `eslint-plugin-react-refresh` | `^0.5.6` | Fast refresh support for React components. |
| `globals` | `^17.12.0` | Global variable typings for ES module context. |
| `typescript-eslint` | `^8.69.0` | ESLint integration for TypeScript AST analysis. |

### 5.3 Build Scripts (Executable Commands):

- `dev`: `vite` — starts local dev server with HMR at `localhost:5173`.
- `build`: `tsc -b && vite build` — sequential type-check (`tsc -b`) then production bundling (`vite build`). Outputs to `dist/`.
- `lint`: `eslint .` — runs the full ESLint pipeline with the configured presets.
- `preview`: `vite preview` — serves the `dist/` bundle for manual QA.

### 5.4 Absent Integrations (by design):

- **No HTTP client libraries**: `node-fetch`, `axios`, or `got` are absent; data is sourced via Vite-resolved static imports.
- **No routing library**: React Router is not installed; routing is handled by the host application’s Vite `resolve`/`base` config or simple pathname matching in `main.tsx`. The module exports a `path` constant for the host to consume:
  ```ts
  // src/modules/dummy-repo/constants.ts
  export const ROUTE_PATH = '/dummy-repo';
  ```
- **No state-management library**: Zustand, Redux, or Jotai are not included. Module-internal state is confined to React component state and the `useRepoInternalData` hook.
- **No external APIs**: As documented in Section 3, no network calls are issued. Any future API integration will be opt-in and defined behind a feature flag.

### 5.5 Engineering Change Log Anchoring:

This design document extends the **"Initial repository onboarding for module: dummy-repo"** change log entry. All component names, type interfaces, and data schemas are newly introduced to support the module's production-grade grounding within the existing `test-project` scaffolding.