# Static Binary Build Process - Code Flow

## Metadata
| Field | Value |
|-------|-------|
| Repository | `github.com/dunglas/frankenphp` |
| Commit | N/A (no git repo initialized) |
| Documented | `2026-01-04` |
| Trigger | `./build-static.sh` (with optional env variables) |
| End State | Standalone `frankenphp-{os}-{arch}` binary in `dist/` |

## Verification Summary
| Status | Count |
|--------|-------|
| VERIFIED | 32 |
| INFERRED | 4 |
| NOT_FOUND | 2 |
| ASSUMED | 1 |

---

## Flow Diagram

```
[./build-static.sh invoked]
           │
           ▼
    ┌─────────────────────────────┐
    │ Step 1: Environment Setup   │
    │ - Detect OS/arch            │
    │ - Set PHP_VERSION           │
    │ - Set default extensions    │
    └─────────────┬───────────────┘
                  │
                  ▼
    ┌─────────────────────────────┐
    │ Step 2: Install Deps        │
    │ - brew install (macOS)      │
    │ - Clone static-php-cli      │
    └─────────────┬───────────────┘
                  │
                  ▼
    ┌─────────────────────────────┐
    │ Step 3: Extension Selection │
    │ - Check PHP_EXTENSIONS env  │
    │ - OR dump from composer     │
    │ - OR use defaultExtensions  │
    └─────────────┬───────────────┘
                  │
                  ▼
    ┌─────────────────────────────┐
    │ Step 4: EMBED App Setup     │
    │ - If EMBED set, add arg     │
    │ - --with-frankenphp-app     │
    └─────────────┬───────────────┘
                  │
                  ▼
    ┌─────────────────────────────┐
    │ Step 5: SPC Commands        │
    │ - spc doctor --auto-fix     │
    │ - spc install-pkg go-xcaddy │
    │ - spc download              │
    │ - spc build --build-frankphp│
    └─────────────┬───────────────┘
                  │
                  ▼
    ┌─────────────────────────────┐
    │ Step 6: Final Binary        │
    │ - Copy to dist/             │
    │ - Verify with version cmd   │
    └─────────────────────────────┘
                  │
                  ▼
[dist/frankenphp-{os}-{arch} ready]
```

---

## Detailed Flow

### Step 1: Script Initialization & Environment Detection

[VERIFIED: build-static.sh:1-15]
```bash
#!/bin/bash

set -o errexit
set -x

if ! type "git" >/dev/null 2>&1; then
	echo "The \"git\" command must be installed."
	exit 1
fi

CURRENT_DIR=$(pwd)

arch="$(uname -m)"
os="$(uname -s | tr '[:upper:]' '[:lower:]')"
[ "$os" = "darwin" ] && os="mac"
```

**Actions:**
- Enable error exit (`errexit`) - script fails on any command error
- Enable debug output (`-x`) - shows each command before execution
- Verify `git` is installed (required)
- Detect CPU architecture (`arm64`, `x86_64`)
- Detect OS and normalize (`darwin` → `mac`, `linux` stays `linux`)

**Data out:**
```
arch = "arm64" | "x86_64"
os = "mac" | "linux"
CURRENT_DIR = "/path/to/frankenphp"
```

---

### Step 2: Configuration Variables

[VERIFIED: build-static.sh:17-31]
```bash
# Supported variables:
# - PHP_VERSION: PHP version to build (default: "8.4")
# - PHP_EXTENSIONS: PHP extensions to build (default: ${defaultExtensions} set below)
# - PHP_EXTENSION_LIBS: PHP extension libraries to build (default: ${defaultExtensionLibs} set below)
# - FRANKENPHP_VERSION: FrankenPHP version (default: current Git commit)
# - EMBED: Path to the PHP app to embed (default: none)
# - DEBUG_SYMBOLS: Enable debug symbols if set to 1 (default: none)
# - MIMALLOC: Use mimalloc as the allocator if set to 1 (default: none)
# - XCADDY_ARGS: Additional arguments to pass to xcaddy
# - RELEASE: [maintainer only] Create a GitHub release if set to 1 (default: none)

# - SPC_REL_TYPE: Release type to download (accept "source" and "binary", default: "source")
# - SPC_OPT_BUILD_ARGS: Additional arguments to pass to spc build
# - SPC_OPT_DOWNLOAD_ARGS: Additional arguments to pass to spc download
# - SPC_LIBC: Set to glibc to build with GNU toolchain (default: musl)
```

