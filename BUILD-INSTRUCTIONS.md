# FrankenPHP with SQL Server - Build Instructions

Build a standalone PHP binary with SQL Server (pdo_sqlsrv) support for macOS.

## Prerequisites

```bash
# Install build tools
brew install go composer unixodbc

# Install Microsoft ODBC Driver
brew tap microsoft/mssql-release https://github.com/Microsoft/homebrew-mssql-release
HOMEBREW_ACCEPT_EULA=Y brew install msodbcsql18
```

## Build

```bash
# Clone FrankenPHP
git clone https://github.com/dunglas/frankenphp.git
cd frankenphp

# Build PHP 8.2 with SQL Server support
PHP_VERSION=8.2 \
PHP_EXTENSIONS="pdo,pdo_sqlsrv" \
PHP_EXTENSION_LIBS="libavif,nghttp2,nghttp3,ngtcp2,brotli" \
./build-static.sh
```

Build takes ~5-10 minutes. The script auto-installs additional dependencies (cmake, bison, etc.) as needed.

## Output

Binary location depends on your Mac:
- Apple Silicon (M1/M2/M3): `dist/frankenphp-mac-arm64`
- Intel: `dist/frankenphp-mac-x86_64`

## Verify

```bash
# Check PHP version
./dist/frankenphp-mac-* php -v

# Verify SQL Server extension loaded
./dist/frankenphp-mac-* php -m | grep -i sql
# Should show: pdo_sqlsrv, sqlsrv
```

## Usage

```bash
# Run PHP file
./dist/frankenphp-mac-* php your-script.php

# Start web server (serves current directory)
./dist/frankenphp-mac-* php-server

# Start with Caddyfile config
./dist/frankenphp-mac-* run
```

## Notes

- The Microsoft ODBC driver (`msodbcsql18`) must be installed on any Mac running this binary
- PHP files are re-read on each request (no restart needed for code changes)
- Binary is architecture-specific (arm64 vs x86_64)
