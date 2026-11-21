# FeedLedger

A Linux-native, offline-first terminal RSS reader written in C++. FeedLedger ingests RSS 2.0 feeds into a local SQLite database and lets you browse them from the command line. Every dependency ships inside the source tree, so it builds, tests and runs on a machine with no internet access.

| | |
|---|---|
| **Binary** | `feedledger` |
| **Platform** | Linux only (GCC or Clang, CMake, Make or Ninja) |
| **Language** | C++20 |
| **Effort** | 12 sessions × 3 hours = 36 hours |
| **Default build mode** | **Offline** (reads local feed files) |
| **Optional build mode** | **Online** (adds libcurl for HTTP/HTTPS downloads), selected at compile time |

---

## 1. Project Summary

### Purpose
FeedLedger lets a user subscribe to RSS 2.0 feeds, pull their articles into SQLite, and list and inspect them from the terminal. It is a systems-style learning and portfolio project: small enough to finish, but it touches parsing, persistence, I/O abstraction, a CLI, and automated testing.

### Two build modes, chosen at compile time
The program has two ways of obtaining feed content. The user picks one when building, and there is **no silent fallback** between them.

| | **Offline mode (default)** | **Online mode** |
|---|---|---|
| CMake flag | `-DFEEDLEDGER_ONLINE=OFF` (default) | `-DFEEDLEDGER_ONLINE=ON` |
| Preprocessor macro | `FEEDLEDGER_ONLINE_MODE=0` | `FEEDLEDGER_ONLINE_MODE=1` |
| Feed sources accepted | `file:///abs/path/feed.xml` or a plain local path | everything offline mode accepts, plus `http://` and `https://` |
| libcurl | not compiled, not linked | compiled and linked |
| Typical use | the no-internet workspace desktop; bundled sample feeds | a machine with internet access, or the loopback demo |
| `http(s)://` source | rejected with a clear message: *"this build is offline; rebuild with -DFEEDLEDGER_ONLINE=ON"* | fetched with libcurl |

The macro is defined in exactly one place (the CMake option) and read in exactly one place (the composition root, plus a `BuildMode.hpp` constant used for `--version` output). No other file contains `#if FEEDLEDGER_ONLINE_MODE`. The libcurl sources are included or excluded by CMake.

### Functional requirements
| # | Feature | CLI |
|---|---|---|
| F1 | Add a feed source | `feedledger add <source> [--name <n>]` |
| F2 | Remove a feed and its articles | `feedledger remove <feed-id>` |
| F3 | List feeds | `feedledger feeds` |
| F4 | Update all feeds (sequential) | `feedledger update` |
| F5 | Parse RSS 2.0 | internal |
| F6 | Store articles, deduplicated by link | internal |
| F7 | List articles | `feedledger list [--feed <id>] [--limit <n>]` |
| F8 | Show one article | `feedledger show <article-id>` |
| F9 | SQLite persistence with versioned migrations | internal |
| F10 | File logging | `feedledger.log` next to the database |
| F11 | Automated tests | GoogleTest + CTest, no network |
| F12 | Help and version | `feedledger help`, `feedledger <command> --help`, `feedledger --version` |

Global option: `--db <path>` (also the `FEEDLEDGER_DB` environment variable). The default is `./feedledger.db`.

### Non-functional requirements
- Builds with a plain `cmake` + `make`/`ninja` on Linux.
- **No network access at configure, build, test or run time** in the default configuration. Nothing is downloaded by CMake (no `FetchContent`).
- No exceptions in business logic; every fallible function returns `Result<T>`.
- Single-threaded, sequential updates.
- Warning-clean at `-Wall -Wextra -Wpedantic -Werror` (GCC and Clang).

### Out of scope
Atom/RSS 1.0, search/FTS, read/unread/starred, categories UI, export, OPML, parallel or background updates, TUI, `open`, feed health statistics, and verification of online mode (it is provided, but acceptance is based on offline mode only).

