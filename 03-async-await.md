# async/await: Async Code That Reads Like English

In [part one](#) we covered the event loop — the engine that makes async JavaScript possible. In [part two](#) we saw how Promises replaced callback hell with flat, chainable code. Now we're at the modern way: `async/await`.

This is the syntax that most JavaScript developers use today. It's built into React hooks, Next.js server components, and virtually every modern codebase. Understanding it deeply — not just copying the pattern — is what separates developers who debug async issues quickly from those who spend hours confused.

---

## What async/await Actually Is

First, the most important thing to understand:

**`async/await` is not a new async mechanism. It is syntax built on top of Promises.**

Under the hood, it's Promises. The engine is identical. The only difference is how the code looks. This matters because any confusion you have about Promises directly carries into `async/await`.

Two keywords:

**`async`** — marks a function as asynchronous. An `async` function always returns a Promise, even if you return a plain value.

**`await`** — pauses execution of the current function until the Promise resolves, then gives you the resolved value. Can only be used inside an `async` function.

```javascript
// Promise version
function loadUser() {
  return fetchUser(1)
    .then(user => fetchPosts(user.id))
    .then(posts => posts);
}

// async/await version — same behaviour, different look
async function loadUser() {
  const user  = await fetchUser(1);
  const posts = await fetchPosts(user.id);
  return posts;
}
```

Both do exactly the same thing. The `async/await` version reads like synchronous code — top to bottom, one line at a time.

---

## await Pauses the Function, Not the Thread

This is critical to understand correctly.

When you write `await somePromise`, JavaScript pauses *that function* and returns control to the event loop. The main thread is completely free. Other code can run, the browser can respond to clicks, animations continue — everything else keeps working.

```javascript
async function loadDashboard() {
  console.log('Starting fetch...');

  const user = await fetchUser(1); // function pauses here
  // everything else in the app keeps running during this pause

  console.log('Got user:', user); // resumes here when Promise resolves
}

loadDashboard();
console.log('This runs immediately'); // runs while loadDashboard is paused
```

Output:
```
Starting fetch...
This runs immediately
Got user: { id: 1, name: 'Steve' }  ← appears 1 second later
```

`await` is not `sleep`. It's a suspension point — the function yields, the thread stays busy with other things.

---

## Error Handling with try/catch

With `.then()` chaining you used `.catch()`. With `async/await` you use the standard `try/catch` — and it works correctly here because the `await` keyword unwraps the Promise result back into synchronous-style code.

```javascript
async function loadDashboard(userId) {
  try {
    const user  = await fetchUser(userId);
    const posts = await fetchPosts(user.id);
    renderUser(user);
    renderPosts(posts);

  } catch (error) {
    // any rejected await in the try block lands here
    showError(error.message);

  } finally {
    hideLoading(); // always runs — success or failure
  }
}
```

Three blocks, three jobs:

- **`try`** — the happy path. What you want to happen.
- **`catch`** — anything fails anywhere in `try`, execution jumps here. One handler for everything — bad network, 404 response, a bug in `renderUser`.
- **`finally`** — runs regardless of outcome. This is critical for cleanup. Without it, if an error occurs, `hideLoading()` never runs and the spinner spins forever.

In the [Crispy Async](https://github.com/emuiga/crispy-async) demo, click the ☠ Trigger Error button. It tries to fetch a user that doesn't exist. The fetch "succeeds" (network-wise), but `response.ok` is false, so we throw manually. The `catch` block shows the error message. `finally` hides the spinner. The entire flow works correctly because of this structure.

---

## Sequential vs Parallel — The Performance Trap

This is where most developers write subtly slow code without realising it.

```javascript
// Sequential — each waits for the one before it
async function loadDashboard(userId) {
  const user     = await fetchUser(userId);    // 1 second
  const posts    = await fetchPosts(userId);   // 1 more second
  const settings = await fetchSettings(userId); // 1 more second
  // Total: ~3 seconds
}
```

Each `await` pauses the function. `fetchPosts` doesn't start until `fetchUser` finishes. `fetchSettings` doesn't start until `fetchPosts` finishes.

But `posts` and `settings` don't depend on `user` — they just need `userId`, which you already have. You're waiting for no reason.

```javascript
// Parallel — all start simultaneously
async function loadDashboard(userId) {
  const user = await fetchUser(userId); // still need this first

  // posts and settings both need userId, don't depend on each other
  const [posts, settings] = await Promise.all([
    fetchPosts(userId),
    fetchSettings(userId)
  ]);
  // Total: ~2 seconds (1s for user + 1s for both others simultaneously)
}
```

`Promise.all` starts both requests at the same time. Total time is the slowest of the two, not the sum. On a dashboard that makes five or six API calls, this difference becomes very noticeable.

The rule: use sequential `await` when each step *depends on the result* of the previous step. Use `Promise.all` when operations are independent.

---

## Common Mistakes

### 1. Forgetting await

```javascript
async function save(data) {
  database.save(data);       // ❌ returns a Promise nobody is waiting for
  console.log('Saved!');     // runs before the save completes
}
```

No error. No warning. The function runs, `console.log` fires, and the database operation finishes sometime later — or maybe fails silently. Always await async operations.

### 2. await inside .forEach()

```javascript
// ❌ forEach doesn't understand async — the awaits are effectively ignored
const ids = [1, 2, 3];
ids.forEach(async (id) => {
  const user = await fetchUser(id);
  console.log(user);
});
console.log('done'); // prints BEFORE any user is fetched
```

`forEach` fires each callback but doesn't wait for the returned Promises. Use `for...of` for sequential, or `.map()` with `Promise.all` for parallel:

```javascript
// Sequential
for (const id of ids) {
  const user = await fetchUser(id);
  console.log(user);
}

// Parallel
const users = await Promise.all(ids.map(id => fetchUser(id)));
```

### 3. async in React's useEffect

```javascript
// ❌ useEffect cannot receive an async function directly
useEffect(async () => {
  const data = await fetchData();
  setData(data);
}, []);

// ✅ define async function inside, then call it
useEffect(() => {
  async function load() {
    const data = await fetchData();
    setData(data);
  }
  load();
}, []);
```

`useEffect` expects either nothing or a cleanup function as its return value. An `async` function always returns a Promise — useEffect doesn't know what to do with it. Always define the async function inside and call it separately.

---

## Putting It All Together

Here is the core of the Crispy Async dashboard, with every concept we've covered across all three articles present in one function:

```javascript
async function loadDashboard(userId) {
  clearUI();
  showLoading();                           // show the async gap

  try {
    const [user, posts] = await Promise.all([  // parallel fetching
      fetchUser(userId),                       // real fetch() Promises
      fetchPosts(userId)
    ]);

    renderUser(user);                      // DOM manipulation
    renderPosts(posts);

  } catch (error) {                        // one handler for all failures
    showError(error.message);

  } finally {
    hideLoading();                         // always runs — gap is closed
  }
}
```

Line by line:
- `showLoading()` — makes the spinner visible, starts the async gap
- `Promise.all` — two network requests, running in parallel
- `await` — pauses this function, thread stays free, spinner keeps spinning
- `catch` — handles any failure from either fetch or either render call
- `finally` — spinner hides no matter what happened

This is production-quality async code. Every concept earned its place.

---

## The Full Picture

Across these three articles, we've built from the ground up:

```
JavaScript is single-threaded
        ↓
Blocking the thread freezes everything
        ↓
The event loop handles async without blocking
        ↓
Callbacks — the original pattern, but nests into chaos
        ↓
Promises — flat chaining, one catch for all errors
        ↓
async/await — same Promises, reads like synchronous code
        ↓
try/catch/finally — familiar error handling that actually works
        ↓
Promise.all — parallel execution for independent operations
```

None of these are separate topics. Each one is built directly on the one before it. When something breaks in async code, you now have the full picture to trace back to where it went wrong.

---

*Part 1: [JavaScript's Hidden Engine: The Event Loop](#)*
*Part 2: [From Callback Hell to Promises](#)*
*Live demo: [Crispy Async](https://github.com/emuiga/crispy-async)*
