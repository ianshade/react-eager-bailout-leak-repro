# react-eager-bailout-leak-repro

Minimal repro: a `useState` setter called repeatedly with a value that never
actually changes leaks memory, even while other components in the tree keep
re-rendering normally.

Open `react-eager-bailout-leak-repro.html` in a browser (loads React 19 from
esm.sh via an import map).

## Why

`dispatch()` always appends an `Update` node to the hook's `queue.pending`
linked list, unconditionally. Separately, if the fiber has no pending work,
React eagerly computes the next state and skips scheduling a re-render
entirely if it's unchanged (`dispatchSetStateInternal`,
[ReactFiberHooks.js:3565-3577](https://github.com/facebook/react/blob/v19.0.0/packages/react-reconciler/src/ReactFiberHooks.js#L3565-L3577)).

That bailout still routes the update through
`enqueueConcurrentHookUpdateAndEagerlyBailout`
([ReactFiberConcurrentUpdates.js:126-150](https://github.com/facebook/react/blob/v19.0.0/packages/react-reconciler/src/ReactFiberConcurrentUpdates.js#L126-L150)),
whose own comment says it plainly:

> the update we just queued will leak until something else happens to
> schedule work (if ever)

The queue is only drained when the fiber actually renders
(`updateReducerImpl`, [ReactFiberHooks.js:1337](https://github.com/facebook/react/blob/v19.0.0/packages/react-reconciler/src/ReactFiberHooks.js#L1337)).
If nothing else forces that (e.g. a `shouldComponentUpdate`/`memo` boundary
upstream blocks incidental re-renders), the list grows forever, regardless
of how much unrelated rendering happens elsewhere in the app.

Same mechanism in React 18. Only fixed for `useReducer`
([PR #22445](https://github.com/facebook/react/pull/22445)), not `useState`.

## Related upstream reports

- [facebook/react#21706](https://github.com/facebook/react/issues/21706):
  `setState(s => unchanged)` retains the updater closure. Closed
  stale/unconfirmed.
- [facebook/react#21692](https://github.com/facebook/react/issues/21692):
  same pattern for `useReducer`. Closed stale/unconfirmed.
- [facebook/react#33998](https://github.com/facebook/react/pull/33998): a
  2026 fix attempt targeting this exact mechanism, referencing both issues
  above plus #28981. Auto-closed stale, never merged.

## Workaround

Don't call the setter at all when the value hasn't changed. Compare before
dispatching, not inside a functional updater (the updater form still
enqueues and still hits the same bailout).
