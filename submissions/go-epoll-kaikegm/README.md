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

1. **Family-Affinity Read Routing & Zero-Syscall EWMA Batch Governor (`src/db.go`, `src/api.go`):**
   - Requests are classified into read families (`GET /api/tags`, `GET /api/articles` unfiltered/tag/author/favorited, `GET /api/articles/feed`, `GET /api/articles/:slug`, `GET /api/articles/:slug/comments`, `GET /api/profiles/:username`, `GET /api/user`) at HTTP frame time before leaving the reactor loop.
   - Each read connection worker owns an independent ring queue (`w.q`) and tracks in-queue counts per `readFamily`. Incoming read jobs route to a worker already accumulating the same family (within `readRouteSlack = 8`), maximizing batch homogeneity.
   - Each read worker tracks an integer Exponentially Weighted Moving Average (`w.ewmaLen = (w.ewmaLen*7 + len(jobs)) >> 3`) of recent batch sizes and releases `collect()` early (`200 µs` max linger) as soon as the dynamic target is reached.

2. **Broadcast, Coalesced `LATERAL`, Shared Feed Hubs, and Shared-Enrichment `MATERIALIZED` CTE Queries (`src/sql.go`, `src/db.go`):**
   - **`groupBroadcast` (`GET /api/tags`):** Executes `stTags` once and broadcasts the serialized JSON response across all requests in the group.
   - **`groupCoalesced` (`unnest(...) WITH ORDINALITY CROSS JOIN LATERAL`):** Collapses up to 32 concurrent single-row or feed/comment reads into a single PostgreSQL statement execution, demultiplexed by `ord` in `rowReader`.
   - **Shared-Enrichment Article List, Slug & Comment CTEs (`stShareListAll`, `stShareListTag`, `stShareListAuthor`, `stShareListFavorite`, `stShareArticleBySlug`, `stShareComments`):** Deduplicates `(filter, limit, offset)` pages and colliding `slug` lookups within the same batch, computes `OFFSET 999 LIMIT 1` short-circuit bounded counts once per distinct filter, aggregates tags once per distinct article ID (`article_tag_arrays`), and enriches author/favorite/following metadata once per distinct article in the batch.
   - **Shared Feed Hubs (`stShareListFeed`):** Identifies popular followees (`hub_floors`) shared by $\ge 2$ viewers in a batch, computes their $(limit+offset)$-th newest article ID as a scan floor (`a2.id >= fl.floor_id`) to prune B-tree index walks, and short-circuits `articlesCount = 1000` (`cap_authors`) whenever a viewer follows a hub author with $\ge 1,000$ articles.

3. **Listener Socket Inheritance & Short-Circuit Count Subqueries (`src/reactor_linux.go`, `src/sql.go`, `src/main.go`):**
   - Sets `TCP_NODELAY`, `SO_KEEPALIVE`, `TCP_KEEPIDLE`, `TCP_KEEPINTVL`, and `TCP_KEEPCNT` once on each listener socket so accepted connections inherit them automatically inside the Linux kernel (eliminating 5 `setsockopt` syscalls per `accept4`), and automatically raises `RLIMIT_NOFILE` soft limit to the hard limit at startup.
   - All `EXISTS (... WHERE follower_id = $1)` and `EXISTS (... WHERE user_id = $1)` subqueries guard on `$1::bigint <> 0`, eliminating B-tree index probes when `viewer == 0` (anonymous traffic).

4. **Zero-Allocation Hot Path (`src/jwt.go`, `src/util.go`, `src/jsonw.go`):**
   - JWT HS256 verification and signing use `zerojwt/v2` (`VerifyWithScratch` / `SignWithScratch`) with per-worker stack/struct scratch buffers.
   - Write request bodies (`POST/PUT`) use `parseFastWrapper` to scan flat JSON payloads in-place without heap allocations.
   - JSON string escaping processes 8 bytes per iteration using a 256-entry lookup table.

## Build and run

```bash
sudo bash install.sh && bash build.sh
DATABASE_URL=postgres://conduit:conduit@127.0.0.1:5432/conduit JWT_SECRET=... HOST=0.0.0.0 PORT=80 bash start.sh
```
