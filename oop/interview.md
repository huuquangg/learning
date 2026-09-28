# Software Engineering Interview Prep Handbook

> Core foundations • Networking • OS services • Coding patterns
>
> **Study tip:** For concept questions, aim for a clear 30–60 second foundation answer first. Go deeper only when the interviewer asks follow-up questions.

## Quick Contents

1. [Interface vs. Abstract Class](#1-interface-vs-abstract-class)
2. [Object-Oriented Programming (OOP)](#2-object-oriented-programming-oop)
3. [Dependency Injection](#3-dependency-injection)
4. [Multithreading & Async Foundations](#4-multithreading--async-foundations)
5. [Networking Foundations](#5-networking-foundations)
6. [Windows Services & Linux Daemons](#6-windows-services--linux-daemons)
7. [Coding Interview Practice](#7-coding-interview-practice)

---

# 1. Interface vs. Abstract Class

> An interface and an abstract class both provide abstraction, but they serve different purposes.

- **Interface:** defines a contract — a set of members methods without implementations.
- **Abstract class:** provides a shared base class that can contain state, abstract members, and common implementation for derived classes.
- **Key distinction:** a class can implement multiple interfaces but can inherit from only one class.
- **Practical use:** expose an interface to consumers, and use an abstract base class internally when implementations need shared behavior.

---

# 2. Object-Oriented Programming (OOP)

OOP organizes software around objects that contain both **state** and **behavior**.

| Pillar | Foundation explanation |
|---|---|
| **Encapsulation** | Protect internal state and expose controlled operations. |
| **Abstraction** | Hide implementation details and expose only what consumers need. |
| **Inheritance** | Allow a class to extend a base class and reuse behavior. |
| **Polymorphism** | Allow different implementations to be accessed through the same abstraction. |

---

# 3. Dependency Injection

> Instead of a class creating its own dependencies, the dependencies are provided from outside — commonly through constructor injection.

This reduces coupling and makes implementations easier to swap, test, and maintain without changing the dependent class.

---

# 4. Multithreading & Async Foundations

## 1. Process and Thread

A **process** is a running application with its own memory.
A **thread** is a unit of execution inside that process.

A process can have multiple threads:

```text
Process
 ├─ Thread 1
 ├─ Thread 2
 └─ Thread 3
```

Threads share the process's memory, so they can work together efficiently, but shared memory can also create concurrency problems.

---

## 2. Multithreading

**Multithreading** means using multiple threads inside one process.

Example:

```text
Thread 1 → handle request A
Thread 2 → handle request B
Thread 3 → background work
```

It's useful when multiple pieces of work need to happen concurrently or in parallel.

The downside is complexity around shared data.

---

## 3. Synchronous

Synchronous means:

> Do one operation and wait until it finishes before continuing.

```csharp
var result = GetData();
Process(result);
```

Flow:

```text
GetData
   ↓
wait
   ↓
Process
```

The current thread is occupied while waiting.

---

## 4. Asynchronous

Asynchronous means:

> Start an operation, and don't block the thread while waiting for it to finish.

```csharp
var result = await GetDataAsync();
```

This is especially useful for:

- database calls
- HTTP requests
- file I/O
- network operations

Important:

> `async` does not mean creating another thread.

Async is mainly about **not blocking a thread while waiting**.

---

## 5. Concurrency vs Parallelism

**Concurrency** means multiple tasks make progress during the same period.

**Parallelism** means multiple tasks literally execute at the same time.

Example:

```text
Concurrency:
Task A → wait → continue
Task B → runs while A waits

Parallelism:
CPU Core 1 → Task A
CPU Core 2 → Task B
```

Async commonly gives you concurrency.

Multiple CPU cores can give you parallelism.

---

## 6. CPU-bound vs I/O-bound

This distinction is very important.

**I/O-bound** work spends most of its time waiting:

```text
Database
HTTP
File
Network
```

Prefer:

```csharp
await SomeOperationAsync();
```

**CPU-bound** work spends time calculating:

```text
Encryption
Compression
Image processing
Large calculation
```

This may benefit from multiple threads or parallel processing.

Good interview sentence:

> Async is mainly useful for I/O-bound operations, while multithreading and parallelism are more relevant for CPU-bound work.

---

## 7. Race condition

A **race condition** happens when multiple threads access shared data and the result depends on timing.

Example:

```csharp
counter++;
```

Two threads may read the same value and overwrite each other's update.

```text
counter = 5

Thread A reads 5
Thread B reads 5

A writes 6
B writes 6

Expected: 7
Actual: 6
```

---

## 8. Synchronization

Synchronization means controlling access to shared resources.

For example:

```csharp
lock (_lock)
{
    counter++;
}
```

Now only one thread can modify the value at a time.

Common synchronization tools:

```text
lock
Semaphore
Mutex
Interlocked
```

For foundation-level interviews, knowing `lock` and `Semaphore` is usually enough.

---

## 9. Lock

A `lock` allows only **one thread at a time** into a critical section.

```csharp
lock (_lock)
{
    UpdateSharedData();
}
```

Use it when several threads modify shared state.

---

## 10. Semaphore

A semaphore lets you limit how many operations can run at once.

For example:

```text
Semaphore = 3
```

means:

```text
Task A → allowed
Task B → allowed
Task C → allowed
Task D → waits
```

Useful when you want concurrency but need to protect resources like a database or external API.

---

## 11. Deadlock

A **deadlock** happens when two threads wait for each other forever.

Example:

```text
Thread A owns Lock 1
Thread B owns Lock 2

Thread A waits for Lock 2
Thread B waits for Lock 1
```

Neither can continue.

A simple prevention rule:

> Always acquire multiple locks in the same order.

Instead of:

```text
A: Lock1 → Lock2
B: Lock2 → Lock1
```

use:

```text
A: Lock1 → Lock2
B: Lock1 → Lock2
```

---

## 12. Task vs Thread

A **Thread** is an actual execution resource.

A **Task** represents work that will complete in the future.

```csharp
Task<User> task = GetUserAsync();
```

A Task does not necessarily have its own thread.

That's an important interview distinction.

---

## The foundation I would memorize

Keep this mental model:

```text
Thread
    ↓
executes code

Multithreading
    ↓
multiple threads

Sync
    ↓
wait and block

Async
    ↓
wait without blocking the thread

Concurrency
    ↓
multiple tasks making progress

Parallelism
    ↓
multiple tasks executing simultaneously

Shared data
    ↓
race condition

Protection
    ↓
lock / semaphore

Wrong locking
    ↓
deadlock
```

And the strongest short interview answer connecting everything is:

> Multithreading means a process uses multiple threads. Concurrency means multiple tasks can make progress at the same time, while parallelism means they physically execute simultaneously. Synchronous code blocks while waiting, whereas asynchronous code allows the thread to do other work during I/O waits. When multiple threads access shared mutable data, we need synchronization such as locks or semaphores to prevent race conditions. Incorrect synchronization can lead to deadlocks.

That is about the level I'd aim for first. Once this foundation feels automatic, deeper topics like `ThreadPool`, `Interlocked`, `ConfigureAwait`, and async state machines become much easier.

---

# 5. Networking Foundations

Your overall mental model is heading in the right direction, but there are a few important corrections—especially **routing, sockets, WebSocket, VPN, and ZTNA**. For an interview, I’d phrase them like this:

| Topic | Interview-ready understanding |
|---|---|
| **TCP vs UDP** | Both are **transport-layer protocols**. **TCP** provides a reliable, ordered byte stream: it establishes a connection, tracks sequence numbers, acknowledges data, and retransmits lost data. **UDP** sends independent datagrams with no built-in guarantee of delivery, ordering, or retransmission, trading reliability features for lower overhead and latency. |
| **DNS** | DNS translates human-friendly domain names such as `google.com` into information computers can use, most commonly IP addresses such as `142.x.x.x`. Think of it as the Internet's distributed naming system. |
| **Routing** | Routing is **not the `/api/users` part of a URL** at the networking level. Network routing decides **which path packets take between networks**, based primarily on destination IP addresses and routing tables. URL routing like `/api/users` happens later at the application/server level. |
| **Ports** | An IP identifies a **machine/network interface**, while a port identifies a particular network service/process endpoint on that machine. For example `192.168.1.10:443`: IP → machine, port `443` → service listening there. |
| **Sockets** | A socket is the **programming abstraction/API** applications use to communicate over the network. You can create a TCP socket or UDP socket. A network connection is commonly identified by protocol + source IP/port + destination IP/port. |
| **HTTP** | HTTP is an **application-layer request/response protocol**. HTTP/1.1 and HTTP/2 normally run over TCP. HTTP itself is stateless: each request contains the information needed to process it, although applications can maintain state using cookies, tokens, sessions, databases, etc. HTTP/3 is different: it runs over QUIC, which uses UDP. |
| **WebSocket** | WebSocket gives the client and server a **persistent, full-duplex connection**, allowing either side to send messages at any time. Important correction: traditional WebSocket normally runs over **TCP, not UDP**. It usually begins with an HTTP handshake and then upgrades the connection to WebSocket. |
| **Firewall** | A firewall enforces rules controlling network traffic. Rules can consider things like source/destination IP, ports, protocol, connection state, application, interface, etc., and decide whether traffic is allowed or blocked. |
| **VPN** | A VPN is **not simply a proxy**. It creates an encrypted tunnel between your device and another network/VPN gateway. It can route some or all network traffic through that tunnel, making your device logically connected to the remote/private network. |
| **ZTNA** | Zero Trust Network Access provides access based on **identity, device posture, policy, and context**, rather than trusting someone merely because they are connected to the corporate network. Typically, it grants access to specific applications/resources instead of giving broad network access like a traditional VPN. |

## The biggest corrections to your current understanding

Your **TCP/UDP** understanding is basically correct, but say **"reliability and ordering"** rather than "TCP doesn't drop messages." Packets can absolutely be lost underneath TCP. TCP detects the loss and retransmits them.

Your **routing** definition is mixing three different concepts:

```text
https://api.example.com:443/users/123
  │            │        │    │
protocol      host     port  application path
                │
                └── DNS resolves this to an IP
                         │
                         ▼
                    10.20.30.40
                         │
                    IP routing
                         │
                         ▼
                     server
                         │
                    TCP port 443
                         │
                    HTTP server
                         │
                   /users/123
```

So think:

**DNS → "What IP?"**

**Routing → "How do packets reach that IP?"**

**Port → "Which service on that machine?"**

**Socket → "How does my program communicate using TCP/UDP?"**

**HTTP path → "Which application resource/handler?"**

And your biggest technical error was here:

> WebSocket ... based on UDP

Change that to:

> **WebSocket normally runs over TCP. It starts with an HTTP handshake and then maintains a persistent bidirectional connection.**

## VPN vs ZTNA

This difference is particularly likely to matter for the endpoint-agent role you're preparing for.

```text
Traditional VPN

Laptop
   │
   │ encrypted tunnel
   ▼
Corporate Network
   ├── Server A
   ├── Server B
   ├── Database
   └── Internal apps
```

Once connected, VPNs traditionally give the device **network-level connectivity**, though firewall rules and segmentation can restrict it.

ZTNA approaches the problem differently:

```text
User + Device
      │
      ├── Who are you?
      ├── Is this device trusted/compliant?
      ├── Are you allowed to access App A?
      │
      ▼
   ZTNA Policy
      │
      ▼
    App A

Not necessarily:
      │
      └──────────► entire corporate network
```

So a strong interview answer would be:

> **"A traditional VPN establishes an encrypted tunnel and usually gives the device network-level access to a private network. ZTNA follows zero-trust principles: it continuously evaluates identity, device posture, and policy, and grants access to specific resources rather than implicitly trusting a device because it's inside the network."**

For your interview, I'd memorize the networking stack in roughly this order:

**DNS → IP → routing → TCP/UDP → ports/sockets → TLS → HTTP/WebSocket → application**

Then separately understand **firewall → VPN → ZTNA**, because those are about **controlling and securing that communication**.

---

# 6. Windows Services & Linux Daemons

Yes. For interview prep, I’d narrow it to these **10 core questions only** and keep every answer at a solid foundation level.

1. **What is a Windows Service?**  
A Windows Service is a background process managed by the **Service Control Manager (SCM)**. It can start automatically with Windows, run without a logged-in user, and be started, stopped, or restarted by the OS.

2. **What is a Linux daemon?**  
A Linux daemon is a long-running background process. On modern Linux systems, it is commonly managed by **systemd**, which controls startup, shutdown, restart behavior, dependencies, and logging.

3. **Windows Service vs Linux daemon?**  
They solve the same problem: running and managing background applications. Windows uses SCM, while Linux commonly uses systemd.

4. **What is a process?**  
A process is a running instance of a program. It has its own memory space, OS resources, security context, and one or more threads.

5. **Process vs thread?**  
A process provides isolation. A thread is an execution unit inside a process. Threads in the same process share memory, which makes communication fast but also creates synchronization problems such as race conditions.

6. **What happens when a service crashes?**  
The process terminates and the OS releases its process resources. SCM or systemd can detect the failure and restart the service based on its recovery configuration. The application itself should also be able to recover its previous state safely.

7. **How should a service shut down?**  
It should use graceful shutdown: stop accepting new work, finish or checkpoint current work, close connections, save important state, release resources, and then exit.

8. **What happens if the network connection is lost?**  
The service should normally keep running. Network failure should not mean service failure. The agent can retry the connection with backoff and synchronize again when connectivity returns.

9. **What is user mode vs kernel mode?**  
Applications normally run in **user mode**, where access is restricted. The operating system kernel runs in **kernel mode**, where it can access hardware and protected system resources. Applications request kernel operations through system calls.

10. **What does least privilege mean for a service?**  
A service should run with only the permissions it actually needs. Running everything as Administrator, root, or SYSTEM increases security risk. Privileged operations should be limited and isolated when possible.

The core mental model to remember is:

```text
Machine boots
    ↓
SCM / systemd
    ↓
Starts background service
    ↓
Process runs
    ↓
Threads execute work
    ↓
Service communicates with OS/network
    ↓
Handles failure gracefully
    ↓
SCM/systemd can restart it if necessary
```

For your interview, this level is enough unless the interviewer starts drilling deeper. The most important concepts are **service lifecycle, process/thread, crash recovery, graceful shutdown, network resilience, user/kernel mode, and least privilege**.

---

# 7. Coding Interview Practice

For interview practice, try explaining each solution aloud in this order: **brute force → what repeated work you can avoid → chosen data structure → edge cases → complexity**.

## 1. Two Sum

Store each number’s index. For the current number, check whether its complement appeared earlier.

```ts
function twoSum(nums: number[], target: number): number[] {
    const seen = new Map<number, number>();

    for (let i = 0; i < nums.length; i++) {
        const complement = target - nums[i];

        if (seen.has(complement)) {
            return [seen.get(complement)!, i];
        }

        seen.set(nums[i], i);
    }

    return [];
}
```

**Time:** O(n). **Space:** O(n).

## 2. Valid Parentheses

Use a stack to remember opening brackets. Each closing bracket must match the latest opening bracket.

```ts
function isValid(s: string): boolean {
    const stack: string[] = [];
    const openingFor: Record<string, string> = {
        ")": "(",
        "]": "[",
        "}": "{",
    };

    for (const char of s) {
        if (char === "(" || char === "[" || char === "{") {
            stack.push(char);
        } else if (stack.pop() !== openingFor[char]) {
            return false;
        }
    }

    return stack.length === 0;
}
```

**Time:** O(n). **Space:** O(n).

## 3. Valid Anagram

Count characters in the first string, then subtract counts using the second.

```ts
function isAnagram(s: string, t: string): boolean {
    if (s.length !== t.length) return false;

    const counts = new Map<string, number>();

    for (const char of s) {
        counts.set(char, (counts.get(char) ?? 0) + 1);
    }

    for (const char of t) {
        const count = counts.get(char) ?? 0;
        if (count === 0) return false;
        counts.set(char, count - 1);
    }

    return true;
}
```

**Time:** O(n). **Space:** O(k), where k is the number of distinct characters.

## 4. Best Time to Buy and Sell Stock

Track the lowest price seen so far and the best profit from selling today.

```ts
function maxProfit(prices: number[]): number {
    let lowest = Infinity;
    let bestProfit = 0;

    for (const price of prices) {
        lowest = Math.min(lowest, price);
        bestProfit = Math.max(bestProfit, price - lowest);
    }

    return bestProfit;
}
```

**Time:** O(n). **Space:** O(1).

## 5. Maximum Subarray

At each position, either extend the previous subarray or start a new one.

```ts
function maxSubArray(nums: number[]): number {
    let current = nums[0];
    let best = nums[0];

    for (let i = 1; i < nums.length; i++) {
        current = Math.max(nums[i], current + nums[i]);
        best = Math.max(best, current);
    }

    return best;
}
```

**Time:** O(n). **Space:** O(1). This is Kadane’s algorithm.

## 6. Longest Substring Without Repeating Characters

Maintain a window with no duplicate characters. Move its left edge past a repeated character.

```ts
function lengthOfLongestSubstring(s: string): number {
    const lastSeen = new Map<string, number>();
    let left = 0;
    let best = 0;

    for (let right = 0; right < s.length; right++) {
        const previous = lastSeen.get(s[right]);

        if (previous !== undefined && previous >= left) {
            left = previous + 1;
        }

        lastSeen.set(s[right], right);
        best = Math.max(best, right - left + 1);
    }

    return best;
}
```

**Time:** O(n). **Space:** O(k).

## 7. Merge Intervals

Sort by start time, then merge each interval into the previous one if they overlap.

```ts
function merge(intervals: number[][]): number[][] {
    if (intervals.length === 0) return [];

    intervals.sort((a, b) => a[0] - b[0]);
    const result: number[][] = [[...intervals[0]]];

    for (let i = 1; i < intervals.length; i++) {
        const last = result[result.length - 1];
        const current = intervals[i];

        if (current[0] <= last[1]) {
            last[1] = Math.max(last[1], current[1]);
        } else {
            result.push([...current]);
        }
    }

    return result;
}
```

**Time:** O(n log n). **Space:** O(n) for the result. The input is sorted in place.

## 8. Reverse Linked List

Redirect each node’s `next` pointer while keeping a reference to the next node.

```ts
function reverseList(head: ListNode | null): ListNode | null {
    let previous: ListNode | null = null;
    let current = head;

    while (current !== null) {
        const next = current.next;
        current.next = previous;
        previous = current;
        current = next;
    }

    return previous;
}
```

**Time:** O(n). **Space:** O(1).

## 9. Linked List Cycle

A fast pointer eventually catches a slow pointer if a cycle exists.

```ts
function hasCycle(head: ListNode | null): boolean {
    let slow = head;
    let fast = head;

    while (fast !== null && fast.next !== null) {
        slow = slow!.next;
        fast = fast.next.next;

        if (slow === fast) return true;
    }

    return false;
}
```

**Time:** O(n). **Space:** O(1).

## 10. Binary Search

Compare the middle element with the target and discard half the search range.

```ts
function search(nums: number[], target: number): number {
    let left = 0;
    let right = nums.length - 1;

    while (left <= right) {
        const mid = left + Math.floor((right - left) / 2);

        if (nums[mid] === target) return mid;
        if (nums[mid] < target) left = mid + 1;
        else right = mid - 1;
    }

    return -1;
}
```

**Time:** O(log n). **Space:** O(1). The array must be sorted.

## 11. Maximum Depth of Binary Tree

The depth of a node is one plus the deeper of its two subtrees.

```ts
function maxDepth(root: TreeNode | null): number {
    if (root === null) return 0;

    return 1 + Math.max(
        maxDepth(root.left),
        maxDepth(root.right)
    );
}
```

**Time:** O(n). **Space:** O(h) for recursion, where h is the tree height.

## 12. Binary Tree Level Order Traversal

Use a queue and process exactly one level at a time.

```ts
function levelOrder(root: TreeNode | null): number[][] {
    if (root === null) return [];

    const result: number[][] = [];
    const queue: TreeNode[] = [root];
    let front = 0;

    while (front < queue.length) {
        const levelSize = queue.length - front;
        const level: number[] = [];

        for (let i = 0; i < levelSize; i++) {
            const node = queue[front++];
            level.push(node.val);

            if (node.left !== null) queue.push(node.left);
            if (node.right !== null) queue.push(node.right);
        }

        result.push(level);
    }

    return result;
}
```

**Time:** O(n). **Space:** O(n). Using `front` avoids repeatedly calling the O(n) `shift()` method.

## 13. Number of Islands

When you find land, count one island and flood-fill all connected land so it cannot be counted again.

```ts
function numIslands(grid: string[][]): number {
    const rows = grid.length;
    if (rows === 0) return 0;

    const cols = grid[0].length;
    let islands = 0;
    const directions = [[1, 0], [-1, 0], [0, 1], [0, -1]];

    for (let row = 0; row < rows; row++) {
        for (let col = 0; col < cols; col++) {
            if (grid[row][col] !== "1") continue;

            islands++;
            const stack: [number, number][] = [[row, col]];
            grid[row][col] = "0";

            while (stack.length > 0) {
                const [r, c] = stack.pop()!;

                for (const [dr, dc] of directions) {
                    const nr = r + dr;
                    const nc = c + dc;

                    if (
                        nr >= 0 && nr < rows &&
                        nc >= 0 && nc < cols &&
                        grid[nr][nc] === "1"
                    ) {
                        grid[nr][nc] = "0";
                        stack.push([nr, nc]);
                    }
                }
            }
        }
    }

    return islands;
}
```

**Time:** O(rows × cols). **Space:** O(rows × cols) in the worst case. This version modifies `grid`.

## 14. Climbing Stairs

To reach step `i`, you can come from step `i − 1` or `i − 2`.

```ts
function climbStairs(n: number): number {
    let twoStepsBack = 1;
    let oneStepBack = 1;

    for (let step = 2; step <= n; step++) {
        const current = oneStepBack + twoStepsBack;
        twoStepsBack = oneStepBack;
        oneStepBack = current;
    }

    return oneStepBack;
}
```

**Time:** O(n). **Space:** O(1).

## 15. House Robber

For each house, choose between skipping it and robbing it after skipping the previous house.

```ts
function rob(nums: number[]): number {
    let twoHousesBack = 0;
    let oneHouseBack = 0;

    for (const money of nums) {
        const current = Math.max(
            oneHouseBack,
            twoHousesBack + money
        );

        twoHousesBack = oneHouseBack;
        oneHouseBack = current;
    }

    return oneHouseBack;
}
```

**Time:** O(n). **Space:** O(1).