### Key design decisions
1. **Build mode is a compile-time switch, default offline.** It is explicit; there is no runtime guessing about connectivity.
2. **One abstraction at the I/O boundary: `IFeedSource`.** `LocalFileSource`, `CurlFeedSource` (online only) and `FakeFeedSource` (tests) implement it. Everything above it is identical in both modes.
3. **One static library `feedledger_core` plus a thin executable `feedledger`**, so tests link the same code the app runs.
4. **Everything vendored** under `third_party/`: SQLite amalgamation, pugixml, GoogleTest, and (online mode only) libcurl.
5. **In-house `Result<T>`** (a thin `std::variant` wrapper with an `std::expected`-like API), so the project does not depend on a C++23 standard library.
6. **Time handling by hand**: `std::chrono::sys_seconds` in memory and ISO-8601 UTC text in SQLite. RFC 822 dates are parsed with a small custom parser (no tz database needed).
7. **No config file and no JSON dependency.** Settings come from `--db` and `FEEDLEDGER_DB`.
8. **Built-in file logger** instead of spdlog/fmt.
9. **`add` never touches the network.** For local sources it only checks that the file exists and is readable.
10. **Empty feed name means "adopt the channel title on first successful update."**

---

## 2. Overall Architecture

**Style:** layered, single-threaded, constructor injection from one composition root (`Application`).

```text
┌────────────────────────────────────────────────────────────────┐
│ Presentation   main.cpp · CommandDispatcher · Commands         │
├────────────────────────────────────────────────────────────────┤
│ Application    Application (composition root, owns the graph)  │
│                chooses the IFeedSource by build mode           │
├────────────────────────────────────────────────────────────────┤
│ Core/Services  FeedManager · ArticleManager · UpdateService    │
├────────────────────────────────────────────────────────────────┤
│ Infrastructure Database · FeedRepository · ArticleRepository   │
│                IFeedSource ─┬─ LocalFileSource   (always)      │
│                             ├─ CurlFeedSource    (online only) │
│                             ├─ RoutingFeedSource (online only) │
│                             └─ FakeFeedSource    (tests only)  │
│                RssParser (pugixml) · Logger                    │
├────────────────────────────────────────────────────────────────┤
│ Foundation     Error/Result · models · util::*                 │
└────────────────────────────────────────────────────────────────┘
```

Dependencies point downward only. Interfaces exist only at the I/O boundary (`IFeedSource`); repositories and managers are concrete classes.

### Subsystems
| Subsystem | Responsibility |
|---|---|
| `cli/` | Tokenize arguments, validate against a declarative command table, route to handlers, print output, return exit codes |
| `app/` | Build and own the object graph, select the feed source for the build mode, run one command |
| `core/` | Business rules: validate and add feeds, query articles |
| `services/` | `UpdateService`: orchestrates fetch → parse → store for each feed |
| `database/` | SQLite RAII wrappers, migrations, SQL for feeds and articles |
| `source/` | Feed content retrieval behind `IFeedSource` |
| `parsers/` | RSS 2.0 XML → `ParsedFeed` |
| `logging/` | Minimal leveled file logger |
| `util/` | Time, string, and source-location helpers |

### Feed source layer
```cpp
struct FetchRequest {
    std::string                location;           // normalized source string
    std::optional<std::string> if_none_match;      // ETag (HTTP only)
    std::optional<std::string> if_modified_since;  // Last-Modified (HTTP only)
};

struct FetchResponse {
    int                        status {200};       // 200, 304, or an HTTP status
    std::string                body;
    std::optional<std::string> etag;
    std::optional<std::string> last_modified;
};

class IFeedSource {
public:
    virtual ~IFeedSource() = default;
    // Transport/IO failures -> Error{Network|Io}. HTTP 4xx/5xx are returned as responses.
    [[nodiscard]] virtual Result<FetchResponse> fetch(const FetchRequest&) = 0;
};
```

| Implementation | Compiled when | Behavior |
|---|---|---|
| `LocalFileSource` | always | Reads `file://` URLs and plain paths; returns status 200 with the file body, or `Io` error if unreadable. Rejects `http(s)` with the "offline build" message. |
| `CurlFeedSource` | `FEEDLEDGER_ONLINE=ON` | libcurl GET with timeouts, redirect limit, size limit, and ETag/Last-Modified capture. |
| `RoutingFeedSource` | `FEEDLEDGER_ONLINE=ON` | Sends `http(s)` locations to `CurlFeedSource` and everything else to `LocalFileSource`. |
| `FakeFeedSource` | tests | Canned responses keyed by location; records requests. |

