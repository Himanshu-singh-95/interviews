# PayPal Staff Software Engineer — Interview Solutions

> **Role:** Staff Software Engineer  
> **Format:** System Design (5 questions) + Coding Challenge (1 question)  
> **Language:** JavaScript / TypeScript

---

## Part 1 — System Design Questions

---

### Q1: Recipe Web App — Scaling Under Traffic

**Problem:** A web app used by home cooks and restaurants stores recipes and ingredients in a central database. It's experiencing slowdowns due to increased traffic.

---

#### Root Cause Analysis

Before jumping to solutions, identify where the bottleneck actually is:

- **Database layer:** slow queries, missing indexes, too many concurrent connections
- **Application layer:** no caching, N+1 query patterns, blocking synchronous operations
- **Infrastructure layer:** single server, no load balancing, no CDN for static assets

---

#### Solution Architecture

**1. Caching Layer (Biggest Win)**

Introduce Redis or Memcached between the app server and database.

- Cache frequently read recipes by ID (`recipe:{id}`)
- Cache search results and category listings with a short TTL (e.g. 60s)
- Use cache-aside pattern: check cache first, fall back to DB on miss, populate cache on read
- Invalidate cache on write (update/delete recipe)

```
Client → App Server → Redis Cache → (miss) → Primary DB
```

**2. Database Optimisation**

- Add indexes on high-cardinality query columns: `user_id`, `category`, `created_at`
- Use `EXPLAIN ANALYZE` to identify slow query plans
- Avoid `SELECT *` — fetch only required columns
- Use connection pooling (e.g. PgBouncer for PostgreSQL) to prevent connection exhaustion

**3. Read Replicas**

- Offload all read traffic (recipe views, search) to one or more read replicas
- Primary DB handles writes only (create, update, delete)
- Application routes queries based on operation type

**4. Horizontal Scaling + Load Balancing**

- Run multiple stateless app server instances behind a load balancer (e.g. NGINX, AWS ALB)
- Store sessions externally (Redis) so any server can handle any request
- Auto-scale based on CPU/request metrics

**5. Async Processing**

- Expensive operations (image resizing, nutrition calculation, search indexing) should be offloaded to a job queue (e.g. BullMQ, SQS + Lambda)
- Return a 202 Accepted immediately, process in background

**6. Pagination**

- Never return unbounded result sets
- Use cursor-based pagination for feeds; offset pagination for static search results

**7. CDN for Static Assets**

- Serve recipe images via CloudFront or Cloudflare
- Set long cache TTLs on immutable assets

---

#### Clarifying Questions to Ask

- What are the SLA targets? (e.g. p99 latency < 500ms)
- Is the bottleneck on reads or writes?
- Do we prioritise availability or strong consistency? (For recipe reads, eventual consistency is fine)
- What is the current database technology and hosting setup?

---

### Q2: Animation Server Farm — Cost Optimisation

**Problem:** A mobile app lets users create drawings and animations. The backend server farm that processes animations is becoming expensive to maintain.

---

#### Cost Reduction Strategy

**1. Autoscaling**

- Use cloud autoscaling groups (AWS ASG, GCP MIG) to provision servers only when needed
- Scale out during peak hours (evenings/weekends), scale in overnight
- Set minimum instance count to 1–2 to maintain responsiveness; max based on budget cap

**2. Spot / Preemptible Instances**

- Animation processing jobs are typically stateless and restartable — ideal for spot instances
- Use spot instances for the worker fleet (70–90% cheaper than on-demand)
- Build retry logic into the job queue so interrupted jobs resume gracefully

**3. Batch Processing During Off-Peak Hours**

- For non-real-time exports (e.g. "save as GIF"), queue the job and process during off-peak
- Notify user via push notification when ready
- Reduces peak load, enabling smaller autoscaling maximums

**4. Client-Side Offloading**

- Evaluate which processing steps can run on the device (preview rendering, low-res playback)
- Only offload final high-quality export or sharing to the server
- Use WebAssembly or native device GPU for lightweight processing

**5. Output Caching / Deduplication**

- Hash input parameters (frames + settings) and cache the output
- If two users export identical animations (e.g. a template), serve from cache
- Store cached outputs in S3/GCS with a TTL; delete on expiry

**6. Compress and Optimise Output**

- Use efficient codecs (H.265/HEVC, AV1) for video; WebP/APNG for image sequences
- Reduce output resolution for preview vs. final export tiers

**7. Reserved Instances for Baseline Load**

- Commit to Reserved Instances for the predictable baseline (e.g. 2–3 always-on servers)
- Use spot/on-demand only for burst traffic

---

#### Architecture Summary