**Key Variables for SQL Server Build:**
| Variable | Purpose | Default |
|----------|---------|---------|
| `PHP_EXTENSIONS` | Extensions to compile | includes `pdo_sqlsrv` |
| `EMBED` | Path to PHP app to embed | none |
| `PHP_VERSION` | PHP version | 8.4 |

---

### Step 3: SPC (static-php-cli) Configuration

[VERIFIED: build-static.sh:33-56]
```bash
# init spc command, if we use spc binary, just use it instead of fetching source
if [ -z "${SPC_REL_TYPE}" ]; then
	SPC_REL_TYPE="source"
fi
# init spc libc
if [ -z "${SPC_LIBC}" ]; then
	if [ "${os}" = "linux" ]; then
		SPC_LIBC="musl"
	fi
fi
# init spc build additional args
if [ -z "${SPC_OPT_BUILD_ARGS}" ]; then
	SPC_OPT_BUILD_ARGS=""
fi
if [ "${SPC_LIBC}" = "musl" ] && [[ "${SPC_OPT_BUILD_ARGS}" != *"--disable-opcache-jit"* ]]; then
	SPC_OPT_BUILD_ARGS="${SPC_OPT_BUILD_ARGS} --disable-opcache-jit"
fi
# init spc download additional args
if [ -z "${SPC_OPT_DOWNLOAD_ARGS}" ]; then
	SPC_OPT_DOWNLOAD_ARGS="--ignore-cache-sources=php-src --retry 5"
	if [ "${SPC_LIBC}" = "musl" ]; then
		SPC_OPT_DOWNLOAD_ARGS="${SPC_OPT_DOWNLOAD_ARGS} --prefer-pre-built"
	fi
fi
```

**macOS Behavior** [INFERRED]:
- `SPC_LIBC` is NOT set on macOS (only on Linux)
- OPcache JIT is enabled on macOS
- Uses native macOS C library

---

### Step 4: PHP Version Resolution

[VERIFIED: build-static.sh:62-77]
```bash
if [ -z "${PHP_VERSION}" ]; then
	get_latest_php_version() {
		input="$1"
		json=$(curl -s "https://www.php.net/releases/index.php?json&version=$input")
		latest=$(echo "$json" | jq -r '.version')

		if [[ "$latest" == "$input"* ]]; then
			echo "$latest"
		else
			echo "$input"
		fi
	}

	PHP_VERSION="$(get_latest_php_version "8.4")"
	export PHP_VERSION
fi
```

**External Call:**
- `GET https://www.php.net/releases/index.php?json&version=8.4`
- Returns latest 8.4.x version (e.g., `8.4.1`)

---

### Step 5: Default Extensions Definition (CRITICAL FOR SQL SERVER)

[VERIFIED: build-static.sh:79-80]
```bash
defaultExtensions="amqp,apcu,ast,bcmath,brotli,bz2,calendar,ctype,curl,dba,dom,exif,fileinfo,filter,ftp,gd,gmp,gettext,iconv,igbinary,imagick,intl,ldap,lz4,mbregex,mbstring,memcache,memcached,mysqli,mysqlnd,opcache,openssl,password-argon2,parallel,pcntl,pdo,pdo_mysql,pdo_pgsql,pdo_sqlite,pdo_sqlsrv,pgsql,phar,posix,protobuf,readline,redis,session,shmop,simplexml,soap,sockets,sodium,sqlite3,ssh2,sysvmsg,sysvsem,sysvshm,tidy,tokenizer,xlswriter,xml,xmlreader,xmlwriter,xsl,xz,zip,zlib,yaml,zstd"
defaultExtensionLibs="libavif,nghttp2,nghttp3,ngtcp2,watcher"
```

**SQL Server Extensions Included:**
- `pdo` - PDO base
- `pdo_sqlsrv` - **SQL Server PDO driver** ✓

**Data shape:**
```
defaultExtensions = comma-separated string of 70+ extensions
```

---

### Step 6: Dependency Installation (macOS)