### Data flow: `feedledger update`
```text
argv → CommandDispatcher → Commands::update → UpdateService::update_all
  → FeedRepository::get_all
  → for each feed:
      IFeedSource::fetch(location, etag, last_modified)
        ├─ 304            → NotModified, record success (HTTP only)
        ├─ 200            → RssParser::parse(body) → ParsedFeed
        │                   ArticleRepository::upsert_many (dedup on link, one transaction)
        │                   FeedRepository::record_fetch_success (adopt title if name empty)
        └─ error / parse  → FeedRepository::record_fetch_failure (last_error, failure_count++)
  → UpdateSummary (feeds ok/failed, articles added/skipped) → printed
  → exit code: 0 all ok · 1 any feed failed
```

### Data model (SQLite schema v1, `PRAGMA user_version = 1`)
```sql
CREATE TABLE feeds (
    id            INTEGER PRIMARY KEY,
    name          TEXT    NOT NULL DEFAULT '',
    url           TEXT    NOT NULL UNIQUE,   -- normalized source (file:///... or http(s)://...)
    category      TEXT,                       -- reserved, unused
    created_at    TEXT    NOT NULL,           -- ISO-8601 UTC
    last_updated  TEXT,
    etag          TEXT,
    last_modified TEXT,
    last_error    TEXT,
    failure_count INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE articles (
    id        INTEGER PRIMARY KEY,
    feed_id   INTEGER NOT NULL,
    guid      TEXT,                           -- stored, not used for dedup yet
    title     TEXT    NOT NULL DEFAULT '',
    link      TEXT    NOT NULL UNIQUE,        -- dedup key
    summary   TEXT    NOT NULL DEFAULT '',
    author    TEXT    NOT NULL DEFAULT '',
    published TEXT    NOT NULL,               -- pubDate, or fetch time if missing/invalid
    read      INTEGER NOT NULL DEFAULT 0,     -- reserved
    starred   INTEGER NOT NULL DEFAULT 0,     -- reserved
    FOREIGN KEY (feed_id) REFERENCES feeds(id) ON DELETE CASCADE
);

CREATE INDEX idx_articles_feed_published ON articles(feed_id, published DESC);
CREATE INDEX idx_articles_published      ON articles(published DESC);
```

### Source tree
```text
feedledger/
├── CMakeLists.txt                  options (FEEDLEDGER_ONLINE), toolchain check, subdirectories
├── README.md
├── cmake/
│   ├── CompilerWarnings.cmake      feedledger_set_warnings(target)
│   └── CheckToolchain.cmake        verifies C++20 features compile
├── third_party/                    vendored; no network needed
│   ├── VERSIONS.md                 pinned versions and licences
│   ├── sqlite/                     sqlite3.c / sqlite3.h (amalgamation)
│   ├── pugixml/                    pugixml.cpp / .hpp
│   ├── googletest/                 vendored source, add_subdirectory
│   └── curl/                       vendored source (used only when FEEDLEDGER_ONLINE=ON)
├── resources/samples/
│   ├── sample_feed.xml             realistic RSS 2.0 (CDATA, several items, mixed date formats)
│   ├── sample_feed_updated.xml     same feed plus new items (shows idempotence + inserts)
│   ├── sample_feed_minimal.xml     missing optional fields (no link/date/author)
│   ├── sample_feed_malformed.xml   broken XML
│   └── sample_feed_atom.xml        Atom document (must be rejected cleanly)
├── scripts/
│   ├── demo_offline.sh             scripted offline demo (no server)
│   └── demo_loopback.sh            optional: python3 http.server on 127.0.0.1 + online build
├── src/
│   ├── CMakeLists.txt              feedledger_core + feedledger
│   ├── main.cpp
│   ├── app/        Application · BuildMode.hpp
│   ├── cli/        CommandDispatcher · Commands
│   ├── core/       Error.hpp (Result) · FeedManager · ArticleManager
│   ├── database/   Database · FeedRepository · ArticleRepository
│   ├── logging/    Logger
│   ├── model/      Feed.hpp · Article.hpp
│   ├── parsers/    RssParser
│   ├── services/   UpdateService
│   ├── source/     IFeedSource.hpp · LocalFileSource · [CurlFeedSource · RoutingFeedSource]
│   └── util/       Time · String · SourceLocator
└── tests/
    ├── CMakeLists.txt
    ├── support/    FakeFeedSource.hpp · TestDb.hpp · TestData.hpp · [LoopbackServer.hpp]
    ├── unit/       Time · String · SourceLocator · RssParser · CommandDispatcher · FeedManager
    ├── integration/ Database · Repositories · UpdateService · Application
    └── e2e/        run the real binary against the sample files
```
Items in `[brackets]` are compiled only in online mode.

