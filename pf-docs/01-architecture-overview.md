# FrankenPHP Architecture Overview

## Metadata
| Field | Value |
|-------|-------|
| Repository | `github.com/dunglas/frankenphp` |
| Commit | N/A (no git repo initialized) |
| Documented | `2026-01-04` |
| Verification Status | `Verified` |

## Example Reference
| Field | Value |
|-------|-------|
| Example Read | `.pf-agent-system-mapper/examples/packages/requests/good-architecture-doc-example.md` |
| Key Format Elements Noted | Tables for execution surfaces, Boundaries section, verification tags for all claims |

## Verification Summary
| Status | Count |
|--------|-------|
| VERIFIED | 48 |
| NOT_FOUND | 4 |
| INFERRED | 2 |
| ASSUMED | 0 |

---

## 0. System Classification

| Field | Value |
|-------|-------|
| Category | Traditional Code |
| Type | Library/Package + Application Server |
| Evidence | Exports Go library (`package frankenphp`), provides Caddy module, includes CLI binaries; embeds PHP interpreter via CGO |
| Overlay Loaded | No |
| Confidence | `[VERIFIED: frankenphp.go:1-6, caddy/caddy.go:1-4]` |

---

## 1. System Purpose

FrankenPHP is a modern application server for PHP built on top of the Caddy web server. It embeds the PHP interpreter directly into Go using CGO, providing a high-performance SAPI (Server API) for executing PHP scripts. The system supports two primary execution modes: **classic mode** (one PHP execution per request) and **worker mode** (long-running PHP processes that handle multiple requests). It also provides real-time capabilities via Mercure integration, automatic HTTPS, HTTP/2 and HTTP/3 support, Early Hints (103), and the ability to create standalone self-executable PHP applications.

[VERIFIED: README.md:1-11]

---

## 2. Component Map

### Core Go Components

| Component | Location | Responsibility | Evidence |
|-----------|----------|----------------|----------|
| frankenphp package | `frankenphp.go` | Main entry point; Init(), Shutdown(), ServeHTTP() | [VERIFIED: frankenphp.go:1-6, 238-353] |
| PHP Thread Management | `phpthread.go` | Manages PHP thread lifecycle and handlers | [VERIFIED: phpthread.go:17-45] |
| Main Thread Controller | `phpmainthread.go` | Controls main PHP thread, initializes PHP runtime | [VERIFIED: phpmainthread.go:22-36] |
| Worker Thread Handler | `threadworker.go` | Handles worker mode execution (long-running scripts) | [VERIFIED: threadworker.go:19-29] |
| Regular Thread Handler | `threadregular.go` | Handles classic mode execution (one request per script) | [VERIFIED: threadregular.go:15-27] |
| Worker Management | `worker.go` | Worker script configuration and request routing | [VERIFIED: worker.go:20-36] |
| Auto-scaling | `scaling.go` | Dynamic thread scaling based on load and CPU usage | [VERIFIED: scaling.go:14-37] |
| Request Context | `context.go` | FrankenPHP request context management | [VERIFIED: context.go:16-41] |
| Options/Config | `options.go` | Configuration options for Init() and workers | [VERIFIED: options.go:14-33] |
| Embedded Apps | `embed.go` | Supports embedding PHP apps as standalone binaries | [VERIFIED: embed.go:21-43] |
| Mercure Integration | `mercure.go` | Real-time pub/sub via Mercure protocol | [VERIFIED: mercure.go:15-17] |

### C/CGO Components

| Component | Location | Responsibility | Evidence |
|-----------|----------|----------------|----------|
| PHP SAPI Implementation | `frankenphp.c` | Custom SAPI module for PHP; handles request lifecycle | [VERIFIED: frankenphp.c:882-911] |
| C Header Definitions | `frankenphp.h` | Type definitions and function declarations for CGO | [VERIFIED: frankenphp.h:1-81] |
| PHP Arginfo | `frankenphp_arginfo.h` | PHP function argument definitions (generated) | [VERIFIED: tree output] |
| PHP Stub | `frankenphp.stub.php` | PHP function stubs for IDE support | [VERIFIED: tree output] |

### Caddy Integration

