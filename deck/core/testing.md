# Testing

The stack's testing story is not a framework choice. It is that **injection is
the seam**, and if the boundaries hold, everything is substitutable without a
mocking library.

## The property to protect

```javascript
createServer({ storage: new MemoryStorage() })
```

Every provider accepts an injected instance *or* a driver configuration. That
duality is what the entire suite rests on. Preserve it in every new provider you
write; the moment one of them can only build its own resource, a whole class of
test stops being possible.

Where credentials are absent, fall back to a mock automatically:

```javascript
const email = config.get('RELAY_API_KEY')
  ? new RelayEmailProvider({ ... })
  : new MockEmailProvider();
```

The payoff is that the full flow — payments, email, media processing — runs
end-to-end locally and in CI with no external services and no test-only branch
inside the domain.

## Test at the layer that has a contract

| Level | Substitutes | Answers |
| --- | --- | --- |
| Unit (domain) | every resource | does the rule hold? |
| Component | the `api` provider | does the real UI render and dispatch correctly? |
| Integration (service) | storage / metadata / identity | does the request produce the right response? |
| End-to-end | nothing | does it actually work? |

**Component tests are worth calling out.** Render the *real* UI factories in a
real browser against a mocked ngin provider — no server, no network. This works
precisely because components are dumb and compositions take an engine: you can
hand them a different one. If a component test needs a running backend,
[a boundary is broken](/?c=%2Fcore%2Fboundaries.md), and the test is telling you
which.

## Testing a refactor

The only useful definition of "this changed no behaviour" is that the tests
which did not change still pass.

> 409 tests passed before the rewrite and 409 passed after.

So keep the old entry point as a facade over the new internals. `createServer`
still builds a server; `beginScan` keeps the signature it always had. Every
existing test drives the public surface exactly as before, and the rewrite
underneath is checkable rather than asserted.

If you cannot arrange that, the refactor is not ready to start.

## Two hazards

**Timing assumptions.** Moving work behind a dispatch moves it a turn of the
event loop later ([Feeds](/?c=%2Fcore%2Ffeeds.md)). Tests that asserted the old
synchronous ordering will fail, and the fix is usually to make the test's
backend genuinely block so the overlap being tested is real — not to loosen the
assertion.

**Shared state between tests.** Where the test pool cannot isolate storage
between cases, tests stay independent by construction: unique emails, unique
ids, per-test collections. Keep doing that in new tests; a suite that only
passes in order will eventually only pass on one machine.
