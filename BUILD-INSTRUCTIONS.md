# FrankenPHP with SQL Server - Build Instructions

Build a standalone PHP binary with SQL Server (pdo_sqlsrv) support for macOS to run Laravel apps natively.

## Fresh Machine Setup

### 1. Install Homebrew (if not installed)

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

After installation, follow the instructions to add Homebrew to your PATH.

### 2. Install Build Dependencies

```bash
brew install go composer unixodbc
```

### 3. Install Microsoft ODBC Driver

```bash
brew tap microsoft/mssql-release https://github.com/microsoft/homebrew-mssql-release
HOMEBREW_ACCEPT_EULA=Y brew install msodbcsql18
```

## Build

```bash
# Clone FrankenPHP
git clone https://github.com/dunglas/frankenphp.git
cd frankenphp

# Build PHP 8.2 with SQL Server + Laravel extensions
PHP_VERSION=8.2 \
PHP_EXTENSIONS="pdo,pdo_sqlsrv,ctype,dom,fileinfo,filter,mbstring,openssl,session,tokenizer,xml,json,bcmath,sockets" \
PHP_EXTENSION_LIBS="libavif,nghttp2,nghttp3,ngtcp2,brotli" \
./build-static.sh
```

Build takes ~10-15 minutes. The script auto-installs additional dependencies (cmake, bison, etc.) as needed.

## Output

Binary location depends on your Mac:
- Apple Silicon (M1/M2/M3): `dist/frankenphp-mac-arm64`
- Intel: `dist/frankenphp-mac-x86_64`

## Verify

```bash
# Create test file
echo '<?php print_r(get_loaded_extensions());' > /tmp/test.php

# Check loaded extensions
./dist/frankenphp-mac-arm64 php-cli /tmp/test.php | grep -i sql
# Should show: pdo_sqlsrv, sqlsrv

# Check PHP version
./dist/frankenphp-mac-arm64 php-cli -v
```

## Running a Laravel App

```bash
# Navigate to your Laravel project and run:
/path/to/frankenphp-mac-arm64 php-server -l :8000 -r public/

# Or with absolute paths:
/path/to/frankenphp-mac-arm64 php-server -l :8000 -r /path/to/laravel/public/
```

Visit `http://localhost:8000` in your browser.

## Notes

- **ODBC Driver Required**: The Microsoft ODBC driver (`msodbcsql18`) must be installed on any Mac running this binary - it cannot be statically linked
- **No Restart Needed**: PHP files are re-read on each request (code changes apply immediately)
- **Architecture Specific**: Binary only works on same architecture it was built on (arm64 vs x86_64)
- **No curl Extension**: This build uses PHP streams for HTTP requests (Guzzle/Laravel HTTP client work fine)

## Troubleshooting

### Build fails with FSEventStream errors
Make sure `PHP_EXTENSION_LIBS` does NOT include `watcher` - it has macOS linking issues.

### Laravel "Class config does not exist" error
You're missing required extensions. Use the full `PHP_EXTENSIONS` list above.

### Clean rebuild
```bash
rm -rf dist/static-php-cli/buildroot dist/static-php-cli/source
# Then run build command again
```