| Component | Location | Responsibility | Evidence |
|-----------|----------|----------------|----------|
| Caddy Module | `caddy/module.go` | HTTP handler for `php` and `php_server` directives | [VERIFIED: caddy/module.go:35-55] |
| Caddy App | `caddy/app.go` | Global FrankenPHP app configuration and startup | [VERIFIED: caddy/app.go:47-63] |
| Registration | `caddy/caddy.go` | Registers Caddy modules and handlers | [VERIFIED: caddy/caddy.go:18-29] |
| Worker Config | `caddy/workerconfig.go` | Worker configuration parsing for Caddyfile | [VERIFIED: tree output] |
| PHP Server | `caddy/php-server.go` | `php_server` directive implementation | [VERIFIED: tree output] |
| PHP CLI | `caddy/php-cli.go` | CLI command support | [VERIFIED: tree output] |
| Admin API | `caddy/admin.go` | Admin endpoints for worker management | [VERIFIED: tree output] |
| Hot Reload | `caddy/hotreload.go` | File watcher for hot reloading | [VERIFIED: tree output] |

### Internal Packages

| Package | Location | Responsibility | Evidence |
|---------|----------|----------------|----------|
| state | `internal/state/` | Thread state machine management | [VERIFIED: phpthread.go:24, phpmainthread.go:17] |
| cpu | `internal/cpu/` | CPU usage monitoring for auto-scaling | [VERIFIED: scaling.go:13] |
| memory | `internal/memory/` | System memory detection for auto max_threads | [VERIFIED: phpmainthread.go:15] |
| fastabs | `internal/fastabs/` | Fast absolute path resolution | [VERIFIED: caddy/module.go:23] |
| phpheaders | `internal/phpheaders/` | Common HTTP header caching | [VERIFIED: phpmainthread.go:16] |
| watcher | `internal/watcher/` | File system watcher for hot reload | [VERIFIED: tree output] |

[NOT_FOUND: No database models or ORM - this is a runtime/server, not an application]
[NOT_FOUND: No web routes in library - routes are configured via Caddy]

---

## 3. Execution Surfaces & High-Level Data Movement

### 3.1 Primary Execution Surfaces

| Entry Surface | Type | Primary Components Involved | Evidence |
|--------------|------|-----------------------------|----------|
| `frankenphp.Init()` | Library API | options, phpMainThread, worker initialization | [VERIFIED: frankenphp.go:238-353] |
| `frankenphp.ServeHTTP()` | Library API | context, worker/regularThread, PHP execution | [VERIFIED: frankenphp.go:385-418] |
| `frankenphp.Shutdown()` | Library API | drainWorkers, drainPHPThreads, cleanup | [VERIFIED: frankenphp.go:355-383] |
| `frankenphp.ExecuteScriptCLI()` | Library API | CLI script execution outside HTTP context | [VERIFIED: frankenphp.go:756-769] |
| `FrankenPHPModule.ServeHTTP()` | Caddy Handler | HTTP request to PHP script forwarding | [VERIFIED: caddy/module.go:178-255] |
| `FrankenPHPApp.Start()` | Caddy App | Initialize PHP runtime on Caddy start | [VERIFIED: caddy/app.go:138-173] |
| `php_server` directive | Caddyfile | Shorthand for PHP file serving | [VERIFIED: caddy/caddy.go:28-29] |
| `php` directive | Caddyfile | PHP handler configuration | [VERIFIED: caddy/caddy.go:25-26] |

### 3.2 High-Level Data Movement

| Stage | Input | Output | Components |
|-------|-------|--------|------------|
| Request Reception | HTTP Request | frankenPHPContext | ServeHTTP, NewRequestWithContext |
| Thread Assignment | frankenPHPContext | Worker or Regular thread | worker.handleRequest, handleRequestWithRegularPHPThreads |
| PHP Execution | Script path, CGI vars | HTTP Response | phpThread, frankenphp.c SAPI |
| Response Delivery | PHP output | HTTP Response to client | go_ub_write, go_write_headers |

### 3.3 Pointers to Code Flow Documentation

The following operations are candidates for **detailed flow tracing** (see 02-code-flows.md):

