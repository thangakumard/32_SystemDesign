# Native Hashing vs Consistent Hashing in Java

## Table of Contents

1. [Overview](#1-overview)
2. [Native Hashing](#2-native-hashing)
   - [2.1 What is Native Hashing?](#21-what-is-native-hashing)
   - [2.2 Basic Formula](#22-basic-formula)
   - [2.3 Java Example](#23-java-example)
   - [2.4 How Data Is Distributed](#24-how-data-is-distributed)
   - [2.5 Pros](#25-pros)
   - [2.6 Cons](#26-cons)
   - [2.7 When to Use](#27-when-to-use)
3. [Consistent Hashing](#3-consistent-hashing)
   - [3.1 What is Consistent Hashing?](#31-what-is-consistent-hashing)
   - [3.2 Hash Ring](#32-hash-ring)
   - [3.3 Java Hash Function Example](#33-java-hash-function-example)
   - [3.4 Key Lookup](#34-key-lookup)
   - [3.5 Adding a Server](#35-adding-a-server)
   - [3.6 Removing a Server](#36-removing-a-server)
   - [3.7 Virtual Nodes](#37-virtual-nodes)
   - [3.8 Pros](#38-pros)
   - [3.9 Cons](#39-cons)
   - [3.10 When to Use](#310-when-to-use)
4. [Native vs Consistent Hashing](#4-native-vs-consistent-hashing)
5. [Real-World Use Cases](#5-real-world-use-cases)
6. [Interview-Friendly Explanation](#6-interview-friendly-explanation)
7. [Key Takeaways](#7-key-takeaways)

---

# 1. Overview

Hashing is commonly used in distributed systems to determine **which server, cache node, partition, or shard should own a particular key**.

For example, suppose we have four Redis nodes:

```text
Redis-1
Redis-2
Redis-3
Redis-4
```

Given a key:

```text
user:12345
```

we need a deterministic way to decide where that key should be stored.

Two common approaches are:

1. **Native hashing / modulo hashing**
2. **Consistent hashing**

The important difference is what happens when the number of servers changes.

With native modulo hashing:

```text
server = hash(key) % numberOfServers
```

Adding or removing a server can cause **a large percentage of keys to move**.

With consistent hashing:

```text
server = first server clockwise from hash(key)
```

Adding or removing a server generally moves only the keys associated with the affected portion of the hash ring.

---

# 2. Native Hashing

## 2.1 What is Native Hashing?

Native hashing in distributed-system discussions usually means a simple **hash + modulo** strategy.

The basic algorithm is:

```text
serverIndex = hash(key) % N
```

where:

- `key` = data key
- `hash(key)` = hash value
- `N` = number of servers
- `serverIndex` = selected server

Example:

```text
hash("user:100") = 17

17 % 4 = 1
```

Therefore:

```text
user:100 -> Server 1
```

> Note: In Java, `HashMap` itself does not use `hash(key) % numberOfServers` for distributing data across machines. This section refers to using a normal hash function with modulo to distribute keys across distributed nodes.

---

## 2.2 Basic Formula

Assume:

```text
Servers = 4

S0
S1
S2
S3
```

For each key:

```text
index = hash(key) % 4
```

Example:

| Key | Hash | Hash % 4 | Server |
|---|---:|---:|---|
| user:101 | 17 | 1 | S1 |
| user:102 | 25 | 1 | S1 |
| user:103 | 42 | 2 | S2 |
| user:104 | 31 | 3 | S3 |
| user:105 | 8 | 0 | S0 |

The same key always maps to the same server as long as the server count remains unchanged.

---

## 2.3 Java Example

```java
public class NativeHashing {

    public static int getServer(String key, int serverCount) {
        int hash = key.hashCode();

        // Prevent negative indexes.
        return Math.floorMod(hash, serverCount);
    }

    public static void main(String[] args) {
        String[] servers = {
            "Server-0",
            "Server-1",
            "Server-2",
            "Server-3"
        };

        String[] keys = {
            "user:101",
            "user:102",
            "user:103",
            "user:104"
        };

        for (String key : keys) {
            int index = getServer(key, servers.length);

            System.out.println(
                key + " -> " + servers[index]
            );
        }
    }
}
```

A simplified hash function could also be implemented as:

```java
static int hash(String key) {
    int hash = 0;

    for (char c : key.toCharArray()) {
        hash = 31 * hash + c;
    }

    return hash;
}
```

This is conceptually similar to Java's traditional `String.hashCode()` calculation.

---

## 2.4 How Data Is Distributed

Suppose:

```text
N = 4
```

The mapping is:

```text
hash % 4

0 -> Server 0
1 -> Server 1
2 -> Server 2
3 -> Server 3
```

Now suppose we add a fifth server:

```text
N = 5
```

The mapping becomes:

```text
0 -> Server 0
1 -> Server 1
2 -> Server 2
3 -> Server 3
4 -> Server 4
```

The same key may now map to a completely different server.

Example:

```text
hash("user:101") = 17

Before:
17 % 4 = 1
user:101 -> Server 1

After:
17 % 5 = 2
user:101 -> Server 2
```

Therefore, changing the number of servers can invalidate many existing cache entries or require significant data movement.

---

## 2.5 Pros

### 1. Very simple

The algorithm is easy to understand:

```text
hash(key) % N
```

### 2. Fast

Hashing and modulo are extremely inexpensive.

Typical lookup:

```text
O(1)
```

### 3. Low implementation complexity

There is no need for:

- hash rings
- virtual nodes
- sorted maps
- ring management

### 4. Good distribution with a good hash function

A high-quality hash function can distribute keys reasonably evenly.

---

## 2.6 Cons

### 1. Poor scalability when nodes change

This is the biggest problem.

Suppose:

```text
100 servers
```

and we add one:

```text
101 servers
```

The formula changes from:

```text
hash(key) % 100
```

to:

```text
hash(key) % 101
```

Many keys will map to different servers.

This can cause:

- cache misses
- massive data movement
- network traffic
- load spikes
- expensive rebalancing

### 2. Cache avalanche risk

Imagine a distributed cache:

```text
Client
   |
   v
hash(key) % N
   |
   +---- Cache 1
   +---- Cache 2
   +---- Cache 3
   +---- Cache 4
```

If one cache node is added, many keys may move.

The newly selected node does not contain those cached values, causing many cache misses simultaneously.

### 3. Node removal has the same problem

If one server fails:

```text
hash(key) % N
```

changes to:

```text
hash(key) % (N - 1)
```

Potentially causing a large redistribution.

---

## 2.7 When to Use

Native hashing is appropriate when:

- the number of nodes rarely changes
- the cluster is relatively static
- simplicity is more important than dynamic scaling
- data movement is inexpensive
- you are partitioning a fixed set of resources

Common examples:

- static sharding
- fixed application partitions
- simple partition routing
- in-memory partitioning
- systems where node membership rarely changes

---

# 3. Consistent Hashing

## 3.1 What is Consistent Hashing?

Consistent hashing is a technique designed to minimize data movement when nodes are added or removed.

Instead of:

```text
hash(key) % N
```

we place both:

- servers
- keys

on a logical circular hash space called a **hash ring**.

Each key is assigned to the first server encountered while moving clockwise around the ring.

Conceptually:

```text
                 Server A
                    *
              --------------- 
           /                   \
         /                       \
        *                         *
   Server D                     Server B
        \                         /
         \                       /
              ---------------
                    *
                 Server C
```

The actual ring usually represents a large integer range, for example:

```text
0 ... 2^32 - 1
```

---

## 3.2 Hash Ring

Assume the hash space is:

```text
0 - 99
```

Servers are placed at:

```text
Server A -> 10
Server B -> 35
Server C -> 60
Server D -> 85
```

The ring is:

```text
             0
             |
       D 85  |  A 10
          \  |  /
           \ | /
            \|/
            /\
           /  \
      C 60     B 35
```

For a key:

```text
hash(key) = 42
```

Move clockwise:

```text
42 -> 60
```

The first server is:

```text
Server C
```

Therefore:

```text
key -> Server C
```

Another key:

```text
hash(key) = 70
```

Moving clockwise:

```text
70 -> 85
```

Therefore:

```text
key -> Server D
```

For:

```text
hash(key) = 90
```

there is no server between 90 and 99, so we wrap around:

```text
90 -> 10
```

Therefore:

```text
key -> Server A
```

---

## 3.3 Java Hash Function Example

A consistent hashing implementation needs a deterministic hash function.

For example:

```java
static int hash(String key) {
    int hash = 0;

    for (char c : key.toCharArray()) {
        hash = 31 * hash + c;
    }

    return Math.floorMod(hash, 100);
}
```

For a real production system, use a well-tested non-cryptographic hash such as:

- MurmurHash
- xxHash
- xxHash3

The important property is that the hash function should distribute keys evenly across the hash space.

---

## 3.4 Key Lookup

A simple Java implementation can use a `TreeMap`.

The `TreeMap` stores:

```text
hash position -> server
```

Example:

```java
import java.util.Map;
import java.util.TreeMap;

public class ConsistentHashing {

    private final TreeMap<Integer, String> ring =
            new TreeMap<>();

    private final int ringSize = 100;

    public void addServer(String server) {
        int position = hash(server);
        ring.put(position, server);
    }

    public String getServer(String key) {
        if (ring.isEmpty()) {
            return null;
        }

        int keyPosition = hash(key);

        // First server clockwise from the key.
        Map.Entry<Integer, String> entry =
                ring.ceilingEntry(keyPosition);

        // Wrap around the ring.
        if (entry == null) {
            entry = ring.firstEntry();
        }

        return entry.getValue();
    }

    private int hash(String key) {
        int hash = 0;

        for (char c : key.toCharArray()) {
            hash = 31 * hash + c;
        }

        return Math.floorMod(hash, ringSize);
    }
}
```

Example usage:

```java
public class Main {

    public static void main(String[] args) {

        ConsistentHashing ch =
                new ConsistentHashing();

        ch.addServer("Server-A");
        ch.addServer("Server-B");
        ch.addServer("Server-C");
        ch.addServer("Server-D");

        System.out.println(
                "user:101 -> " +
                ch.getServer("user:101")
        );

        System.out.println(
                "user:102 -> " +
                ch.getServer("user:102")
        );
    }
}
```

The lookup operation is approximately:

```text
O(log N)
```

because `TreeMap` is a balanced tree.

---

## 3.5 Adding a Server

This is where consistent hashing becomes useful.

Suppose the ring contains:

```text
A -> 10
B -> 35
C -> 60
D -> 85
```

Now add:

```text
E -> 50
```

Previously:

```text
35 -----> 60
 B         C
```

Keys between:

```text
35 and 50
```

previously belonged to:

```text
Server C
```

After adding E:

```text
35 -----> 50 -----> 60
 B          E         C
```

Only keys in that affected range move:

```text
Server C -> Server E
```

Most other keys remain where they were.

This is the primary advantage over modulo hashing.

---

## 3.6 Removing a Server

Suppose:

```text
A -> 10
B -> 35
C -> 60
D -> 85
```

Remove:

```text
C
```

Keys previously mapped to C move clockwise to:

```text
D
```

Other keys do not need to move.

Therefore, node failure or removal causes relatively limited redistribution.

---

## 3.7 Virtual Nodes

A major improvement to basic consistent hashing is the use of **virtual nodes**, also called replicas.

Without virtual nodes, a server may own a very large portion of the ring.

Example:

```text
A ------------------------ B
       C
```

The distribution may be uneven.

Instead of placing each physical server once:

```text
Server A
Server B
Server C
```

we place multiple virtual nodes:

```text
A#1
A#2
A#3
...
A#100

B#1
B#2
B#3
...
B#100
```

Example:

```java
for (int i = 0; i < virtualNodes; i++) {
    String virtualNode = server + "#" + i;
    int position = hash(virtualNode);

    ring.put(position, server);
}
```

Now each physical server owns many small ranges.

This provides better load distribution.

### Why virtual nodes matter

Without virtual nodes:

```text
Server A -> 45% of ring
Server B -> 10%
Server C -> 45%
```

With virtual nodes:

```text
Server A -> ~33%
Server B -> ~34%
Server C -> ~33%
```

The exact distribution depends on the hash function and number of virtual nodes.

---

## 3.8 Pros

### 1. Minimal data movement

Adding or removing a node affects only a portion of keys.

This is the primary benefit.

### 2. Excellent for dynamic clusters

Nodes can be:

- added
- removed
- replaced
- scaled horizontally

without remapping the entire key space.

### 3. Good for distributed caches

It minimizes cache invalidation caused by topology changes.

### 4. Supports horizontal scaling

You can add nodes as traffic increases.

### 5. Virtual nodes improve distribution

Multiple virtual nodes per physical server reduce hotspots.

---

## 3.9 Cons

### 1. More complex

Compared with:

```text
hash(key) % N
```

you need:

- a hash ring
- node membership
- ring updates
- lookup logic
- possibly virtual nodes

### 2. Lookup is more expensive

A simple modulo lookup is approximately:

```text
O(1)
```

A `TreeMap`-based consistent hash lookup is:

```text
O(log N)
```

Although this is generally fast enough, it is technically more expensive.

### 3. Uneven distribution without virtual nodes

A poor hash distribution or too few nodes can create hotspots.

Virtual nodes are normally used to solve this.

### 4. Membership changes require coordination

All clients/nodes must have a consistent view of the ring.

Otherwise:

```text
Client A -> Server B

Client B -> Server C
```

could happen for the same key.

Distributed systems therefore need a mechanism for maintaining cluster membership.

### 5. Does not automatically guarantee perfect load balancing

Consistent hashing minimizes movement; it does not guarantee perfectly equal traffic.

Hot keys can still overload a server.

---

## 3.10 When to Use

Consistent hashing is a strong choice when:

- nodes dynamically scale
- nodes can fail
- minimizing data movement matters
- the system is distributed
- cache/data locality is important

Common use cases include:

- distributed caches
- Redis clusters
- Memcached
- distributed databases
- object storage
- CDNs
- distributed key-value stores
- sharded services
- service routing
- request/session affinity

---

# 4. Native vs Consistent Hashing

| Feature | Native / Modulo Hashing | Consistent Hashing |
|---|---|---|
| Basic formula | `hash(key) % N` | Hash ring + clockwise lookup |
| Complexity | Simple | More complex |
| Lookup | O(1) | O(log N) with TreeMap |
| Adding node | Many keys may move | Only a portion moves |
| Removing node | Many keys may move | Only affected range moves |
| Dynamic scaling | Poor | Excellent |
| Load distribution | Good with good hash | Good with virtual nodes |
| Implementation | Very easy | More involved |
| Virtual nodes | Not required | Commonly used |
| Best for | Static clusters | Dynamic distributed systems |
| Cache use | Possible | Very common |
| Failure handling | Poor redistribution behavior | Better redistribution behavior |

---

# 5. Real-World Use Cases

## 5.1 Distributed Cache

Imagine:

```text
Application
     |
     v
+----+----+----+
|    |    |    |
v    v    v    v
R1   R2   R3   R4
```

Keys:

```text
user:1
user:2
user:3
...
```

can be distributed using consistent hashing.

This is especially useful when cache nodes are frequently added or removed.

---

## 5.2 Memcached

A client can use consistent hashing to decide which Memcached node stores a key.

For example:

```text
GET user:100

        |
        v
hash("user:100")
        |
        v
consistent hash ring
        |
        v
Memcached-3
```

If `Memcached-5` is added, only a subset of keys needs to move.

---

## 5.3 Redis / Distributed Key-Value Stores

Consistent hashing can be used conceptually for distributed key routing.

However, an important distinction is that some production systems implement their own partitioning schemes rather than a textbook consistent-hash ring.

For example, Redis Cluster uses **16384 hash slots**, not a traditional virtual-node consistent hash ring.

So in an interview, avoid saying:

> "Redis Cluster uses consistent hashing."

A more accurate statement is:

> "Redis Cluster uses hash-slot-based partitioning, while consistent hashing is another common technique for distributing keys across dynamic nodes."

---

## 5.4 CDN / Request Routing

Consistent hashing can be useful when requests should repeatedly reach the same backend/cache node.

Example:

```text
Client
   |
   v
Load Balancer
   |
   v
Consistent Hash(key)
   |
   +---- Server A
   +---- Server B
   +---- Server C
```

The key might be:

```text
userId
sessionId
objectId
URL
```

This can improve:

- cache locality
- session affinity
- backend efficiency

---

## 5.5 Distributed Storage

A distributed storage system may need to decide:

```text
Which node owns object X?
```

Hashing can determine the responsible node.

Consistent hashing is useful when storage nodes are dynamically added or removed.

---

# 6. Interview-Friendly Explanation

A concise Senior/Staff-level answer:

> **Native hashing** usually means using `hash(key) % N` to select a server. It is extremely simple and provides O(1) routing, but it has a major problem: when N changes, the modulo result changes for a large number of keys. That can cause significant data movement and cache misses.
>
> **Consistent hashing** places both servers and keys on a logical hash ring. A key is assigned to the first server clockwise from the key's hash position. When a server is added or removed, only the keys in the affected ring segment need to move, so it provides much better scalability for dynamic clusters.
>
> In production, consistent hashing commonly uses **virtual nodes** so that each physical server owns many small portions of the ring, improving distribution and reducing hotspots.

### Example

```text
Native:

hash(key) % 4

              Add Server
                  |
                  v
hash(key) % 5

Many keys change servers.
```

Versus:

```text
Consistent Hashing:

             A
          /     \
        D         B
         \       /
            C

Add E:

             A
          /     \
        D         B
         \   E   /
            C

Only keys in E's affected range move.
```

---

# 7. Key Takeaways

### Native / Modulo Hashing

Remember:

```text
server = hash(key) % N
```

**Best when:**

```text
N is stable
```

**Main problem:**

```text
Changing N => lots of keys move
```

---

### Consistent Hashing

Remember:

```text
key -> hash ring -> first server clockwise
```

**Best when:**

```text
Nodes are dynamically added/removed
```

**Main advantage:**

```text
Node change => only a small portion of keys move
```

**Use virtual nodes for:**

```text
Better distribution
+
Reduced hotspots
+
More balanced load
```

---

## Final Mental Model

Think of the two approaches this way:

```text
                 HASHING
                    |
          +---------+---------+
          |                   |
       Native             Consistent
       Hashing              Hashing
          |                   |
     hash(key) % N       Hash Ring
          |                   |
       O(1) lookup       O(log N)*
          |                   |
   Simple/static       Dynamic clusters
          |                   |
   Node changes cause    Limited key
   lots of movement      movement
```

`*` O(log N) applies to the common `TreeMap` implementation shown here. Other implementations can provide different lookup characteristics.

### Rule of thumb

```text
Static number of nodes
        |
        v
Native hashing

Dynamic number of nodes
        |
        v
Consistent hashing
```
