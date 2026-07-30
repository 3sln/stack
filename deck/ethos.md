# Ethos

Everything else in this deck is downstream of these. They are not style
preferences; each one is here because its absence caused a specific bug in a
specific project, and the fix generalised.

## 1. Declare what you need. Never reach for it.

A route that receives one eighteen-key context object is using a service
locator. Nothing records what it actually uses, so nothing stops it using more,
and the dependency graph exists only in the author's head.

```javascript
// Checkable. A test can assert this list; a reviewer can read it.
static deps = ['vfs', 'tasks', 'lifecycle'];
```

A declared list is data. It can be asserted on, walked, and refused when it
contains a cycle — by name, rather than by producing `undefined` three calls
later.

## 2. Lifetimes belong to the container, not to your discipline.

`try/finally` is a promise you make to yourself. The container's version is
checked by construction: it releases whether the action returns, throws, or is
aborted.

> The claim-release in the collection scan was a `finally` block, and it had
> already been written wrong once — releasing too early. As a provider, `obtain`
> takes the lease and `release` gives it back, and there is no third path.

The same argument retires two other hand-maintained lists: **build order** (each
provider names what it needs and the container walks it) and **teardown order**
(the container disposes in reverse construction order, so a resource is always
torn down before the things it was built from). Both used to be statements in
sequence that had to agree and silently didn't have to.

## 3. Make the grant the object, not a check you remembered to run.

If authorization is a step *before* the work, then the check and the use are
separate, and nothing carries the grant forward. A caller who asserted `read`
still holds an unrestricted handle and a raw id, and the destructive call is one
line away.

Instead, let the thing you obtain *be* the permission:

```javascript
// A read handle has no `remove` — not because calling it is checked,
// but because there is no `remove` to call.
const node = await access.node(id, 'read');
```

Because that resolution happens in `obtain`, denial happens during
`container.lease()` — **before `execute` runs at all**. An action cannot proceed
unauthorized because it cannot obtain its dependencies.

This generalises past authorization. Any rule you would otherwise have to
remember is better expressed as an object whose surface only permits the legal
moves.

## 4. One model of a rule, in one place.

Two implementations of the same rule agree while everything is wired correctly
and diverge exactly when it is not — and the more permissive one wins.

> An access provider decided capability implication locally: `write` implies
> `read`. The service that owned the ACL disagreed — only `admin` implies
> anything. So a `write` handle was handing out `read` to a principal who had
> never been granted it. Two models of one rule; the wrong one winning silently.

Ask the owner. Do not re-derive.

Related: derive state, never dual-write it. A context key written in two places
is a key that is wrong in one of them.

## 5. Prefer configuration to inference.

Decide from what you were told, not from what happens to be present. "Is a
service wired up?" and "should this deployment enforce ACLs?" give the same
answer right up until the moment they don't — and that moment is when
enforcement quietly stops.

## 6. The platform is a provider.

Domain code never imports `env`, the Cloudflare bindings, an AWS SDK, werift, or
`Bun.spawn`. Every external capability is an interface with at least two
implementations: the real one and the one used in tests.

This is what makes a codebase runnable in more than one place — Node, Bun,
Workers, a test harness — and it is not a refactor you can do later, because by
then the platform is in four hundred files.

## 7. Seams are events, not return values.

Work that has something to say *now* and something else to say *later* has a
feed, not a return value. The alternative is inventing a `{ task, alreadyRunning,
done }` tuple, and then a second polling mechanism bolted on beside it, and then
discovering both were describing the same thing badly.

## 8. The edge is dumb.

A UI component, an HTTP route, and a line of CLI output are the same role: they
render what they are given and signal intent. None of them decides anything. The
value of this rule is entirely in how boring it makes the outermost layer, which
is the layer that changes most often.

## 9. A migration you cannot verify should not be started.

> 409 tests passed before the rewrite and 409 passed after, which is the only
> useful definition of "this changed no behaviour".

Keep the old entry point as a facade so the existing tests drive the new code
unchanged. If you cannot arrange that, the refactor is not ready to begin.

## 10. Be honest about what a change cost.

Every one of these choices has a price, and the price belongs in the comment
next to the code:

> A dispatch costs a turn of the event loop. `dispatch()` returns synchronously
> and the action runs on a later macrotask, so the claim is now taken one turn
> later. Two requests in the same tick can both run, back to back, instead of
> the second being turned away. That is wasteful, not incorrect — the claim
> still guarantees they never overlap, which is the property that matters.

A comment that only lists benefits is marketing. Write down the boundary you
moved, so the next person knows what they are standing on.