```
Mobile App
    │
    ├─ Preview/playback → on-device
    │
    └─ Export request → Job Queue (SQS)
                            │
                    Spot Worker Fleet (autoscaled)
                            │
                    Output Cache (S3 + Redis hash check)
                            │
                    Push Notification to user
```

---

### Q3: Sports Stats Service — Unreliable Third-Party API

**Problem:** A sports statistics aggregation service is heavily dependent on a single third-party API that is unreliable.

---

#### Resilience Strategy

**1. Circuit Breaker Pattern**

- Wrap all calls to the third-party API in a circuit breaker (e.g. using `opossum` in Node.js)
- States: Closed (normal) → Open (failing, reject immediately) → Half-Open (probe recovery)
- Prevents cascading failures when the upstream is down

```typescript
import CircuitBreaker from 'opossum';

const breaker = new CircuitBreaker(fetchSportsData, {
  timeout: 3000,          // fail if no response in 3s
  errorThresholdPercentage: 50,
  resetTimeout: 30000,    // retry after 30s
});

breaker.fallback(() => getCachedData());
```

**2. Caching with TTL**

- Cache every successful API response in Redis with a sensible TTL (e.g. 60s for live scores, 24h for historical stats)
- On API failure, serve stale cache with a `data_freshness` timestamp surfaced to the user
- Differentiate between live data (short TTL) and historical data (long TTL / permanent store)

**3. Multiple / Fallback Providers**

- Integrate a secondary sports data provider (e.g. Sportradar, SportsDB)
- On primary failure, transparently switch to secondary
- Abstract behind a `SportsDataProvider` interface so the app code is provider-agnostic

**4. Retry with Exponential Backoff + Jitter**

```typescript
async function fetchWithRetry(url: string, retries = 3): Promise<Response> {
  for (let attempt = 0; attempt < retries; attempt++) {
    try {
      return await fetch(url);
    } catch (err) {
      if (attempt === retries - 1) throw err;
      const delay = Math.min(1000 * 2 ** attempt + Math.random() * 100, 10000);
      await new Promise(r => setTimeout(r, delay));
    }
  }
}
```

**5. Graceful Degradation**

- If all providers fail and cache is stale, show last known data with a visible "Last updated X minutes ago" indicator
- Never show a blank screen or crash — partial data is better than no data

**6. Alerting & Monitoring**

- Track error rate, p99 latency, and cache hit ratio on the upstream dependency
- Alert on-call when error rate exceeds threshold (e.g. >5% over 5 minutes)
- Build a status page or internal dashboard for API health

**7. Local Historical Data Store**

- Store historical match results in your own database
- Live scores depend on the API; historical queries never do

---

#### Clarifying Questions to Ask

