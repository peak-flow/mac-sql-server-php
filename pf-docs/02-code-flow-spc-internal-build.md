# SPC Internal Build Process - Code Flow

## Metadata
| Field | Value |
|-------|-------|
| Repository | `github.com/crazywhalecc/static-php-cli` |
| Commit | `cloned at runtime by build-static.sh` |
| Documented | `2026-01-04` |
| Trigger | `spc build --build-frankenphp ...` |
| End State | `buildroot/bin/frankenphp` binary |

## Verification Summary
| Status | Count |
|--------|-------|
| VERIFIED | 45 |
| INFERRED | 3 |

---

## Flow Diagram

```
[spc build command]
        │
        ▼
┌───────────────────────────────┐
│ BuildPHPCommand::handle()     │
│ - Parse extensions/libs args  │
│ - Create Builder by OS        │
│ - Resolve dependencies        │
└───────────┬───────────────────┘
            │
            ▼
┌───────────────────────────────┐
│ proveLibs()                   │
│ - Load library classes        │
│ - Calculate dependencies      │
│ - Order by dependency graph   │
└───────────┬───────────────────┘
            │
            ▼
┌───────────────────────────────┐
│ proveExts()                   │
│ - Extract PHP source          │
│ - Extract extension sources   │
│ - Create Extension objects    │
│ - Check dependencies          │
└───────────┬───────────────────┘
            │
            ▼
┌───────────────────────────────┐
│ setupLibs()                   │
│ - For each library:           │
│   - Extract source            │
│   - Patch if needed           │
│   - Build (cmake/make/etc)    │
│   - Install to buildroot      │
└───────────┬───────────────────┘
            │
            ▼
┌───────────────────────────────┐
│ buildPHP()                    │
│ - Run buildconf               │
│ - Run ./configure             │
│ - Build embed SAPI (libphp.a) │
└───────────┬───────────────────┘
            │
            ▼
┌───────────────────────────────┐
│ buildFrankenphp()             │
│ - Set CGO environment         │
│ - Run xcaddy build            │
│ - Link libphp.a + Go code     │
└───────────┬───────────────────┘
            │
            ▼
┌───────────────────────────────┐
│ deploySAPIBinary()            │
│ - Extract debug symbols       │
│ - Strip binary                │
│ - Copy to buildroot/bin       │
└───────────┴───────────────────┘
            │
            ▼
[buildroot/bin/frankenphp ready]
```

---

## Detailed Flow

### Step 1: Command Entry Point

[VERIFIED: src/SPC/command/BuildPHPCommand.php:21-22]
```php
#[AsCommand('build', 'build PHP', ['build:php'])]
class BuildPHPCommand extends BuildCommand
```

**Symfony Console command** registered as `build` (alias `build:php`).

---

### Step 2: Parse Input Arguments

[VERIFIED: src/SPC/command/BuildPHPCommand.php:54-61]
```php
public function handle(): int
{
    // transform string to array
    $libraries = array_map('trim', array_filter(explode(',', $this->getOption('with-libs'))));
    // transform string to array
    $shared_extensions = array_map('trim', array_filter(explode(',', $this->getOption('build-shared'))));
    // transform string to array
    $static_extensions = $this->parseExtensionList($this->getArgument('extensions'));
```

**Data in:**
```
extensions = "pdo,pdo_sqlsrv,ctype,..."  (from command line)
--with-libs = "libavif,nghttp2,..."
--build-frankenphp = true
--enable-zts = true
```

---

### Step 3: Parse Build Target Rules

[VERIFIED: src/SPC/command/BuildPHPCommand.php:269-280]
```php
private function parseRules(array $shared_extensions = []): int
{
    $rule = BUILD_TARGET_NONE;
    $rule |= ($this->getOption('build-cli') ? BUILD_TARGET_CLI : BUILD_TARGET_NONE);
    $rule |= ($this->getOption('build-micro') ? BUILD_TARGET_MICRO : BUILD_TARGET_NONE);
    $rule |= ($this->getOption('build-fpm') ? BUILD_TARGET_FPM : BUILD_TARGET_NONE);
    $rule |= $this->getOption('build-embed') || !empty($shared_extensions) ? BUILD_TARGET_EMBED : BUILD_TARGET_NONE;
    $rule |= ($this->getOption('build-frankenphp') ? (BUILD_TARGET_FRANKENPHP | BUILD_TARGET_EMBED) : BUILD_TARGET_NONE);
    ...
}
```