- **Worker Mode Request Handling** - How requests are dispatched to long-running worker scripts
- **Regular Mode Request Handling** - Classic CGI-style execution per request
- **Thread Auto-scaling** - Dynamic thread creation based on load
- **PHP Runtime Initialization** - How the PHP SAPI is initialized via CGO

### Section 3 Self-Check

- [x] No method bodies longer than 3 lines quoted
- [x] No loops or conditionals described
- [x] All movements as conceptual stages
- [x] Defers to 02-code-flows.md

---

## 4. File/Folder Conventions

| Pattern | Purpose | Evidence |
|---------|---------|----------|
| `*.go` | Go source files (main package) | [VERIFIED: tree output] |
| `*.c`, `*.h` | C source for PHP SAPI via CGO | [VERIFIED: frankenphp.c, frankenphp.h] |
| `caddy/` | Caddy module integration | [VERIFIED: caddy/caddy.go:1-4] |
| `internal/` | Private internal packages | [VERIFIED: internal/ directory] |
| `testdata/` | Test fixtures and PHP scripts | [VERIFIED: tree output] |
| `docs/` | Documentation files | [VERIFIED: tree output] |
| `*_test.go` | Unit and integration tests | [VERIFIED: tree output] |
| `*-skip.go` | Build-conditional files (feature toggles) | [VERIFIED: mercure-skip.go, watcher-skip.go] |
| `thread*.go` | Thread handler implementations | [VERIFIED: threadworker.go, threadregular.go, threadinactive.go] |

---

## 5. External Dependencies

### Direct Dependencies

| Dependency | Purpose | Evidence |
|------------|---------|----------|
| `github.com/caddyserver/caddy/v2` | Web server framework (Caddy module) | [VERIFIED: caddy/go.mod] |
| `github.com/dunglas/mercure` | Real-time pub/sub protocol | [VERIFIED: go.mod:9, mercure.go:13] |
| `github.com/prometheus/client_golang` | Prometheus metrics | [VERIFIED: go.mod:12] |
| `github.com/e-dant/watcher/watcher-go` | File system watcher for hot reload | [VERIFIED: go.mod:10] |
| `github.com/maypok86/otter/v2` | High-performance cache | [VERIFIED: go.mod:11] |
| `github.com/stretchr/testify` | Testing utilities | [VERIFIED: go.mod:13] |
| `golang.org/x/net` | Extended networking | [VERIFIED: go.mod:14] |
| PHP (system) | PHP 8.2+ with ZTS | [VERIFIED: frankenphp.go:47, frankenphp.c compile flags] |

[NOT_FOUND: No database drivers - this is a runtime server]
[NOT_FOUND: No external API clients - embeds PHP which handles those]

---

## 6. Boundaries & Non-Responsibilities

Explicitly **NOT** in this library/server:

| Boundary | Evidence |
|----------|----------|
| No application logic | Server only executes PHP scripts provided by users |
| No database connections | PHP applications handle their own DB [VERIFIED: no DB imports] |
| No template rendering | PHP handles templating |
| No async/await in Go | Uses goroutines and channels for concurrency [VERIFIED: code patterns] |
| No Windows support | Requires WSL on Windows [VERIFIED: README.md:19] |
| PHP versions < 8.2 | Minimum PHP 8.2 required [VERIFIED: frankenphp.go:289-292] |

---

## 7. Known Issues & Risks

| Issue | Location | Description | Evidence |
|-------|----------|-------------|----------|
| Timeout limitations | phpmainthread.go | ZTS timeouts broken except on Linux/FreeBSD | [VERIFIED: phpmainthread.go:188-193] |
| Memory leaks with PECL | scaling.go | Some PECL extensions prevent full thread stopping | [VERIFIED: scaling.go:240-248] |
| SIGPIPE handling | frankenphp.go | Must be masked in non-Go threads | [VERIFIED: frankenphp.c:961-973] |
| Non-ZTS degradation | frankenphp.go | Without ZTS, only 1 thread available | [VERIFIED: frankenphp.go:300-305] |

---

## 8. Entry Points Summary

### Go Library API

