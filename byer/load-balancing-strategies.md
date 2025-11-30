# Load Balancing Strategies — Simple Explanations & Examples

Below are common load‑balancing strategies explained simply with short comments and tiny JavaScript examples for each. Real proxies (Nginx, HAProxy, Envoy) implement these at scale and handle many edge cases; these examples illustrate the core idea.

---

## 1) Round Robin
- Description: Cycle through servers evenly. Good when servers are similar.  
- Pros: Simple, fair when all servers equal.  
- Cons: Doesn’t consider server load or capacity.

Example:
```javascript
// Round Robin selector
const servers = ['s1', 's2', 's3'];
let rrIndex = 0;

function pickRoundRobin() {
  // pick server, then advance index
  const server = servers[rrIndex];
  rrIndex = (rrIndex + 1) % servers.length;
  return server;
}

// Usage
console.log(pickRoundRobin()); // s1
console.log(pickRoundRobin()); // s2
console.log(pickRoundRobin()); // s3
console.log(pickRoundRobin()); // s1
```

---

## 2) Weighted Round Robin
- Description: Like round robin but servers appear proportional to weight (capacity). Use when servers have different capacities.  
- Pros: Better distribution for heterogeneous servers.  
- Cons: Needs weights; still ignores current runtime load.

Example:
```javascript
// Weighted Round Robin selector (simple expansion)
const weightedServers = [
  { name: 's1', weight: 3 }, // will get ~3x traffic
  { name: 's2', weight: 1 },
  { name: 's3', weight: 2 }
];

let wrrList = [];
weightedServers.forEach(s => {
  for (let i = 0; i < s.weight; i++) wrrList.push(s.name);
});
let wrrIndex = 0;

function pickWeightedRoundRobin() {
  const server = wrrList[wrrIndex];
  wrrIndex = (wrrIndex + 1) % wrrList.length;
  return server;
}
```

---

## 3) Least Connections
- Description: Send request to server with fewest active connections. Good for long-lived requests or variable workloads.  
- Pros: Adapts to current load.  
- Cons: Needs tracking of active connections; slightly more complex.

Example:
```javascript
// Least Connections selector
const connections = {
  s1: 5, // currently 5 active requests
  s2: 2,
  s3: 8
};

function pickLeastConnections() {
  // choose server with minimal active connections
  return Object.keys(connections).reduce((minServer, s) =>
    connections[s] < connections[minServer] ? s : minServer
  , Object.keys(connections)[0]);
}

console.log(pickLeastConnections()); // s2
```

---

## 4) IP Hash (Source IP)
- Description: Hash client IP to pick a backend — same client goes to same server (session affinity).  
- Pros: Simple sticky behavior; deterministic.  
- Cons: Uneven distribution if clients skew; not resilient to server changes.

Example:
```javascript
// IP Hash selector (simple)
const servers = ['s1', 's2', 's3'];

function simpleHash(str) {
  let h = 0;
  for (let i = 0; i < str.length; i++) h = (h * 31 + str.charCodeAt(i)) >>> 0;
  return h;
}

function pickByIpHash(clientIp) {
  const hash = simpleHash(clientIp);
  return servers[hash % servers.length];
}

console.log(pickByIpHash('10.0.0.5')); // deterministic server for that IP
```

---

## 5) Consistent Hashing
- Description: Hash request key (IP, session id, user id) to a ring so small changes (server add/remove) cause minimal remapping. Great for caches and sharding.  
- Pros: Stable mapping; minimizes remapping on topology changes.  
- Cons: More complex; use virtual nodes for balance.

Example (conceptual):
```javascript
// Conceptual consistent hashing (small demo)
const crypto = require('crypto');
const servers = ['s1','s2','s3'];

// Build ring (hash -> server)
const ring = servers.map(s => ({
  hash: parseInt(crypto.createHash('md5').update(s).digest('hex').slice(0,8), 16),
  server: s
})).sort((a,b) => a.hash - b.hash);

function pickConsistent(key) {
  const keyHash = parseInt(crypto.createHash('md5').update(key).digest('hex').slice(0,8), 16);
  for (const node of ring) {
    if (node.hash >= keyHash) return node.server;
  }
  return ring[0].server; // wrap
}

console.log(pickConsistent('user:123')); // stable mapping for key
```

---

## 6) Random
- Description: Select a server at random. Simple and sometimes effective.  
- Pros: Extremely simple, can spread load reasonably when traffic is high.  
- Cons: No guarantees at low traffic; ignores capacity and health.

Example:
```javascript
// Random selector
const servers = ['s1','s2','s3'];
function pickRandom() {
  return servers[Math.floor(Math.random() * servers.length)];
}
console.log(pickRandom());
```

---

## 7) Least Response Time (Adaptive)
- Description: Choose server with lowest recent average response time. Adapts to server performance and network latency.  
- Pros: Improves user-perceived latency.  
- Cons: Requires metrics collection and smoothing logic.

Example:
```javascript
// Choose server with lowest average response time
const stats = {
  s1: { avgMs: 120 },
  s2: { avgMs: 80 },
  s3: { avgMs: 200 }
};

function pickLowestLatency() {
  return Object.entries(stats).sort((a,b) => a[1].avgMs - b[1].avgMs)[0][0];
}
console.log(pickLowestLatency()); // s2
```

---

## 8) Health-aware / Failover (Primary-Backup)
- Description: Prefer healthy servers; fall back to backups. Combine health checks with any strategy to avoid routing to unhealthy nodes.  
- Pros: Avoids sending traffic to unhealthy servers.  
- Cons: Requires health checking and state management.

Example (health + round robin):
```javascript
// Health-aware Round Robin
let servers = [
  { name: 's1', healthy: true },
  { name: 's2', healthy: false },
  { name: 's3', healthy: true }
];
let idx = 0;

function pickHealthyRoundRobin() {
  const healthy = servers.filter(s => s.healthy).map(s => s.name);
  if (healthy.length === 0) throw new Error('No healthy servers');
  const server = healthy[idx % healthy.length];
  idx++;
  return server;
}
```

---

## Quick Guidance: Which to use when?
- Round Robin / Weighted RR: simple setups; different capacities → weighted RR.  
- Least Connections: long-lived requests (WebSockets, streaming).  
- Least Response Time: prioritize latency/UX; needs monitoring.  
- IP Hash / Sticky: session affinity without cookies.  
- Consistent Hashing: distributed caches or sharded data stores.  
- Health-aware: always combine health checks with any algorithm.  
- Random: simplicity or when combined with health checks and many requests.

---

## Next steps
- Want nginx / HAProxy / Envoy config snippets for any of these strategies?  
- Or a tiny runnable Node.js proxy implementing one strategy end-to-end?  
Tell me which and I’ll add it in a follow-up file.