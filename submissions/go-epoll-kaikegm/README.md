# Go: epoll + pgx with family-routed query coalescing, shared-enrichment CTEs, and zerojwt

**Front: direct**

A Conduit server in pure Go (`CGO_ENABLED=0`) built on a custom multi-loop `epoll` reactor, zero-allocation request/response path, and PostgreSQL query coalescing. Source is in `src/`.

| Piece | Version |
|---|---|
| Go | 1.27 (`-trimpath -ldflags="-s -w"`) |
| HTTP | Custom epoll reactor (`SO_REUSEPORT`, `TCP_DEFER_ACCEPT`, `EPOLLONESHOT`) and zero-copy HTTP/1.1 parser |
| Postgres driver | `github.com/jackc/pgx/v5` v5.11.0 (`pgconn` extended binary protocol + pipeline mode) |
| JSON | Custom binary-row encoder (8-byte unrolled string escaper) + zero-alloc flat JSON body scanner with `encoding/json` fallback |
| JWT | `github.com/KaikeGM/zerojwt/v2` v2.2.0 (`VerifyWithScratch` / `SignWithScratch`, 0 allocs/op) with `github.com/golang-jwt/jwt/v5` fallback |
| bcrypt | 4-lane bit-sliced/interleaved pure-Go Eksblowfish (cost 10) with `golang.org/x/crypto/bcrypt` fallback |

## Key Optimizations (`go-epoll-kaikegm`)

1. **Same-CPU Go $\leftrightarrow$ PostgreSQL Adaptive Balancer & Family-Affinity Routing (`src/db.go`, `src/main.go`):**
   - Requests are classified into read families (`GET /api/tags`, `GET /api/articles` unfiltered/tag/author/favorited, `GET /api/articles/feed`, `GET /api/articles/:slug`, `GET /api/articles/:slug/comments`, `GET /api/profiles/:username`, `GET /api/user`) at HTTP frame time before leaving the reactor loop.
   - Each read connection worker owns an independent ring queue (`w.q`) and tracks in-queue counts per `readFamily`. Incoming read jobs route to a worker already accumulating the same family (within `readRouteSlack = 8`), maximizing batch homogeneity.
   - **Adaptive Same-CPU Balancer:** Tracks active PostgreSQL read/write flights (`activeReads`, `activeWrites`) and per-worker execution latency EWMA (`w.ewmaUs`) alongside batch-size EWMA (`w.ewmaLen`). When PostgreSQL is loaded on the shared 8-vCPU host (`activeReads + activeWrites >= maxReadConc`), the balancer gates concurrent pipelined read batches and dynamically raises the coalescing target and linger window (up to `450 µs`), producing larger deduplicated CTE / `UNNEST` batches with fewer PostgreSQL executions.
   - Each PostgreSQL session configures `work_mem = 32MB`, `random_page_cost = 1.1`, `cpu_index_tuple_cost = 0.0005`, and `effective_io_concurrency = 200` at startup.

2. **Coalesced `UNNEST` Write Group-Commit & B-Tree Monotonic Key Sorting (`src/sql.go`, `src/db.go`):**
   - **Batched Write Coalescing (`stBatchFollow`, `stBatchUnfollow`, `stBatchFavorite`, `stBatchUnfavorite`, `stBatchCommentCreate`):** Collapses multiple concurrent `POST/DELETE` follow, favorite, and comment-creation requests in a write commit batch into a single `UNNEST(...) WITH ORDINALITY` statement per write type, cutting PostgreSQL planner/executor overhead and index lock acquisitions while preserving exact per-request `404` / `200` / `201` semantics.
   - **B-Tree Monotonic Batch Sorting (`sortGroupByKey`):** Sorts coalesced read and write groups in-place by their primary B-tree index key (`slug`, `username`, `tag`, `offset`) before binding `UNNEST` array parameters so PostgreSQL index probes walk B-tree leaf pages monotonically and avoid row-lock contention on concurrent writes.

3. **Broadcast, Coalesced `LATERAL`, Shared Feed Hubs, and Shared-Enrichment `MATERIALIZED` CTE Queries (`src/sql.go`, `src/db.go`):**
   - **`groupBroadcast` (`GET /api/tags`):** Executes `stTags` once and broadcasts the serialized JSON response across all requests in the group.
   - **`groupCoalesced` (`unnest(...) WITH ORDINALITY CROSS JOIN LATERAL`):** Collapses up to 64 concurrent single-row or feed/comment reads into a single PostgreSQL statement execution, demultiplexed by `ord` in `rowReader`.
   - **Shared-Enrichment Article List, Slug & Comment CTEs (`stShareListAll`, `stShareListTag`, `stShareListAuthor`, `stShareListFavorite`, `stShareArticleBySlug`, `stShareComments`):** Deduplicates `(filter, limit, offset)` pages and colliding `slug` lookups within the same batch, computes `OFFSET 999 LIMIT 1` short-circuit bounded counts once per distinct filter, aggregates tags once per distinct article ID (`article_tag_arrays`), and enriches author/favorite/following metadata once per distinct article in the batch.
   - **Shared Feed Hubs (`stShareListFeed`):** Identifies popular followees (`hub_floors`) shared by $\ge 2$ viewers in a batch, computes their $(limit+offset)$-th newest article ID as a scan floor (`a2.id >= fl.floor_id`) to prune B-tree index walks, and short-circuits `articlesCount = 1000` (`cap_authors`) whenever a viewer follows a hub author with $\ge 1,000$ articles.

4. **L1D Cache-Aligned 4-Lane `bcryptx`, Listener Socket Inheritance & Zero-Allocation Hot Path (`src/internal/bcryptx/bcryptx.go`, `src/reactor_linux.go`, `src/jwt.go`, `src/jsonw.go`):**
   - **Cache-Set-Offset 4-Lane `bcryptx`:** Pads each Blowfish `state` struct from `4,168` bytes to `4,224` bytes (`66` cache lines) so all 4 interleaved lanes (`16.9 KB` total) fit inside a single core's L1D cache without 4 KB cache-set aliasing.
   - Sets `TCP_NODELAY`, `SO_KEEPALIVE`, `TCP_KEEPIDLE`, `TCP_KEEPINTVL`, and `TCP_KEEPCNT` once on each listener socket so accepted connections inherit them automatically inside the Linux kernel (eliminating 5 `setsockopt` syscalls per `accept4`), and automatically raises `RLIMIT_NOFILE` soft limit to the hard limit at startup.
   - JWT HS256 verification and signing use `zerojwt/v2` (`VerifyWithScratch` / `SignWithScratch`) with per-worker stack/struct scratch buffers; write request bodies (`POST/PUT`) use `parseFastWrapper` in-place; JSON string escaping processes 8 bytes per iteration.

## Build and run

```bash
sudo bash install.sh && bash build.sh
DATABASE_URL=postgres://conduit:conduit@127.0.0.1:5432/conduit JWT_SECRET=... HOST=0.0.0.0 PORT=80 bash start.sh
```
