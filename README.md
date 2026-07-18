# pgsplit-bench

Benchmark monolithic Postgres against compute/storage-separated Postgres (Neon) side by side — write throughput, read latency, WAL efficiency, branch/clone time, and storage footprint — all runnable locally, for free.

> **Status:** early / scaffolding in progress. This README is the project spec; the harness and dashboard are being built out. See the [roadmap](#roadmap).

> **New to WAL, full-page writes, or compute/storage separation?** Start with the **[Concepts guide](docs/concepts.md)** — it explains the theory these experiments test, from Postgres fundamentals up, with every claim referenced.

## Why

Postgres deployments are moving from a **monolith** — one machine where the query engine, buffer cache, WAL, and data files all live together (this is what a classic managed instance like Azure PostgreSQL Flexible Server gives you) — toward **compute/storage separation**, where stateless compute streams WAL to a durable storage service that keeps full page history on object storage. This is the architecture behind Neon and, by extension, Databricks Lakebase.

The claims made for the separated model are specific and testable: higher write throughput (via eliminating full-page writes), near-instant branching regardless of database size, copy-on-write storage, and scale-to-zero. `pgsplit-bench` runs **both architectures for real** on one machine and measures the differences, so you can see which claims hold up, by how much, and where the monolith actually wins.

Note: Databricks Lakebase itself is a managed cloud service and isn't free to run. Its engine is the open-source [Neon](https://github.com/neondatabase/neon) storage stack, which *can* be self-hosted locally — so that's the separated-storage target used here.

## Architecture

Two targets, both speaking the Postgres wire protocol, on the same host under the same resource limits:

```
Monolith                          Separated — self-hosted Neon

┌───────────────────┐             ┌───────────────────┐
│ Postgres          │             │ Postgres (compute)│  stateless,
│  ├ query engine   │             │  ├ query engine   │  scales to zero
│  ├ buffers        │             │  ├ buffers        │
│  ├ WAL   ──┐      │             │  └ WAL ──┐        │
│  └ data files◄┘   │             └──────────┼────────┘
│      (local disk) │                        │ WAL over the network
└───────────────────┘                        ▼
                                   ┌───────────────────┐
compute + storage fused;          │ safekeepers (×3)  │  durable WAL,
a commit = one local fsync        │  Paxos quorum     │  commit on quorum
                                   └─────────┬─────────┘
                                             │ WAL stream
                                             ▼
                                   ┌───────────────────┐
                                   │ pageserver        │  rebuilds pages
                                   │  base + WAL deltas│  from the WAL stream
                                   └─────────┬─────────┘
                                             │ layer files
                                             ▼
                                   ┌───────────────────┐
                                   │ object storage    │  cheap, durable
                                   │  (S3 / MinIO)     │
                                   └───────────────────┘
```

The monolith is one fused node; the separated target splits stateless Postgres compute from a durable, tiered storage service. [Databricks Lakebase](https://www.databricks.com/blog/announcing-lakebase-public-preview) is the managed product built on this design — here we self-host its open-source engine (Neon) so it runs locally and free. For the theory behind every box above, see the **[Concepts guide](docs/concepts.md)**.

| Target | What it is | Shape |
| --- | --- | --- |
| **monolith** | `postgres:17` in Docker | Compute + storage fused in one node; commits `fsync` to local disk |
| **separated** | Self-hosted Neon stack (compute + safekeeper + pageserver + MinIO) | Stateless compute; commits stream WAL to a safekeeper quorum; pages rebuilt on read from object storage |

Both compute layers are pinned to the same Postgres major version so version differences aren't a confound.

## Experiments

Each experiment isolates one architectural difference and produces one chart. The trustworthy signal is the **shape** of each result, not laptop-specific absolute numbers.

1. **Write throughput + WAL efficiency** — the "5x writes" claim. Run write-heavy `pgbench` at rising client counts; record TPS and WAL bytes per transaction. Includes a control on the monolith with `full_page_writes` on vs off, which isolates the exact mechanism the separated architecture exploits (compute never persists data pages to local disk ⇒ torn pages can't occur ⇒ FPW can be safely disabled).

2. **Read latency under concurrency** — `pgbench -S` (read-only) at 1/4/8/16/32 clients; record p50/p95/p99.

3. **Branch vs restore, by dataset size** — for databases of 100 MB / 1 GB / 5 GB / 10 GB, time "make a test copy": on the monolith via `pg_dump | pg_restore`, on the separated target via a branch. Expected shape: monolith climbs ~linearly with size, separated stays flat at seconds.

4. **Storage footprint of a copy** — measure disk used by the second copy immediately after experiment 3, then after mutating X% of rows, to show copy-on-write growth in proportion to changes rather than total size.

5. **Cold start / cost of idle** — suspend the separated compute and time resume → first successful query. The managed automatic scale-to-zero (idle detection, autoscaler) is not part of the self-hostable OSS stack, so here it's simulated by stopping the compute container and restarting it — which still measures the quantity that matters: the cold-start latency you pay for not keeping the node running. The monolith has no scale-to-zero equivalent; this reframes the comparison around the cost of an always-on node.

## Datasets

- **`pgbench` built-in** (TPC-B-like, scaled with `-s`) for the throughput and latency numbers — standard and apples-to-apples.
- **Synthetic time-series** — a generated `meter_reading(meter_id, tenant_id, ts, value, accuracy)` table (millions of rows) for the branching and storage experiments, giving realistic insert-heavy load and range scans.

## Visualization

A self-contained `dashboard.html` (Chart.js) reads `results.json` and renders:

- write TPS and WAL-per-transaction bars,
- latency percentile lines vs client count,
- the branch-time-vs-size chart (the hero: one line climbing, one flat),
- a storage-footprint panel.

No server, no account — open the file in a browser.

## Getting started

> Target interface; commands land as the harness is built.

```bash
# 1. Bring up both targets
docker compose up -d          # monolith + self-hosted Neon stack (+ MinIO)

# 2. Run the benchmark suite
python bench/run.py --scope full --dataset both --trials 3

# 3. View results
open dashboard.html           # reads results.json
```

## Planned project structure

```
pgsplit-bench/
├── docker-compose.yml        # monolith + Neon stack + MinIO
├── bench/
│   ├── run.py                # orchestrates experiments, writes results.json
│   ├── experiments/          # one module per experiment above
│   └── datagen/              # synthetic meter-reading generator
├── dashboard.html            # Chart.js results viewer
├── results.json              # produced by a run
└── README.md
```

## Interpreting results

- **Match the setup fairly.** Same Postgres major version; pin the separated target's *compute* container to the same CPU/memory limits as the monolith; warm-up runs discarded; multiple trials reported as medians.
- **The separated target is a whole stack, not one container.** Alongside its compute node it runs a pageserver, three safekeepers, a storage broker, and MinIO — always-on processes that consume extra host CPU, memory, and disk. That overhead is intrinsic to the architecture, and it's why "runnable for free" here means local reproduction of the *shapes*, not a claim about production cost or total resource use.
- **Expect the monolith to win some.** The separated target routes every commit across an intra-host hop to the safekeeper quorum instead of a local `fsync`, so single-node write latency can be higher. That's the cost of the separation, not a misconfiguration — and it's part of what the benchmark is meant to reveal.
- **Trust the curves, not the constants.** A laptop won't reproduce cloud absolute numbers. The flat-vs-linear branching curve and the WAL reduction from disabling full-page writes are the durable takeaways.

## Roadmap

- [ ] `docker-compose` for both targets (monolith + self-hosted Neon + MinIO)
- [ ] Benchmark runner and results schema
- [ ] Experiment 1: write throughput + WAL efficiency (incl. FPW control)
- [ ] Experiment 2: read latency percentiles
- [ ] Experiment 3: branch vs restore by size
- [ ] Experiment 4: copy-on-write storage footprint
- [ ] Experiment 5: cold start / scale-to-zero
- [ ] Synthetic meter-reading data generator
- [ ] Chart.js dashboard
- [ ] Optional third target: Neon free cloud tier (adds the network-latency dimension)

## Acknowledgements

Built on the open-source [Neon](https://github.com/neondatabase/neon) storage engine, which makes a genuine compute/storage-separated Postgres runnable locally.

## License

MIT (see [`LICENSE`](LICENSE)).