### CLI contract
- Exit codes: `0` success · `1` runtime failure (including any feed failing during `update`) · `2` usage error.
- `add` accepts `file:///abs/path.xml`, an absolute path, or a relative path. Plain paths are converted to a normalized `file:///` URL before storage. A missing or unreadable local file is an error.
- `list` defaults to 50 articles (max 1000), newest first. `show` prints article metadata and a plain-text summary with HTML tags stripped at display time.
- `--version` prints the version and the build mode (`offline` or `online`).

### Error handling
`Result<T>` everywhere. `ErrorCode` values: `InvalidArgument`, `NotFound`, `AlreadyExists`, `Network`, `Parse`, `Database`, `Io`, `Unsupported`, `Internal`. Errors are mapped to exit codes only in `Commands`. Exceptions are caught only in `main` as a last resort.

### Testing strategy
| Level | What it covers | Tools |
|---|---|---|
| Unit | time parsing (ISO-8601, RFC 822 variants), string helpers, source normalization, RSS parsing (good, minimal, malformed, Atom), CLI parsing and exit codes | GoogleTest |
| Integration | migrations (idempotent), repository CRUD, uniqueness, cascade delete, `UpdateService` with `FakeFeedSource` (insert/skip counts, failure bookkeeping, idempotent reruns), `Application` wiring on in-memory SQLite | GoogleTest |
| End-to-end | runs the real `feedledger` binary on a temp database: `add → update → list → show → update again → remove` against `resources/samples/*.xml` | CTest + shell/GoogleTest driver |
| Loopback (online build only, label `loopback`; optional, not part of acceptance) | in-process 127.0.0.1 test server serves a sample; `CurlFeedSource` fetches it | GoogleTest |

All default tests need no network. The loopback test uses only the loopback interface and is excluded from the default offline build.

---

## 3. Implementation Plan (12 sessions × 3 hours)

| # | Focus | Done when |
|---|---|---|
| 1 | Linux baseline, vendoring, build skeleton | Configure, build and `ctest` work offline with a trivial test |
| 2 | Foundations: `Result`, models, time, strings, source locator, logger | Utility unit tests pass |
| 3 | SQLite foundation and migrations | Migration is idempotent; DB tests pass |
| 4 | Repositories | CRUD, uniqueness, cascade and no-duplicate tests pass |
| 5 | RSS parser and sample feeds | Parser tests pass on all sample files |
| 6 | Feed source layer (offline) and build-mode switch | `LocalFileSource` and `FakeFeedSource` tests pass; offline build rejects `http(s)` |
| 7 | Online mode: libcurl source (optional; not part of acceptance) | Offline build is unaffected; online code compiles where libcurl is available |
| 8 | Update service and managers | Insert/skip counts, failures and idempotent reruns verified |
| 9 | CLI core: dispatcher, `help`, `add`, `remove`, `feeds` | CLI tests pass; exit codes consistent |
| 10 | CLI data commands: `update`, `list`, `show` | Commands work on real SQLite; output deterministic |
| 11 | Application wiring, end-to-end tests, demo scripts | Offline demo script runs end to end |
| 12 | Hardening, documentation, final verification | Clean offline build; full `ctest` pass; evidence captured |

