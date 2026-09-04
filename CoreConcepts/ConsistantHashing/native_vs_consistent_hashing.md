# Deep Dive: Native Hashing vs. Consistent Hashing

Hashing is a fundamental computer science concept used to map data of arbitrary size to fixed-size values. When scaling distributed systems, the choice of hashing architecture significantly impacts performance, load balancing, and cluster stability. 

This document explores **Native (Traditional) Hashing** and **Consistent Hashing**, detailing their mechanics, concrete hash function implementations, pros, cons, and real-world application scenarios.

---

## 1. Native Hashing (Traditional Modulo Hashing)

### 1.1 How It Works
Native hashing distributes keys across a fixed set of $N$ nodes (or buckets) using a standard hash function combined with the modulo operator ($\%$):

$$\text{Node Index} = \text{Hash}(key) \pmod N$$

1. The input $key$ (e.g., string, integer) is passed through a hashing function (like MD5, SHA-256, or MurmurHash3) to yield an integer score.
2. The score is modulo-divided by $N$, where $N$ is the current number of active storage nodes or cache servers.
3. The key is assigned to the node corresponding to the resulting index ($0$ to $N-1$).

### 1.2 Hash Function Example (Python)

Below is an implementation of a native hashing mechanism using standard hashlib / custom hash functions to keep dependencies clean.

```python
import hashlib


class NativeHashRing:

    def __init__(self, nodes):
        """Initialize with a list of active node identifiers/addresses."""
        self.nodes = sorted(nodes)

    def get_node(self, key: str) -> str:
        """Find the designated node for a given key using native modulo hashing."""
        if not self.nodes:
            raise ValueError("No nodes available.")

        # Step 1: Generate numeric hash integer from string key (using MD5/SHA256 converted to int)
        hash_val = int(hashlib.md5(key.encode("utf-8")).hexdigest(), 16)

        # Step 2: Modulo by total number of nodes N
        node_index = hash_val % len(self.nodes)

        # Step 3: Return target node
        return self.nodes[node_index]


# --- Usage Example ---
nodes = ["Node_0", "Node_1", "Node_2"]
native_cluster = NativeHashRing(nodes)

keys = ["user_1024", "session_abc", "img_9882", "order_55"]
print("--- Native Hashing Assignments (N=3) ---")
for k in keys:
    print(f"Key: '{k}' -> Assigned to: {native_cluster.get_node(k)}")
```

### 1.3 Pros and Cons

| **Pros** | **Cons** |
| :--- | :--- |
| **Simplicity**: Extremely straightforward to implement and reason about ($O(1)$ lookup). | **Massive Cache Invalidation (Rehash Storms)**: Changing $N$ (adding/removing a node) causes almost **all** keys ($K \cdot \frac{N-1}{N}$) to remap to new nodes. |
| **Uniform Distribution**: High-quality hash algorithms ensure uniform key distribution across all $N$ nodes. | **High Resharding Cost**: Rescheduling data across a network when resizing causes heavy I/O and latency spikes. |
| **Zero Overhead**: No extra data structures (like trees or sorted rings) are needed in memory. | **Downtime / Cache Stampede Risk**: Mass cache misses upon cluster scaling can overwhelm back-end database layers. |

---

## 2. Consistent Hashing

### 2.1 How It Works
Consistent Hashing decouples the mapping from the fixed count of nodes $N$. Instead of mapping keys directly to node indices via modulo, both **keys** and **nodes** are mapped onto a shared, abstract circular space called a **Hash Ring** (ranging from $0$ to $2^{32}-1$ or $2^{64}-1$).

1. **The Hash Ring**: Imagine a continuous integer ring from $0$ to $2^{32}-1$.
2. **Node Mapping**: Each physical server is hashed into one or more positions on the ring using its ID or IP address.
3. **Key Mapping**: A key is hashed to a position on the same ring.
4. **Key Assignment**: To locate the server for a key, traverse the ring **clockwise** from the key's position until encountering the first node.
5. **Virtual Nodes (VNodes)**: To prevent non-uniform distribution (hotspots), each physical node is assigned multiple "virtual nodes" across the ring (e.g., `Node_A#1`, `Node_A#2`, `Node_A#3`).

When a node is added or removed, **only $K/N$ keys are remapped** on average, where $K$ is the total number of keys and $N$ is the number of nodes.

### 2.2 Hash Function Example (Python)

Below is a complete implementation of a **Consistent Hash Ring with Virtual Nodes** using binary search (`bisect`) for $O(\log N)$ node lookups.

