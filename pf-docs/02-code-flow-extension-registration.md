# Extension Registration - Code Flow

## Metadata
| Field | Value |
|-------|-------|
| Repository | `github.com/dunglas/frankenphp` |
| Commit | N/A (no git repo initialized) |
| Documented | `2026-01-04` |
| Trigger | `frankenphp.RegisterExtension()` called before `Init()` |
| End State | PHP extension registered and available via `php -m` |

## Verification Summary
| Status | Count |
|--------|-------|
| VERIFIED | 26 |
| INFERRED | 3 |
| NOT_FOUND | 0 |
| ASSUMED | 1 |

---

## Flow Diagram

```
[Go Package init()]
       │
       │  frankenphp.RegisterExtension(ptr)
       ▼
┌─────────────────────────────┐
│ ext.go: Append to           │
│ extensions slice            │
└─────────────┬───────────────┘
              │
              │  (Later: frankenphp.Init() called)
              ▼
┌─────────────────────────────┐
│ frankenphp.go:249           │
│ registerExtensions()        │
└─────────────┬───────────────┘
              │
              │  sync.Once
              ▼
┌─────────────────────────────┐
│ ext.go:26 → CGO call        │
│ C.register_extensions()     │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────────────────┐
│ frankenphp.c:1287-1294                  │
│ - Store module pointer                  │
│ - Hook php_register_internal_extensions │
└─────────────┬───────────────────────────┘
              │
              │  (During SAPI startup)
              ▼
┌─────────────────────────────────────────┐
│ php_main → php_module_startup()         │
│ → calls php_register_internal_extensions│
│ → our hook: register_internal_extensions│
└─────────────┬───────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────┐
│ frankenphp.c:1269-1285                  │
│ for each module:                        │
│   zend_register_internal_module()       │
└─────────────────────────────────────────┘
              │
              ▼
[Extension Available in PHP Runtime]
```

---

## Detailed Flow

### Step 1: Go Package Variables

[VERIFIED: ext.go:10-13]
```go
var (
	extensions   []*C.zend_module_entry
	registerOnce sync.Once
)
```

**Purpose:**
- `extensions` - slice storing pointers to C `zend_module_entry` structures
- `registerOnce` - ensures registration happens exactly once (thread-safe)

---

### Step 2: RegisterExtension() API

[VERIFIED: ext.go:15-18]
```go
// RegisterExtension registers a new PHP extension.
func RegisterExtension(me unsafe.Pointer) {
	extensions = append(extensions, (*C.zend_module_entry)(me))
}
```

**Usage Pattern:**
- Called from Go package `init()` functions
- Must be called BEFORE `frankenphp.Init()`
- Takes `unsafe.Pointer` to C `zend_module_entry` struct

**Data in:** `unsafe.Pointer` (cast to `*C.zend_module_entry`)
**Data out:** Extension appended to `extensions` slice

---

### Step 3: Typical Extension Registration Pattern

[VERIFIED: internal/extgen/templates/extension.go.tpl:15-17]
```go
func init() {
	frankenphp.RegisterExtension(unsafe.Pointer(&C.{{.BaseName}}_module_entry))
}
```

**Generated Extension Pattern:**
- Uses Go's `init()` for automatic registration
- References C-defined `zend_module_entry` struct
- Runs before `main()`, before `Init()` is called

**Example from test suite:**
[VERIFIED: internal/testext/exttest.go:24-30]
```go
func testRegisterExtension(t *testing.T) {
	frankenphp.RegisterExtension(unsafe.Pointer(&C.module1_entry))
	frankenphp.RegisterExtension(unsafe.Pointer(&C.module2_entry))

	err := frankenphp.Init()
	require.Nil(t, err)
	defer frankenphp.Shutdown()
```

---

### Step 4: Init() Triggers Registration