- What is the acceptable data staleness for users? (Live match = 0s; yesterday's scores = hours)
- Should users be informed when they're seeing stale data?
- Is there budget for a paid secondary data provider?

---

### Q4: Smart Freezer — Migrating to Offline Microcontroller

**Problem:** A smart freezer's data logging and calculation logic runs on an Ethernet-connected computer. It needs to be migrated to an embedded microcontroller with no network access.

---

#### Key Constraints

| Resource | Networked Computer | Microcontroller |
|---|---|---|
| CPU | Multi-core GHz | Single-core MHz |
| RAM | GBs | KBs–MBs |
| Storage | HDD/SSD (TBs) | Flash (KBs–MBs) |
| Network | Ethernet / WiFi | None |
| Power | AC mains | DC / battery |

---

#### Migration Considerations

**1. Memory-Efficient Data Structures**

- Replace dynamic structures (arrays of objects, hashmaps) with fixed-size structs
- Use circular/ring buffers for time-series temperature logs — overwrite oldest entries when full
- Pre-allocate all buffers at startup; avoid heap fragmentation (avoid `malloc`/`free` in loops)

**2. Persistent Storage Strategy**

- Write logs to onboard Flash (EEPROM, SD card, or NAND Flash)
- Use a wear-levelling strategy — Flash cells degrade with repeated writes; rotate write positions
- Structure storage as a circular log with a header sector storing the write pointer
- Compress readings if space is tight (delta encoding: store temperature delta from previous reading instead of absolute values)

**3. Low-Power Operation**

- Use deep sleep modes between sensor reads
- Wake on timer interrupt (e.g. every 60 seconds) to log temperature
- Duty-cycle any peripherals (LCD display, LEDs) to only power on when needed

**4. Reliable Error Handling**

- Use checksums (CRC16/CRC32) on stored records to detect corruption
- Include a watchdog timer to auto-reset on firmware hang
- Log error codes to storage for later diagnostics

**5. Firmware Updates (No Network)**

- Support firmware updates via SD card or USB mass storage
- On boot, check for an `update.bin` file on the SD card; validate checksum before applying
- Keep a fallback firmware partition so a failed update doesn't brick the device

**6. Security (Offline)**

- Encrypt any sensitive data stored locally (e.g. usage patterns)
- Validate firmware update packages with a cryptographic signature

**7. Preserving Original Functionality**

- Map every calculation the computer performed to a microcontroller-safe equivalent
- Profile the most expensive calculations and optimise (integer math instead of float where possible)
- Test thoroughly with production-representative data before hardware swap

---

### Q5: Short-Video App with TTL — Capacity Planning

**Problem:** Users upload 1-minute videos with a configurable TTL (5 min – 24 hours). Predict resource needs as the app scales.

---

#### Capacity Planning Model

**Key Variables to Measure**

| Variable | Why It Matters |
|---|---|
| Daily Active Users (DAU) | Drives upload and view volume |
| Upload rate (videos/user/day) | Direct storage and compute demand |
| Average video size (MB) | Storage and bandwidth per upload |
| TTL distribution | Affects average stored video lifetime |
| View-to-upload ratio | CDN egress bandwidth |
| Peak concurrency factor | Determines infrastructure burst capacity |

**Storage Estimation Example**

```
Assumptions:
  100,000 DAU
  Each user uploads 2 videos/day
  Average video = 50 MB (1 min, 1080p, H.264)
  Average TTL = 6 hours

Daily uploads:    100,000 × 2 = 200,000 videos
Daily ingress:    200,000 × 50 MB = 10 TB/day
Avg stored at any time:
  200,000 × (6h / 24h) = 50,000 videos × 50 MB = 2.5 TB live storage
```

**Transcoding Requirements**

- Each uploaded video needs to be transcoded into multiple resolutions (1080p, 720p, 360p)
- Transcoding is CPU-intensive; estimate ~1–2× real-time on a standard instance
- At 200,000 uploads/day: ~200,000 minutes of video to process per day
- Use a managed transcoding service (AWS Elemental MediaConvert, Zencoder) or a spot-instance worker fleet

**CDN & Egress Bandwidth**

```
View-to-upload ratio: 10× (each video watched 10 times avg)
Daily views: 2,000,000
Avg video served: 50 MB
Daily egress: 2,000,000 × 50 MB = 100 TB/day
```

CDN is non-negotiable at this scale. CloudFront, Fastly, or Cloudflare will significantly reduce origin egress cost.

**TTL-Based Deletion**

- Run a scheduled cleanup job (every 5 minutes) to delete expired videos from storage and CDN
- Purge CDN cache on expiry to prevent stale serving
- Soft-delete in DB first; hard-delete from object storage asynchronously

**Scaling Triggers**

- Set autoscaling alarms on: upload queue depth, transcoding worker CPU, CDN hit rate, storage utilisation
- Plan infrastructure to handle 3–5× peak above daily average (viral spikes)

**Compliance & Security**

- Implement content moderation pipeline (async) before videos go live
- Ensure GDPR/CCPA-compliant deletion — TTL expiry must purge all copies including backups
- Encrypt videos at rest (AES-256) and in transit (TLS 1.3)

---

## Part 2 — Coding Challenge

---

### Problem: Tile Hand Validator (Mahjong-style)

**Problem Statement:**

Given a string of digit characters (`"0"–"9"`) representing a hand of tiles, determine whether the entire hand can be arranged into:
- Any number of **triples** (three identical tiles), AND
- Exactly **one pair** (two identical tiles)

All tiles must be used. Return `true` if valid, `false` otherwise.

**Examples:**

```
"88844"   → true   (triple: 8,8,8 | pair: 4,4)
"88844444" → true  (triple: 8,8,8 | triple: 4,4,4 | pair: 4,4) — wait, let's check...
             freq: {8:3, 4:5} — 4s: try pair (freq 3), 3 % 3 = 0 ✓ → true
"8884"    → false  (can't form a valid pair + triples)
"88"      → false  (pair only, no triples — total tiles not divisible by (3n+2))
"22333"   → true   (pair: 2,2 | triple: 3,3,3)
""        → false  (need at least one pair)
```

---

### Solution

#### Approach

1. Build a **frequency map** of all tiles
2. Total tile count must satisfy `count % 3 === 2` (necessary condition: 3n + 2 tiles total)
3. For each tile that appears **at least twice**, try using it as the **pair**:
   - Subtract 2 from its frequency
   - Check if all remaining frequencies are divisible by 3
   - Restore the frequency and try the next candidate
4. If any candidate pair produces a valid remainder, return `true`

#### Complexity

- **Time:** O(n + d²) where n = number of tiles, d = number of distinct digits (max 10) → effectively O(n)
- **Space:** O(d) → O(1) since d ≤ 10

---

#### TypeScript Implementation

```typescript
function isValidHand(hand: string): boolean {
  // Edge case: minimum valid hand is 5 tiles (1 triple + 1 pair)
  // and total must be 3n + 2 for some n >= 0
  if (hand.length < 2 || (hand.length - 2) % 3 !== 0) {
    return false;
  }

  // Build frequency map
  const freq = new Map<string, number>();
  for (const tile of hand) {
    freq.set(tile, (freq.get(tile) ?? 0) + 1);
  }

  // Try each tile as the pair
  for (const [tile, count] of freq) {
    if (count < 2) continue;

    // Use this tile as the pair
    freq.set(tile, count - 2);

    // Check if all remaining frequencies are divisible by 3
    if (allDivisibleByThree(freq)) {
      return true;
    }

    // Restore frequency for next iteration
    freq.set(tile, count);
  }

  return false;
}

function allDivisibleByThree(freq: Map<string, number>): boolean {
  for (const count of freq.values()) {
    if (count % 3 !== 0) return false;
  }
  return true;
}
```

---

#### Test Cases

```typescript
// Test runner
function test(hand: string, expected: boolean): void {
  const result = isValidHand(hand);
  const status = result === expected ? '✓ PASS' : '✗ FAIL';
  console.log(`${status}  isValidHand("${hand}") → ${result}  (expected: ${expected})`);
}

test("88844",     true);   // triple 8s + pair 4s
test("22333",     true);   // pair 2s + triple 3s
test("112233",    true);   // pairs become triples? No: 3+3+2=8 tiles, (8-2)%3=2 ✓
                            // try 1 as pair: {1:0, 2:2, 3:2} — 2%3≠0 ✗
                            // try 2 as pair: {1:2, 2:0, 3:2} — 2%3≠0 ✗
                            // try 3 as pair: {1:2, 2:2, 3:1} — 2%3≠0 ✗ → false
test("112233",    false);  // corrected
test("1112223",   true);   // triple 1s + triple 2s + pair... wait: 7 tiles, (7-2)%3=5%3≠0 → false
test("1112223",   false);  // corrected — 7 tiles invalid
test("11122233",  true);   // 8 tiles, (8-2)%3=0 ✓
                            // try 1 as pair: {1:0, 2:3, 3:2} — 2%3≠0 ✗
                            // try 2 as pair: {1:2, 2:1, 3:2} — not all %3=0 ✗
                            // try 3 as pair: {1:3, 2:3, 3:0} — all %3=0 ✓ → true
test("8884",      false);  // 4 tiles, (4-2)%3=2%3≠0 → false (length check catches it)
test("88",        false);  // only 2 tiles — pair but no triple → (2-2)%3=0 but
                            // need at least 1 triple? Actually 0 triples is valid...
                            // re-check: n>=0, so "88" → true (0 triples, 1 pair)
test("88",        true);   // 0 triples + 1 pair is valid per the problem
test("",          false);  // empty hand
test("1",         false);  // single tile
```

---

#### Edge Cases Handled

| Case | Handling |
|---|---|
| Empty string | Length check returns `false` immediately |
| Total length not `3n + 2` | Early return `false` |
| All same tile (e.g. `"11111"`) | Tries `1` as pair, remaining `3` → divisible by 3 → `true` |
| Multiple valid pair candidates | Loop finds the first valid one and returns `true` |
| No valid pair exists | All iterations fail, returns `false` |
| Single digit repeated (e.g. `"11"`) | 0 triples + 1 pair → `true` |

---

#### Alternative: Iterative (No Map Mutation)

If mutating the map feels fragile, here's a pure version using a plain array:

```typescript
function isValidHandV2(hand: string): boolean {
  if (hand.length < 2 || (hand.length - 2) % 3 !== 0) return false;

  const freq = new Array(10).fill(0);
  for (const ch of hand) freq[parseInt(ch)]++;

  for (let digit = 0; digit <= 9; digit++) {
    if (freq[digit] < 2) continue;

    freq[digit] -= 2;
    const valid = freq.every(count => count % 3 === 0);
    freq[digit] += 2;

    if (valid) return true;
  }

  return false;
}
```

This version is cleaner for interviews — a fixed-size array of 10 is easier to reason about than a Map.

---

## Summary

| Question | Core Concepts |
|---|---|
| Q1 — Recipe App Scaling | Caching, read replicas, indexing, horizontal scaling, async jobs |
| Q2 — Animation Cost | Autoscaling, spot instances, client offloading, batch processing |
| Q3 — Unreliable API | Circuit breaker, retry/backoff, fallback providers, graceful degradation |
| Q4 — Microcontroller Migration | Embedded constraints, ring buffers, wear levelling, watchdog, OTA |
| Q5 — TTL Video App | Capacity planning, transcoding, CDN, TTL deletion, compliance |
| Coding — Tile Hand | Frequency map, pair trial, divisibility check, O(n) time O(1) space |