[VERIFIED: build-static.sh:110-125]
```bash
if type "brew" >/dev/null 2>&1; then
	if ! type "composer" >/dev/null; then
		packages="composer"
	fi
	if ! type "go" >/dev/null 2>&1; then
		packages="${packages} go"
	fi
	if [ -n "${RELEASE}" ] && ! type "gh" >/dev/null 2>&1; then
		packages="${packages} gh"
	fi

	if [ -n "${packages}" ]; then
		# shellcheck disable=SC2086
		brew install --formula --quiet ${packages}
	fi
fi
```

**macOS Dependencies:**
- `composer` - PHP package manager (for static-php-cli)
- `go` - Go compiler (for building FrankenPHP)
- `gh` - GitHub CLI (only for releases)

---

### Step 7: static-php-cli Setup

[VERIFIED: build-static.sh:127-148]
```bash
if [ "${SPC_REL_TYPE}" = "binary" ]; then
	mkdir -p static-php-cli/
	cd static-php-cli/
	if [[ "${arch}" =~ "arm" ]]; then
		dl_arch="aarch64"
	else
		dl_arch="${arch}"
	fi
	curl -o spc -fsSL "https://dl.static-php.dev/static-php-cli/spc-bin/nightly/spc-linux-${dl_arch}"
	chmod +x spc
	spcCommand="./spc"
elif [ -d "static-php-cli/src" ]; then
	cd static-php-cli/
	git pull
	composer install --no-dev -a --no-interaction
	spcCommand="./bin/spc"
else
	git clone --depth 1 https://github.com/crazywhalecc/static-php-cli --branch main
	cd static-php-cli/
	composer install --no-dev -a --no-interaction
	spcCommand="./bin/spc"
fi
```

**Flow Branches:**
1. **Binary mode** (Linux only): Downloads pre-built `spc` binary
2. **Existing source**: Pulls latest, runs `composer install`
3. **Fresh clone**: Clones repo, runs `composer install`

**External Call:**
- `git clone https://github.com/crazywhalecc/static-php-cli`

**Data out:**
```
spcCommand = "./bin/spc" | "./spc"
```

---

### Step 8: Extension Selection Logic

[VERIFIED: build-static.sh:150-160]
```bash
# Extensions to build
if [ -z "${PHP_EXTENSIONS}" ]; then
	# enable EMBED mode, first check if project has dumped extensions
	if [ -n "${EMBED}" ] && [ -f "${EMBED}/composer.json" ] && [ -f "${EMBED}/composer.lock" ] && [ -f "${EMBED}/vendor/installed.json" ]; then
		cd "${EMBED}"
		# read the extensions using spc dump-extensions
		PHP_EXTENSIONS=$(${spcCommand} dump-extensions "${EMBED}" --format=text --no-dev --no-ext-output="${defaultExtensions}")
	else
		PHP_EXTENSIONS="${defaultExtensions}"
	fi
fi
```

**Extension Selection Priority:**
1. Environment variable `PHP_EXTENSIONS` if set
2. Auto-detect from `EMBED` app's composer dependencies
3. Fall back to `defaultExtensions` (includes `pdo_sqlsrv`)

**For SQL Server, ensure either:**
- Use default (already includes `pdo_sqlsrv`)
- OR set `PHP_EXTENSIONS="pdo,pdo_sqlsrv,..."` explicitly

---

### Step 9: Library Dependencies

[VERIFIED: build-static.sh:162-177]
```bash
# Additional libraries to build
if [ -z "${PHP_EXTENSION_LIBS}" ]; then
	PHP_EXTENSION_LIBS="${defaultExtensionLibs}"
fi

# The Brotli library must always be built as it is required by http://github.com/dunglas/caddy-cbrotli
if ! echo "${PHP_EXTENSION_LIBS}" | grep -q "\bbrotli\b"; then
	PHP_EXTENSION_LIBS="${PHP_EXTENSION_LIBS},brotli"
fi

# The mimalloc library must be built if MIMALLOC is true
if [ -n "${MIMALLOC}" ]; then
	if ! echo "${PHP_EXTENSION_LIBS}" | grep -q "\bmimalloc\b"; then
		PHP_EXTENSION_LIBS="${PHP_EXTENSION_LIBS},mimalloc"
	fi
fi
```