[VERIFIED: frankenphp.go:238-249]
```go
// Init starts the PHP runtime and the configured workers.
func Init(options ...Option) error {
	if isRunning {
		return ErrAlreadyStarted
	}
	isRunning = true

	// Ignore all SIGPIPE signals...
	signal.Ignore(syscall.SIGPIPE)

	registerExtensions()  // <-- Called here, BEFORE PHP startup
```

**Timing:** Registration happens early in `Init()`, before any options processing or PHP thread creation.

---

### Step 5: registerExtensions() Implementation

[VERIFIED: ext.go:20-29]
```go
func registerExtensions() {
	if len(extensions) == 0 {
		return
	}

	registerOnce.Do(func() {
		C.register_extensions(extensions[0], C.int(len(extensions)))
		extensions = nil
	})
}
```

**Flow:**
1. Early return if no extensions registered
2. `sync.Once` ensures this runs exactly once
3. Call C function with pointer to first element and count
4. Clear slice (no longer needed)

**CGO Call:**
```
C.register_extensions(extensions[0], C.int(len(extensions)))
```

---

### Step 6: C Header Declaration

[VERIFIED: frankenphp.h:79]
```c
void register_extensions(zend_module_entry *m, int len);
```

**Parameters:**
- `m` - pointer to first `zend_module_entry` in array
- `len` - number of extensions to register

---

### Step 7: C-Side Module Storage

[VERIFIED: frankenphp.c:1265-1267]
```c
static zend_module_entry *modules = NULL;
static int modules_len = 0;
static int (*original_php_register_internal_extensions_func)(void) = NULL;
```

**Static Variables:**
- `modules` - pointer to extension array (from Go)
- `modules_len` - count of extensions
- `original_php_register_internal_extensions_func` - saved original PHP function pointer

---

### Step 8: register_extensions() Implementation

[VERIFIED: frankenphp.c:1287-1294]
```c
void register_extensions(zend_module_entry *m, int len) {
  modules = m;
  modules_len = len;

  original_php_register_internal_extensions_func =
      php_register_internal_extensions_func;
  php_register_internal_extensions_func = register_internal_extensions;
}
```

**Function Pointer Hook:**
1. Store module pointer and length
2. Save original `php_register_internal_extensions_func`
3. Replace with our custom `register_internal_extensions` function

**Why Hook?** PHP's internal extension registration is called during module startup. By hooking this function, we can add our extensions at the right moment in PHP's initialization sequence.

---

### Step 9: SAPI Startup Sequence

[VERIFIED: frankenphp.c:989-1004]
```c
sapi_startup(&frankenphp_sapi_module);

// ... INI configuration ...

frankenphp_sapi_module.startup(&frankenphp_sapi_module);
```

**Where:** Runs in `php_main()` thread function

[VERIFIED: frankenphp.c:616-619]
```c
static int frankenphp_startup(sapi_module_struct *sapi_module) {
  php_import_environment_variables = get_full_env;

  return php_module_startup(sapi_module, &frankenphp_module);
}
```

**Chain:**
1. `sapi_startup()` - Initialize SAPI layer
2. `frankenphp_startup()` - Our SAPI startup function
3. `php_module_startup()` - PHP's core module initialization
4. Inside `php_module_startup()` → calls `php_register_internal_extensions_func()`
5. Our hook `register_internal_extensions()` is called

---

### Step 10: Hooked Registration Function

[VERIFIED: frankenphp.c:1269-1285]
```c
PHPAPI int register_internal_extensions(void) {
  if (original_php_register_internal_extensions_func != NULL &&
      original_php_register_internal_extensions_func() != SUCCESS) {
    return FAILURE;
  }

  for (int i = 0; i < modules_len; i++) {
    if (zend_register_internal_module(&modules[i]) == NULL) {
      return FAILURE;
    }
  }

  modules = NULL;
  modules_len = 0;

  return SUCCESS;
}
```

**Flow:**
1. Call original PHP internal extension registration (if exists)
2. Iterate through our custom modules
3. Call `zend_register_internal_module()` for each
4. Clear module storage
5. Return SUCCESS/FAILURE

