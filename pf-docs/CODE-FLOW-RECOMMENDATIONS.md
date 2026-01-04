# Code Flow Recommendations: FrankenPHP

> Generated: 2026-01-04
> Based on: `pf-docs/01-architecture-overview.md`
> **Focus**: Creating standalone PHP binary with SQL Server drivers for macOS (no Docker)

---

## Summary

| Flow | Priority | Components | Effort | Relevance to SQL Server Binary |
|------|----------|------------|--------|--------------------------------|
| Static Binary Build Process | **High** | build-static.sh, static-php-cli, embed.go | Medium | **Critical** - how extensions are compiled |
| Embedded App Extraction | **High** | embed.go, init(), untar() | Low | **Critical** - how app is packaged |
| Extension Registration | **High** | ext.go, frankenphp.c | Low | **Critical** - how SQL Server drivers load |
| PHP Runtime Initialization | Medium | frankenphp.go:Init(), phpmainthread.go | Medium | Understanding startup |

---

## Recommended Flows

### 1. Static Binary Build Process (Priority: **Critical**)

**Why document this?**
This is THE flow for your use case. Understanding how `build-static.sh` uses `static-php-cli` to compile PHP with extensions (including `pdo_sqlsrv`) into a static binary is essential for creating a standalone binary on macOS.

**Trigger**: Running `./build-static.sh` with `PHP_EXTENSIONS` including `pdo_sqlsrv`

**Key components**:
- `build-static.sh` - orchestrates the entire static build process
- `static-php-cli` (external) - actually compiles PHP and extensions statically
- `xcaddy` - builds Caddy with FrankenPHP module
- `frankenphp/caddy/frankenphp/main.go` - final binary entry point

**Key files to start tracing**:
- `frankenphp/build-static.sh:79` - default extensions list (includes `pdo_sqlsrv`)
- `frankenphp/build-static.sh:151-159` - extension detection from composer
- `frankenphp/build-static.sh:179-186` - EMBED app packaging
- `frankenphp/build-static.sh:212` - final spc build command with `--build-frankenphp`

**Critical Variables for SQL Server**:
```bash
# These are KEY for your use case:
PHP_EXTENSIONS="pdo,pdo_sqlsrv"  # Must include pdo_sqlsrv
PHP_EXTENSION_LIBS="..."         # May need unixODBC library
```

**Prompt to use**:
```
Create code flow documentation for FrankenPHP covering:
Static Binary Build Process - from build-static.sh through static-php-cli to final frankenphp binary

Focus on:
1. How PHP_EXTENSIONS variable controls which extensions are compiled
2. How static-php-cli downloads and compiles extension sources
3. How the final binary is assembled with xcaddy
4. macOS-specific considerations (arch detection, brew dependencies)

Reference the architecture overview at pf-docs/01-architecture-overview.md
Start tracing from frankenphp/build-static.sh:1
```

---

### 2. Embedded App Extraction Flow (Priority: **High**)

**Why document this?**
If you want a truly standalone binary that includes your PHP application code (not just the runtime), you need to understand how FrankenPHP embeds and extracts apps at startup.

**Trigger**: Binary startup when `embeddedApp []byte` is non-empty

**Key components**:
- `embed.go` - Go's `//go:embed` directive and extraction logic
- `app.tar` / `app_checksum.txt` - embedded tar archive
- `init()` function - automatic extraction on startup

**Key files to start tracing**:
- `frankenphp/embed.go:23-27` - embed directives
- `frankenphp/embed.go:29-43` - init() function extracts to temp
- `frankenphp/embed.go:48-149` - untar() extraction logic

**Data Movement**:
```
Binary startup → init() called → embeddedApp checked →
untar() to temp dir → EmbeddedAppPath set → app ready to serve
```

**Prompt to use**:
```
Create code flow documentation for FrankenPHP covering:
Embedded App Extraction - from binary startup through app extraction to serving

Focus on:
1. How app.tar is embedded using Go's embed directive
2. The init() automatic extraction sequence
3. How EmbeddedAppPath is used by Caddy module
4. Checksum-based caching to avoid re-extraction

Reference the architecture overview at pf-docs/01-architecture-overview.md
Start tracing from frankenphp/embed.go:29
```

---

### 3. Extension Registration Flow (Priority: **High**)

**Why document this?**
Understanding how PHP extensions (like `pdo_sqlsrv`) are registered and loaded is critical for debugging extension loading issues on macOS.

**Trigger**: `frankenphp.Init()` → `registerExtensions()`

**Key components**:
- `ext.go` - Go-side extension registration
- `frankenphp.c:register_extensions()` - C-side registration
- `frankenphp.h` - C type definitions

