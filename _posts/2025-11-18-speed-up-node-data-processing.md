---
layout: post
title: Speed Up Data Processing in Node with p-limit and Worker Threads
author: tatut
excerpt: >
  We'll explore practical strategies to accelerate a real-world Node application that reads several large JSON files from disk, processes their contents, and outputs the results as large number of smaller JSON files.
tags:
 - Node
 - JavaScript
 - TypeScript
 - plimit
 - worker_threads
---

When I started writing this blog post, I realized that Node.js is now the same age I was when I was only _thinking_ about becoming a software developer. Node.js was originally designed to let web servers use JavaScript, the same language powering websites, creating a unified development experience. While Node.js is well-suited for I/O driven web servers, it faces challenges with CPU-intensive computations due to its single-threaded event loop. When heavy computations run on the main thread, they can block other events and degrade overall performance.

In this blog post, we'll explore practical strategies to accelerate a real-world Node application that reads several large JSON files from disk, processes their contents, and outputs the results as large number of smaller JSON files. We'll introduce two powerful Node.js tools: `p-limit` library for managing concurrent I/O operations, and `worker_threads` for parallelizing CPU-bound tasks. By combining these techniques, we can significantly reduce processing time and make Node applications more scalable.

## Limiting Factors in Node

To make sense of how to optimize performance in Node.js, it is essential to first understand the underlying limitations. At its core, Node.js is built on the V8 JavaScript engine, which provides high-speed execution of JavaScript code, but even highly optimized code cannot match the speed of native code. This means that Node.js is hardly the fastest tool available for raw data processing, so why use it for that in the first place? Well, it is a natural choice for handling JSON-based data due to its native support for JavaScript and rich node_modules ecosystem. It's also widely used tool and can be easily picked up by virtually any web developer.

Node's architecture is fundamentally event-driven and single-threaded. This works fine for web interfaces, but can be seen as a limitation for pure data processing. In simple terms, single-threaded means that only a single JavaScript event or function can be on execution at a time. Even if you put your code in a Promise-returning function makes no exception, since its synchronous parts (= non I/O code) is still run sequentially and blocks the event loop from executing other tasks. Luckily, Node.js is still efficient for handling I/O operations such as reading/writing files and handling network traffic. These operations are delegated to run by the underlying system (libuv thread pool) and thus do not overload Node's main event loop.

To optimize our Node process, we are going to maximize the performance of parallel I/O operations and also parallelize CPU-bound operations. How much of an increase can we expect to get? That is a very good question, but it is difficult to give any precise prediction as it largely depends on the use case. If numbers must be given, we were able to decrease the processing time of our JSON files from 40 seconds to just 14 seconds by using these techniques.

## p-limit

[p-limit](github.com/sindresorhus/p-limit) is a lightweight library for controlling concurrency in asynchronous operations. In other words, it can be used to easily limit the number of concurrently run I/O operations. Wait a second... how can we speed things up by _limiting_ processing? Well, while Node is good for running multiple I/O tasks in parallel, running _too many_ parallel operations at once can overwhelm system resources or hit OS limits.

First, let's consider this code:

```js
for (const file of files) {
   const result = await processData(file);
   await fs.promises.writeFile(result.path, result.data))
}
```

Here we process each file one-by-one and write it's result to the disk. We continue only after waiting the file operation to complete. This works and it's simple, but does not have much parallel processing going on. Let's improve it by running file write operations in the background:

```js
for (const file of files) {
   const result = await processData(file);
   fs.promises.writeFile(result.path, result.data))
}
```

Here we do not wait for file write to complete, but schedule it and immediately continue processing the next file. Assuming we have a large number of files to write, running this would eventually cause good number of the files to be sent to **libuv** to be handled at once - potentially way too much. While libuv has some limits of how many write operations it does in parallel by default, the remaining operations are put into queue, which consumes a lot of memory.

Thus, what we actually want to do is to run multiple file write operations in parallel and in the background, but avoid running _too many_ and cause high memory peaks. `p-limit` solves this by allowing you to specify the maximum number of concurrently run promises, ensuring that only a set number of tasks run at any given time while the rest wait in a queue.

The right amount concurrent operations depends on the use case and the capability of the underlying hardware. I would suggest beginning with a value of 4-6 and finding the sweet spot manually by testing with real-world data and hardware.

Here is an example of using `p-limit` to run a limited number of multiple file write operations concurrently. The idea is that we set a limit of how many promises (i.e `writeFile` operations) we want to run in parallel (in this case, **four**). There is also a pending queue, which makes sure that if we have thousands of files to write, we do not push all of them to `pLimit` at once. Instead, we allow up to **eight** operations to wait in the queue. Increasing this number allows data processing to run longer before hitting the pending queue limit, but also increases memory usage.

```js
import { promises as fs } from "fs";

import pLimit from "p-limit";

const MAX_ITEMS_IN_PENDING_QUEUE = 8; // Allow up to 8 operations to wait in the queue
const writePool = pLimit(4); // Allow up to 4 concurrent operations
const pendingWrites = new Set();

interface DataFile {
  path: string,
  data: string
}

function waitForFreeCapacityInPendingQueue() {
  return new Promise((resolve) => {
    const check = () => {
      if (writePool.pendingCount < MAX_ITEMS_IN_PENDING_QUEUE) {
        resolve(undefined);
      } else {
        // Wait until pending queue is free
        setTimeout(check, 0);
      }
    };

    check();
  });
}


async function enqueueWriteToFile(file: DataFile) {
  await waitForFreeCapacityInPendingQueue();
  const task = writePool(() => fs.writeFile(file.path, file.data));
  pendingWrites.add(task);
  task.finally(() => pendingWrites.delete(task));
}

async function waitForFileWritesToComplete() {
  await Promise.all([...pendingWrites]);
}

async function main() {
  // Assuming that `files` comes from somewhere
  for (file of files) {
    // Assuming that `processFile` is implemented somewhere
    const processed = await processFile(file);
    await enqueueWriteToFile(processed)
  }

  await waitForFileWritesToComplete();
}
```

