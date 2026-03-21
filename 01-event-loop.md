# JavaScript's Hidden Engine: The Event Loop

Have you ever clicked a button on a website and the whole page froze? Or wondered how JavaScript can fetch data from a server without stopping everything else from working? The answer lives in one of the most misunderstood parts of JavaScript — the event loop.

This is the first article in a three-part series on async JavaScript. By the end of all three, you'll understand not just *how* to write async code, but *why* it works the way it does.

---

## JavaScript Has One Worker

Here is the most important sentence in this entire article:

**JavaScript can only do one thing at a time.**

One thread. One call stack. One thing executing at any moment.

This is not an accident or a limitation someone forgot to fix. It was a deliberate design decision made in 1995 when Brendan Eich created JavaScript for browsers. The reasoning was simple: if two threads could modify a webpage at the same time, you'd have chaos — buttons half-clicked, text half-updated, the DOM in a permanently broken state. One thread meant a simpler, safer model.

But one thread creates a problem.

---

## The Call Stack

Before we get to that problem, you need to understand the call stack — JavaScript's way of tracking what it's currently doing.

Think of it as a stack of plates. Every time you call a function, a plate gets added to the top. When the function finishes and returns, that plate is removed. JavaScript can only look at the top plate at any time.

```javascript
function add(a, b) {
  return a + b;
}

function calculate() {
  const result = add(2, 3);
  console.log(result);
}

calculate();
```

Here's what happens in the call stack:

```
calculate() called    → Stack: [calculate]
add(2, 3) called      → Stack: [calculate, add]
add returns 5         → Stack: [calculate]
console.log(5) called → Stack: [calculate, console.log]
console.log returns   → Stack: [calculate]
calculate returns     → Stack: []  ← empty, done
```

Clean. Sequential. Predictable.

Now — what if one of those functions takes three seconds to run?

---

## The Blocking Problem

```javascript
console.log('Start');

// Imagine this is reading a large file, or doing heavy computation
for (let i = 0; i < 3_000_000_000; i++) {}

console.log('End');
```

For those three seconds, the call stack is completely occupied by that loop. Nothing else can happen. No clicks register. No animations run. The browser cannot even redraw the page.

To users, the tab is frozen.

This is called **blocking the call stack**, and it's catastrophic in real applications. A web app that freezes while fetching data from a server — even for half a second — feels broken.

This is the problem async JavaScript exists to solve.

---

## The Solution: Handing Work Off

JavaScript's runtime — the browser or Node.js — has a trick. It can take certain slow operations and hand them off to be handled outside the JavaScript thread entirely.

Timers, network requests, file reads — these are managed by the browser's internal C++ APIs (or Node.js's libuv library). They run in parallel, completely separate from your JavaScript thread.

Your JavaScript thread stays free.

When that external work finishes, the result needs to come back to JavaScript. That's where the event loop comes in.

---

## The Event Loop

Here is the full picture:

```
┌────────────────────────────────────────┐
│             YOUR JS CODE               │
│                                        │
│   ┌──────────────┐                     │
│   │  CALL STACK  │ ← JS runs here      │
│   └──────┬───────┘                     │
│          │ when empty, checks:          │
│          ▼                             │
│   ┌──────────────────┐                 │
│   │ MICROTASK QUEUE  │ ← Promises      │
│   └──────┬───────────┘                 │
│          ▼                             │
│   ┌──────────────────┐                 │
│   │ MACROTASK QUEUE  │ ← setTimeout    │
│   └──────────────────┘                 │
└────────────────────────────────────────┘

         ↕ slow work handled by:

┌────────────────────────────────────────┐
│        BROWSER / NODE.JS APIS          │
│  setTimeout, fetch, file reads...      │
│  When done → push to queue             │
└────────────────────────────────────────┘
```

The event loop does one thing, forever: when the call stack is empty, check the microtask queue. If something's there, run it. Keep running until the microtask queue is empty. Then take one item from the macrotask queue and run it. Repeat.

Let's see it in action:

```javascript
console.log('Start');

setTimeout(() => {
  console.log('setTimeout');
}, 0);

console.log('End');
```

Output:
```
Start
End
setTimeout
```

Even with a delay of zero milliseconds, `setTimeout` doesn't run immediately. It gets handed off to the browser, which registers the timer, then pushes the callback to the macrotask queue when it fires. By the time the callback runs, the synchronous code has already finished.

---

## Microtasks vs Macrotasks

There are two queues, and they're not equal. The microtask queue (where Promise callbacks live) always gets fully drained before the event loop picks up the next macrotask (where `setTimeout` callbacks live).

```javascript
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
console.log('4');
```

Output:
```
1
4
3
2
```

`3` (Promise/microtask) always runs before `2` (setTimeout/macrotask), even though both were scheduled at roughly the same time.

---

## Why This Matters for Real Applications

Every time you fetch data from an API in a real application, this is what happens:

1. `fetch()` is called — JavaScript hands the network request to the browser
2. The call stack is immediately free — your app stays responsive
3. The user can scroll, click, type — nothing is blocked
4. When the server responds, the browser pushes your callback to the queue
5. The event loop picks it up when the stack is empty
6. Your code runs with the data

In our [Crispy Async](https://github.com/emuiga/crispy-async) demo, you can watch this happen live. Click "Load User 1" — a spinner appears immediately while the fetch is in flight, and the card renders once the data arrives. That spinner is the async gap made visible: the time between the request leaving JavaScript and the response coming back.

---

## The Key Takeaways

- JavaScript is single-threaded — one call stack, one thing at a time
- Blocking the call stack freezes the entire page
- Slow operations (network, timers, file reads) are handled outside the JS thread by the browser or Node.js
- The event loop moves results from the queues back to the call stack when it's free
- Microtasks (Promises) always run before macrotasks (setTimeout)

In the next article, we'll look at the first solution developers reached for when async JavaScript was born: callbacks — and why they eventually created a problem of their own.

---

*Part 2: [From Callback Hell to Promises](#)*
*Part 3: [async/await: Async Code That Reads Like English](#)*
*Live demo: [Crispy Async](https://github.com/emuiga/crispy-async)*
