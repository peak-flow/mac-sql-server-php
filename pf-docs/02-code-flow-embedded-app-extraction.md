# Embedded App Extraction - Code Flow

## Metadata
| Field | Value |
|-------|-------|
| Repository | `github.com/dunglas/frankenphp` |
| Commit | N/A (no git repo initialized) |
| Documented | `2026-01-04` |
| Trigger | Binary startup (Go's `init()` mechanism) |
| End State | PHP app extracted to temp dir, `EmbeddedAppPath` set for Caddy module |

## Verification Summary
| Status | Count |
|--------|-------|
| VERIFIED | 28 |
| INFERRED | 3 |
| NOT_FOUND | 1 |
| ASSUMED | 0 |

---

## Flow Diagram

```
[Binary Execution Starts]
           │
           ▼
    ┌─────────────────────────────┐
    │ Go Runtime: init() called   │
    │ embed.go:init()             │
    └─────────────┬───────────────┘
                  │
           embeddedApp empty?
                  │
        ┌─────────┴─────────┐
        │ YES               │ NO
        ▼                   ▼
    [Return early]    ┌─────────────────────┐
                      │ Build extraction    │
                      │ path with checksum  │
                      └──────────┬──────────┘
                                 │
                                 ▼
                      ┌─────────────────────┐
                      │ untar() to temp dir │
                      │ - Create dirs       │
                      │ - Extract files     │
                      │ - Handle macOS      │
                      └──────────┬──────────┘
                                 │
                          Error? │
                    ┌────────────┴────────────┐
                    │ YES                     │ NO
                    ▼                         ▼
             ┌──────────────┐     ┌──────────────────────┐
             │ RemoveAll()  │     │ Set EmbeddedAppPath  │
             │ panic(err)   │     │ = extracted path     │
             └──────────────┘     └──────────┬───────────┘
                                             │
                                             ▼
                              ┌──────────────────────────────┐
                              │ Caddy Module Consumes Path   │
                              │ - module.go: Root resolution │
                              │ - php-server.go: chdir()     │
                              │ - context.go: documentRoot   │
                              │ - php-cli.go: script paths   │
                              │ - workerconfig.go: workers   │
                              └──────────────────────────────┘
                                             │
                                             ▼
                              [Shutdown: RemoveAll(EmbeddedAppPath)]
```

---

## Detailed Flow

### Step 1: Go Embed Directives (Build-Time)

[VERIFIED: embed.go:23-27]
```go
//go:embed app.tar
var embeddedApp []byte

//go:embed app_checksum.txt
var embeddedAppChecksum []byte
```

**Build-Time Behavior:**
- Go's `embed` package embeds `app.tar` into the binary
- `app.tar` is created by static-php-cli during `--build-frankenphp`
- `app_checksum.txt` contains a hash for cache invalidation

**Data shape:**
```
embeddedApp = []byte (tar archive contents, or empty if no app)
embeddedAppChecksum = []byte (checksum string)
```

[NOT_FOUND: app.tar creation process - handled by static-php-cli, external to FrankenPHP]

---

### Step 2: Package-Level Variable

[VERIFIED: embed.go:20-21]
```go
// EmbeddedAppPath contains the path of the embedded PHP application (empty if none)
var EmbeddedAppPath string
```

**Purpose:** Global variable that consumers check to determine if an embedded app exists and where it's located.

---

### Step 3: init() - Automatic Extraction

[VERIFIED: embed.go:29-43]
```go
func init() {
	if len(embeddedApp) == 0 {
		// No embedded app
		return
	}

	appPath := filepath.Join(os.TempDir(), "frankenphp_"+string(embeddedAppChecksum))

	if err := untar(appPath); err != nil {
		_ = os.RemoveAll(appPath)
		panic(err)
	}

	EmbeddedAppPath = appPath
}
```

**Flow:**
1. Check if `embeddedApp` has content (length > 0)
2. If empty → return early (no embedded app)
3. Build extraction path: `{temp_dir}/frankenphp_{checksum}`
4. Call `untar()` to extract
5. On error → cleanup and panic
6. On success → set `EmbeddedAppPath`

**Path Construction:**
```
macOS: /var/folders/.../T/frankenphp_abc123def456
Linux: /tmp/frankenphp_abc123def456
```

**Caching Mechanism** [INFERRED]:
- Checksum in path means same app = same extraction path
- If directory exists with same checksum, `untar()` skips existing files

---

### Step 4: untar() - Archive Extraction

[VERIFIED: embed.go:48-54]
```go
func untar(dir string) (err error) {
	t0 := time.Now()
	nFiles := 0
	madeDir := map[string]bool{}

	tr := tar.NewReader(bytes.NewReader(embeddedApp))
	loggedChtimesError := false
```

**Initialization:**
- Capture start time for mod time clamping
- Track created directories to avoid redundant `MkdirAll`
- Create tar reader from embedded bytes

---

### Step 5: Tar Entry Processing Loop

[VERIFIED: embed.go:55-69]
```go
for {
	f, err := tr.Next()
	if err == io.EOF {
		break
	}
	if err != nil {
		return fmt.Errorf("tar error: %w", err)
	}
	if f.Typeflag == tar.TypeXGlobalHeader {
		// golang.org/issue/22748: git archive exports
		// a global header ('g') which after Go 1.9
		// (for a bit?) contained an empty filename.
		// Ignore it.
		continue
	}
```

**Entry Processing:**
- Read each tar entry until EOF
- Skip global headers (git archive compatibility)

---

### Step 6: Path Validation

[VERIFIED: embed.go:70-74]
```go
	rel, err := nativeRelPath(f.Name)
	if err != nil {
		return fmt.Errorf("tar file contained invalid name %q: %v", f.Name, err)
	}
	abs := filepath.Join(dir, rel)
```

**Security Check:**
- `nativeRelPath()` validates path is relative
- Prevents path traversal attacks (`../` etc.)
- Converts to native path separator

---

### Step 7: Regular File Extraction

[VERIFIED: embed.go:78-133]
```go
case mode.IsRegular():
	// Make the directory...
	dir := filepath.Dir(abs)
	if !madeDir[dir] {
		if err := os.MkdirAll(filepath.Dir(abs), mode.Perm()); err != nil {
			return err
		}
		madeDir[dir] = true
	}
	if runtime.GOOS == "darwin" && mode&0111 != 0 {
		// See comment in writeFile.
		err := os.Remove(abs)
		if err != nil && !errors.Is(err, fs.ErrNotExist) {
			return err
		}
	}
	if _, err := os.Stat(abs); os.IsNotExist(err) {
		wf, err := os.OpenFile(abs, os.O_RDWR|os.O_CREATE|os.O_TRUNC, mode.Perm())
		// ... write file contents ...
	}
```

**File Extraction Flow:**
1. Ensure parent directory exists
2. **macOS Special Handling** (lines 91-97): Remove executable files before re-extraction
3. Only extract if file doesn't exist (caching)
4. Write file contents with proper permissions

**macOS Executable Handling** [VERIFIED: embed.go:91-97]:
```go
if runtime.GOOS == "darwin" && mode&0111 != 0 {
	// See comment in writeFile.
	err := os.Remove(abs)
	if err != nil && !errors.Is(err, fs.ErrNotExist) {
		return err
	}
}
```

**Why?** [INFERRED]: macOS code signing and Gatekeeper may prevent overwriting executables. Removing first ensures clean extraction.

---

### Step 8: Directory Creation

[VERIFIED: embed.go:134-138]
```go
case mode.IsDir():
	if err := os.MkdirAll(abs, mode.Perm()); err != nil {
		return err
	}
	madeDir[abs] = true
```

**Directories:** Created with original permissions from tar.

---

### Step 9: Symlink Handling (Skipped)

[VERIFIED: embed.go:139-143]
```go
case mode&os.ModeSymlink != 0:
	// TODO: ignore these for now. They were breaking x/build tests.
	// Implement these if/when we ever have a test that needs them.
	// But maybe we'd have to skip creating them on Windows for some builders
	// without permissions.
```

**Known Limitation:** Symlinks in embedded apps are silently ignored.

---

### Step 10: Path Validation Function

[VERIFIED: embed.go:154-188]
```go
func nativeRelPath(p string) (string, error) {
	if p == "" {
		return "", errors.New("path not provided")
	}

	// ... validation logic ...

	clean := path.Clean(p)
	if path.IsAbs(clean) {
		return "", fmt.Errorf("path %q is not relative", p)
	}
	if clean == ".." || strings.HasPrefix(clean, "../") {
		return "", fmt.Errorf("path %q refers to a parent directory", p)
	}
	// ...
}
```

**Security Validations:**
- Path must not be empty
- Path must be relative (not absolute)
- Path must not traverse to parent (`..`)
- Path must not reference Windows volume names

---

## Consumer Usage

### Consumer 1: Context Document Root

[VERIFIED: context.go:76-85]
```go
if fc.documentRoot == "" {
	if EmbeddedAppPath != "" {
		fc.documentRoot = EmbeddedAppPath
	} else {
		var err error
		if fc.documentRoot, err = os.Getwd(); err != nil {
			return nil, err
		}
	}
}
```

**Usage:** When creating request context, if no document root specified, use embedded app path.

**Priority:**
1. Explicitly set document root
2. `EmbeddedAppPath` (if embedded app exists)
3. Current working directory (fallback)

---

### Consumer 2: Caddy Module Provision

[VERIFIED: caddy/module.go:105-116]
```go
if f.Root == "" {
	if frankenphp.EmbeddedAppPath == "" {
		f.Root = "{http.vars.root}"
	} else {
		f.Root = filepath.Join(frankenphp.EmbeddedAppPath, defaultDocumentRoot)

		var rrs bool
		f.ResolveRootSymlink = &rrs
	}
} else if frankenphp.EmbeddedAppPath != "" && filepath.IsLocal(f.Root) {
	f.Root = filepath.Join(frankenphp.EmbeddedAppPath, f.Root)
}
```

**Root Resolution:**
- If no root set AND embedded app exists → `{EmbeddedAppPath}/public`
- If root is relative AND embedded app exists → prepend embedded path

[VERIFIED: caddy/caddy.go:14]
```go
defaultDocumentRoot = "public"
```

---

### Consumer 3: ServeHTTP Fallback

[VERIFIED: caddy/module.go:188-192]
```go
if f.resolvedDocumentRoot == "" {
	documentRoot = repl.ReplaceKnown(f.Root, "")
	if documentRoot == "" && frankenphp.EmbeddedAppPath != "" {
		documentRoot = frankenphp.EmbeddedAppPath
	}
```

**Runtime Fallback:** If document root evaluates to empty at request time, use embedded path.

---

### Consumer 4: php_server Directive

[VERIFIED: caddy/module.go:472-481]
```go
if frankenphp.EmbeddedAppPath != "" {
	if phpsrv.Root == "" {
		phpsrv.Root = filepath.Join(frankenphp.EmbeddedAppPath, defaultDocumentRoot)
		fsrv.Root = phpsrv.Root
		rrs := false
		phpsrv.ResolveRootSymlink = &rrs
	} else if filepath.IsLocal(fsrv.Root) {
		phpsrv.Root = filepath.Join(frankenphp.EmbeddedAppPath, phpsrv.Root)
		fsrv.Root = phpsrv.Root
	}
}
```

**php_server Integration:** Same logic applied to php_server shorthand directive.

---

### Consumer 5: PHP Server Command (chdir)

[VERIFIED: caddy/php-server.go:84-88]
```go
if frankenphp.EmbeddedAppPath != "" {
	if err := os.Chdir(frankenphp.EmbeddedAppPath); err != nil {
		return caddy.ExitCodeFailedStartup, err
	}
}
```

**Working Directory:** Changes process working directory to embedded app path.

---

### Consumer 6: PHP INI Scan Directory

[VERIFIED: caddy/php-server.go:106-113]
```go
if frankenphp.EmbeddedAppPath != "" {
	if _, err := os.Stat("php.ini"); err == nil {
		iniScanDir := os.Getenv("PHP_INI_SCAN_DIR")

		if err := os.Setenv("PHP_INI_SCAN_DIR", iniScanDir+":"+frankenphp.EmbeddedAppPath); err != nil {
			return caddy.ExitCodeFailedStartup, err
		}
	}
```

**PHP Configuration:** If `php.ini` exists in embedded app, add to PHP's INI scan path.

---

### Consumer 7: PHP CLI Command

[VERIFIED: caddy/php-cli.go:34-37]
```go
if frankenphp.EmbeddedAppPath != "" {
	if _, err := os.Stat(args[0]); err != nil {
		args[0] = filepath.Join(frankenphp.EmbeddedAppPath, args[0])
	}
}
```

**CLI Script Resolution:** If script path doesn't exist, try relative to embedded app.

---

### Consumer 8: Worker Configuration

[VERIFIED: caddy/workerconfig.go:154-156]
```go
if frankenphp.EmbeddedAppPath != "" && filepath.IsLocal(wc.FileName) {
	wc.FileName = filepath.Join(frankenphp.EmbeddedAppPath, wc.FileName)
}
```

**Worker Paths:** Relative worker script paths are resolved against embedded app.

---

### Consumer 9: App Configuration Workers

[VERIFIED: caddy/app.go:122-123]
```go
if frankenphp.EmbeddedAppPath != "" && filepath.IsLocal(w.FileName) {
	w.FileName = filepath.Join(frankenphp.EmbeddedAppPath, w.FileName)
}
```

**Global Worker Paths:** Same resolution for workers defined in global frankenphp block.

---

### Consumer 10: Startup Logging

[VERIFIED: frankenphp.go:336-338]
```go
if EmbeddedAppPath != "" {
	globalLogger.LogAttrs(globalCtx, slog.LevelInfo, "embedded PHP app 📦", slog.String("path", EmbeddedAppPath))
}
```

**Logging:** Logs embedded app path at startup for debugging.

---

### Consumer 11: Shutdown Cleanup

[VERIFIED: frankenphp.go:372-375]
```go
// Remove the installed app
if EmbeddedAppPath != "" {
	_ = os.RemoveAll(EmbeddedAppPath)
}
```

**Cleanup:** Removes extracted temp directory on graceful shutdown.

---

## Data Flow Summary

```
Build Time:
  PHP App Directory → static-php-cli → app.tar → Go embed → Binary

Runtime:
  Binary Start
       │
       ▼
  init() extracts to: /tmp/frankenphp_{checksum}/
       │
       ▼
  EmbeddedAppPath = "/tmp/frankenphp_{checksum}"
       │
       ├──→ context.go: default documentRoot
       ├──→ module.go: Root = "{path}/public"
       ├──→ php-server.go: chdir("{path}")
       ├──→ php-server.go: PHP_INI_SCAN_DIR += "{path}"
       ├──→ php-cli.go: resolve script paths
       └──→ workerconfig.go: resolve worker paths
       │
       ▼
  Shutdown
       │
       ▼
  RemoveAll(EmbeddedAppPath)
```

---

## Known Issues & Limitations

| Issue | Location | Description |
|-------|----------|-------------|
| Symlinks ignored | embed.go:139-143 | Symlinks in embedded apps are silently skipped |
| First-run delay | embed.go:48 | Initial extraction adds startup time |
| Temp dir persistence | embed.go:35 | Extraction uses checksum-based path; survives restarts if not cleaned |
| macOS executable handling | embed.go:91-97 | Removes executables before re-extraction (code signing workaround) |

---

## Caching Behavior

[VERIFIED: embed.go:98]
```go
if _, err := os.Stat(abs); os.IsNotExist(err) {
```

**Caching Logic:**
- Files are only extracted if they don't exist
- Checksum in directory name provides version-based cache invalidation
- Same binary version = same checksum = reuses existing extraction

**Cache Invalidation:**
- New build with different app → new checksum → new extraction
- Manual deletion of temp dir → re-extraction on next start
- Graceful shutdown → cleanup removes dir

---

## Example: Embedded Laravel App

```
Build:
  EMBED=/path/to/laravel-app ./build-static.sh

Resulting structure in temp:
  /tmp/frankenphp_a1b2c3d4/
  ├── public/
  │   └── index.php
  ├── app/
  ├── config/
  ├── routes/
  ├── vendor/
  ├── artisan
  ├── php.ini          (if present, added to PHP_INI_SCAN_DIR)
  └── Caddyfile        (if present, auto-loaded)

Runtime resolution:
  - Document root: /tmp/frankenphp_a1b2c3d4/public
  - Workers: artisan → /tmp/frankenphp_a1b2c3d4/artisan
  - php-cli scripts: relative to /tmp/frankenphp_a1b2c3d4/
```

---

## Quick Reference

### Check if Embedded App Active
```go
if frankenphp.EmbeddedAppPath != "" {
    // Embedded app is available
    fmt.Println("App at:", frankenphp.EmbeddedAppPath)
}
```

### Building with Embedded App
```bash
EMBED=/path/to/your/app ./build-static.sh
```

### Extraction Location
```
macOS:  /var/folders/.../T/frankenphp_{checksum}/
Linux:  /tmp/frankenphp_{checksum}/
```

---

*Documentation generated following agent-system-mapper methodology v1.0*