**Key files to start tracing**:
- `frankenphp/ext.go:16-18` - `RegisterExtension()` function
- `frankenphp/ext.go:20-29` - `registerExtensions()` called from Init
- `frankenphp/frankenphp.go:249` - where extensions are registered in Init()
- `frankenphp/frankenphp.h:79` - `register_extensions()` C declaration

**Prompt to use**:
```
Create code flow documentation for FrankenPHP covering:
Extension Registration - how PHP extensions are registered across Go/C boundary

Focus on:
1. RegisterExtension() Go API for external extensions
2. CGO boundary crossing to register_extensions()
3. When in the Init() sequence this happens
4. How static-compiled extensions differ from dynamically loaded ones

Reference the architecture overview at pf-docs/01-architecture-overview.md
Start tracing from frankenphp/frankenphp.go:249
```

---

### 4. PHP Runtime Initialization (Priority: Medium)

**Why document this?**
Understanding the full Init() sequence helps debug startup issues, especially on macOS where ZTS and extension loading can have platform-specific behavior.

**Trigger**: `frankenphp.Init(options...)`

**Key components**:
- `frankenphp.go:Init()` - main initialization
- `phpmainthread.go` - main PHP thread startup
- `frankenphp.c:frankenphp_startup()` - C SAPI initialization

**Key files to start tracing**:
- `frankenphp/frankenphp.go:238-353` - Init() function
- `frankenphp/phpmainthread.go:41-79` - initPHPThreads()
- `frankenphp/phpmainthread.go:105-125` - start() SAPI initialization

**Prompt to use**:
```
Create code flow documentation for FrankenPHP covering:
PHP Runtime Initialization - from Init() through SAPI startup to ready state

Focus on:
1. Option processing and validation
2. PHP version and ZTS checks
3. Main thread creation via CGO
4. Thread pool initialization
5. macOS-specific timeout handling (ZTS issues)

Reference the architecture overview at pf-docs/01-architecture-overview.md
Start tracing from frankenphp/frankenphp.go:238
```

---

## macOS + SQL Server: Practical Guide

Based on the code analysis, here's what you need to know for your specific use case:

### Building on macOS with SQL Server Drivers

**Prerequisites** (from `build-static.sh:110-125`):
```bash
# Installed via brew automatically:
brew install composer go
# For SQL Server, you'll also need:
brew install unixodbc
```

**Build Command**:
```bash
cd frankenphp

# Option 1: Include pdo_sqlsrv in default set (already included!)
./build-static.sh

# Option 2: Minimal set with SQL Server only
PHP_EXTENSIONS="pdo,pdo_sqlsrv,mysqli,mysqlnd,opcache,ctype,session" \
  ./build-static.sh

# Option 3: Embed your app
EMBED=/path/to/your/php/app \
  PHP_EXTENSIONS="pdo,pdo_sqlsrv" \
  ./build-static.sh
```

### Key Observations from Code

1. **pdo_sqlsrv IS in default extensions** (`build-static.sh:79`)
   - The default build ALREADY includes `pdo_sqlsrv`

2. **macOS architecture detection** (`build-static.sh:13-15`):
   ```bash
   arch="$(uname -m)"  # arm64 or x86_64
   os="mac"
   ```

3. **static-php-cli handles ODBC dependencies**
   - When you specify `pdo_sqlsrv`, static-php-cli downloads Microsoft's ODBC drivers
   - This is handled externally to FrankenPHP code

4. **Final binary location** (`build-static.sh:221-222`):
   ```bash
   bin="dist/frankenphp-mac-${arch}"
   ```

---

## Skip These (Low Value for Your Use Case)

| Flow | Why Skip |
|------|----------|
| Worker Mode Request Handling | Only needed if using workers; standalone binary works in regular mode |
| Auto-scaling Thread Logic | Optimization concern, not relevant to building the binary |
| Mercure Real-time | Not relevant to SQL Server connectivity |
| Hot Reload Watcher | Development feature, not needed for production standalone |
| Caddy Directive Parsing | Configuration detail, not build process |

---

## Notes

1. **Suggested Order**: Build Process → Embedded App → Extension Registration → Runtime Init

2. **External Dependency**: Much of the actual static compilation happens in `static-php-cli` (external project). For deep SQL Server driver debugging, you may need to trace into that project.

3. **macOS ZTS Limitation**: Note from `phpmainthread.go:188-193` - timeouts are broken on macOS with ZTS. This won't affect SQL Server queries but be aware for long-running operations.

4. **Testing Your Build**: After building, verify SQL Server support:
   ```bash
   ./dist/frankenphp-mac-arm64 php -m | grep -i sql
   # Should show: pdo_sqlsrv, sqlsrv
   ```

---

*Generated following agent-system-mapper methodology v1.0*
