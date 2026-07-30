# Shapes

The [core cards](/?c=%2Fintroduction.md) describe what every 3sln application
has. This section describes the three recognisable forms they take.

A shape is not a framework or a template. It is a set of answers to four
questions that a whole class of application answers the same way.

## The four questions

1. **What is the composition root, and when does it run?** Once at startup, or
   once per request?
2. **What is the surface?** A DOM tree, a `Response`, a stream of terminal
   output.
3. **How long does a resource live?** The process, a request, a connection.
4. **Are there subscribers?** This decides whether Queries earn their keep.

## The three shapes

| | [Single-page app](/?c=%2Fshapes%2Fspa.md) | [Service](/?c=%2Fshapes%2Fservice.md) | [Headless](/?c=%2Fshapes%2Fheadless.md) |
| --- | --- | --- | --- |
| Composition root | `main.js`, once | `createServer` once; a lease per request | `run.js`, once |
| Surface | dodo components | routes returning `Response` | terminal / a socket |
| Resource lifetime | the tab | process (singletons) + request (leases) | the process |
| Queries | everywhere | rarely — one-shot reads are not subscriptions | one, and it *is* the process |
| dodo | yes | no | no |

## Telling them apart

The mistake worth avoiding is assuming an application is one shape because the
last one was. Most real products are **two or three shapes in one repository**,
and the seam between them is a deliberate design decision rather than an
accident of deployment.

A worked example: an audiobook host is a **service** (a Worker serving an API
and generating HTML) plus two **single-page apps** (a player and a publishing
console) that ship as separate bundles into pages the service renders. A
self-hostable drive is a **service** (runtime-agnostic, `Request → Response`)
plus one large **SPA** workbench. An agent tether is a **headless process** (a
desktop daemon), a **service** (signalling), and an **SPA** (the phone PWA) —
three shapes, three packages, one protocol package shared between them.

What travels across those seams is data and protocol modules. What must never
travel is a provider or a platform import.

## Not every page is an SPA

A page rendered by the server with a small bundle attached is a perfectly good
3sln application, and often the right one. The service generates the HTML,
inlines a bootstrap payload, and the client picks it up:

```javascript
const bootEl = document.getElementById('storia-bootstrap');
const boot = bootEl ? JSON.parse(bootEl.textContent || '{}') : {};
```

That client is still an engine, still has providers, still dispatches actions —
it simply does not own routing or the first paint. Everything in the
[SPA card](/?c=%2Fshapes%2Fspa.md) applies except the parts about owning the
whole document.

## What is shared no matter the shape

- Providers wrap every platform capability, with a mock alongside the real one.
- Dependencies are declared, never located.
- The composition root is the only file that knows concrete implementations.
- The surface is dumb.

If a card in this section appears to contradict one of those, the card is wrong.