### Session 1 — Linux baseline, vendoring and build skeleton
**Deliverables**
- Repository layout and top-level `CMakeLists.txt` (C++20, no extensions, `CMAKE_EXPORT_COMPILE_COMMANDS`).
- `third_party/` populated with SQLite amalgamation, pugixml and GoogleTest (and libcurl source, unused for now), plus `VERSIONS.md`. Fetching these is done once on any machine with internet before the project reaches the offline desktop.
- `cmake/CheckToolchain.cmake` (fails with a clear message if C++20 support is missing) and `cmake/CompilerWarnings.cmake`.
- `FEEDLEDGER_ONLINE` option (default `OFF`) that defines `FEEDLEDGER_ONLINE_MODE` and is printed at configure time.
- `feedledger_core` static library, a `feedledger` executable that prints `--version`, and one passing GoogleTest registered with CTest.
- No `FetchContent`, `ExternalProject` or `find_package` that reaches the network.

**Done when:** on a machine with the network disabled, `cmake -S . -B build && cmake --build build && ctest --test-dir build` succeeds with zero warnings.

### Session 2 — Foundations
**Deliverables**
- `core/Error.hpp`: `ErrorCode`, `Error`, `Result<T>`, `Result<void>`, `make_error`.
- Models: `Feed`, `Article`, `ParsedFeed`.
- `util/Time`: `TimePoint`, ISO-8601 format/parse, RFC 822 parser (named zones, numeric offsets, optional seconds and weekday), fallback behavior for invalid dates.
- `util/String`: trim, case-insensitive compare, `parse_int`, `strip_html`.
- `util/SourceLocator`: classifies input as local file or HTTP(S), normalizes plain paths to `file:///…`, validates URLs.
- `logging/Logger`: leveled file logger (`debug|info|warn|error`), no console sink.

**Done when:** unit tests for every helper pass, including edge cases (empty input, bad dates, relative paths, `file://` with encoded characters).

### Session 3 — SQLite foundation
**Deliverables**
- `Database` RAII wrapper (open file or `:memory:`, `PRAGMA foreign_keys=ON`, WAL for file DBs), `Statement` (prepare/bind/step/column helpers), `Transaction` (RAII, rollback on destruction).
- Migration runner on `PRAGMA user_version`; schema v1 as above.
- Every SQLite error converted to `Error{Database}` with the SQLite message.

**Done when:** opening the same database twice leaves it at version 1 with no errors; transaction commit/rollback tests pass; no SQL is built by concatenating user data.

### Session 4 — Repositories
**Deliverables**
- `FeedRepository`: add, get by id, get all, remove, record fetch success/failure, adopt title.
- `ArticleRepository`: `upsert_many` (single transaction, `INSERT … ON CONFLICT(link) DO NOTHING`, returns inserted/skipped counts), query by feed with limit, get by id.

**Done when:** tests cover CRUD, duplicate feed URL, duplicate article link, cascade delete, ordering by `published`, and repeated upserts that add nothing.

### Session 5 — RSS parsing and sample feeds
**Deliverables**
- `RssParser` on pugixml: channel title; per item `title`, `link`, `guid`, `description` (including CDATA), `author`/`dc:creator`, `pubDate`.
- Defensive rules: items without a link are skipped (counted), invalid or missing dates fall back to fetch time, Atom/RDF and non-XML input return `Error{Parse}` with a clear message.
- Sample files in `resources/samples/` (see the source tree above). If real feeds can be captured once on an internet-connected machine, replace or add to the hand-written samples; they do not need to be real to satisfy the plan.

**Done when:** parser tests pass offline against every sample, including malformed and Atom inputs.

### Session 6 — Feed source layer (offline) and build-mode switch
**Deliverables**
- `IFeedSource`, `LocalFileSource`, `FakeFeedSource`.
- `app/BuildMode.hpp` (`constexpr` mode name from the macro) and the source selection in `Application`.
- Offline-mode behavior: `add` and `update` reject `http(s)` sources with the explicit "rebuild with -DFEEDLEDGER_ONLINE=ON" error.

**Done when:** tests pass for reading bundled samples via both plain paths and `file://` URLs, missing and unreadable files, oversized files, and the `http(s)` rejection.