**Key insight:** `--build-frankenphp` automatically sets `BUILD_TARGET_EMBED` because FrankenPHP requires `libphp.a`.

---

### Step 4: Create OS-Specific Builder

[VERIFIED: src/SPC/builder/BuilderProvider.php:22-31]
```php
public static function makeBuilderByInput(InputInterface $input): BuilderBase
{
    ini_set('memory_limit', '4G');

    self::$builder = match (PHP_OS_FAMILY) {
        'Windows' => new WindowsBuilder($input->getOptions()),
        'Darwin' => new MacOSBuilder($input->getOptions()),
        'Linux' => new LinuxBuilder($input->getOptions()),
        'BSD' => new BSDBuilder($input->getOptions()),
        default => throw new WrongUsageException('Current OS "' . PHP_OS_FAMILY . '" is not supported yet'),
    };
    ...
}
```

**Calls:** `new MacOSBuilder($options)` on macOS

---

### Step 5: Resolve Dependencies

[VERIFIED: src/SPC/command/BuildPHPCommand.php:127]
```php
[$extensions, $libraries, $not_included] = DependencyUtil::getExtsAndLibs(
    array_merge($static_extensions, $shared_extensions),
    $libraries,
    $include_suggest_ext,
    $include_suggest_lib
);
```

[VERIFIED: src/SPC/util/DependencyUtil.php:31-62]
```php
public static function platExtToLibs(): array
{
    $exts = Config::getExts();
    $libs = Config::getLibs();
    $dep_list = [];
    foreach ($exts as $ext_name => $ext) {
        // convert ext-depends value to ext@xxx
        $ext_depends = Config::getExt($ext_name, 'ext-depends', []);
        $ext_depends = array_map(fn ($x) => "ext@{$x}", $ext_depends);
        // merge ext-depends with lib-depends
        $lib_depends = Config::getExt($ext_name, 'lib-depends', []);
        $depends = array_merge($ext_depends, $lib_depends, ['php']);
        ...
    }
}
```

**Dependency Resolution Algorithm:**
1. Read `ext.json` and `lib.json` config files
2. Build dependency graph (ext→lib, lib→lib)
3. Topological sort to get build order
4. Return ordered arrays of extensions and libraries

---

### Step 6: Prove Libraries (proveLibs)

[VERIFIED: src/SPC/builder/unix/UnixBuilderBase.php:36-81]
```php
public function proveLibs(array $sorted_libraries): void
{
    // search all supported libs
    $support_lib_list = [];
    $classes = FileSystem::getClassesPsr4(
        ROOT_DIR . '/src/SPC/builder/' . osfamily2dir() . '/library',
        'SPC\builder\\' . osfamily2dir() . '\library'
    );
    foreach ($classes as $class) {
        if (defined($class . '::NAME') && $class::NAME !== 'unknown' && Config::getLib($class::NAME) !== null) {
            $support_lib_list[$class::NAME] = $class;
        }
    }
    ...
    // add lib object for builder
    foreach ($sorted_libraries as $library) {
        ...
        $lib = new ($support_lib_list[$library])($this);
        $this->addLib($lib);
    }
}
```

**Library Class Discovery:**
- Scans `src/SPC/builder/macos/library/` for PHP files
- Each file defines a library class with `NAME` constant
- Examples: `brotli.php`, `openssl.php`, `curl.php`

---

### Step 7: Prove Extensions (proveExts)