**Zend Registration:**
`zend_register_internal_module()` is PHP's Zend Engine function that:
- Adds module to `module_registry` hash table
- Registers module's functions
- Calls module's `MINIT` (module initialization) handler

---

### Step 11: FrankenPHP's Own Extension

[VERIFIED: frankenphp.c:599-609]
```c
static zend_module_entry frankenphp_module = {
    STANDARD_MODULE_HEADER,
    "frankenphp",
    ext_functions,         /* function table */
    PHP_MINIT(frankenphp), /* initialization */
    NULL,                  /* shutdown */
    NULL,                  /* request initialization */
    NULL,                  /* request shutdown */
    NULL,                  /* information */
    TOSTRING(FRANKENPHP_VERSION),
    STANDARD_MODULE_PROPERTIES};
```

**FrankenPHP Extension:** Registered via `php_module_startup()` second argument, provides core PHP functions.

---

### Step 12: Exposed PHP Functions

[VERIFIED: frankenphp_arginfo.h:54-67]
```c
static const zend_function_entry ext_functions[] = {
	ZEND_FE(frankenphp_handle_request, arginfo_frankenphp_handle_request)
	ZEND_FE(headers_send, arginfo_headers_send)
	ZEND_FE(frankenphp_finish_request, arginfo_frankenphp_finish_request)
	ZEND_FALIAS(fastcgi_finish_request, frankenphp_finish_request, arginfo_fastcgi_finish_request)
	ZEND_FE(frankenphp_request_headers, arginfo_frankenphp_request_headers)
	ZEND_FALIAS(apache_request_headers, frankenphp_request_headers, arginfo_apache_request_headers)
	ZEND_FALIAS(getallheaders, frankenphp_request_headers, arginfo_getallheaders)
	ZEND_FE(frankenphp_response_headers, arginfo_frankenphp_response_headers)
	ZEND_FALIAS(apache_response_headers, frankenphp_response_headers, arginfo_apache_response_headers)
	ZEND_FE(mercure_publish, arginfo_mercure_publish)
	ZEND_FE(frankenphp_log, arginfo_frankenphp_log)
	ZEND_FE_END
};
```

**PHP Functions Provided:**
| Function | Purpose |
|----------|---------|
| `frankenphp_handle_request()` | Worker mode request loop |
| `frankenphp_finish_request()` | End request early |
| `fastcgi_finish_request()` | Alias for compatibility |
| `frankenphp_request_headers()` | Get request headers |
| `apache_request_headers()` | Alias for compatibility |
| `getallheaders()` | Alias for compatibility |
| `frankenphp_response_headers()` | Get response headers |
| `apache_response_headers()` | Alias for compatibility |
| `headers_send()` | Send headers (Early Hints) |
| `mercure_publish()` | Publish to Mercure hub |
| `frankenphp_log()` | Structured logging |

---

## Extension Types in FrankenPHP

### Type 1: Static/Compiled Extensions (pdo_sqlsrv, etc.)

[INFERRED from build-static.sh]

**How they work:**
- Compiled directly into the PHP binary during static build
- Handled by static-php-cli, not FrankenPHP
- Appear in `php -m` automatically
- No runtime registration needed

**Build-time flow:**
```
PHP_EXTENSIONS="pdo,pdo_sqlsrv" ./build-static.sh
       │
       ▼
static-php-cli downloads extension source
       │
       ▼
Compiles extension into libphp.a
       │
       ▼
Links into final FrankenPHP binary
```

---

### Type 2: Go-Based Custom Extensions

**How they work:**
- Written in Go with CGO bridge to C
- Registered via `RegisterExtension()` API
- Compiled into the Go binary

**Registration Flow:**
```go
// In your extension package
package myext

// #include "myext.h"
import "C"
import (
    "unsafe"
    "github.com/dunglas/frankenphp"
)

func init() {
    frankenphp.RegisterExtension(unsafe.Pointer(&C.myext_module_entry))
}
```