### Session 7 — Online mode: libcurl source
**Deliverables**
- `CurlFeedSource`: RAII for the global init and easy handle; GET with `Accept` header, connect/total timeouts, redirect limit, response-size limit, ETag/Last-Modified capture, conditional request headers; `CURLOPT_PROTOCOLS_STR` restricted to `http,https`.
- `RoutingFeedSource`.
- CMake: libcurl built from `third_party/curl` only when `FEEDLEDGER_ONLINE=ON`, with a minimal feature set (no zlib/brotli/zstd, no LDAP, no HTTP/2). TLS is enabled only if a system TLS library is found; otherwise the build is **HTTP-only**, which is enough for the loopback demo. `-DFEEDLEDGER_CURL_SYSTEM=ON` links the system libcurl instead.
- `tests/support/LoopbackServer.hpp` (POSIX sockets on 127.0.0.1, ephemeral port) and the `loopback`-labelled test.

**Done when:** the offline build is unaffected (no libcurl compiled or linked) and the online code is in place. Online mode is not part of the success verification; the loopback test and an online-mode build are best-effort checks where the tooling allows.

### Session 8 — Update service and managers
**Deliverables**
- `UpdateService::update_all` implementing the data flow above, with `UpdateSummary` (per-feed status, added, skipped, error text).
- `FeedManager` (add with validation and duplicate detection, remove, list) and `ArticleManager` (list with `ArticleQuery`, get).

**Done when:** integration tests with `FakeFeedSource` verify first update inserts, second update inserts nothing, `sample_feed_updated.xml` adds only the new items, a failing feed is recorded without stopping the others, and an empty feed name adopts the channel title.

### Session 9 — CLI core
**Deliverables**
- `CommandDispatcher` and `CommandSpec` table; `--flag value` and `--flag=value`; `--` terminator; per-command `--help`; global `--db`.
- `Commands`: `help`, `add`, `remove`, `feeds`.
- Clear user-facing messages for bad ids, unknown commands, duplicates and offline-mode rejections.

**Done when:** CLI tests assert output text and exit codes `0`/`1`/`2` for success, runtime failure and usage errors.

### Session 10 — CLI data commands
**Deliverables**
- `update` (prints one line per feed plus totals), `list` (`--feed`, `--limit`, aligned columns, truncation of long titles), `show` (metadata and cleaned summary).
- Deterministic output: stable ordering, no timestamps from "now" in test paths.

**Done when:** commands operate on real SQLite data and tests verify output for empty database, one feed, several feeds, filtering and limits.

### Session 11 — Application wiring, end-to-end tests, demo scripts
**Deliverables**
- `Application` composition root and `main.cpp` finalized.
- End-to-end CTest scenarios that run the built binary against `resources/samples/`.
- `scripts/demo_offline.sh`: builds (default mode), then `add → update → feeds → list → show → update → remove`.
- `scripts/demo_loopback.sh`: optional online-mode demo with `python3 -m http.server 8000 --bind 127.0.0.1` serving `resources/samples/` (skips with a message if `python3` is absent).

**Done when:** `scripts/demo_offline.sh` completes on the workspace desktop and the second `update` reports zero new articles.

### Session 12 — Hardening, documentation, verification buffer
**Deliverables**
- Edge cases: empty feed, huge description, non-UTF-8 input, duplicate links across feeds, read-only database directory.
- Final README section with build/run instructions and the limitation statement.
- Fresh-clone verification of the offline build, a recorded transcript of the offline demo, and `ctest` output.
- Buffer for whatever slipped from earlier sessions. **Do not add deferred features here.**

**Done when:** the success criteria in Section 4 are all met.

### Pacing and risk
- If toolchain or vendoring problems appear, spend the extra time in Session 1 and shorten Sessions 11–12.
- Highest-risk sessions: 3 (SQLite), 7 (libcurl build) and 10–11 (CLI integration). Session 7 is optional for acceptance; if it overruns, the offline deliverable is still complete.

---

## 4. Success Criteria

