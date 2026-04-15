# Bloom Filters and Probabilistic Data Structures

**Computer Science Fundamentals Series**

Bloom filters · HyperLogLog · Count-Min Sketch · Cuckoo filters · False positives · Cardinality estimation

*Mid-level software engineer track -- 20 slides*

---

## Table of Contents

1. [Why Probabilistic Data Structures?](#slide-02--why-probabilistic-data-structures)
2. [Bloom Filter -- Core Idea](#slide-03--bloom-filter--core-idea)
3. [Bloom Filter -- Insert & Query](#slide-04--bloom-filter--insert--query)
4. [False Positive Probability Analysis](#slide-05--false-positive-probability-analysis)
5. [Optimal Number of Hash Functions](#slide-06--optimal-number-of-hash-functions)
6. [Counting Bloom Filters](#slide-07--counting-bloom-filters)
7. [Cuckoo Filters](#slide-08--cuckoo-filters)
8. [Quotient Filters](#slide-09--quotient-filters)
9. [HyperLogLog -- Cardinality Estimation](#slide-10--hyperloglog--cardinality-estimation)
10. [HyperLogLog -- How It Works](#slide-11--hyperloglog--how-it-works)
11. [Count-Min Sketch](#slide-12--count-min-sketch)
12. [MinHash -- Jaccard Similarity](#slide-13--minhash--jaccard-similarity)
13. [Skip Lists](#slide-14--skip-lists)
14. [Reservoir Sampling](#slide-15--reservoir-sampling)
15. [Comparison Table](#slide-16--comparison-table)
16. [Applications -- Databases & Caches](#slide-17--applications--databases--caches)
17. [Applications -- Networks & Analytics](#slide-18--applications--networks--analytics)
18. [Design Trade-offs](#slide-19--design-trade-offs)
19. [Summary & Further Reading](#slide-20--summary--further-reading)

---

## Slide 02 -- Why Probabilistic Data Structures?

### The exactness tax

Exact data structures (hash tables, balanced trees, sorted arrays) guarantee correct answers, but their space and time costs grow linearly -- or worse -- with data size.

- Storing 1 billion 64-bit keys in a hash set costs ~8 GB of RAM minimum
- Exact distinct count over a stream requires storing every unique element
- Exact set intersection over two billion-element sets requires materialising both

### The probabilistic bargain

Trade a small, bounded error rate for dramatic savings in space and time:

- **Bloom filter** -- 1 billion membership checks in ~1.2 GB (vs ~8 GB exact)
- **HyperLogLog** -- count distinct over billions in ~12 KB
- **Count-Min Sketch** -- frequency estimation in fixed space regardless of stream size

### Key properties

- **One-sided errors** -- most structures produce false positives but never false negatives (or vice versa)
- **Sub-linear space** -- memory grows much slower than the data set
- **Tunable accuracy** -- you choose the error rate and allocate space accordingly

---

## Slide 03 -- Bloom Filter -- Core Idea

### Invented by Burton Bloom, 1970

A Bloom filter answers one question: "Is element x in the set?"

- **"Probably yes"** -- the element might be in the set (false positive possible)
- **"Definitely no"** -- the element is certainly not in the set (no false negatives)

### Components

| Component | Description |
|-----------|------------|
| **Bit array** | `m` bits, all initialised to 0 |
| **Hash functions** | `k` independent hash functions, each mapping an element to a position in `[0, m)` |

### Intuition

Think of it as `k` different hash tables compressed into a single bit array. Each hash function "votes" on whether the element is present. If all `k` votes say yes, we report membership -- but collisions can cause false positives.

> A Bloom filter with 10 bits per element and 7 hash functions achieves a false positive rate of ~0.8%.

---

## Slide 04 -- Bloom Filter -- Insert & Query

### Insert(x)

For each hash function `h_i` (where `i = 1..k`):
1. Compute `h_i(x) mod m`
2. Set `bit_array[h_i(x) mod m] = 1`

### Query(x)

For each hash function `h_i` (where `i = 1..k`):
1. Compute `h_i(x) mod m`
2. If `bit_array[h_i(x) mod m] == 0` for any `i`, return **definitely not in set**
3. If all bits are `1`, return **probably in set**

### Walk-through

```
m = 16 bits, k = 3 hash functions

Insert "cat":  h1("cat")=2, h2("cat")=7, h3("cat")=13
  Bit array: 0 0 1 0 0 0 0 1 0 0 0 0 0 1 0 0

Insert "dog":  h1("dog")=4, h2("dog")=7, h3("dog")=11
  Bit array: 0 0 1 0 1 0 0 1 0 0 0 1 0 1 0 0

Query "cat":   positions 2,7,13 all = 1 -> probably yes
Query "fish":  h1=2, h2=4, h3=9 -> bit 9 = 0 -> definitely no
Query "bird":  h1=2, h2=7, h3=11 -> all 1 -> FALSE POSITIVE
```

> Deletion is impossible in a standard Bloom filter -- clearing a bit might affect other elements.

---

## Slide 05 -- False Positive Probability Analysis

### Probability derivation

After inserting `n` elements into an `m`-bit array with `k` hash functions:

- Probability a specific bit is still 0 after all insertions: `(1 - 1/m)^(kn)`
- Approximation: `e^(-kn/m)`
- False positive probability: `(1 - e^(-kn/m))^k`

### Compact formula

```
FPP ≈ (1 - e^(-kn/m))^k
```

### Practical numbers

| Bits per element (m/n) | k (optimal) | False positive rate |
|------------------------|-------------|-------------------|
| 4 | 3 | ~4.7% |
| 8 | 6 | ~2.2% |
| 10 | 7 | ~0.82% |
| 16 | 11 | ~0.046% |
| 24 | 17 | ~0.0015% |

> Doubling the bits per element roughly squares the false positive rate (cuts it dramatically). Each additional ~1.44 bits per element halves the FPP at optimal k.

---

## Slide 06 -- Optimal Number of Hash Functions

### The trade-off

- **Too few hash functions** -- high false positive rate because fewer bits are checked
- **Too many hash functions** -- bit array fills up faster, increasing false positives
- **Optimal k** balances these two effects

### Formula

```
k_opt = (m/n) * ln(2) ≈ 0.693 * (m/n)
```

At optimal k, the false positive rate simplifies to:

```
FPP_min = (1/2)^k ≈ (0.6185)^(m/n)
```

### Practical sizing

Given a desired false positive rate `p` and expected number of elements `n`:

```
m = -n * ln(p) / (ln(2))^2
k = (m/n) * ln(2)
```

**Example:** 1 million elements, 1% false positive rate:
- `m = 9,585,059 bits` (~1.14 MB)
- `k = 7 hash functions`

> In practice, use two hash functions and derive k values via `h_i(x) = h1(x) + i * h2(x)` (Kirsch-Mitzenmacker optimisation). Performance of k=1 double-hashing matches k independent functions.

---

## Slide 07 -- Counting Bloom Filters

### Problem

Standard Bloom filters do not support deletion -- you cannot safely clear a bit without potentially removing other elements.

### Solution: replace bits with counters

Instead of a single bit per position, use a small counter (typically 4 bits):

- **Insert(x)** -- increment each counter at `h_i(x)` positions
- **Delete(x)** -- decrement each counter at `h_i(x)` positions
- **Query(x)** -- all counters at `h_i(x)` positions are > 0? -> probably in set

### Trade-offs

| Aspect | Standard Bloom | Counting Bloom |
|--------|---------------|----------------|
| **Space** | 1 bit per slot | 4 bits per slot (4x overhead) |
| **Deletion** | Not supported | Supported |
| **Counter overflow** | N/A | Rare with 4-bit counters (probability ~1.37 * 10^-15 per counter for optimal k) |
| **False negatives** | Impossible | Possible if counter wraps to 0 on overflow |

> Counting Bloom filters are used in network routing tables and distributed caches where elements must be removed.

---

## Slide 08 -- Cuckoo Filters

### Improvement over Bloom filters

Cuckoo filters store compact *fingerprints* in a cuckoo hash table. Named after cuckoo hashing, where elements can be displaced to alternative locations.

### Structure

- Array of *buckets*, each holding a small number of slots (typically 4)
- Each element is represented by a fingerprint (e.g., 8-12 bits)
- Two candidate buckets: `b1 = hash(x)`, `b2 = b1 XOR hash(fingerprint(x))`

### Operations

- **Insert** -- place fingerprint in `b1` or `b2`; if both full, evict an existing entry to its alternative bucket (cuckoo displacement)
- **Lookup** -- check if fingerprint exists in `b1` or `b2`
- **Delete** -- remove one matching fingerprint from `b1` or `b2`

### Advantages over Bloom filters

| Feature | Bloom Filter | Cuckoo Filter |
|---------|-------------|--------------|
| **Deletion** | Not supported | Supported natively |
| **Space efficiency** | Good | Better (at FPP < 3%) |
| **Lookup speed** | k memory accesses | 2 memory accesses |
| **Max load factor** | ~50% (bits set) | ~95% (bucket occupancy) |

> Cuckoo filters are the modern default for membership queries when deletion is needed or when the false positive rate target is below ~3%.

---

## Slide 09 -- Quotient Filters

### Cache-friendly alternative

Quotient filters store elements in a single compact hash table using quotienting:

- Hash each element to a `p`-bit fingerprint
- Split into `q`-bit **quotient** (bucket index) and `r`-bit **remainder** (stored value)
- `p = q + r`

### Key properties

- **Cache-friendly** -- all data in a contiguous array; no pointer chasing
- **Mergeable** -- two quotient filters can be merged in linear time (unlike Bloom filters)
- **Resizable** -- can be doubled by redistributing remainders
- **Supports deletion** -- with counting variants

### Metadata bits

Each slot uses 3 extra bits to handle collisions via linear probing:

| Bit | Purpose |
|-----|---------|
| `is_occupied` | A canonical slot for some element's quotient |
| `is_continuation` | Part of a run (same quotient) |
| `is_shifted` | Stored away from its canonical slot |

> Quotient filters shine in on-disk scenarios (LSM trees, SSTables) because they are cache-line-aligned and support efficient merging during compaction.

---

## Slide 10 -- HyperLogLog -- Cardinality Estimation

### Problem

Count the number of distinct elements in a massive stream. Exact counting requires `O(n)` space. HyperLogLog does it in `O(1)` -- about 12 KB for billions of elements.

### Core insight

If you hash elements uniformly, the maximum number of leading zeros in any hash tells you roughly `log2(n_distinct)`.

- If the longest run of leading zeros is `k`, estimate `2^k` distinct elements
- Single observation is noisy; HyperLogLog uses many independent observations

### Real-world usage

| System | Use Case |
|--------|---------|
| **Redis** | `PFADD` / `PFCOUNT` -- native HyperLogLog commands |
| **BigQuery** | `APPROX_COUNT_DISTINCT()` -- default uses HLL++ |
| **Presto / Trino** | `approx_distinct()` |
| **Spark** | `approx_count_distinct()` |
| **PostgreSQL** | `hll` extension |

> HyperLogLog is arguably the most widely deployed probabilistic data structure in production systems today.

---

## Slide 11 -- HyperLogLog -- How It Works

### Algorithm

1. Hash each element to a uniform binary string
2. Use the first `b` bits to select one of `m = 2^b` registers (buckets)
3. Count the leading zeros in the remaining bits; store the maximum in that register
4. Estimate: `E = alpha_m * m^2 * (sum of 2^(-register[j]))^(-1)`

### Corrections

- **Small range correction** -- when many registers are empty, use linear counting
- **Large range correction** -- when estimate exceeds `2^32 / 30`, apply hash collision correction
- **HLL++** (Google, 2013) -- improved bias correction and sparse representation for small cardinalities

### Space and accuracy

| Registers (m) | Bits (b) | Memory | Standard Error |
|---------------|----------|--------|---------------|
| 64 | 6 | 192 B | ~13% |
| 256 | 8 | 768 B | ~6.5% |
| 1024 | 10 | 3 KB | ~3.25% |
| 4096 | 12 | 12 KB | ~1.6% |
| 16384 | 14 | 48 KB | ~0.8% |

> Standard error is `1.04 / sqrt(m)`. Doubling registers halves the error but doubles memory.

### Mergeability

HyperLogLog registers can be merged by taking element-wise maximums -- perfect for distributed counting across shards.

---

## Slide 12 -- Count-Min Sketch

### Problem

Estimate the frequency of any element in a data stream using sub-linear space. Exact counting requires a counter per distinct element.

### Structure

- 2D array of counters: `d` rows (hash functions) x `w` columns (width)
- `d` independent hash functions, one per row

### Operations

- **Update(x, c)** -- for each row `i`, increment `counters[i][h_i(x)]` by `c`
- **Query(x)** -- return `min(counters[i][h_i(x)])` across all rows

### Error bounds

- Estimate is always >= true count (over-estimates due to collisions)
- With probability `1 - delta`: error <= `epsilon * ||stream||_1`
- Space: `w = ceil(e / epsilon)`, `d = ceil(ln(1 / delta))`

### Example

```
d=3 rows, w=8 columns

Update("cat", 5):  row 0 col 3 += 5, row 1 col 7 += 5, row 2 col 1 += 5
Update("dog", 3):  row 0 col 5 += 3, row 1 col 3 += 3, row 2 col 1 += 3

Query("cat"): min(col3=5, col7=5, col1=8) = 5  (exact in this case)
Query("dog"): min(col5=3, col3=3, col1=8) = 3  (exact in this case)
```

> Count-Min Sketch is used for heavy-hitter detection, traffic monitoring, and frequency capping in ad tech.

---

## Slide 13 -- MinHash -- Jaccard Similarity

### Problem

Estimate how similar two sets are without comparing them element by element. The Jaccard similarity of sets A and B is:

```
J(A, B) = |A ∩ B| / |A ∪ B|
```

### MinHash algorithm

1. Choose `k` independent hash functions
2. For each set, compute the minimum hash value under each function
3. The fraction of matching minimums estimates Jaccard similarity

### Why it works

```
Pr[min(h(A)) = min(h(B))] = J(A, B)
```

The probability that the minimum hash values agree exactly equals the Jaccard similarity. Averaging over `k` hash functions reduces variance.

### Locality-Sensitive Hashing (LSH)

Combine MinHash signatures into bands for fast approximate nearest-neighbour search:

- Divide signature into `b` bands of `r` rows
- Two items are candidate pairs if they agree in all rows of at least one band
- Tune `b` and `r` to control the S-curve threshold

### Applications

- **Near-duplicate detection** -- web pages, documents, code clones
- **Recommendation systems** -- user-item similarity at scale
- **Plagiarism detection** -- comparing document shingle sets

---

## Slide 14 -- Skip Lists

### A probabilistic alternative to balanced trees

Skip lists use randomised multi-level linked lists to achieve `O(log n)` expected search, insert, and delete -- without the complexity of AVL/red-black tree rotations.

### Structure

- **Level 0** -- a sorted linked list containing all elements
- **Level 1+** -- each element is promoted to the next level with probability `p` (typically 1/2)
- Search starts at the top level and drops down when overshooting

### Expected complexity

| Operation | Expected Time | Worst Case |
|-----------|--------------|-----------|
| Search | `O(log n)` | `O(n)` |
| Insert | `O(log n)` | `O(n)` |
| Delete | `O(log n)` | `O(n)` |
| Space | `O(n)` | `O(n log n)` |

### Why probabilistic?

The structure's balance depends on coin flips during insertion, not deterministic rebalancing. In expectation, the number of levels is `O(log n)` and the search path length is `O(log n)`.

### Real-world usage

- **Redis** sorted sets (ZSET) -- skip lists as the primary index structure
- **LevelDB / RocksDB** -- memtable implemented as a skip list
- **Lucene** -- concurrent skip lists for in-memory posting lists

> Skip lists are often preferred over balanced BSTs in concurrent settings because fine-grained locking is simpler with linked list nodes than with tree rotations.

---

## Slide 15 -- Reservoir Sampling

### Problem

Select `k` items uniformly at random from a stream of unknown length `n`, using only `O(k)` memory.

### Algorithm (Vitter's Algorithm R)

```
reservoir = first k items from stream

for i = k+1 to n:
    j = random integer in [1, i]
    if j <= k:
        reservoir[j] = stream[i]
```

### Why it works

Each element ends up in the reservoir with probability `k/n`. Proof by induction:
- Element `i` is chosen with probability `k/i`
- Previously selected elements survive with probability `1 - 1/i` each step
- Combined: `k/n` for all elements

### Variants

| Variant | Use Case |
|---------|---------|
| **Weighted reservoir sampling** | Non-uniform probability proportional to weights |
| **Distributed reservoir sampling** | Merge reservoirs from parallel streams |
| **Algorithm L** (Li, 1994) | Optimal: skips geometrically many elements, `O(k(1 + log(n/k)))` random numbers |

### Applications

- Sampling log lines for analysis without storing the full stream
- A/B test random assignment from an unbounded user stream
- Random row selection from a database table scan

---

## Slide 16 -- Comparison Table

### Probabilistic data structures at a glance

| Structure | Space | Error Type | Supports Delete | Mergeable | Primary Operation |
|-----------|-------|-----------|----------------|-----------|------------------|
| **Bloom Filter** | `O(n)` bits | False positive | No | OR | Membership |
| **Counting Bloom** | `O(n)` counters | False positive | Yes | Add counters | Membership |
| **Cuckoo Filter** | `O(n)` fingerprints | False positive | Yes | No | Membership |
| **Quotient Filter** | `O(n)` remainders | False positive | Yes (variant) | Yes | Membership |
| **HyperLogLog** | `O(1)` ~12 KB | +/- ~1.6% | N/A | Max | Cardinality |
| **Count-Min Sketch** | `O(1/epsilon)` | Over-estimate | N/A | Add | Frequency |
| **MinHash** | `O(k)` per set | +/- similarity | N/A | Union | Similarity |
| **Skip List** | `O(n)` | None (exact) | Yes | No | Ordered ops |
| **Reservoir Sampling** | `O(k)` | Sampling variance | N/A | Merge | Random sample |

> The "space" column shows asymptotic behaviour. The key insight: membership filters use bits, not pointers -- orders of magnitude smaller than exact sets.

---

## Slide 17 -- Applications -- Databases & Caches

### Databases

- **LSM-tree SSTables** -- Bloom filters on each SSTable avoid unnecessary disk reads during point lookups. LevelDB, RocksDB, Cassandra, HBase all use them.
- **PostgreSQL** -- uses Bloom indexes (extension) for multi-column equality filters
- **BigQuery / Presto** -- HyperLogLog for `APPROX_COUNT_DISTINCT` over petabyte-scale tables
- **Merge joins** -- Count-Min Sketch to estimate join selectivity in query planners

### Caches

- **CDN edge caches** -- Bloom filter gates whether to fetch from origin; avoids cache pollution for one-hit-wonder requests
- **Browser safe browsing** -- Chrome uses a Bloom filter to check URLs against a malware list locally before making a network request
- **Squid proxy** -- cache digests are Bloom filters exchanged between sibling proxies for cooperative caching

### Distributed systems

- **Cassandra** -- Bloom filter per SSTable to determine which SSTables might contain a key
- **Bitcoin SPV clients** -- Bloom filters for lightweight transaction filtering without downloading the full blockchain
- **HDFS** -- HyperLogLog for file-system analytics across petabytes

---

## Slide 18 -- Applications -- Networks & Analytics

### Network routers and security

- **Packet deduplication** -- Bloom filters detect duplicate packets in network switches at line rate
- **DDoS mitigation** -- Count-Min Sketch identifies heavy-hitter source IPs for rate limiting
- **Firewall ACLs** -- Bloom filters for fast IP/port membership checks before consulting full rules

### Spell checkers and NLP

- **Spell checkers** -- store a dictionary in a Bloom filter; check candidate words without loading the full dictionary into memory
- **Web crawlers** -- track visited URLs with a Bloom filter to avoid re-crawling (Google, Bing)
- **Shingling for plagiarism** -- MinHash over word n-grams to detect near-duplicate documents

### Analytics and streaming

- **Unique visitor counting** -- HyperLogLog for real-time unique user counts (Redis `PFCOUNT`)
- **Frequency capping in ads** -- Count-Min Sketch to limit how many times a user sees an ad
- **Sampling dashboards** -- reservoir sampling for real-time percentile estimation over high-volume event streams
- **Stream joins** -- Bloom filters to pre-filter one side of a stream-stream join, reducing wasted processing

---

## Slide 19 -- Design Trade-offs

### Choosing the right structure

| Scenario | Recommended | Reasoning |
|----------|------------|-----------|
| Membership queries, no deletion | **Bloom filter** | Simplest, well-understood, minimal space |
| Membership with deletion | **Cuckoo filter** | Better space efficiency at low FPP, native delete |
| On-disk membership (LSM compaction) | **Quotient filter** | Cache-friendly, mergeable, resizable |
| Distinct count over stream | **HyperLogLog** | 12 KB for billions of elements, mergeable |
| Frequency estimation | **Count-Min Sketch** | Bounded error, constant space, additive merge |
| Set similarity at scale | **MinHash + LSH** | Sub-linear candidate generation |
| Ordered data, concurrent access | **Skip list** | Simple locking, no rebalancing |
| Random sample from stream | **Reservoir sampling** | O(k) memory, provably uniform |

### Common pitfalls

- **Under-sizing Bloom filters** -- leads to unacceptable false positive rates as the filter fills
- **Ignoring hash quality** -- poor hash functions create correlated bits, inflating FPP
- **Forgetting about growth** -- standard Bloom filters cannot be resized; use scalable Bloom filters or plan capacity
- **Treating estimates as exact** -- probabilistic structures give estimates; propagate uncertainty through your system

---

## Slide 20 -- Summary & Further Reading

### Key takeaways

- Probabilistic data structures trade bounded, tuneable error for dramatic space and time savings
- Bloom filters remain the workhorse for membership queries -- simple, fast, well-analysed
- HyperLogLog is the standard for cardinality estimation in databases, analytics, and streaming systems
- Count-Min Sketch is the go-to for frequency estimation in streaming and network monitoring
- Cuckoo filters improve on Bloom filters when deletion is needed or when FPP targets are below 3%
- MinHash enables similarity estimation at scale for document deduplication and recommendation
- Skip lists prove that randomisation can replace complex balancing logic with simpler, concurrent-friendly structures
- Always size your structures based on expected data volume and acceptable error rate -- under-provisioning is the most common mistake

### Recommended reading

| Source | Description |
|--------|------------|
| **Bloom (1970)** | "Space/Time Trade-offs in Hash Coding with Allowable Errors" -- the original paper |
| **Flajolet et al. (2007)** | "HyperLogLog: the analysis of a near-optimal cardinality estimation algorithm" |
| **Fan et al. (2014)** | "Cuckoo Filter: Practically Better Than Bloom" -- CMU paper introducing cuckoo filters |
| **Cormode & Muthukrishnan (2005)** | "An Improved Data Stream Summary: The Count-Min Sketch and its Applications" |
| **Mitzenmacher & Upfal** | *Probability and Computing* -- rigorous treatment of randomised algorithms and data structures |
| **Redis Documentation** | [redis.io/docs](https://redis.io/docs/) -- HyperLogLog, Bloom filter module, sorted sets (skip lists) |
