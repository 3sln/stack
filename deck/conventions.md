# Conventions

Small, boring, non-negotiable. These exist so that reading unfamiliar code in
this stack requires no orientation.

## Naming

- **UI factories are `lowerCamelCase`.** `saveButton`, never `SaveButton`. We
  are not in React; a capitalised identifier here signals a class, and a factory
  is not one.
- **Actions are imperative, Queries are nominal.** `AddTodo`, `ScanCollection`;
  `Todos`, `PlayerState`. If an action's name is a noun, it is probably doing
  two things.
- **Providers are named for the resource, not the technology.** `storage`, not
  `r2`; `db`, not `d1`. The whole point is that the concrete choice is invisible
  from the inside.

## Modules

- **UI factories use `export default`.** One factory per file.
- **Actions and Queries for a domain live together** in `bl/<domain>.js`. Do not
  split them into `actions/` and `queries/` folders — they change together and
  are read together.
- **Import dodo through the project's own shared module**, never from the
  package directly, so every vnode reconciles through one dodo instance.

```javascript
import { dodo, watch, cell } from '../shared/stack.js';   // ✅
import * as dodo from '@3sln/dodo';                        // ❌ in app code
```

- **Package specifiers are scoped**: `@3sln/ngin`, `@3sln/dodo`, `@3sln/bab`,
  `@3sln/js-tools`. Bare `ngin` is wrong wherever it appears, and so is a deep
  import past a package's `exports` — the build gives every exported subpath its
  own entry point, and reaching past them hands out a second copy of the package
  ([Delivery](/?c=%2Fdelivery.md)).

## Data

- **Timestamps are epoch milliseconds.** Everywhere, in every project, with no
  exceptions and no local `Date` objects in stored shapes.
- **Secrets are stored hashed.** Tokens and tickets as SHA-256; plaintext is
  returned exactly once, at creation.

## Comments

The house style is to write down **why**, and specifically what the alternative
cost. A comment that restates the code is noise; a comment that records the bug
this shape prevents is the most valuable thing in the file.

Where a decision has a price, the price goes in the comment next to it. See
[Ethos §11](/?c=%2Fethos.md).

## Anti-patterns

Named because each has actually happened.

| ❌ | Why |
| --- | --- |
| `PascalCase` UI factories | reads as a class; we are not React |
| `engine` in a component | the boundary is the whole design |
| `onClick`-style callback props | use `.on({ click })`, and [bubble events upward](/?c=%2Fcore%2Fboundaries.md) |
| Callbacks or acks passed down for upward signalling | native browser events already do this, across arbitrary depth, with nothing threaded |
| A `ui` handle that can dispatch | gives every component the engine it is not supposed to have — and is always a symptom of the row below |
| A per-render handle argument threaded to every component | inject UI capabilities into the **factory**, once, at composition; a channel accumulates |
| A `platform/` or `services/` folder beside `bl/` | state with a registry, a lifetime or an `observe()` belongs in the engine ([Ethos §7](/?c=%2Fethos.md)) |
| A hand-rolled `subject`/`observe`/`set` | that is a Query, without the teardown |
| A manual `rerender()` | state that never entered the reactive graph |
| Command dispatch or plugin lifecycle in the UI layer | a registry Provider plus an Action; `engine.dispatch` must be able to reach them |
| Treating an injected provider as the resource | providers get *providers*; consumers get *resources* |
| Actions dispatching Actions | there is no engine in `deps` on purpose; use `engineFeed` |
| `await engine.dispatch(...)` | returns a feed, not a promise — resolves immediately. Use `.next('complete')` |
| A one-eighteen-key `ctx` object | a service locator wearing dependency injection's clothes |
| Splitting `bl/` into `actions/` + `queries/` | they change together |
| Named exports for UI factories | `export default`, one per file |
| `.style({...})` on a vnode | there is no such helper; use `$styling` |
| A bare hyphenated prop (`'aria-label'`) | bare props are assigned as properties; use `$attrs` |
| `bl/` importing from `ui/` | wrong direction; the registration belongs in the composition root |
| Reaching past a façade into its internals | the invariant has one home or it has none |
| Dual-writing derived state | derive it from one projection; two writers means one is wrong |
| Inferring config from whether a service is present | decide from configuration ([Ethos §5](/?c=%2Fethos.md)) |
| Swallowing a mirror write with `.catch(() => {})` | two stores diverge invisibly |
| `npm` / `npx` in a Bun project | `bun` / `bunx`; a `packageManager` pin guards it |
| `?v=` on an asset URL | a cache key nothing downstream can reason about; the filename carries the hash ([Delivery](/?c=%2Fdelivery.md)) |
| `immutable` on a stable-named file | the browser stops re-checking it and keeps importing a build that no longer exists |
| Two `Cache-Control` rules matching one path | Cloudflare appends and the strictest wins, so `no-cache` quietly beats the caching you wanted |
| A hand-maintained import map in the repository | it is build output; a checked-in one drifts the moment a dependency moves |
| A deep import into a dependency's internals | a second copy of the package, and the singleton it was is not one any more |
