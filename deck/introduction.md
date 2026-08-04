# The 3sln Stack

A set of small libraries, and a way of using them, for building applications
that stay legible as they grow.

## The libraries

- **[ngin](https://ngin.3sln.com)** — dependency injection, actions and queries.
  Three independent layers (`@3sln/ngin/providers`, `/actions`, `/queries`), so
  you can take only the part you need.
- **[dodo](https://dodo.3sln.com)** — a functional VDOM, with reactivity,
  context, observation, animation and scoped styling as optional modules the
  core neither imports nor knows about.
- **[bab](https://github.com/3sln/bab)** — localization. The source text is the
  key, a translation is a `String`, and lookup is a pluggable catalogue.
- **[js-tools](https://github.com/3sln/js-tools)** — the builder and development
  server the client applications share. Unbundled ES modules, content-addressed,
  resolved by an import map.
- **[deck](https://deck.3sln.com)** — the documentation and playground tool this
  deck is built with.

ngin runs anywhere JavaScript does. dodo runs where there is a DOM. That
asymmetry is the shape of the whole stack: **every application has a provider
graph and a domain; only some of them have a screen.**

## How this deck is organised

The stack does not prescribe one application. It prescribes a small number of
ideas that hold in all of them, and then a handful of recognisable **shapes**
those ideas take.

- **[Ethos](/?c=%2Fethos.md)** — the principles the rest of the deck is
  downstream of. Read this one first.
- **Core** — concepts that apply to every application, with or without a screen:
  [providers](/?c=%2Fcore%2Fproviders.md),
  [actions and queries](/?c=%2Fcore%2Factions-and-queries.md),
  [feeds](/?c=%2Fcore%2Ffeeds.md),
  [boundaries](/?c=%2Fcore%2Fboundaries.md) and
  [testing](/?c=%2Fcore%2Ftesting.md).
- **Shapes** — how those concepts land in a particular kind of application:
  [the single-page app](/?c=%2Fshapes%2Fspa.md),
  [the service](/?c=%2Fshapes%2Fservice.md),
  [the headless process](/?c=%2Fshapes%2Fheadless.md), and an
  [overview](/?c=%2Fshapes%2Foverview.md) of how to tell which one you are
  writing.
- **[Delivery](/?c=%2Fdelivery.md)** — how a client application actually
  reaches a browser: what is bundled, what is not, and the two outages that
  decided it.
- **[Conventions](/?c=%2Fconventions.md)** and
  **[Structure](/?c=%2Fstructure.md)** — naming, file layout, and the
  anti-patterns worth naming out loud.

If you are looking for the API of a specific function, it is in that library's
own deck, not here. This deck is about how the pieces are put together.

## The one-paragraph version

Resources — a database, a socket, a microphone, a child process — are owned by
**Providers**, which declare what they depend on and hand back a resource with a
lifetime the container manages. Logic lives in **Actions** (do a thing) and
**Queries** (observe a thing), which declare the resources they need and receive
them already obtained. Everything at the edge — a UI component, an HTTP route, a
line of terminal output — is dumb: it renders what it is given and signals
intent. A single **composition root** knows the concrete world and wires the
three together. Nothing below the composition root knows what platform it is
running on.
