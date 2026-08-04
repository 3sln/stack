# Structure

There is no single mandated tree. There is a **recurring set of names**, and
where they sit depends on the [shape](/?c=%2Fshapes%2Foverview.md) and on how
many shapes the repository holds.

## The names

Wherever they appear, they mean the same thing:

| Folder | Contains | May import |
| --- | --- | --- |
| `providers/` | Provider classes wrapping resources | other providers |
| `bl/` | Actions + Queries, one file per domain | provider *interfaces* |
| `domain/` | pure server-side logic over resources | provider interfaces |
| `ui/components/` | dumb dodo factories | dodo, other components |
| `ui/compositions/` | engine-aware wiring | `bl/`, components, dodo |
| `routes/` | `Request → Response` handlers | `domain/`, http helpers |
| `shared/` | code crossing a shape boundary: types, protocol, formatting | nothing platform-specific |

`ui/components` never imports `bl/`. `bl/` never imports `ui/`. Everything else
is arrangement.

One folder that should **not** appear: a `platform/` or `services/` directory of
stateful objects sitting beside `bl/` and consumed directly by the UI. Whatever
is in it — command dispatch, a plugin host, registries, settings — is provider,
action and query material that has drifted out of the engine, and the drift is
what forces a context handle into every component
([Boundaries](/?c=%2Fcore%2Fboundaries.md)). Genuinely stateless helpers do not
need a home of their own; put them in `lib/`.

## A single-app client

The classic layout, and the right default when the repository is one app:

```text
src/
├── bl/                  # todos.js, auth.js — Actions + Queries per domain
├── providers/           # browser capabilities
├── ui/
│   ├── components/      # pure factories, export default
│   └── compositions/    # inject the engine
├── shared/
│   └── stack.js         # the one dodo instance + the Cell seam
└── main.js              # the composition root
```

A small app may flatten `bl/` into `state/` with `actions.js`, `queries.js` and
`router.js` beside the engine. That is fine at that size; the split by domain
earns its place as domains appear.

## A service

The surface, the logic, and the graph are three different things:

```text
src/
├── engine/
│   ├── providers/       # core.js (the graph), access.js (authorization)
│   └── actions/         # long-running work with feeds
├── domain/              # pure logic per domain
├── providers/           # platform wrappers, where not yet in the graph
├── routes/              # one file per resource; dumb
├── adapters/            # node.js, bun.js, worker.js — runtime seams
├── lib/                 # http helpers, signing, html escaping
├── router.js
└── container.js         # the composition root: the only file that knows the platform
```

## Several shapes in one repository

Once there is more than one shape, each gets a package and the seam between them
gets an explicit one:

```text
packages/
├── core/                # runtime-agnostic domain + interfaces
├── server/              # the service (adapters/, engine/, routes/)
├── web/                 # the SPA
├── desktop/             # the headless process
├── protocol/            # the wire format both ends share
└── plugin-sdk/          # a published boundary
```

The rule that makes this work: **`protocol/` and `core/` may be imported by
anyone; nothing else crosses.** A provider never leaves its package.

An app with two clients against one backend puts the clients side by side and
shares only what is genuinely shared:

```text
src/
├── worker/              # the service
├── client/
│   ├── player/          # SPA — bl/, providers/, ui/
│   ├── console/         # SPA — bl/, providers/, ui/
│   └── shared/          # stack.js, api.js, formatting, icons
└── shared/              # constants used by both worker and clients
```

Note the two `shared/` folders and that they are different: one is shared
between clients, one crosses the client/server boundary. Only the second may be
imported by the worker.

## Where the build config goes

`jstools.config.js` at the repository root, beside `package.json` — it names
which directories ship, so it belongs where a reader looks first, not buried in
`scripts/`. The build script beside it stays small: it calls the builder and
emits whatever else the deployment needs ([Delivery](/?c=%2Fdelivery.md)).

The `include` list in it is a boundary, not a convenience. `src/worker` and
`src/server` are not client code, and publishing them to the asset tree puts the
whole domain layer on a public URL.

## Where the deck goes

A `deck/` subdirectory with its own `package.json`, documenting the project it
sits in. It is a sub-project, not part of the build.

## The test that matters

You can move any of these folders. What you cannot do is let an import point the
wrong way. If you are unsure whether a layout is acceptable, do not compare it
to the trees above — check the arrows in
[Boundaries](/?c=%2Fcore%2Fboundaries.md).
