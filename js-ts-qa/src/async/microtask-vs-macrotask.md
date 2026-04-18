# Q: Can you explain the difference between the microtask queue and the macrotask queue?

**Answer:**

Both the **microtask queue** and the **macrotask queue** (often just called the "task queue") are deeply integrated into the JavaScript **Event Loop**. They dictate the strict priority order in which asynchronous callbacks are executed.

### 1. The Microtask Queue
The microtask queue has **absolute, highest priority** over the macrotask queue. 
Its purpose is to execute small, immediate background tasks exactly before the Event Loop is allowed to continue, render the UI, or look at the macrotask queue.

**What goes into the microtask queue?**
*   **Promises** (`.then()`, `.catch()`, `.finally()`)
*   `process.nextTick()` (in Node.js)
*   `queueMicrotask()`
*   `MutationObserver` callbacks

### 2. The Macrotask Queue
The macrotask queue holds larger, more generic operations that the browser or host environment schedules. 

**What goes into the macrotask queue?**
*   `setTimeout()`
*   `setInterval()`
*   `setImmediate()` (in Node.js)
*   I/O operations (like fetching network data)
*   UI rendering and event callbacks (like clicks or keyboard presses)

### How they interact (The Execution Order)
1.  The Event Loop executes the current synchronous code (the call stack) until it is completely empty.
2.  Once empty, it checks the **microtask queue**. It executes *every single task* inside the microtask queue until it's completely empty. (If a microtask schedules another microtask, it will process that one too!).
3.  Only when the microtask queue is 100% empty, the Event Loop takes exactly **one** task from the **macrotask queue** and executes it.
4.  After that one macrotask finishes, it loops back and checks the microtask queue again.

### Classic Interview Example

```javascript
console.log('1. Script start'); // Synchronous

setTimeout(() => { // Macrotask
  console.log('2. setTimeout'); 
}, 0);

Promise.resolve().then(() => { // Microtask
  console.log('3. Promise 1'); 
}).then(() => { // Microtask chained
  console.log('4. Promise 2'); 
});

console.log('5. Script end'); // Synchronous
```

**Output Order:**
1.  `1. Script start` *(Sync)*
2.  `5. Script end` *(Sync)*
3.  `3. Promise 1` *(Microtask Queue)*
4.  `4. Promise 2` *(Microtask Queue emptied)*
5.  `2. setTimeout` *(Macrotask Queue picked up last)*