```python
import bisect
import hashlib


class ConsistentHashRing:

    def __init__(self, num_replicas: int = 100):
        """
        :param num_replicas: Number of virtual nodes per physical node.
        """
        self.num_replicas = num_replicas
        self.ring = []  # Sorted list of virtual node hashes
        self.vnode_map = {}  # Maps virtual node hash -> physical node string

    def _hash(self, key: str) -> int:
        """Returns an integer hash derived from MD5."""
        return int(hashlib.md5(key.encode("utf-8")).hexdigest(), 16)

    def add_node(self, node: str):
        """Add a physical node and its virtual replicas to the ring."""
        for i in range(self.num_replicas):
            vnode_key = f"{node}-vnode-{i}"
            vnode_hash = self._hash(vnode_key)

            # Insert into ring while maintaining sorted order
            bisect.insort(self.ring, vnode_hash)
            self.vnode_map[vnode_hash] = node

    def remove_node(self, node: str):
        """Remove a physical node and its virtual replicas from the ring."""
        for i in range(self.num_replicas):
            vnode_key = f"{node}-vnode-{i}"
            vnode_hash = self._hash(vnode_key)

            index = bisect.bisect_left(self.ring, vnode_hash)
            if index < len(self.ring) and self.ring[index] == vnode_hash:
                del self.ring[index]
                del self.vnode_map[vnode_hash]

    def get_node(self, key: str) -> str:
        """Find the designated node for a key by moving clockwise on the ring."""
        if not self.ring:
            return None

        key_hash = self._hash(key)

        # Find the first virtual node hash >= key_hash
        idx = bisect.bisect_right(self.ring, key_hash)

        # If we reached the end of the ring, wrap around to index 0
        if idx == len(self.ring):
            idx = 0

        return self.vnode_map[self.ring[idx]]


# --- Usage Example ---
ch_cluster = ConsistentHashRing(num_replicas=3)

# Add physical servers
for server in ["Server_A", "Server_B", "Server_C"]:
    ch_cluster.add_node(server)

keys = ["user_1024", "session_abc", "img_9882", "order_55"]
print("--- Consistent Hashing Initial Assignments ---")
for k in keys:
    print(f"Key: '{k}' -> Assigned to: {ch_cluster.get_node(k)}")

# Dynamic Scaling Demonstration
print("\n--- Scaling: Removing Server_B ---")
ch_cluster.remove_node("Server_B")
for k in keys:
    print(f"Key: '{k}' -> Re-assigned to: {ch_cluster.get_node(k)}")
```

### 2.3 Pros and Cons

| **Pros** | **Cons** |
| :--- | :--- |
| **Minimal Keys Remapped**: Adding or removing a server shifts only $\sim K/N$ keys, minimizing network thrashing. | **Higher Algorithm Complexity**: Requires ring structures, sorted arrays/trees, and binary searches ($O(\log N)$ lookup). |
| **Horizontal Scalability**: Allows seamless auto-scaling of dynamic caching networks (e.g., Memcached/Redis clusters). | **Potential Memory Overhead**: Maintaining thousands of virtual nodes per physical machine consumes memory metadata. |
| **Load Balancing via VNodes**: Virtual nodes prevent hotspots by distributing server responsibility evenly across the hash space. | **Non-trivial Heterogeneity Management**: Requires explicit weighting/tuning of virtual node ratios for asymmetric hardware. |

---

## 3. Comparison Summary

| Metric / Feature | Native (Modulo) Hashing | Consistent Hashing |
| :--- | :--- | :--- |
| **Key Formula** | `Hash(key) % N` | Clockwise lookup on ring ($0$ to $2^{b}-1$) |
| **Lookup Time Complexity** | $O(1)$ | $O(\log N)$ (with Binary Search/TreeMap) |
| **Impact of Adding Node** | Re-maps almost $100\%$ of keys | Re-maps only $\sim 1/N$ of total keys |
| **Impact of Removing Node** | Re-maps almost $100\%$ of keys | Re-maps only keys belonging to removed node |
| **System Overhead** | Minimal (Zero extra structures) | Moderate (Requires sorted ring & virtual nodes) |
| **Hotspot Vulnerability** | Low (if hash function is good) | Solved explicitly via **Virtual Nodes** |

---

## 4. When and Where Each Hashing Strategy is Used

### 4.1 Native Hashing Use Cases
Native hashing is ideal when the number of buckets or nodes is **static**, strictly bounded, or when cache loss is negligible and simplicity/speed is paramount.

1. **In-Memory Hash Tables / HashMaps**: Standard programming language data structures (Python `dict`, Java `HashMap`, C++ `std::unordered_map`) use modulo-based bucket placement because the dataset resides on a single machine.
2. **Fixed Data Partitioning**: Partitioning database tables across a known, permanent number of shards (e.g., static database sharding with 16 fixed tables).
3. **Internal Load Balancing with Sticky Sessions**: Short-lived request routing where node counts rarely change, or where state can be easily recreated.

### 4.2 Consistent Hashing Use Cases
Consistent hashing is the standard architecture for **elastic, high-availability, distributed systems** where nodes frequently join, leave, or fail.

1. **Distributed Caches**: 
   * **Memcached** / **Redis Cluster**: Prevents cache stampedes when scaling cache nodes up or down.
2. **NoSQL Distributed Databases**:
   * **Apache Cassandra** & **Amazon DynamoDB**: Uses a consistent hash ring to partition data across distributed storage nodes smoothly.
3. **Content Delivery Networks (CDNs)**:
   * **Akamai** / **Cloudflare**: Routes requests for web assets to edge servers; if an edge server fails, only its assigned traffic shifts to neighboring edge servers.
4. **Distributed Object Storage**:
   * **OpenStack Swift** & **Ceph**: Maps data blocks/objects to storage disks reliably during drive failures or expansion.
5. **Microservices Load Balancing**:
   * System routers (e.g., Envoy, HAProxy) using consistent hashing for stateful RPC connections or distributed rate limiting.
