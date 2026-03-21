# From Callback Hell to Promises

In the [previous article](#), we learned that JavaScript is single-threaded and uses the event loop to handle async operations without freezing the page. Now the question is: how do you actually *write* code that works asynchronously?

The answer has evolved over the years. This article covers the first two generations — callbacks and promises — and the specific problem that forced JavaScript developers to move from one to the other.

---

## Generation 1: Callbacks

A callback is simply a function you pass to another function, to be called later when some async work is done.

```javascript
function fetchUser(userId, callback) {
  setTimeout(() => {
    const user = { id: userId, name: 'Steve' };
    callback(user); // "I'm done — here's your data"
  }, 1000);
}

fetchUser(1, function(user) {
  console.log('Got user:', user);
});
```

The pattern: *I can't give you the data right now, but give me a function — I'll call it when it's ready.*

For a single async operation, this works perfectly fine. The problem appears when async operations depend on each other.

---

## The Pyramid of Doom

In a real application, async operations are rarely isolated. You might need to:

1. Authenticate a user
2. Use their ID to fetch their profile
3. Use their role to get their permissions
4. Use their permissions to load their dashboard

Each step depends on the result of the previous one. With callbacks, that looks like this:

```javascript
authenticateUser(credentials, function(user) {
  getProfile(user.id, function(profile) {
    getPermissions(profile.role, function(permissions) {
      loadDashboard(permissions, function(data) {
        render(data);
      });
    });
  });
});
```

This is **callback hell** — also called the Pyramid of Doom. And this is only four levels deep. Real applications often need more.

The problems are structural:

**It drifts right.** Every nested operation adds another level of indentation. The actual logic gets buried inside layers of function wrappers.

**Error handling is a nightmare.** Each level needs its own error check. There's no single place to catch failures from the entire chain.

**Parallelism is awkward.** If two operations don't depend on each other, running them simultaneously requires manual coordination — tracking counters, checking if both have finished.

**It's hard to read.** The flow of the program doesn't match the way you'd describe it in plain English.

---

## The Node.js Convention: Error-First Callbacks

Before Promises arrived, Node.js established a convention for handling errors in callbacks. The first argument is always the error (null if no error), and the second is the data:

```javascript
fetchUser(1, function(error, user) {
  if (error) {
    console.log('Failed:', error.message);
    return;
  }
  console.log('Got user:', user);
});
```

This is called an **error-first callback**. It's better than nothing, but you still have to remember to handle the error at every single level of a nested chain. Forget one, and a failure silently disappears.

---

## Generation 2: Promises

A Promise is an object that represents a value that isn't available yet. It says: *I don't have the result right now, but I promise I'll have it — or I'll tell you if I failed.*

Every Promise is in exactly one of three states:

- **Pending** — work in progress, no result yet
- **Fulfilled** — success, has a resolved value
- **Rejected** — failed, has an error reason

Once a Promise settles (fulfills or rejects), it never changes state. A resolved Promise stays resolved.

### Creating a Promise

```javascript
function fetchUser(userId) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (userId < 0) {
        reject(new Error('Invalid userId'));
        return;
      }
      resolve({ id: userId, name: 'Steve' });
    }, 1000);
  });
}
```

The `new Promise()` constructor takes an executor function with two arguments: `resolve` (call this on success) and `reject` (call this on failure). You write your async work inside, then call one of them when it's done.

Important: `reject` is not automatic. You have to write the condition yourself and call it explicitly. A Promise that never calls `reject` will simply stay pending forever if something goes wrong.

### Consuming a Promise

```javascript
fetchUser(1)
  .then(user => {
    console.log('Got user:', user);
    return user.id;           // pass a value to the next step
  })
  .then(id => {
    console.log('User ID:', id);
  })
  .catch(error => {
    console.log('Failed:', error.message);
  })
  .finally(() => {
    console.log('Done — success or failure');
  });
```

Three things to know about `.then()` chaining:

1. Each `.then()` returns a new Promise, so you can chain them
2. Whatever you `return` from a `.then()` becomes the input to the next `.then()`
3. If you return a Promise from `.then()`, the chain waits for it to resolve before continuing

### The Problem Promises Solve

Compare the nested callback version to the Promise version of the same flow:

```javascript
// Callbacks — nested, hard to follow
authenticateUser(credentials, function(user) {
  getProfile(user.id, function(profile) {
    getPermissions(profile.role, function(permissions) {
      loadDashboard(permissions, function(data) {
        render(data);
      });
    });
  });
});

// Promises — flat, readable
authenticateUser(credentials)
  .then(user => getProfile(user.id))
  .then(profile => getPermissions(profile.role))
  .then(permissions => loadDashboard(permissions))
  .then(data => render(data))
  .catch(error => showError(error));
```

The Promise version reads almost like plain English. And that single `.catch()` at the end handles errors from any step in the entire chain — no need to check at every level.

---

## Running Things in Parallel

When multiple async operations don't depend on each other, you can run them simultaneously with `Promise.all`:

```javascript
// Sequential — ~2 seconds total (wasteful)
const user  = await fetchUser(1);
const posts = await fetchPosts(1);

// Parallel — ~1 second total
const [user, posts] = await Promise.all([
  fetchUser(1),
  fetchPosts(1)
]);
```

`Promise.all` takes an array of Promises, starts them all at the same time, and resolves when all of them have resolved. If any one rejects, the whole thing rejects.

There are other variants for different situations:

| Method | Behaviour |
|---|---|
| `Promise.all` | Waits for all, fails if any fail |
| `Promise.allSettled` | Waits for all, gives you each result regardless |
| `Promise.race` | Resolves/rejects as soon as the first settles |
| `Promise.any` | Resolves as soon as the first fulfills |

In the [Crispy Async](https://github.com/emuiga/crispy-async) demo, the user card and posts are fetched in parallel using `Promise.all`. Both requests start at the same time — so the total wait is only as long as the slower of the two.

---

## One Trap: try/catch Doesn't Catch Promise Rejections

A common mistake when first learning Promises:

```javascript
// ❌ This does NOT catch Promise rejections
try {
  fetchUser(1).then(user => console.log(user));
} catch (error) {
  console.log('caught:', error);
}
```

`try/catch` catches synchronous errors only. By the time the Promise rejects (after an async operation), the `try/catch` block has already finished executing and is gone. The rejection happens in the future — `try/catch` isn't watching anymore.

For Promises, use `.catch()` on the chain. Or — as we'll see in the next article — use `try/catch` inside an `async` function where it works correctly.

---

## The Key Takeaways

- Callbacks work for simple cases but create unreadable nested code for sequential operations
- Error-first callbacks are a convention, not a guarantee — errors can still be missed
- A Promise represents a future value in one of three states: pending, fulfilled, or rejected
- `.then()` chaining keeps async code flat and readable
- One `.catch()` at the end of a chain handles errors from any step
- `Promise.all` runs multiple operations in parallel — use it when operations don't depend on each other
- `try/catch` does not catch Promise rejections — use `.catch()` or `async/await`

Promises solved callback hell. But `.then()` chaining still feels a bit removed from how you'd naturally write synchronous code. That's what the third generation fixed.

---

*Part 1: [JavaScript's Hidden Engine: The Event Loop](#)*
*Part 3: [async/await: Async Code That Reads Like English](#)*
*Live demo: [Crispy Async](https://github.com/emuiga/crispy-async)*