### Verified on the Linux desktop in offline mode (no internet)
1. **Native Linux build:** a fresh configure, build and test run uses no network and produces a runnable `feedledger` binary.
2. **`ctest` is green** with zero warnings at `-Werror`.
3. **SQLite core works:** the database is created and migrated, feeds and articles are stored and queried, and repeated updates create no duplicates.
4. **Parsing core works:** all bundled RSS 2.0 samples parse correctly, including missing optional fields; malformed XML and Atom produce clean errors, never crashes.
5. **CLI core works:** all commands behave as specified, with consistent exit codes `0`/`1`/`2`.
6. **Offline demo is reproducible:** `scripts/demo_offline.sh` runs `add → update → list → show → update` against bundled samples, and the second update adds nothing.
7. **Build mode is explicit:** the default build is offline and rejects `http(s)` sources with a clear message; `--version` shows the mode.

Online mode is not part of the success verification. It is built as an optional extension and is not required for acceptance.

### First milestone (minimum convincing slice, after Session 6)
Linux build and tests run; a bundled sample feed is ingested from a local path; articles are persisted in SQLite; `feeds` and `list` display the stored data.

---

## 5. Environment, Limitations and Risks

### Build and run
```bash
# Default: offline mode
cmake -S . -B build && cmake --build build
ctest --test-dir build --output-on-failure
./build/feedledger --db demo.db add resources/samples/sample_feed.xml
./build/feedledger --db demo.db update
./build/feedledger --db demo.db list
./build/feedledger --db demo.db show 1

# Optional: online mode
cmake -S . -B build-online -DFEEDLEDGER_ONLINE=ON && cmake --build build-online
./build-online/feedledger --version      # reports: online
```

### What will not work within the desktop limits
| Limit | Impact | Handling |
|---|---|---|
| No internet | No public feeds; no dependency downloads | Bundled samples; all dependencies vendored; online mode compiled in only on request |
| Unknown compiler/library versions | Newer language features may be missing | C++20 baseline, in-house `Result<T>`, toolchain check in Session 1 |
| No external services | No hosted DB, containers, or remote APIs | Single-process CLI with a local SQLite file |
| Loopback tooling may be missing | `python3` may be absent | Offline mode needs no server; the loopback test uses an in-process server |

### Stated limitation
Live fetching from public internet RSS feeds is the intended real-world behavior of online mode, but it is not verified on the offline desktop. Acceptance is based on offline mode, which proves the **source → read → parse → store → query** pipeline using bundled local feeds.

### Risks
| Risk | Impact | Mitigation |
|---|---|---|
| Vendored libcurl is awkward to build | Session 7 overruns | Minimal feature set; HTTP-only fallback; system-libcurl option; offline mode never depends on it |
| Vendored sources missing when the project is moved to the offline desktop | Build fails at configure | Session 1 checks that `third_party/` is complete and fails early with a clear message |
| Hand-written samples are too clean | Parser misses real-world cases | Include CDATA, mixed date formats and missing fields; replace or add real snapshots when an internet machine is available |
| Real feeds with non-UTF-8 text or bad dates | Odd rows | Lenient date parser with fetch-time fallback; known limitation for non-UTF-8 |
| Global unique `link` drops cross-feed duplicates | An article appears under one feed only | Accepted |
| Database lives in the current directory by default | Data scattered | `--db` / `FEEDLEDGER_DB` documented |

---

## 6. Engineering Conventions
- Namespaces `feedledger` and `feedledger::util`; types `PascalCase`; functions and members `snake_case`; private members end with `_`; `#pragma once`.
- Third-party headers (`sqlite3.h`, `pugixml.hpp`, `curl/curl.h`) are included only from `.cpp` files, never from project headers.
- `Result<T>` is `[[nodiscard]]`; every result is checked or propagated; no `.value()` without a prior check.
- RAII for every external resource (SQLite handle and statements, transactions, curl global state and easy handle, files).
- `const`-correct, `explicit` single-argument constructors, `final` where not meant for inheritance.
- No raw `new`/`delete`, no `shared_ptr`, no `GLOB` in CMake, no `std::endl`.
- `std::cout`/`std::cerr` only in `main`, `CommandDispatcher` and `Commands`.
- Review checkpoint for every session: zero warnings, tests for new code, no network use in default tests, no deferred features introduced.
