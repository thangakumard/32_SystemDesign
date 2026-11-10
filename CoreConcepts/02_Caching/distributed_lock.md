# Distributed Locking with Redis and Lua Scripts

## Table of Contents
- [Q1: Why release a Redis distributed lock using a Lua script?](#q1-why-release-a-redis-distributed-lock-using-a-lua-script)
  - [The Danger of Non-Atomic Lock Release](#the-danger-of-non-atomic-lock-release)
    - [Why checking value in application code fails (TOCTOU)](#why-checking-value-in-application-code-fails-toctou)
  - [How a Lua Script Solves the Problem](#how-a-lua-script-solves-the-problem)
    - [How this protects the lock](#how-this-protects-the-lock)
- [Q2: Step-by-Step Walkthrough of the Lua Script](#q2-step-by-step-walkthrough-of-the-lua-script)
  - [Step 1: Redis looks up the current value](#step-1-redis-looks-up-the-current-value-get)
  - [Step 2: The script compares the tokens](#step-2-the-script-compares-the-tokens)
  - [Step 3: The script skips the deletion](#step-3-the-script-skips-the-deletion)
  - [Why this saves the day](#why-this-saves-the-day)
- [Q3: Can clients use any programming language? Why does it say Lua script?](#q3-can-clients-use-any-programming-language-why-does-it-say-lua-script)
  - [1. The Problem: Redis lacks a single "Conditional Delete" command](#1-the-problem-redis-lacks-a-single-conditional-delete-command)
  - [2. The Solution: Moving the logic inside Redis with Lua](#2-the-solution-moving-the-logic-inside-redis-with-lua)
  - [Summary of Roles](#summary-of-roles)

---

## Q1: Why release a Redis distributed lock using a Lua script?

### Question
When using Redis as a distributed lock, why should a client release the lock with a Lua script that checks the stored token before deleting the key?

### Answer
When using Redis as a distributed lock, releasing the lock requires checking a unique token (often a random UUID) before deleting the key. Doing this inside a **Lua script** ensures that the check and the deletion happen **atomically**.

Without a Lua script, a race condition can cause a client to accidentally delete **another client's lock**, completely breaking mutual exclusion.

### The Danger of Non-Atomic Lock Release

Imagine a distributed lock implemented with a unique token and an expiration time (TTL) to prevent deadlocks if a client crashes. A failure happens through the following sequence:

1. **Client A** acquires the lock, setting a unique token (`uuid-A`) and a TTL of 10 seconds.
2. **Client A** experiences a long delay (such as a heavy Garbage Collection pause, a network hiccup, or a CPU spike) that lasts longer than 10 seconds.
3. Because 10 seconds have passed, Redis automatically expires and deletes the lock key.
4. **Client B** requests the lock, successfully acquires it, and sets its own token (`uuid-B`). Client B is now safely performing critical section work.
5. **Client A** finally wakes up from its pause. Unaware that its lock expired, Client A finishes its task and attempts to release its lock.

#### Why checking value in application code fails (TOCTOU)

You might think Client A could simply run a `GET` command first to check if the token matches `uuid-A`, and if it does, run `DEL`.

This introduces a **Time-Of-Check to Time-Of-Use (TOCTOU) race condition**:

* Between the moment Client A executes `GET` (and verifies the token matches) and the moment it executes `DEL`, the lock could expire and be grabbed by Client B.
* Client A would still proceed to execute `DEL`, wiping out Client B's active lock and allowing a third client to jump in.

### How a Lua Script Solves the Problem

Redis evaluates Lua scripts **atomically**. When you send a Lua script to Redis, no other commands or scripts can run until it finishes.

A safe lock-release Lua script looks like this:

```lua
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

#### How this protects the lock:

1. **The Check and Delete are Inseparable:** Redis checks the value of the key and deletes it in a single, indivisible atomic operation.
2. **No Interleaving:** It is impossible for another client to acquire the lock between the token check and the deletion. If Client A's token no longer matches (because the lock expired and was taken by Client B), the script evaluates to false, and the `DEL` command is skipped entirely.

---

## Q2: Step-by-Step Walkthrough of the Lua Script

### Question
Can you explain the Lua script step by step using Client A and Client B?

### Answer
Assuming **Client A** paused, its lock expired, and **Client B** took over with token `"uuid-B"`, Client A wakes up and sends the script to Redis with:
* `KEYS[1]` = `"my_lock"`
* `ARGV[1]` = `"uuid-A"`

### Step 1: Redis looks up the current value (`get`)
```lua
if redis.call("get", "my_lock") == "uuid-A" then
```
Redis looks up `"my_lock"`. Because Client B is currently holding the lock, Redis gets **`"uuid-B"`**.

### Step 2: The script compares the tokens
The script checks if `"uuid-B"` equals `"uuid-A"`. They **do not match**.

### Step 3: The script skips the deletion
The `if` condition fails, skipping `DEL` completely and executing:
```lua
else
    return 0
end
```
The script returns `0` to Client A, signaling that the lock was not released.

### Why this saves the day
Because Redis runs Lua scripts completely atomically, no other client can sneak in between the `GET` check and the `DEL` command. Client B's lock remains safely intact.

---

## Q3: Can clients use any programming language? Why does it say Lua script?

### Question
A client can use any programming language, right? Why does it specifically say "Lua script"?

### Answer
Yes, your **application code** (the backend server handling business logic) can be written in any language—such as Python, Java, Go, Node.js, or C#. 

However, **Lua** is used as a script that runs **directly inside the Redis engine itself**.

### 1. The Problem: Redis lacks a single "Conditional Delete" command
Normally, Redis commands are separate:
* `GET` reads a value.
* `DEL` deletes a key.

If your application (written in Python or Java) handles the logic, it must make **two network round-trips** to Redis:
1. `GET my_lock` $\rightarrow$ Returns `"uuid-A"`.
2. App checks if `"uuid-A" == "uuid-A"`, then sends `DEL my_lock`.

Between those two network calls, another client can acquire the lock. Your application code cannot prevent other clients from interacting with Redis during that gap.

### 2. The Solution: Moving the logic inside Redis with Lua
Instead of executing two separate network requests from your app language, your app sends a snippet of Lua code to Redis: *"Hey Redis, run this check and delete locally in memory."*

Redis natively embeds a Lua interpreter. When Redis runs a Lua script, it freezes all other client commands until the script completes, guaranteeing full atomicity.

### Summary of Roles
* **Application Code (Java, Python, Go, etc.):** Handles business logic, talks to databases, and decides *when* to acquire or release a lock.
* **Lua Script:** A lightweight script passed to Redis so that Redis can execute multiple commands (`GET` + `DEL`) as a single, un-interruptible atomic operation on the server.