[VERIFIED: src/SPC/builder/BuilderBase.php:146-198]
```php
public function proveExts(array $static_extensions, array $shared_extensions = [], ...): void
{
    ...
    if (!$skip_extract) {
        $this->emitPatchPoint('before-php-extract');
        SourceManager::initSource(sources: ['php-src'], source_only: true);
        $this->emitPatchPoint('after-php-extract');
        if ($this->getPHPVersionID() >= 80000) {
            $this->emitPatchPoint('before-micro-extract');
            SourceManager::initSource(sources: ['micro'], source_only: true);
            $this->emitPatchPoint('after-micro-extract');
        }
        $this->emitPatchPoint('before-exts-extract');
        SourceManager::initSource(exts: [...$static_extensions, ...$shared_extensions]);
        $this->emitPatchPoint('after-exts-extract');
        // patch micro
        SourcePatcher::patchMicro();
    }

    foreach ([...$static_extensions, ...$shared_extensions] as $extension) {
        $class = AttributeMapper::getExtensionClassByName($extension) ?? Extension::class;
        /** @var Extension $ext */
        $ext = new $class($extension, $this);
        ...
        $this->addExt($ext);
    }
}
```

**Source Extraction Flow:**
1. Extract `php-src` tarball to `source/php-src/`
2. Extract `phpmicro` to `source/php-src/sapi/micro/`
3. Extract each external extension source

---

### Step 8: Setup Libraries (Build C Libraries)

[VERIFIED: src/SPC/builder/BuilderBase.php:56-72]
```php
public function setupLibs(): void
{
    // build all libs
    foreach ($this->libs as $lib) {
        $starttime = microtime(true);
        $status = $lib->setup($this->getOption('rebuild', false));
        match ($status) {
            LIB_STATUS_OK => logger()->info('lib [' . $lib::NAME . '] setup success, took ' . round(microtime(true) - $starttime, 2) . ' s'),
            LIB_STATUS_ALREADY => logger()->notice('lib [' . $lib::NAME . '] already built'),
            LIB_STATUS_INSTALL_FAILED => logger()->error('lib [' . $lib::NAME . '] install failed'),
            ...
        };
    }
}
```

[VERIFIED: src/SPC/builder/LibraryBase.php:175-207]
```php
public function tryBuild(bool $force_build = false): int
{
    ...
    if ($force_build) {
        ...
        // extract first if not exists
        if (!is_dir($this->source_dir)) {
            $this->getBuilder()->emitPatchPoint('before-library[ ' . static::NAME . ']-extract');
            SourceManager::initSource(libs: [static::NAME], source_only: true);
            $this->getBuilder()->emitPatchPoint('after-library[ ' . static::NAME . ']-extract');
        }

        if (!$this->patched && $this->patchBeforeBuild()) {
            file_put_contents($this->source_dir . '/.spc.patched', 'PATCHED!!!');
        }
        $this->getBuilder()->emitPatchPoint('before-library[ ' . static::NAME . ']-build');
        $this->build();  // <-- Abstract method, implemented by each library
        $this->installLicense();
        ...
    }
}
```

**Library Build Flow (per library):**
1. Extract source tarball
2. Apply patches if needed
3. Call `build()` method (cmake, autoconf, or custom)
4. Install headers/libs to `buildroot/`

---

### Step 9: Build PHP with Extensions

[VERIFIED: src/SPC/builder/macos/MacOSBuilder.php:81-175]
```php
public function buildPHP(int $build_target = BUILD_TARGET_NONE): void
{
    $this->emitPatchPoint('before-php-buildconf');
    SourcePatcher::patchBeforeBuildconf($this);

    shell()->cd(SOURCE_PATH . '/php-src')->exec(getenv('SPC_CMD_PREFIX_PHP_BUILDCONF'));

    $this->emitPatchPoint('before-php-configure');
    SourcePatcher::patchBeforeConfigure($this);

    ...
    $this->seekPhpSrcLogFileOnException(fn () => shell()->cd(SOURCE_PATH . '/php-src')->exec(
        getenv('SPC_CMD_PREFIX_PHP_CONFIGURE') . ' ' .
            ($enableCli ? '--enable-cli ' : '--disable-cli ') .
            ($enableFpm ? '--enable-fpm ' : '--disable-fpm ') .
            ($enableEmbed ? "--enable-embed={$embed_type} " : '--disable-embed ') .
            ...
            $this->makeStaticExtensionArgs() . ' ' .
            $envs_build_php
    ));

    $this->emitPatchPoint('before-php-make');
    SourcePatcher::patchBeforeMake($this);

    $this->cleanMake();

    if ($enableEmbed) {
        logger()->info('building embed');
        ...
        $this->buildEmbed();
    }
    if ($enableFrankenphp) {
        logger()->info('building frankenphp');
        $this->buildFrankenphp();
    }
}
```