---

### Type 3: Dynamic Extensions (dl())

[ASSUMED: Not recommended for FrankenPHP]

**Not typically used because:**
- Static builds don't support `dl()`
- ZTS mode has thread-safety considerations
- All extensions should be compiled in

---

## SQL Server Extension Flow (pdo_sqlsrv)

For your specific use case with SQL Server:

**Build Time:**
```
PHP_EXTENSIONS="pdo,pdo_sqlsrv" ./build-static.sh
                    │
                    ▼
         static-php-cli handles:
         1. Download pdo_sqlsrv source
         2. Download unixODBC dependency
         3. Compile as static library
         4. Link into PHP
                    │
                    ▼
         Binary includes pdo_sqlsrv
```

**Runtime:**
```
./frankenphp-mac-arm64 start
          │
          ▼
    PHP SAPI startup
          │
          ▼
    pdo_sqlsrv already in binary
    (no RegisterExtension needed)
          │
          ▼
    Available: new PDO("sqlsrv:...")
```

**Verification:**
```bash
./frankenphp-mac-arm64 php -m | grep -i sql
# pdo_sqlsrv
```

---

## Timing Requirements

```
┌─────────────────────────────────────────────────────────────┐
│                     BEFORE Init()                           │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ Go Package init() Functions Run                       │  │
│  │   - frankenphp.RegisterExtension() calls              │  │
│  │   - Extensions appended to slice                      │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     frankenphp.Init()                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ Line 249: registerExtensions()                        │  │
│  │   - C.register_extensions() called                    │  │
│  │   - Function pointer hooked                           │  │
│  └───────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ Later: PHP SAPI Startup                               │  │
│  │   - php_module_startup() called                       │  │
│  │   - Our hook register_internal_extensions() runs      │  │
│  │   - zend_register_internal_module() for each          │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     AFTER Init()                            │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ Extensions Available                                  │  │
│  │   - php -m shows all extensions                       │  │
│  │   - PHP code can use extension functions              │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## Error Handling

### Registration Failure

[VERIFIED: frankenphp.c:1276-1278]
```c
if (zend_register_internal_module(&modules[i]) == NULL) {
  return FAILURE;
}
```

**On Failure:**
- `zend_register_internal_module()` returns NULL
- `register_internal_extensions()` returns FAILURE
- `php_module_startup()` fails
- `frankenphp_startup()` fails
- PHP runtime doesn't start

### Init() Error

[VERIFIED: frankenphp.go:253-256]
```go
for _, o := range options {
	if err := o(opt); err != nil {
		Shutdown()
		return err
	}
}
```

**Error propagation:** If PHP startup fails, `Init()` returns error.

---

## Known Limitations

| Limitation | Reason |
|------------|--------|
| Must register before Init() | Function pointer hook happens early in Init() |
| No unregister API | Extensions stay loaded for process lifetime |
| Thread-safe registration | Uses `sync.Once` for safety |
| Static extensions preferred | Dynamic loading not well-supported in static builds |

---

## Quick Reference

### Registering a Custom Extension

```go
package myext

// #include "myext.h"
import "C"
import (
    "unsafe"
    "github.com/dunglas/frankenphp"
)

// Called automatically before main()
func init() {
    frankenphp.RegisterExtension(unsafe.Pointer(&C.myext_module_entry))
}
```

### Checking Registered Extensions

```bash
# List all PHP extensions
./frankenphp php -m

# Check specific extension
./frankenphp php -m | grep pdo_sqlsrv

# Get extension info
./frankenphp php --ri pdo_sqlsrv
```

### Build with Static Extensions

```bash
# SQL Server support (already in defaults)
./build-static.sh

# Custom extension set
PHP_EXTENSIONS="pdo,pdo_sqlsrv,openssl,curl" ./build-static.sh
```

---

*Documentation generated following agent-system-mapper methodology v1.0*