The code begins from `main`, which is assumed to process thousands of files. After a file is processed, it is scheduled for writing via `enqueueWriteToFile`. This function either accepts the file to be written or forces the main loop to wait until pending queue has some free capacity available. If the file is accepted, data processing can continue while the actual write operation is run in the background.

It's important to note that `p-limit` is mainly designed to help running I/O operations more effectively. It does not help with CPU-bound tasks since it does not magically make JavaScript multi-threaded. For long-running CPU-bound tasks, we are going to look at `worker_threads` module next.

## Worker Threads

Node.js has `worker_threads` module, which enables true parallelism by running code inside Workers. This practically frees us from the single-threaded limitation of Node. Hooray, problems solved! But there is catch: running code in workers is _kind of_ running another Node inside Node. Each worker has it's own execution context, call stack, heap memory and event loop. This makes worker threads completely separated, but also requires some effort to setup.

Since running a task in a Worker requires its own "Node", initialising a Worker _practically_ requires initialising it with its own script. Here is a (very) simplified example.

First, we create the worker file and import `processData` from the main application:

```ts
// worker_types.ts
interface WorkerParams {
  a: number;
  b: number;
}
```

```ts
// worker.ts
import { workerData, parentPort } from "node:worker_threads";
import { processData } from "data/process"

// Process input from main thread
const params = workerData as WorkerParams;
const result = processData(params);

// Send result back to the main thread
parentPort?.postMessage(result);
```

We can compile this worker using `esbuild`, with a command like this: `npx esbuild src/worker.ts --bundle --outdir=dist --format=esm --platform=node"`.

Finally, we call the worker from the main thread:

```ts
// main.ts
import { fileURLToPath } from "node:url";
import { Worker } from "node:worker_threads";
import * as path from "path";

import { WorkerParams } from "worker_types";

const filename = fileURLToPath(import.meta.url);
const dirname = path.dirname(filename);

// Parameters to send to the worker
const params: WorkerParams = { a: 5, b: 7 };

// Create the worker (must point to already compiled JS file)
const worker = new Worker(path.join(dirname, "worker.js"), {
  workerData: params,
});

worker.on("message", async (result) => {
  console.log("Result from worker:", result);
  await enqueueWriteToFile({path: "hello.txt", data: result});
});

worker.on('error'), worker.on("exit", code => {
  console.error("Worker failed with code", code);
});
```

While this method works, it requires us to manually initialise a new Worker every time we want to process something in parallel. Also, if we want to run multiple different processes in parallel, the worker must be tuned to handle different tasks and return different results. These things can be simplified by using a library called [workerpool](https://github.com/josdejong/workerpool).

The idea in **workerpool** is roughly the same as what we previously saw with queued file write operations: we keep a pool of workers "warm", ready to be used for offloading multiple long-running computations in the background. If more tasks are sheduled to be offloaded than there are free workers available, we can use a manually defined pending queue to force the main thread to wait before a new worker can be initialised.

Whether you decide to use **workerpool** or implement your own solutions, here are a couple tips I hope I would have known when starting to mess with Workers:

- You are free to import stuff from your main application code to your worker, but everything you import gets compiled in the final `worker.js`. This can result in large bundles and potentially unwanted code if not done carefully. For example, if you introduce a worker pool in your main application code, and you accidentally import some code from there in your worker, **every worker gets its own worker pool**. Probably not what you want. This can be avoided by doing `import { isMainThread } from "worker_threads"` and checking if `isMainThread` is `false` when ever something should never be imported by a worker.
- Worker's JS code must be built separately every time you modify it and always before running tests (I'm not happy remembering spending a day figuring out why some changes in my worker code did not do anything). If you use Vitest, you can use its `setup.ts` file to run code like `execSync("npm run build:worker", { stdio: "inherit" });` to get a fresh worker every time before running tests.
- Worker threads have their own isolated memory and cannot reference objects from the main thread (okay, there is `SharedArrayBuffer`, but it's a bit advanced concept). When you instantiate a worker and pass in some parameters, these parameters **passed by value** (i.e. structured clone), not by reference. Thus, one needs to be careful about high memory peaks when using workers.
- Make it easy to run the same code either sequentially or in parallel. Sequentially run code is much easier to debug than parallel and both should give you a similar result.

## Conclusion

Here is a quick summary of the tools we covered:

**p-limit** is ideal for managing concurrency in I/O-bound tasks, such as reading or writing files and making HTTP requests. By limiting the number of simultaneous operations, you prevent resource exhaustion and avoid hitting system limits. The code is only slightly more complex than standard async code, making `p-limit` a practical choice for many I/O-heavy workloads.

**worker_threads** is designed for CPU-bound tasks that would otherwise block the event loop, virtually any heavy data processing. Worker threads run in parallel, leveraging multiple CPU cores for true concurrency. However, using worker threads increases code complexity and memory usage, as each thread has its own context and communication overhead. Using it also requires some planning of _how_ to actually divide complex computation into workers effectively. For heavy parallel data processing, I recommend taking a look at [workerpool](https://github.com/josdejong/workerpool) library.

Both `p-limit` and `worker_threads` are powerful tools for improving Node.js performance, but they serve different purposes and come with trade-offs.
Using either or both always increases code complexity and memory usage. Use them only when you see clear advantages and are ready to accept the increased code complexity.
