# Delivery

How the code gets into the browser. This is a card about
[`@3sln/js-tools`](https://github.com/3sln/js-tools) — the builder and the
development server the stack's client applications share — and about the two
outages that shaped it.

Every rule here exists because its absence produced **a blank page with only the
background painted**, in production, with a green build.

## Nothing is bundled that does not have to be

Two kinds of code have two different shapes, so they get two treatments.

| | Project modules | Dependencies |
| --- | --- | --- |
| Shipped as | one file per module, as authored | bundled, split by entry point |
| Named | `playerApp.4f2c1ab9de.js` | `3sln_dodo-QK3PZ7.js` |
| Changed by | an edit to that one file | a version bump |

Project modules change one at a time. A bundle means every edit invalidates
every file, and the whole point of content addressing is that a change costs
what it actually changed.

Dependencies are the opposite: they change rarely, and they must not be
duplicated. **Splitting them is about identity, not size.** Bundling
`@3sln/dodo` and `@3sln/dodo/reactive` as two independent bundles gives each its
own private copy of dodo's internals — and a Cell created through one is then
not recognised by the other. The same argument applies to `@3sln/ngin`: two
copies means two provider registries. One entry point per subpath in the
package's `exports`, with what they share factored into a chunk, is what keeps a
package a singleton.

## No source is rewritten

Import statements stay exactly as authored. An **import map** does all the
redirection:

```json
{
  "imports": {
    "@3sln/dodo":              "/assets/vendor/3sln_dodo-QK3PZ7.js",
    "/assets/shared/stack.js": "/assets/shared/stack.4f2c1ab9de.js"
  }
}
```

The second key is the interesting one. A browser resolves a relative specifier
to a URL *before* it consults the map, so `../shared/stack.js` from a sibling
module has already become `/assets/shared/stack.js` by the time the map is asked
— and one key per module intercepts every neighbour import in the graph.

That is why the source tree is **mirrored** under the asset root rather than
hashed in place. One prefix covers the lot, which is also what lets a single
cache rule mark the whole tree immutable.

## Nothing is cache-busted

No `?v=`. Ever.

> `BUILD_ID` was never actually set, so every listener's service worker
> precached `?v=dev` once and pinned that build permanently — and the cache name
> it compared against never changed either, so its own sweep had nothing to
> sweep.

A query string is a cache key that nothing downstream can reason about. A
content hash in the filename is one that everything can: **a changed file is a
changed URL**. Nothing is ever overwritten, so nothing needs invalidating, and a
browser still holding a poisoned copy of an old URL simply never asks for it
again — it recovers on the next deploy without anyone clearing anything.

## `immutable` only where it is true

> The rule that caused the outage was `immutable` on the catch-all, covering
> mutable, unhashed module URLs.

A stable-named entry point marked `immutable` is one the browser will not
re-check. After a deploy it goes on importing modules from a build that no
longer exists — and because a failed import is a **link-time** failure, the
whole graph dies before a line of it runs.

Two rules, then, and they are not symmetric:

- the content-addressed asset tree: `max-age=31536000, immutable`
- everything with a stable name: `no-cache`

And on Cloudflare, a third thing to know: matching rules are **appended**, not
replaced, and the strictest value wins. A `Cache-Control` on a catch-all arrives
alongside the one on the asset root as `no-cache, …, immutable`, and `no-cache`
wins — silently throwing away the caching this whole arrangement exists to
enable. So exactly one rule may set `Cache-Control` for any given path, and the
catch-all carries security headers only.

## The page carries one tag

A page needs three things before it can run: the import map, the entry
stylesheet and the entry module. All three are content-addressed, so all three
change on most builds — which is a lot of coupling to hand to a hand-written
`index.html`, or to a worker rendering HTML per request.

So the build emits a **wire-up script** that installs all three, and the page
says:

```html
<script src="/@wireup/player.js"></script>
```

The build rewrites that `src` to the hashed URL. The dev server answers the same
URL with development paths in it. The page is byte-identical in the repository
and in production, and nothing downstream of it has to know what a hash is.

Three things about it that are load-bearing:

- **It is a classic script, and it must come before any module script.** An
  import map has to be installed before the first module load is triggered.
  Since the wire-up appends the entry module itself, everything after it is in
  order by construction.
- **A module script inserted into the DOM is not deferred.** Unlike one written
  in the markup, it runs the moment it has loaded — which can be before the body
  exists, intermittently, depending on the network. The wire-up starts the fetch
  immediately with `modulepreload` (which follows the import map, so it warms
  the whole unbundled graph) and appends the script at `DOMContentLoaded`.
- **An import map is inline script content as far as CSP is concerned**, even
  created through the DOM. The wire-up carries its own `nonce` across.

## Development is the same description, served live

The builder and the dev server read **one config**. Two code paths disagreeing
about which files are project modules, which packages are dependencies, or where
an entry point lives is precisely the class of bug you find after deploying.

What development does differently is only what the map says:

- **Project modules are served as they are.** That is what makes hot module
  replacement possible at all — the file the browser is running is the file on
  disk, so there is something to replace.
- **ES module dependencies are served straight out of `node_modules`**, so a
  stack trace names the real file and a linked package is edited and reloaded
  like any other source.
- **Anything that is not already ESM is converted by esbuild and cached** —
  together, with splitting, and with every directly-served ESM specifier marked
  external, so a converted package reaches its ESM dependencies through the map
  rather than inlining a second copy of them.
- **Bare specifiers still resolve through the import map**, not through a
  resolver plugin. Development is not the one place where something other than
  the map resolves the graph.

## The graph is checked before the build reports success

```
build failed: 1 specifier(s) in the shipped graph are not in the import map:
  jszip — imported by src/client/console/bl/lpfImport.js
```

> jszip states its browser entry as a *map of redirects* rather than a filename.
> A resolver that hands the map over as a path resolves nothing, the package
> disappears from the import map, and the build stays green. The failure was a
> blank page at runtime, in the one code path that lazily imported it.

Every bare specifier reachable from an entry has to be in the map. The walk is
done by esbuild rather than a regex, so dynamic imports and re-exports are
covered, and a miss names the file that imported it.

## What stays in the project

The builder is not a framework. It emits the client; a project's own build
script emits everything around it, with the build result in hand:

```javascript
const result = await build(config);
// result.buildId, .entries, .imports, .modules, .copied, .html
```

That is where a static site, a sitemap check, a service worker with the build id
stamped into its cache name, or a `_headers` file listing a project's own stable
names belongs. Storia's build script is thirty lines; donki's is a hundred,
almost all of it landing pages and crawler files.

## Which side of the line

| Concern | Whose |
| --- | --- |
| Content addressing, splitting, the import map, the wire-up | the builder |
| Which directories ship, and which are server-side | the config |
| HTML, service workers, sitemaps, redirects, headers content | the project |
| Serving anything that is not the app | the project's dev middleware |

The one that catches people out is the second row. `include: ['client',
'shared']` is not a tidiness setting: publishing a `worker/` or `server/`
directory to the asset tree puts the whole domain layer on a public URL.