**Required Libraries:**
- `brotli` - Always added (required by Caddy cbrotli)
- `libavif,nghttp2,nghttp3,ngtcp2,watcher` - Default set

[NOT_FOUND: unixODBC library not explicitly mentioned]
- SQL Server driver (`pdo_sqlsrv`) requires unixODBC
- [ASSUMED: static-php-cli handles ODBC dependencies internally when pdo_sqlsrv is requested]

---

### Step 10: EMBED App Configuration

[VERIFIED: build-static.sh:179-186]
```bash
# Embed PHP app, if any
if [ -n "${EMBED}" ] && [ -d "${EMBED}" ]; then
	if [[ "${EMBED}" != /* ]]; then
		EMBED="${CURRENT_DIR}/${EMBED}"
	fi
	# shellcheck disable=SC2089
	SPC_OPT_BUILD_ARGS="${SPC_OPT_BUILD_ARGS} --with-frankenphp-app='${EMBED}'"
fi
```

**If EMBED is set:**
1. Convert relative path to absolute
2. Add `--with-frankenphp-app` argument to build command

**This triggers the embed.go flow at runtime** (see Step 14).

---

### Step 11: Xcaddy Module Configuration

[VERIFIED: build-static.sh:201]
```bash
export SPC_CMD_VAR_FRANKENPHP_XCADDY_MODULES="--with github.com/dunglas/mercure/caddy --with github.com/dunglas/vulcain/caddy --with github.com/dunglas/caddy-cbrotli"
```

**Additional Caddy Modules:**
- `mercure/caddy` - Real-time pub/sub
- `vulcain/caddy` - HTTP/2 push
- `caddy-cbrotli` - Brotli compression

---

### Step 12: SPC Build Commands

[VERIFIED: build-static.sh:203-212]
```bash
# Build FrankenPHP
${spcCommand} doctor --auto-fix
for pkg in ${SPC_OPT_INSTALL_ARGS}; do
	${spcCommand} install-pkg "${pkg}"
done
# shellcheck disable=SC2086
${spcCommand} download --with-php="${PHP_VERSION}" --for-extensions="${PHP_EXTENSIONS}" --for-libs="${PHP_EXTENSION_LIBS}" ${SPC_OPT_DOWNLOAD_ARGS}
export FRANKENPHP_SOURCE_PATH="${CURRENT_DIR}"
# shellcheck disable=SC2086,SC2090
${spcCommand} build --enable-zts --build-embed --build-frankenphp ${SPC_OPT_BUILD_ARGS} "${PHP_EXTENSIONS}" --with-libs="${PHP_EXTENSION_LIBS}"
```

**SPC Command Sequence:**

| Command | Purpose |
|---------|---------|
| `spc doctor --auto-fix` | Check and fix build environment |
| `spc install-pkg go-xcaddy` | Install xcaddy for building Caddy |
| `spc download` | Download PHP source, extension sources, library sources |
| `spc build` | Compile everything into static binary |

**Critical Build Flags:**
```
--enable-zts          # Thread-safe PHP (required for FrankenPHP)
--build-embed         # Build PHP as embeddable library
--build-frankenphp    # Build FrankenPHP binary (not just PHP)
```

**Data passed to spc build:**
```
FRANKENPHP_SOURCE_PATH = path to FrankenPHP source
PHP_EXTENSIONS = comma-separated extensions including pdo_sqlsrv
PHP_EXTENSION_LIBS = comma-separated libraries
```

[NOT_FOUND: Internal spc build process - external to FrankenPHP codebase]

---

### Step 13: Final Binary Assembly

[VERIFIED: build-static.sh:219-224]
```bash
cd ../..

bin="dist/frankenphp-${os}-${arch}"
cp "dist/static-php-cli/buildroot/bin/frankenphp" "${bin}"
"${bin}" version
"${bin}" build-info
```

**Actions:**
1. Copy compiled binary from spc buildroot to dist/
2. Name format: `frankenphp-mac-arm64` or `frankenphp-linux-x86_64`
3. Verify binary works with `version` command
4. Show build info (extensions, PHP version)

**Data out:**
```
Binary location: dist/frankenphp-{os}-{arch}
```

---

