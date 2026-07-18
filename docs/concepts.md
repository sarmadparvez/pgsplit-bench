# Concepts: from monolithic Postgres to compute/storage separation

This is the "read this first" guide for `pgsplit-bench`. The benchmark exists to test a set of
claims made about a newer database architecture; this document explains the theory those claims
rest on, building up from Postgres fundamentals so the experiments make sense before you run them.

It is written to be read top to bottom. Every non-obvious factual claim is footnoted to a primary
source (PostgreSQL docs, the original research papers, or the engineering write-ups from the teams
that built these systems).

**The path we'll take:**

1. [How a classic Postgres stores and writes data](#1-how-a-classic-postgres-stores-and-writes-data) — pages, the buffer, and the durability problem
2. [Write-Ahead Logging (WAL)](#2-write-ahead-logging-wal) — the mechanism that makes writes durable
3. [Checkpoints](#3-checkpoints) — how dirty pages get to disk and WAL gets recycled
4. [Torn pages and full-page writes](#4-torn-pages-and-full-page-writes) — the safety feature the benchmark centers on
5. [The monolith, and what it costs](#5-the-monolith-and-what-it-costs) — why copies, scaling, and idle are expensive
6. [The shift: separating compute from storage](#6-the-shift-separating-compute-from-storage) — the idea, and where you've seen it before
7. [How Neon actually works](#7-how-neon-actually-works) — safekeepers, pageserver, object storage
8. [Why full-page writes can be turned off here](#8-why-full-page-writes-can-be-turned-off-here) — the flagship result
9. [Branching and copy-on-write](#9-branching-and-copy-on-write) — instant clones, and why
10. [Scale-to-zero and cold starts](#10-scale-to-zero-and-cold-starts)
11. [What the monolith still wins](#11-what-the-monolith-still-wins) — the honest counterpoint
12. [Concept → experiment map](#12-concept--experiment-map)
13. [Glossary](#13-glossary)
14. [References](#14-references)

---

## 1. How a classic Postgres stores and writes data

Postgres stores table and index data in files on disk, divided into fixed-size **pages** (also
called blocks) of **8 KB** by default.[^pg-page] A page is the unit of I/O: Postgres reads and
writes whole 8 KB pages, never individual rows.

It does not read and write those pages directly from disk on every query. Instead it keeps a
region of RAM called **shared buffers** (the `shared_buffers` setting) that caches pages. When a
query needs a page, Postgres looks in shared buffers first; on a miss it reads the page from disk
into a buffer. When a query *modifies* a row, it modifies the page **in memory** and marks that
buffer **dirty** — changed in RAM but not yet written back to disk.

```
Reads and writes go through RAM, not straight to disk

        a query wants a row on page 42
                │
                ▼
   ┌────────────────────────────────┐
   │ shared_buffers (RAM)           │
   │  ┌────┐ ┌────┐ ┌────┐ ┌────┐   │   * = dirty: changed in
   │  │ 17 │ │ 42*│ │ 08 │ │ 91 │   │       RAM, not yet on disk
   │  └────┘ └────┘ └────┘ └────┘   │
   └───────▲────────────┬───────────┘
      miss │ read       │ dirty pages written back
           │            ▼ later, in bulk (at a checkpoint)
   ┌────────────────────────────────┐
   │ data files on disk (8 KB pages)│
   │   …08…17…42…91…                │
   └────────────────────────────────┘
```

This is fast, but it creates a durability problem. If the server loses power right after a
transaction commits, the change lives only in a dirty page in RAM, which is gone. Yet the "D" in
ACID says a committed transaction must survive a crash. Flushing every modified 8 KB page to disk
at every commit would be correct but slow — random 8 KB writes scattered across the data files,
synchronously, on the commit path.

The way out of this bind is the single most important idea in the whole guide.

## 2. Write-Ahead Logging (WAL)

Postgres solves durability with **Write-Ahead Logging**: before any change is written to the data
files, a record describing that change is written to a sequential log — the **WAL** — and flushed
to durable storage.[^pg-wal-intro] The rule is in the name: the log is written *ahead of* the data.

The key consequence is stated directly in the PostgreSQL documentation: because the WAL record is
safely on disk, **the data pages themselves do not have to be flushed to disk on every
commit.**[^pg-wal-intro] If the server crashes, Postgres replays ("redoes") the WAL on restart to
reconstruct any changes that hadn't yet made it into the data files. The data pages can be written
back lazily, later, in bulk.

Why this is a win:

- **Sequential, not random.** WAL is appended to the end of a log. One sequential fsync at commit
  replaces many scattered random writes to the data files.[^pg-wal-intro]
- **One flush covers many changes.** A single transaction touching many pages produces WAL that is
  flushed once.

Every WAL record has a unique, monotonically increasing address called an **LSN** (Log Sequence
Number) — essentially the byte offset of that record in the log stream.[^pg-wal-internals] The LSN
is how Postgres (and, later, replicas and the Neon storage layer) answer the question "which
version of this page am I looking at?" Hold onto that idea; it's the hinge the separated
architecture turns on.

WAL was built for crash recovery, but the same log turns out to be a general-purpose change
stream. Postgres **streaming replication** works by shipping the primary's WAL to standby servers,
which replay it to stay in sync.[^pg-ha] The log *is* the description of everything that changed —
a fact that the next architecture takes much further.

```
Classic write path (monolith)

  client
    │  COMMIT
    ▼
 ┌─────────────────────────────────────────────┐
 │ Postgres process                            │
 │                                             │
 │  modify page in shared_buffers  (dirty)     │
 │  append change to WAL                       │
 │         │                                   │
 │         ▼  fsync at commit                  │
 │   ┌───────────┐        ┌───────────────┐    │
 │   │  WAL log  │        │  data files   │    │
 │   │ (disk)    │        │ (disk, 8 KB   │    │
 │   └───────────┘        │  pages)       │    │
 │                        └───────────────┘    │
 │                        flushed later, in    │
 │                        bulk, at checkpoints │
 └─────────────────────────────────────────────┘
```

## 3. Checkpoints

If data pages are flushed "later, in bulk," when exactly? At a **checkpoint**. A checkpoint is a
point at which Postgres guarantees that all dirty pages up to a certain LSN have been written to
the data files.[^pg-checkpoints] After a checkpoint, the WAL written before it is no longer needed
for crash recovery and can be recycled or removed.

```
Checkpoints bound how much WAL a crash forces you to replay

  WAL: ──▶───────────────▼───────────────────▼───────────▶ now
                     checkpoint            checkpoint
                        (A)                   (B)          ✗ crash
                                                           │
        on restart, replay WAL only from B ───────────────┘
        (everything ≤ B is already in the data files;
         WAL written before B can be recycled)
```

Checkpoints matter for two reasons that come up later:

- They **bound crash recovery time**: on restart, Postgres only has to replay WAL written since the
  last checkpoint, not the entire history.[^pg-checkpoints]
- They are the reference point for the full-page-write mechanism in the next section — the "first
  time a page is touched after a checkpoint" is a specific, important moment.

## 4. Torn pages and full-page writes

Here is the subtle failure mode that experiment 1 is built around.

Postgres writes 8 KB pages, but the disk underneath deals in smaller **sectors** — commonly 512
bytes, so an 8 KB page is ~16 sectors. A single 8 KB write is therefore not atomic at the hardware
level. If power is lost mid-write, some sectors of the page may be updated while others still hold
old data — a **torn page**.[^pg-wal-reliability] The PostgreSQL documentation puts it plainly:

> "PostgreSQL typically writes 8192 bytes, or 16 sectors, at a time … the process of writing could
> fail due to power loss at any time, meaning some of the 512-byte sectors were written while
> others were not."[^pg-wal-reliability]

A torn page is dangerous because WAL records are usually *deltas* — "change these 12 bytes at this
offset." Replaying a delta on top of a half-written page produces silent corruption. To prevent
this, Postgres uses **full-page writes** (`full_page_writes`, on by default): the first time a page
is modified after a checkpoint, Postgres writes a **complete copy of the 8 KB page** into the WAL,
not just the delta.[^pg-wal-reliability][^pg-wiki-fpw] During recovery it can restore the whole page
from that image, then safely apply subsequent deltas.

```
An 8 KB page is ~16 disk sectors; a crash mid-write can tear it

  8 KB page  =  [s0][s1][s2][s3] … [s15]      (512-byte sectors)

  power lost mid-write:
        written        │      NOT written
      ┌────┬────┬────┬─┴──┐        ┌────┬────┬────┐
      │ s0 │ s1 │ s2 │ s3 │  …     │s13 │s14 │s15 │
      │new │new │new │new │        │old │old │old │   ← half new, half old
      └────┴────┴────┴────┘        └────┴────┴────┘
                       = TORN PAGE

  Replaying a small WAL delta onto a torn page → silent corruption.

  full_page_writes: on the FIRST change after a checkpoint, copy the
  WHOLE page into the WAL so recovery can rebuild it before deltas apply:

  WAL: … [ ██ full 8 KB image of page 42 ██ ] [δ][δ][δ] …
              ▲ heavy, but makes the page recoverable
```

This is correct and safe, but it is **expensive**: those full 8 KB images dominate WAL volume on
write-heavy workloads. The documentation notes the escape hatch — if the storage layer *guarantees*
that partial page writes can never be observed, full-page writes can be turned off:

> "If you have file-system software that prevents partial page writes (e.g., ZFS), you can turn off
> this page imaging by turning off the `full_page_writes` parameter."[^pg-wal-reliability]

That single sentence is the seam the separated architecture pries open. Keep it in mind through
sections 7 and 8.

## 5. The monolith, and what it costs

Everything so far describes a **monolith**: one machine (or one VM/container) where the query
engine, the shared-buffer cache, the WAL, and the data files all live together on local disk. A
classic managed instance — Azure Database for PostgreSQL Flexible Server, Amazon RDS, or
`postgres:17` in a container — is this shape. It is simple, well-understood, and for single-node
write latency it is hard to beat, because a commit is one local `fsync`.

But fusing compute and storage into one node has structural costs:

- **Copies are physical and scale with size.** To make a second copy of a database you must move
  the bytes — `pg_dump | pg_restore`, `pg_basebackup`, or a file copy. A 10 GB database takes
  roughly ten times as long to copy as a 1 GB one. There is no cheap "give me another one of these."
- **Compute and storage scale together.** Need more CPU for a busy hour? You move to a bigger node,
  and its disk comes along whether you needed more or not (and vice versa).
- **No scale-to-zero.** The node is either running (and billed) or off (and unreachable). An idle
  database still occupies a live machine.

```
Copying a monolith database means moving all its bytes

   1 GB db   ─ pg_dump | pg_restore ─▶  ▓                    t
  10 GB db   ─ pg_dump | pg_restore ─▶  ▓▓▓▓▓▓▓▓▓▓          ~10t
                                        └── copy time grows with size ──┘
```

None of these are bugs. They are consequences of the architecture. The question `pgsplit-bench`
asks is: what happens if you *unfuse* compute from storage — and what do you pay for it?

## 6. The shift: separating compute from storage

**If you come from the data-warehouse world, you already know the punchline.** The defining move of
the modern cloud warehouse — Snowflake, BigQuery, Databricks — was to **decouple storage from
compute** so each scales independently and compute can be spun up and down on demand. Snowflake's
2016 SIGMOD paper describes exactly this: a multi-cluster, shared-data architecture with storage and
compute as separate, independently elastic tiers.[^snowflake] That is why you can resize a warehouse
without moving data, and suspend it when idle.

The harder problem is doing the same thing for a **transactional** (OLTP) database, where you can't
relax durability or accept warehouse-style latency. The breakthrough came from **Amazon Aurora**,
whose 2017 SIGMOD paper is built on a slogan that captures the whole idea: **"the log is the
database."**[^aurora] Instead of the compute node writing data pages to storage, it ships only WAL
(redo log) records to a distributed storage tier, and **pushes the work of applying that log — of
materializing pages — down into storage.**[^aurora] The compute node is thereby freed from owning
the durable data files at all.

**Neon** is the open-source Postgres implementation of this idea, and it is the separated-storage
target in this benchmark. It keeps *unmodified-enough* Postgres as the compute layer but replaces
the storage engine with a purpose-built, disaggregated one.[^neon-arch] Databricks **Lakebase** is a
managed product built on Neon — Databricks acquired Neon in 2025 — which is why Neon and Databricks
publish nearly identical findings about this architecture.[^databricks-neon][^databricks-lakebase]

```
Monolith                          Separated (Neon)

┌───────────────────┐             ┌───────────────────┐
│ Postgres          │             │ Postgres (compute)│  stateless
│  ├ query engine   │             │  ├ query engine   │  streams WAL out,
│  ├ buffers        │             │  ├ buffers        │  caches pages locally
│  ├ WAL   ──┐      │             │  └ WAL ──┐        │
│  └ data files◄┘   │             └──────────┼────────┘
│      (local disk) │                        │ WAL over network
└───────────────────┘                        ▼
                                   ┌───────────────────┐
compute + storage fused           │ Safekeepers (x3)  │  durable WAL,
one fsync to local disk           │  Paxos quorum     │  quorum commit
                                   └─────────┬─────────┘
                                             │ WAL stream
                                             ▼
                                   ┌───────────────────┐
                                   │ Pageserver        │  materializes pages
                                   │  (base + deltas)  │  from WAL
                                   └─────────┬─────────┘
                                             │ layer files
                                             ▼
                                   ┌───────────────────┐
                                   │ Object storage    │  (S3 / MinIO)
                                   │  (durable, cheap) │
                                   └───────────────────┘
```

## 7. How Neon actually works

Neon splits Postgres into a stateless **compute** layer and a durable **storage** layer with two
components.[^neon-arch]

**Compute** is a Postgres process. Crucially, it does **not** own a durable local data directory. It
still has shared buffers and a local cache, but the authoritative copy of the data does not live on
its disk.[^neon-arch][^neon-fpw]

**Safekeepers** provide durable WAL. Instead of flushing WAL to a local file, the compute node
**streams WAL records over the network to a quorum of safekeepers** (typically three). A transaction
is durable once a **quorum acknowledges** the WAL, coordinated with a Paxos-based protocol.[^neon-arch]
This is the separated architecture's replacement for the monolith's local `fsync` — and, as
section 11 notes, it is also where the monolith's latency advantage comes from.

**Pageserver** is where "the log is the database" becomes concrete. The pageserver consumes the WAL
stream and **materializes page versions** by combining a **base image** of a page with the stream of
**delta** (WAL) records that apply to it.[^neon-arch] It organizes this history into layer files and
uploads them to **object storage** (S3, or MinIO for a local self-hosted stack), which is the cheap,
durable bottom of the stack.[^neon-arch]

```
The pageserver rebuilds any version of a page from a base image + WAL deltas

  base image               deltas (WAL) applied in LSN order
  of page 42         ┌───────┬───────┬───────┬───────┐
  @ LSN 100     +    │ Δ@105 │ Δ@112 │ Δ@130 │ Δ@148 │  ──▶  page 42 @ LSN 148
  ──────────         └───────┴───────┴───────┴───────┘       (the exact version
                                                              the reader asked for)
```

**Read path.** The compute node prefers local access: memory first, then its local NVMe cache. Only
on a miss does it ask the pageserver for a page, which **reconstructs the exact version** (identified
by LSN) from the base image plus deltas and returns it over the network.[^neon-arch] This network
round trip on a cold read is the read-side cost of separation.

```
Compute reads local-first; only a miss pays for the network

  needs page 42
        │
        ▼
  ┌────────────┐  miss  ┌────────────┐  miss   ┌──────────────────┐
  │ RAM        │──────▶ │ local NVMe │───────▶ │ pageserver       │
  │ buffers    │        │ cache      │ network │ reconstruct page │
  └────────────┘        └────────────┘ ◀────── │ @ LSN, return it │
     hit? done             hit? done    page    └──────────────────┘
```

## 8. Why full-page writes can be turned off here

Now the payoff. Recall section 4: full-page writes exist to defend against **torn pages on local
disk**, and the docs say they can be disabled if the storage layer makes partial page writes
impossible.[^pg-wal-reliability]

In Neon, the compute node **never persists a canonical data page to local disk at all** — it streams
WAL to the safekeeper quorum, and the pageserver owns page materialization. As Neon puts it: "there
is no local-disk page to tear, the failure mode FPW was designed to prevent simply does not
exist."[^neon-fpw] So `full_page_writes` can be turned **off**, and the fat 8 KB page images vanish
from the WAL stream.

```
Turning off full-page writes empties the WAL of fat page images

  full_page_writes = ON            full_page_writes = OFF  (Neon)
  ┌───────────────────────┐        ┌───────────────────────┐
  │ [██ 8 KB image ██]    │        │ [δ]                   │
  │ [δ][δ][δ]             │        │ [δ][δ][δ]             │
  │ [██ 8 KB image ██]    │        │ [δ][δ]                │
  │ [δ][δ]                │        │ [δ]                   │
  └───────────────────────┘        └───────────────────────┘
      ~58 KB WAL / txn                < 4 KB WAL / txn   (~94% less)

  δ = small delta record       ██ = full 8 KB page image
```

The measured effect, from Neon's own engineering write-up, is large:

- **WAL per transaction dropped from ~58 KB to under 4 KB — a ~94% reduction.**[^neon-fpw]
- **Write throughput improved up to ~4.5× on 32-vCPU instances** (about 2.8× on 16-vCPU, ~20% on
  4-vCPU — the benefit grows with core count).[^neon-fpw]
- **p99 read latencies dropped ~30–50%**, partly because the WAL stream the pageserver ingests is far
  smaller.[^neon-fpw]

This is the origin of the **"5× faster writes"** headline Databricks uses for Lakebase; their write-up
describes the same mechanism.[^databricks-5x] There is one important catch, and both teams call it
out: naively removing full-page images makes a page's **delta chain** grow without bound, which would
slow reads (the pageserver has more deltas to replay). The fix is **image-generation pushdown** — the
**pageserver** generates a fresh full-page image once a page accumulates too many deltas, instead of
the compute node paying for it in WAL on every post-checkpoint touch.[^neon-fpw][^databricks-5x] The
cost of correctness didn't disappear; it moved from the write path into the storage layer, where it's
cheaper.

```
The pageserver (not the WAL) now caps how long a delta chain grows

  page 42:  base ─Δ─Δ─Δ─Δ─Δ─Δ─Δ─Δ─▶   chain too long?
                                       │
                     pageserver mints a fresh base image
                                       ▼
            new base ─Δ─Δ─▶     reads stay fast, and compute never
                                paid for it in WAL on the write path
```

Two honesty notes for anyone reading these numbers:

- These are **vendor benchmarks** on cloud hardware. The whole point of `pgsplit-bench` is to
  reproduce the *shape* of the result yourself — that's why experiment 1 includes a control that
  toggles `full_page_writes` on the monolith, isolating exactly this mechanism.
- The "5×" is workload- and hardware-dependent (write-heavy, high core count). Expect a smaller
  multiplier on a laptop.

## 9. Branching and copy-on-write

Because the pageserver already stores page history keyed by LSN, making a **branch** is nearly free.
A Neon branch is a **copy-on-write clone**: it shares all of the parent's existing data up to the
branch point, and only **writes after the branch are stored as new deltas**.[^neon-branching] Nothing
is copied at creation time, so a branch is effectively **instant regardless of database size**, and
creating one **puts no load on the parent**.[^neon-branching]

```
A branch shares the parent's history up to an LSN; only its writes diverge

  parent (main)  ●───●───●───●───●───●───●──▶  keeps advancing
                             │
                 branch point (an LSN)     ← 0 bytes copied; history shared
                             │
  branch                     └──◇───◇───◇──▶  only these writes are new
                                              deltas, owned by the branch
```

Contrast this with the monolith, where "make me a test copy" means physically moving the bytes
(`pg_dump | pg_restore` or `pg_basebackup`) — time and space that grow with the database. That
contrast is the hero chart of this project (experiment 3): the monolith's copy time climbs roughly
linearly with size while the separated target stays flat at seconds. Experiment 4 then measures the
**copy-on-write** claim directly — a fresh branch consumes almost no extra storage until you start
mutating rows, at which point its footprint grows in proportion to the *changes*, not the total
database size.[^neon-branching]

## 10. Scale-to-zero and cold starts

If compute is stateless and all durable state lives in safekeepers, the pageserver, and object
storage, then an idle database can **suspend its compute entirely** and resume later without losing
data. Neon and Lakebase both offer this **scale-to-zero**.[^databricks-lakebase] The monolith has no
equivalent — its node is always on or fully off — so the fair comparison is not "who starts faster"
but "what does an always-on node cost when nothing is happening." Experiment 5 measures the other
side of that trade: the **cold-start latency** from a scaled-to-zero compute to its first successful
query, which is the price you pay for not running (and paying for) an idle machine.

```
Idle compute suspends; durable state stays in the storage tier

  compute:  ▓▓ active ▓▓ │····· suspended, not billed ·····│ ▓ cold start ▓ ▶
                                                            └─▶ first query
  storage:  ●── WAL (safekeepers) · pages (pageserver) · objects (S3) ───────▶
            durable the whole time — nothing to rebuild from scratch

  experiment 5 measures the gap:  resume ──▶ first successful query
```

## 11. What the monolith still wins

A guide that only listed the separated architecture's wins would be marketing, not a benchmark spec.
The separation is a **trade-off**, and the monolith wins real things:

- **Single-node write latency.** A monolith commit is one local `fsync`. The separated target must
  send each commit's WAL across an (intra-host, in this benchmark) network hop and wait for a
  **safekeeper quorum** to acknowledge.[^neon-arch] That extra hop can make individual commit latency
  *higher* than the monolith's — this is the cost of durability-via-replication, not a
  misconfiguration.
- **Cold reads.** A page that isn't in the compute node's local cache costs a **pageserver round
  trip** to reconstruct and fetch.[^neon-arch] The monolith reads it from local disk (or its own
  buffer cache) with no network involved.
- **Operational simplicity.** One process, one disk, no safekeeper quorum or pageserver to run.

```
Where the monolith's write-latency edge comes from

  monolith commit:   Postgres ──fsync──▶ local disk        ✓ durable (one local hop)

  separated commit:  Postgres ──WAL over network──▶ safekeeper ┐
                                                    safekeeper ├ wait for quorum ack
                                                    safekeeper ┘
                     ✓ durable only after a quorum responds — one extra network hop
```

This is why the README insists on trusting **curves, not constants**: the durable, reproducible
takeaways are the *shapes* — flat-vs-linear branch time, and the WAL collapse when full-page writes
are disabled — not laptop-specific absolute numbers, and certainly not a single "X is faster" verdict.

## 12. Concept → experiment map

| Concept (section) | Experiment in the README |
| --- | --- |
| Full-page writes & WAL volume (§4, §8) | **1** — write throughput + WAL efficiency, with the FPW on/off control |
| Buffer cache, read path, pageserver round trips (§1, §7, §11) | **2** — read latency under concurrency |
| Copy-on-write branching vs physical copy (§9) | **3** — branch vs `pg_dump`/`pg_restore` by dataset size |
| Copy-on-write storage growth (§9) | **4** — storage footprint of a copy after mutation |
| Stateless compute & scale-to-zero (§10) | **5** — cold start / cost of idle |

## 13. Glossary

- **Page / block** — the fixed 8 KB unit Postgres reads and writes.[^pg-page]
- **Shared buffers** — Postgres's in-RAM cache of pages; modified pages are "dirty."
- **WAL (Write-Ahead Log)** — the sequential log of changes, written before data files, that provides
  durability and crash recovery.[^pg-wal-intro]
- **LSN (Log Sequence Number)** — a monotonically increasing address identifying a position in the
  WAL; used to name page versions.[^pg-wal-internals]
- **Checkpoint** — a point guaranteeing all dirty pages up to some LSN are on disk, after which older
  WAL can be recycled.[^pg-checkpoints]
- **Torn page** — a page left half-written by a crash mid-write, because 8 KB isn't atomic at the
  sector level.[^pg-wal-reliability]
- **Full-page write (FPW)** — writing a whole page image into WAL on first modification after a
  checkpoint, to survive torn pages.[^pg-wal-reliability]
- **Safekeeper** — Neon component that stores WAL durably via a quorum-acknowledged (Paxos)
  protocol.[^neon-arch]
- **Pageserver** — Neon component that materializes page versions from a base image plus WAL deltas
  and persists them to object storage.[^neon-arch]
- **Base image / delta** — a materialized page snapshot, plus the WAL records applied on top of it.[^neon-arch]
- **Branch** — a copy-on-write clone of a database at an LSN; shares parent data, stores only its own
  writes as deltas.[^neon-branching]
- **Scale-to-zero** — suspending stateless compute while durable state persists in the storage tier.[^databricks-lakebase]

## 14. References

**PostgreSQL documentation**
- Database page layout / 8 KB blocks — <https://www.postgresql.org/docs/current/storage-page-layout.html>
- Write-Ahead Logging, introduction — <https://www.postgresql.org/docs/current/wal-intro.html>
- Reliability: torn pages & `full_page_writes` — <https://www.postgresql.org/docs/current/wal-reliability.html>
- `full_page_writes` parameter — <https://www.postgresql.org/docs/current/runtime-config-wal.html#GUC-FULL-PAGE-WRITES>
- WAL internals / LSN — <https://www.postgresql.org/docs/current/wal-internals.html>
- WAL configuration & checkpoints — <https://www.postgresql.org/docs/current/wal-configuration.html>
- High availability & streaming replication — <https://www.postgresql.org/docs/current/high-availability.html>
- Wiki: Full page writes — <https://wiki.postgresql.org/wiki/Full_page_writes>

**Foundational papers**
- Verbitski et al., *Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases*, SIGMOD 2017 — <https://dl.acm.org/doi/10.1145/3035918.3056101> ([open PDF](http://hpts.ws/papers/2017/aurora.pdf))
- Dageville et al., *The Snowflake Elastic Data Warehouse*, SIGMOD 2016 — <https://dl.acm.org/doi/10.1145/2882903.2903741>

**Neon (the separated-storage engine used here)**
- Architecture overview — <https://neon.com/docs/introduction/architecture-overview>
- Branching (copy-on-write) — <https://neon.com/docs/introduction/branching>
- "Everyone gets faster writes: We turned off FPW's in Neon" — <https://neon.com/blog/turning-off-fpw-for-faster-writes>
- Architecture decisions in Neon — <https://neon.com/blog/architecture-decisions-in-neon>

**Databricks Lakebase (managed product built on Neon)**
- How lakebase architecture delivers 5× faster Postgres writes — <https://www.databricks.com/blog/how-lakebase-architecture-delivers-5x-faster-postgres-writes>
- Databricks + Neon — <https://www.databricks.com/blog/databricks-neon>
- Announcing Lakebase Public Preview — <https://www.databricks.com/blog/announcing-lakebase-public-preview>

---

[^pg-page]: PostgreSQL Documentation, "Database Page Layout" — the default page/block size is 8 KB (`BLCKSZ`), and I/O is done in whole pages. <https://www.postgresql.org/docs/current/storage-page-layout.html>

[^pg-wal-intro]: PostgreSQL Documentation, "Write-Ahead Logging (WAL)": WAL records are flushed before data-file changes, so data pages need not be flushed on every commit, and a single sequential log flush replaces many random data-file writes. <https://www.postgresql.org/docs/current/wal-intro.html>

[^pg-wal-internals]: PostgreSQL Documentation, "WAL Internals": describes the Log Sequence Number (LSN) as the byte position in the WAL stream. <https://www.postgresql.org/docs/current/wal-internals.html>

[^pg-checkpoints]: PostgreSQL Documentation, "WAL Configuration": checkpoints flush dirty buffers and allow older WAL to be recycled; recovery replays WAL only since the last checkpoint. <https://www.postgresql.org/docs/current/wal-configuration.html>

[^pg-wal-reliability]: PostgreSQL Documentation, "Reliability" (§ Reliability and the Write-Ahead Log): explains torn pages ("PostgreSQL typically writes 8192 bytes, or 16 sectors, at a time … some of the 512-byte sectors were written while others were not"), how full-page images guard against them, and that `full_page_writes` may be turned off when the filesystem prevents partial page writes (e.g., ZFS). <https://www.postgresql.org/docs/current/wal-reliability.html>

[^pg-wiki-fpw]: PostgreSQL Wiki, "Full page writes": full-page images are written on the first modification of a page after a checkpoint. <https://wiki.postgresql.org/wiki/Full_page_writes>

[^pg-ha]: PostgreSQL Documentation, "High Availability, Load Balancing, and Replication": streaming replication ships and replays WAL on standby servers. <https://www.postgresql.org/docs/current/high-availability.html>

[^snowflake]: Dageville et al., "The Snowflake Elastic Data Warehouse," SIGMOD 2016 — the multi-cluster shared-data design decouples storage and compute into independently elastic tiers. <https://dl.acm.org/doi/10.1145/2882903.2903741>

[^aurora]: Verbitski et al., "Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases," SIGMOD 2017 — "the log is the database"; only redo log records cross the network and the log applicator is pushed down to the storage tier, with writes acknowledged by a 4-of-6 storage quorum. <https://dl.acm.org/doi/10.1145/3035918.3056101>

[^neon-arch]: Neon Docs, "Architecture overview": stateless compute streams WAL to a quorum of safekeepers (Paxos-based) instead of flushing to local disk; the pageserver materializes page versions from base images plus WAL deltas and stores layer files in object storage; reads prefer local cache and fall back to a pageserver request. <https://neon.com/docs/introduction/architecture-overview>

[^neon-fpw]: Neon Blog, "Everyone gets faster writes: We turned off FPW's in Neon": because compute is stateless and streams WAL to a Paxos quorum of safekeepers, "there is no local-disk page to tear"; disabling full-page writes cut WAL per transaction from ~58 KB to under 4 KB (~94%), improved write throughput up to ~4.5× on 32-vCPU instances, and dropped p99 read latency ~30–50%; unbounded delta chains are avoided via pageserver-side image generation. <https://neon.com/blog/turning-off-fpw-for-faster-writes>

[^neon-branching]: Neon Docs, "Branching": a branch is a copy-on-write clone that shares parent data up to the branch point; writes to a branch are saved as deltas, and creating a branch does not load or affect the parent. <https://neon.com/docs/introduction/branching>

[^databricks-5x]: Databricks Blog, "How lakebase architecture delivers 5× faster Postgres writes": describes the same disable-FPW mechanism and pageserver-side image generation used to keep delta chains bounded. <https://www.databricks.com/blog/how-lakebase-architecture-delivers-5x-faster-postgres-writes>

[^databricks-neon]: Databricks Blog, "Databricks + Neon": Databricks' acquisition of Neon and the serverless Postgres architecture (branching, elastic scaling) behind Lakebase. <https://www.databricks.com/blog/databricks-neon>

[^databricks-lakebase]: Databricks Blog, "Announcing Lakebase Public Preview": Lakebase is managed Postgres built on Neon, providing copy-on-write branching, serverless autoscaling, and scale-to-zero. <https://www.databricks.com/blog/announcing-lakebase-public-preview>