**PHP Build Sequence:**
```
1. ./buildconf --force           # Generate configure script
2. ./configure --enable-embed... # Configure with extensions
3. make clean                    # Clean previous build
4. make embed                    # Build libphp.a
5. make install-*                # Install to buildroot
```

---

### Step 10: Generate Extension Configure Args

[VERIFIED: src/SPC/builder/BuilderBase.php:247-269]
```php
public function makeStaticExtensionArgs(): string
{
    $ret = [];
    foreach ($this->getExts() as $ext) {
        ...
        $arg ??= $ext->getConfigureArg();
        logger()->info($ext->getName() . ' is using ' . $arg);
        $ret[] = trim($arg);
    }
    logger()->debug('Using configure: ' . implode(' ', $ret));
    return implode(' ', $ret);
}
```

[VERIFIED: src/SPC/builder/Extension.php:76-88]
```php
public function getEnableArg(bool $shared = false): string
{
    $escapedPath = ...;
    $_name = str_replace('_', '-', $this->name);
    return match ($arg_type = Config::getExt($this->name, 'arg-type', 'enable')) {
        'enable' => '--enable-' . $_name . ($shared ? '=shared' : '') . ' ',
        'enable-path' => '--enable-' . $_name . '=' . ($shared ? 'shared,' : '') . $escapedPath . ' ',
        'with' => '--with-' . $_name . ($shared ? '=shared' : '') . ' ',
        'with-path' => '--with-' . $_name . '=' . ($shared ? 'shared,' : '') . $escapedPath . ' ',
        ...
    };
}
```

**Example Output:**
```
--enable-pdo --with-pdo-sqlsrv --enable-ctype --enable-dom --enable-mbstring ...
```

---

### Step 11: Build FrankenPHP Binary

[VERIFIED: src/SPC/builder/unix/UnixBuilderBase.php:404-466]
```php
protected function buildFrankenphp(): void
{
    GlobalEnvManager::addPathIfNotExists(GoXcaddy::getPath());
    $this->processFrankenphpApp();
    $nobrotli = $this->getLib('brotli') === null ? ',nobrotli' : '';
    $nowatcher = $this->getLib('watcher') === null ? ',nowatcher' : '';
    $xcaddyModules = getenv('SPC_CMD_VAR_FRANKENPHP_XCADDY_MODULES');
    $frankenphpSourceDir = getenv('FRANKENPHP_SOURCE_PATH') ?: SOURCE_PATH . '/frankenphp';

    $xcaddyModules = "--with github.com/dunglas/frankenphp={$frankenphpSourceDir} " .
        "--with github.com/dunglas/frankenphp/caddy={$frankenphpSourceDir}/caddy {$xcaddyModules}";
    ...

    $env = [...[
        'CGO_ENABLED' => '1',
        'CGO_CFLAGS' => clean_spaces($cflags),
        'CGO_LDFLAGS' => "{$this->arch_ld_flags} {$staticFlags} {$config['ldflags']} {$libs}",
        'XCADDY_GO_BUILD_FLAGS' => '-buildmode=pie ' .
            '-ldflags \"-linkmode=external ' . $extLdFlags . ' ' .
            '-X \'github.com/caddyserver/caddy/v2.CustomVersion=FrankenPHP ' .
            "v{$frankenPhpVersion} PHP {$libphpVersion} Caddy'\\\" " .
            "-tags={$muslTags}nobadger,nomysql,nopgx{$nobrotli}{$nowatcher}",
        'LD_LIBRARY_PATH' => BUILD_LIB_PATH,
    ], ...GoXcaddy::getEnvironment()];
    shell()->cd(BUILD_BIN_PATH)
        ->setEnv($env)
        ->exec("xcaddy build --output frankenphp {$xcaddyModules}");

    $this->deploySAPIBinary(BUILD_TARGET_FRANKENPHP);
}
```

**CGO Environment Variables:**
| Variable | Purpose |
|----------|---------|
| `CGO_ENABLED=1` | Enable C interop for Go |
| `CGO_CFLAGS` | C compiler flags (include paths, defines) |
| `CGO_LDFLAGS` | Linker flags (library paths, static libs) |
| `XCADDY_GO_BUILD_FLAGS` | Go build flags (link mode, tags) |

