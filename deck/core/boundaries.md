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

  return div(
    button('Send').on({ click: () => fire('send-text', this._input.value) }),
  ).classes('controls');
});
```

The composition listens and translates:

```javascript
export default ({ engine, controlBar }) => alias(state =>
  div(
    controlBar(state).on({
      'send-text': e => engine.dispatch(new SendText(e.detail)),
    }),
  ).classes('app'));
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
  div(
    div().classes('progress-fill').style({ width: `${pct(fraction)}%` }),
  ).classes('progress');
```

## What a component may be given

"Components get nothing but data" is too strong. A component genuinely needs
things that are not data and not business logic: the dodo API itself, a theme,
sub-components, a formatter. The engine is the wrong vehicle for these — a
Provider handing out `dodo` is a category error, and it would put the render API
behind an async `obtain`.

Two mechanisms, both correct.

**Module import of one shared instance.** The default, and what every app here
does for dodo: a single `runtime.js` or `shared/stack.js` exports the configured
instance, and components import it.

```javascript
import { dd } from '../../runtime.js';
const { div, span, button } = dd;
```

This is not coupling to the app; it is coupling to the stack. It is also what
guarantees every vnode reconciles through one dodo instance.

**Factory injection at the composition root.** For anything an app might need
more than one of — a theme, a swappable sub-component, a registry of openers a
particular build ships.

```javascript
export default ({ theme, rowMenu }) => alias(function (item) { /* … */ });
```

The dependency is fixed once, where the app is assembled, and is visible in one
place.

The line is **what** is injected, not whether injection happens: it must be a UI
capability. The moment the injected object can reach the engine, a business
service, or a registry of application behaviour, it is not a UI capability — it
is the engine with extra steps.

The test that settles it: **could this component render in a component test with
no engine?** A theme and a dodo instance keep that true. A handle with `go` and
`platform` on it does not.

### Timing matters as much as content

Factory injection happens **once**, at composition. Passing a handle as a
per-render argument to every component means every call site must already hold
it, so it has to be threaded to every level of the tree — and an object that must
reach everywhere inevitably accumulates everything, because adding a field to it
is free and adding a parameter is not.

That is the difference between `({ theme }) => alias(…)` and
`function launcher(state, ui)`. The first names its dependency and fixes it; the
second is a channel, and a channel with `engine` on it is not a UI capability.

## Two deviations, both debt

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

**A context handle that dispatches.** A `ui` object carrying `go(action)`,
`exec(commandId)`, `rerender()` and direct references to `engine`, `app` and
`platform`, passed to every component as `component(state, ui)`.

Note precisely what makes this the wrong side of the line above, because it is
two separate failures and only one of them is about injection:

- **It is passed per render, not at composition.** The signature is
  `export default function activityBar(state, ui)`, re-supplied on every call.
  Had it been `({ theme, icons }) => alias(…)`, fixed once at the root, the
  threading argument would hold and the object would have stayed small.
- **What it carries is not a UI capability.** `engine`, `app` and `platform` are
  on it. No amount of getting the timing right would fix that.

And it is tempting to defend the whole thing as a pragmatic answer to
prop-threading in a deep tree. It is not. **The handle is a symptom; the disease
is a second architecture sitting above the engine.**

Look at what it reaches. In the app where this pattern grew, `platform/` is
roughly five thousand lines of services: a command registry with an `execute`,
a plugin host owning iframes and message ports, a contribution registry,
keybindings, settings, context keys, navigation, overlays. Eleven of them
hand-roll their own `observe()`, and the same `subject`/`observe`/`set` plumbing
is copy-pasted nine times.

Every one of those is something ngin already has a name for:

| Built in `platform/` | Should be |
| --- | --- |
| `CommandService` — a `handlers` Map plus `execute(id, …)` | a registry **Provider**, plus an `ExecuteCommand` **Action** |
| `PluginHost` — iframes, ports, install/uninstall/reconcile | a **Provider** (`obtain`/`release`/`dispose`) plus Actions |
| contribution / keybinding registries | **Providers** |
| settings, context keys, navigation, overlay state | **Queries**, with Actions to change them |
| nine copies of `subject` + `observe` + `set` | a Query. That is what a Query is. |

Once the command registry lives outside the engine, `engine.dispatch` cannot
reach it — so something has to carry `platform` down to the components, and that
something is the handle. The threading problem it solves was created by moving
this logic out of the engine in the first place.

Move it back and the handle has nothing left to carry. `ui.exec(id)` becomes
`engine.dispatch(new ExecuteCommand(id))` in a composition. `ui.go` was only ever
`engine.dispatch` with a shorter name. `ui.rerender()` — the manual repaint for
state that never entered a reactive graph — stops being expressible, which is the
point. Components go back to `(state)` plus bubbling events.

The failure this predicts is not hypothetical. Search state ended up with three
owners, one of them a component writing a service's `set()` directly and forcing
a manual `rerender()`. It took an architecture audit to catch, and it has since
been pulled back behind actions — by convention, not by construction, so nothing
stops it recurring.

The rule worth extracting: **anything with a registry, a lifetime, or an
`observe()` belongs in the engine.** Pure helpers — an expression parser, a
formatter, a URL builder — can live wherever they are used. Everything else is a
Provider, an Action or a Query, and if it is none of those it should not be
holding state that the UI reads.
