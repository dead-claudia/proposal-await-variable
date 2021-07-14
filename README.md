# Implicitly awaited variables

A simple `await` modifier to variable declarations to make efficient `async` functions far simpler to write in practice.

## Example usage

```js
// Fetch thing A
await let resultA = fetch("thing A")
await let a = resultA.json()

// Fetch thing B - done in parallel with `resultA` because it doesn't depend on it!
await let resultB = fetch("thing B")
await let b = resultB.json()

// If an error occurs, it stops here, not above.
console.log([a, b])
```

This could be shortened to this and still work equivalently:

```js
await let a = (await fetch("thing A")).json()
await let b = (await fetch("thing B")).json()
console.log([a, b])
```

## Concept

```js
// Source
await const a = expr1
await let b = expr2
independentFunction()
foo(a)
for (const item of coll) bar(item, b)
baz(a)
test(c)

// Naive transpilation to `async`/`await` (roughly)
let _pendingValue$ = {}
let a = _pendingValue$
let b = _pendingValue$
let c = _pendingValue$
const a$promise$ = Promise.resolve(expr1)
const b$promise$ = Promise.resolve(expr2)
const c$promise$ = Promise.resolve(expr3)
independentFunction()
foo(a === _pendingValue$ ? (a = await a$promise$) : a)
for (const item of coll) bar(item, b === _pendingValue$ ? (b = await a$promise$) : b)
baz(a === _pendingValue$ ? (a = await a$promise$) : a)
test(c === _pendingValue$ ? (c = await c$promise$) : c)
```

The idea is that you specify the variable as async, and it gets implicitly awaited before first use. The basic production is `await var a = b` (or similar with `let` or `const`).

- Uses are implicitly awaited, and while it might not be immediately clear the order, the point of this is when order *doesn't* matter beyond promise variable dependencies. (I anticipate this would largely become the default for many use cases, as it basically means you don't hardly ever need `Promise.all` anymore.)
- Synchronous errors in awaited variables' initializers always translate to rejections, for consistency. They're still stored, but the error is just deferred until initial use.
- Assignments change the promise that's currently awaited, and immediate values assigned to `await` variables are just used as-is after failing the initial `then` access. The existing promise is subsequently ignored after assignment.
  - A future cancellation proposal should issue a cancellation whenever this replacement occurs, if the promise hasn't yet resolved (but only if).
  - I'm extremely not tied to this semantic - I just want *a* behavior here. Anything is better than nothing, and this is just a start.
- Awaited variables are implicitly awaited, with errors thrown as applicable, before being placed in child function contexts, regardless of type. This ensures the looseness in timing is purely restricted to the function they're declared in.
  - I'm also open to banning such references outright, but that would also risk confusion as users would be wondering why they can't just use what looks like an ordinary variable reference. There's already enough magic, and I don't want to add even more apparent magic than what already exists.
  - Allowing async references to propagate in non-resolved state to child `async` functions and similar is a *bad* idea: it's not clear why `Promise.all(users.map(x => doSomething(x, asyncVar)))`  shouldn't be functionally equivalent to `Promise.all(users.map(async x => doSomething(x, asyncVar)))` just because `asyncVar` was defined differently, and I don't expect anyone would intuitively think otherwise absent deep knowledge of awaited variables' semantics.
  - Note: this does not need to extend to inner `async do` blocks as specified in https://github.com/tc39/proposal-async-do-expressions - those can just carry on normally, as it isn't spiritually that different and isn't an entirely new context.
- You can't `yield` inside awaited variables, but you may `await`. In this case, the `await` only blocks the completion of that expression, not subsequent statements. It continues much like how `async do` works, just at the expression level.
- A future cancellation proposal should issue a cancellation to all pending promises whenever an exception propagates past their scope.
- Awaited variables not used by the end of the function simply continue. If they result in rejections, those will simply translate to unhandled rejections as applicable (this will become handy once `async do` expressions come along).

> In reality, this should incorporate a few basic optimizations as well:
> 1. It should do the same microtask optimization current `await` does
> 2. It should attempt to extract the variable out immediately on resolution and immediately resume execution from the current suspension point if the value resolved corresponds to the relevant name. So if it's waiting on `b` above, and `b`'s backing promise resolves, it should continue execution immediately in the next microtask, using that value. It shouldn't wait on `c`.
> 3. Implementation shouldn't generate suspensions for values that it's trivially determinable they can only be already fulfilled. For instance, the second `a` should just use the resolution value directly.

## Rationale

I'm seeing an outgrowth of a long series of what look to be extremely hacky proposals

- https://github.com/tc39/proposal-await.ops: `await.all` as basically syntax
  - I could see the mild utility for collections, but not for static cases. And plus, unlike that, this doesn't require an IC check for `Array.prototype[@@iterator]` just to optimize it correctly.
- https://github.com/ajvincent/proposal-await-dictionary: `Promise.all` with dictionary object support
  - This is useful in a few unique situations, but the use case becomes extremely niche with this proposal.
- Several other community proposals I've seen.
  - https://es.discourse.group/t/concurrent-async-and-normal-await-evaluation-blocks/827 has several distinct proposals (and I've linked it here in a response there).
  - https://es.discourse.group/t/asyncawait-a-function-declaration-keyword-to-have-callers-automatically-await-result/54
  - The use case listed for https://es.discourse.group/t/proposal-optional-await-ej-await-true-false/493 is entirely resolved by this
  - https://es.discourse.group/t/synchronous-exceptions-thrown-from-complex-expressions-create-abandoned-promises-solutions/663: All promises are executed, and rejections are simply handled as they come up.
    - See my above note regarding cancellation - there really isn't a mechanism for dealing with this cleanly without some way to cancel promise actions. But of course, this proposal makes the hole clearly defined and obvious, and allows it to be filled *after* the initial proposal.

This also replaces nearly every use for `async do` that isn't essentially a `do` expression that needs an inner `await` somewhere. The first example was pulled [from the `async do` proposal itself](https://github.com/tc39/proposal-async-do-expressions), and at the time of writing initially looked like this:

```js
Promise.all([
  async do {
    let result = await fetch("thing A");
    await result.json();
  },
  async do {
    let result = await fetch("thing B");
    await result.json();
  },
]).then(([a, b]) => console.log([a, b]));
```

This would pair well with it, too - here's the original example, rewritten to use both this proposal and that:

```js
// Fetch thing A
await let a = async do {
  let result = await fetch("thing A")
  await result.json()
}

// Fetch thing B - done in parallel with `resultA` because it doesn't depend on it!
await let b = async do {
  let result = await fetch("thing B")
  await result.json()
}

// If an error occurs, it stops here, not above.
console.log([a, b])
```