**xcaddy Build Command:**
```bash
xcaddy build --output frankenphp \
  --with github.com/dunglas/frankenphp=/path/to/source \
  --with github.com/dunglas/frankenphp/caddy=/path/to/source/caddy \
  --with github.com/dunglas/mercure/caddy \
  --with github.com/dunglas/vulcain/caddy \
  --with github.com/dunglas/caddy-cbrotli
```

---

### Step 12: Process Embedded App (Optional)

[VERIFIED: src/SPC/builder/unix/UnixBuilderBase.php:357-381]
```php
protected function processFrankenphpApp(): void
{
    $frankenphpSourceDir = getenv('FRANKENPHP_SOURCE_PATH') ?: SOURCE_PATH . '/frankenphp';
    ...
    $frankenphpAppPath = $this->getOption('with-frankenphp-app');

    if ($frankenphpAppPath) {
        if (!is_dir($frankenphpAppPath)) {
            throw new WrongUsageException("The path provided to --with-frankenphp-app is not a valid directory: {$frankenphpAppPath}");
        }
        $appTarPath = $frankenphpSourceDir . '/app.tar';
        logger()->info("Creating app.tar from {$frankenphpAppPath}");

        shell()->exec('tar -cf ' . escapeshellarg($appTarPath) . ' -C ' . escapeshellarg($frankenphpAppPath) . ' .');

        $checksum = hash_file('md5', $appTarPath);
        file_put_contents($frankenphpSourceDir . '/app_checksum.txt', $checksum);
    } else {
        ...
        file_put_contents($frankenphpSourceDir . '/app.tar', '');
        file_put_contents($frankenphpSourceDir . '/app_checksum.txt', '');
    }
}
```

**If `--with-frankenphp-app` is set:**
1. Create `app.tar` from the specified directory
2. Generate MD5 checksum
3. Both files are embedded via Go's `//go:embed` directive

---

### Step 13: Deploy Final Binary

[VERIFIED: src/SPC/builder/unix/UnixBuilderBase.php:301-313]
```php
protected function deploySAPIBinary(int $type): string
{
    $src = match ($type) {
        BUILD_TARGET_CLI => SOURCE_PATH . '/php-src/sapi/cli/php',
        BUILD_TARGET_MICRO => SOURCE_PATH . '/php-src/sapi/micro/micro.sfx',
        BUILD_TARGET_FPM => SOURCE_PATH . '/php-src/sapi/fpm/php-fpm',
        BUILD_TARGET_CGI => SOURCE_PATH . '/php-src/sapi/cgi/php-cgi',
        BUILD_TARGET_FRANKENPHP => BUILD_BIN_PATH . '/frankenphp',
        default => throw new SPCInternalException("Deployment does not accept type {$type}"),
    };
    $dst = BUILD_BIN_PATH . '/' . basename($src);
    return $this->deployBinary($src, $dst);
}
```

[VERIFIED: src/SPC/builder/unix/UnixBuilderBase.php:127-167]
```php
public function deployBinary(string $src, string $dst, bool $executable = true): string
{
    ...
    // copy binary
    shell()->exec('cp ' . escapeshellarg($src) . ' ' . escapeshellarg($dst));

    // extract debug info
    $this->extractDebugInfo($dst);

    // strip
    if (!$this->getOption('no-strip')) {
        $this->stripBinary($dst);
    }

    // UPX for linux
    $upx_option = $this->getOption('with-upx-pack');
    if ($upx_option && PHP_OS_FAMILY === 'Linux' && $executable) {
        ...
        shell()->exec(getenv('UPX_EXEC') . " --best {$dst}");
    }

    return $dst;
}
```

**Deployment Steps:**
1. Copy binary from build location to `buildroot/bin/`
2. Extract debug symbols to `buildroot/debug/`
3. Strip binary (remove symbols) unless `--no-strip`
4. Compress with UPX on Linux (optional)

---

### Step 14: Sanity Check

