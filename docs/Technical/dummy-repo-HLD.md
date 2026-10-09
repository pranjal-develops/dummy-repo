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
- No backend logic, no runtime database, and no external API calls for repo data fetching in the initial onboarding phase. All repo data is sourced via typed static JSON fixtures or internal state. Additionally, PR #2 introduces a centralized API client module (`src/api.ts`) for backend module-operation requests, which coexists as an extensibility layer.
- Designed for incremental extension: future phases may plug in a REST/GraphQL backend, but the current contract is entirely frontend-local.

**Primary Responsibility Boundary:**
- The module owns the component tree from `DummyRepoPage` down to leaf atoms (`RepoHeader`, `CommitListItem`, `StatsBar`).
- Data fetching, normalization, and error handling are orchestrated by the custom hook `useRepoInternalData`.
- Rendering pipeline: `main.tsx` → `App.tsx` → `<DummyRepoModule />` → component hierarchy → DOM.

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

## 3. API Contracts & Endpoint Specifications

The module does not expose or consume external HTTP APIs for repo data in the current onboarding. The "contract" is defined through **TypeScript interface contracts** and **static JSON data schemas** resolved at compile time by Vite.

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
  author: string