### Step 14: Embedded App Extraction (Runtime)

When the built binary runs with an embedded app:

[VERIFIED: embed.go:23-27]
```go
//go:embed app.tar
var embeddedApp []byte

//go:embed app_checksum.txt
var embeddedAppChecksum []byte
```

**Build-time:** static-php-cli packages app directory into `app.tar` and generates `app_checksum.txt`

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

**Runtime Flow:**
1. `init()` runs automatically at binary startup
2. Check if `embeddedApp` has content
3. Create temp directory with checksum name (caching)
4. Extract tar to temp directory
5. Set `EmbeddedAppPath` for use by Caddy module

[VERIFIED: embed.go:91-97] - macOS-specific handling:
```go
if runtime.GOOS == "darwin" && mode&0111 != 0 {
	// See comment in writeFile.
	err := os.Remove(abs)
	if err != nil && !errors.Is(err, fs.ErrNotExist) {
		return err
	}
}
```

**macOS Consideration:** Executable files are removed before extraction to handle code signing issues.

---

### Step 15: Caddy Module Entry Point

[VERIFIED: caddy/frankenphp/main.go:1-16]
```go
package main

import (
	caddycmd "github.com/caddyserver/caddy/v2/cmd"

	// plug in Caddy modules here.
	_ "github.com/caddyserver/caddy/v2/modules/standard"
	_ "github.com/dunglas/caddy-cbrotli"
	_ "github.com/dunglas/frankenphp/caddy"
	_ "github.com/dunglas/mercure/caddy"
	_ "github.com/dunglas/vulcain/caddy"
)

func main() {
	caddycmd.Main()
}
```

**Import Side Effects:**
- Each `_` import registers its module with Caddy
- `frankenphp/caddy` registers PHP handling
- `mercure/caddy` registers real-time features
- `vulcain/caddy` registers HTTP/2 push

---

## External Calls Summary

| Call | Location | Purpose |
|------|----------|---------|
| `curl https://www.php.net/releases/` | build-static.sh:65 | Get latest PHP version |
| `git clone static-php-cli` | build-static.sh:144 | Get build tool |
| `spc download` | build-static.sh:209 | Download PHP/extension sources |
| `spc build` | build-static.sh:212 | Compile static binary |

---

## SQL Server Specific Notes

### Extension Inclusion
[VERIFIED: build-static.sh:79]
- `pdo_sqlsrv` IS in the default extensions list
- No additional configuration needed for basic SQL Server support

### Build Command for SQL Server Only
```bash
PHP_EXTENSIONS="pdo,pdo_sqlsrv,ctype,session,mbstring,openssl" \
  ./build-static.sh
```

### Verify SQL Server Support
```bash
./dist/frankenphp-mac-arm64 php -m | grep -i sql
# Expected output:
# pdo_sqlsrv
# sqlsrv (if included)
```

---

## Known Issues

1. **[INFERRED]** unixODBC dependency is not explicitly documented
   - SQL Server drivers require ODBC
   - static-php-cli likely handles this internally

2. **[VERIFIED: build-static.sh:47-49]** OPcache JIT disabled on musl Linux
   - Only affects Linux builds, not macOS

3. **[VERIFIED: embed.go:139-143]** Symlinks not supported in embedded apps
   ```go
   case mode&os.ModeSymlink != 0:
       // TODO: ignore these for now. They were breaking x/build tests.
   ```

4. **[INFERRED]** First run extracts to temp directory
   - May have slight startup delay on first execution
   - Subsequent runs use cached extraction (checksum-based)

---

## Quick Reference: Building with SQL Server for macOS

```bash
cd frankenphp

# Option 1: Full default build (already includes pdo_sqlsrv)
./build-static.sh

# Option 2: With your embedded app
EMBED=/path/to/your/app ./build-static.sh

# Option 3: Minimal with SQL Server
PHP_EXTENSIONS="pdo,pdo_sqlsrv,ctype,session,mbstring,json,openssl,curl" \
  ./build-static.sh

# Result
ls -la dist/frankenphp-mac-*
# frankenphp-mac-arm64 (Apple Silicon)
# frankenphp-mac-x86_64 (Intel)
```

---

*Documentation generated following agent-system-mapper methodology v1.0*