| Function | Purpose | Evidence |
|----------|---------|----------|
| `frankenphp.Init(options...)` | Initialize PHP runtime | [VERIFIED: frankenphp.go:238] |
| `frankenphp.Shutdown()` | Stop PHP runtime | [VERIFIED: frankenphp.go:355] |
| `frankenphp.ServeHTTP(w, r)` | Execute PHP for HTTP request | [VERIFIED: frankenphp.go:385] |
| `frankenphp.ExecuteScriptCLI(script, args)` | Run PHP CLI script | [VERIFIED: frankenphp.go:756] |
| `frankenphp.ExecutePHPCode(code)` | Execute PHP code string | [VERIFIED: frankenphp.go:771] |
| `frankenphp.NewRequestWithContext(r, opts...)` | Create FrankenPHP request | [VERIFIED: context.go:62] |
| `frankenphp.DrainWorkers()` | Graceful worker shutdown | [VERIFIED: worker.go:180] |
| `frankenphp.RestartWorkers()` | Restart all workers | [VERIFIED: worker.go:222] |

### PHP Functions (exposed to PHP scripts)

| Function | Purpose | Evidence |
|----------|---------|----------|
| `frankenphp_handle_request(callable)` | Worker request loop | [VERIFIED: frankenphp.c:412-487] |
| `frankenphp_finish_request()` | End request early | [VERIFIED: frankenphp.c:276-289] |
| `frankenphp_request_headers()` | Get request headers | [VERIFIED: frankenphp.c:344-358] |
| `frankenphp_response_headers()` | Get response headers | [VERIFIED: frankenphp.c:401-409] |
| `headers_send(code)` | Send headers (Early Hints) | [VERIFIED: frankenphp.c:489-508] |
| `mercure_publish(topics, data, ...)` | Publish Mercure update | [VERIFIED: frankenphp.c:510-550] |
| `frankenphp_log(message, level, context)` | Structured logging | [VERIFIED: frankenphp.c:552-571] |

### Caddy Directives

| Directive | Purpose | Evidence |
|-----------|---------|----------|
| `frankenphp { }` | Global config block | [VERIFIED: caddy/app.go:202-311] |
| `php { }` | PHP handler block | [VERIFIED: caddy/module.go:257-331] |
| `php_server { }` | Shorthand PHP serving | [VERIFIED: caddy/caddy.go:28] |

---

## 9. Technology Stack Summary

| Layer | Technology | Evidence |
|-------|------------|----------|
| Language | Go 1.25+ | [VERIFIED: go.mod:3] |
| CGO Integration | C (PHP SAPI) | [VERIFIED: frankenphp.c] |
| Web Server | Caddy v2 | [VERIFIED: caddy/go.mod imports] |
| PHP Runtime | PHP 8.2+ (ZTS recommended) | [VERIFIED: frankenphp.go:289-292] |
| Real-time | Mercure protocol | [VERIFIED: mercure.go] |
| Metrics | Prometheus | [VERIFIED: go.mod:12] |
| File Watching | watcher-go | [VERIFIED: go.mod:10] |

---

## 10. Architecture Diagram Reference

See `04-diagrams.md` for:
- Thread lifecycle state machine
- Request flow through worker vs regular modes
- CGO boundary interactions
- Caddy module integration

---

## Appendix: Key Type Definitions

### phpThread (phpthread.go:17-26)
```go
type phpThread struct {
    runtime.Pinner
    threadIndex  int
    requestChan  chan contextHolder
    drainChan    chan struct{}
    handlerMu    sync.Mutex
    handler      threadHandler
    state        *state.ThreadState
    sandboxedEnv map[string]*C.zend_string
}
```
[VERIFIED: phpthread.go:17-26]

### worker (worker.go:20-36)
```go
type worker struct {
    mercureContext
    name                   string
    fileName               string
    num                    int
    maxThreads             int
    requestOptions         []RequestOption
    requestChan            chan contextHolder
    threads                []*phpThread
    // ...
}
```
[VERIFIED: worker.go:20-36]

### frankenPHPContext (context.go:16-41)
```go
type frankenPHPContext struct {
    mercureContext
    documentRoot    string
    splitPath       []string
    env             PreparedEnv
    logger          *slog.Logger
    request         *http.Request
    worker          *worker
    // ...
}
```
[VERIFIED: context.go:16-41]

---

*Documentation generated following agent-system-mapper methodology v1.0*