[VERIFIED: src/SPC/builder/unix/UnixBuilderBase.php:275-295]
```php
// sanity check for frankenphp
if (($build_target & BUILD_TARGET_FRANKENPHP) === BUILD_TARGET_FRANKENPHP) {
    logger()->info('running frankenphp sanity check');
    $frankenphp = BUILD_BIN_PATH . '/frankenphp';
    if (!file_exists($frankenphp)) {
        throw new ValidationException(
            "FrankenPHP binary not found: {$frankenphp}",
            validation_module: 'FrankenPHP sanity check'
        );
    }
    $prefix = PHP_OS_FAMILY === 'Darwin' ? 'DYLD_' : 'LD_';
    [$ret, $output] = shell()
        ->setEnv(["{$prefix}LIBRARY_PATH" => BUILD_LIB_PATH])
        ->execWithResult("{$frankenphp} version");
    if ($ret !== 0 || !str_contains(implode('', $output), 'FrankenPHP')) {
        throw new ValidationException(...);
    }
}
```

**Verification:** Runs `frankenphp version` to confirm binary works.

---

## Key Configuration Files

| File | Purpose |
|------|---------|
| `config/ext.json` | Extension metadata (dependencies, build args) |
| `config/lib.json` | Library metadata (source URLs, dependencies) |
| `config/source.json` | Download URLs for sources |
| `config/pre-built.json` | Pre-built library locations |

---

## Build Artifacts

```
buildroot/
├── bin/
│   ├── frankenphp          # Final binary
│   ├── php                 # CLI (if built)
│   └── php-config          # PHP config script
├── lib/
│   ├── libphp.a            # Static PHP library
│   ├── libssl.a            # OpenSSL
│   ├── libbrotli*.a        # Brotli compression
│   └── pkgconfig/          # pkg-config files
├── include/
│   └── php/                # PHP headers
├── debug/
│   └── frankenphp.dwarf    # Debug symbols (macOS)
└── license/
    └── */                  # License files per library
```

---

## macOS-Specific Considerations

### Watcher Library Issue
[VERIFIED: 2026-01-04 build test]
```php
$nowatcher = $this->getLib('watcher') === null ? ',nowatcher' : '';
```
- If `watcher` library not built, adds `nowatcher` tag
- Watcher requires CoreServices framework (FSEventStream)
- Static build cannot link CoreServices dynamically
- **Fix:** Exclude `watcher` from `PHP_EXTENSION_LIBS`

### Framework Linking
[VERIFIED: src/SPC/builder/macos/MacOSBuilder.php:48-74]
```php
public function getFrameworks(bool $asString = false): array|string
{
    ...
    /** @var MacOSLibraryBase $lib */
    foreach ($libs as $lib) {
        array_push($frameworks, ...$lib->getFrameworks());
    }
    ...
}
```
- macOS libraries may require system frameworks
- Extensions can declare required frameworks in `ext.json`

---

## Known Issues

1. **[VERIFIED]** Memory limit set to 4GB for complex builds
   ```php
   ini_set('memory_limit', '4G');
   ```

2. **[VERIFIED]** FrankenPHP requires ZTS (thread-safe) PHP
   ```php
   if (!$this->getOption('enable-zts')) {
       throw new WrongUsageException('FrankenPHP SAPI requires ZTS enabled PHP');
   }
   ```

3. **[INFERRED]** Extension dependency cycles may cause build failures
   - DependencyUtil uses topological sort
   - Circular dependencies would throw exception

4. **[VERIFIED]** Library build order matters
   - Libraries built in dependency order
   - e.g., `zlib` before `curl` (curl depends on zlib)

---

## Debug Tips

### View Full Build Log
```bash
# SPC logs to stdout with -x flag
./build-static.sh 2>&1 | tee build.log
```

### Check Library Installation
```bash
# Verify library was built
ls -la dist/static-php-cli/buildroot/lib/lib*.a

# Check headers
ls -la dist/static-php-cli/buildroot/include/
```

### Rebuild Single Library
```bash
# Delete library marker and rebuild
rm dist/static-php-cli/buildroot/lib/libbrotli*.a
rm -rf dist/static-php-cli/source/brotli
# Run build again - will rebuild brotli
```

---

*Documentation generated following agent-system-mapper methodology v1.0*
