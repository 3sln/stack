# Boundaries

The stack has four roles. Every application has all four; only their names
change. Getting the *direction* right matters far more than getting the folder
names right.

## The four roles

| Role | Owns | Knows about |
| --- | --- | --- |
| **Resources** (Providers) | the outside world — a database, a socket, a mic, a child process | nothing above it |
| **Domain** (Actions, Queries, pure modules) | the rules | resource *interfaces* only |
| **Composition** | wiring | everything — this is the only place that may |
| **Surface** | rendering and signalling intent | what it was handed |

## The same four in three shapes

The insight that keeps this deck honest: **a UI component and an HTTP route are
the same role.** Both are the outermost layer, both are dumb, and both are the
layer that changes most often.

| Role | Single-page app | Service | Headless process |
| --- | --- | --- | --- |
| Resources | `providers/` — IndexedDB, audio, WebRTC, mic | `providers/` — storage, metadata, KV, identity | `providers/` — child process, socket, registry |
| Domain | `bl/` — Actions + Queries | `domain/` + engine Actions | `bl/` — a long-lived Query |
| Composition | `main.js` | `createServer` / `createContainer` | `run.js` |
| Surface | `ui/components/` | `routes/` | terminal output |

## The dependency rule

Arrows point one way: **Surface → Composition → Domain → Resources.** Nothing
points back up.

Concretely:

- Domain code never imports a UI module, a route, or a platform SDK.
- Surface code never imports a provider.
- `bl/` must not import from `ui/`. (When it does, it is usually a registration
  call that belongs in the composition root instead.)
- Only the composition root names concrete implementations.

The last one is the load-bearing rule. If `createContainer` is the only file
that mentions D1, R2, werift or `Bun.spawn`, the whole application is portable
and testable. If those names leak downward, neither is true any more and no
amount of interface design recovers it.

## Where the boundary usually breaks

Three failure modes, all observed in these projects, all worth naming:

**1. The service-locator context object.** One eighteen-key `ctx` passed to
every route. Technically the route "receives" its dependencies, but nothing
records which ones it uses and nothing stops it reaching for more. Declared
`deps` is the fix — see [Ethos §1](/?c=%2Fethos.md).

**2. The surface reaching around the domain.** A component that calls
`someService.set(...)` directly, or fetches and then hangs results off a shared
object with a manual re-render. The state then has three owners and the action
layer is decorative. The enabler is almost always a service exposing a public
`set()` — expose *intent* methods, not setters.

**3. Dual-written derived state.** A flag maintained by hand at every mutation
site, which one site forgets. Derive it from a single projection instead. If two
places can write it, one of them is wrong and you will not find out from a type.

## How the surface signals intent

Three sanctioned mechanisms. They differ in how much the component is allowed to
know, and the right choice depends on how reusable the component is meant to be.

**Native browser events, bubbling upward.** This is the mechanism. A component
announces what happened; whoever is listening decides what it means.

```javascript
// Must be a `function`: dodo invokes an alias builder as
// `builder.apply(hostElement, args)`, and an arrow function's `this` is
// fixed at definition, so apply cannot set it. (Arrows *nested inside*
// are fine — they inherit the `this` apply just bound.)
export default alias(function (state) {
  const fire = (type, detail) =>
    this.dispatchEvent(new CustomEvent(type, { bubbles: true, detail }));

  return div({ className: 'controls' },
    button('Send').on({ click: () => fire('send-text', this._input.value) }));
});
```

The composition listens and translates:

```javascript
export default ({ engine, controlBar }) => alias(state =>
  div({ className: 'app' },
    controlBar(state).on({
      'send-text': e => engine.dispatch(new SendText(e.detail)),
    })));
```

We use the platform's own upward channel because it already exists and already
works. Bubbling means an intermediate component does not have to know or forward
anything; the event crosses arbitrary depth without a single intervening module
mentioning it. Nothing needs threading, so no intermediate signature changes
when a leaf gains a new thing to say.

A purely presentational piece with nothing to announce needs none of this — it
is a function of data returning a vnode, and that is not an exception to the
rule so much as the case where the rule has nothing to do:

```javascript
export const progressBar = fraction =>
  div({ className: 'progress' },
    div({ className: 'progress-fill', $styling: { width: `${pct(fraction)}%` } }));
```

### Two deviations, both debt

Neither of these is a sanctioned alternative. Both exist in our projects and
both should be migrated toward events.

**Callbacks and acks passed down as arguments.** A component taking an
`onSave`-style function is a React habit, not a dodo one. It re-couples the
component to its caller's vocabulary, forces every intermediate level to accept
and forward a prop it does not care about, and gives up bubbling for nothing.
Where an app instead avoids the problem by inlining every interactive element
into its compositions, the components folder ends up holding only decoration
while the compositions grow to a thousand lines — the boundary has not been
respected, it has been evacuated.

**A context handle that dispatches.** A `ui` object carrying `go(action)` —
alongside `engine`, `platform` and a `rerender()` escape hatch — passed to every
component as `component(state, ui)`. It removes the threading problem, and in a
shell with deep trees and plugin-contributed components that problem is real:
otherwise every intermediate signature changes whenever a leaf gains something
new to say.

The cost is that *dispatch actions, never mutate state* becomes a convention
rather than a structural guarantee, because the engine and every service are
right there on the handle. In practice the convention slipped — search state
ended up with three owners, one of them a component writing a service's `set()`
directly and forcing a manual `rerender()` — and it took an architecture audit
to notice. It has since been pulled back behind actions. That is the argument in
miniature: the boundary held only as long as someone was watching it, which is
exactly what a boundary is supposed to make unnecessary.
