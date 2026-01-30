# Installing OpenClaw on Alpine Linux (Termux/proot-distro)

This guide covers installing OpenClaw on Alpine Linux aarch64 running under Termux via proot-distro.

## Prerequisites

- Termux with proot-distro
- Alpine Linux installed (`proot-distro install alpine`)
- Node.js 22+ installed

## The Problems

### 1. Official installer rejects Alpine/Termux

The official installer at `https://openclaw.bot/install.sh` only recognizes `linux-gnu` systems. It fails on Alpine (`linux-musl`) and Termux (`linux-android`) with "Unsupported operating system".

**Fix:** This repo includes a patched `install.sh` that adds musl and android detection.

### 2. Native clipboard module missing musl binary

OpenClaw depends on `@mariozechner/clipboard`, a native Node.js module. This package does not publish a prebuilt binary for `linux-arm64-musl` (Alpine on ARM64). The binary generated during npm install is incorrectly linked against glibc, causing this error:

```
Error relocating clipboard.linux-arm64-musl.node: gnu_get_libc_version: symbol not found
```

**Fix:** Rebuild the native module from source (see below).

## Solution

### Using the patched installer

You can use the patched installer from this repo instead of the official one:

```bash
curl -fsSL https://raw.githubusercontent.com/jumasheff/openclaw-alpine-termux-installation/main/install.sh | bash
```

This will get past the OS detection, but you'll still need to rebuild the clipboard module (steps 2-4 below).

### Manual installation

Rebuild the native clipboard module from source using Alpine's Rust toolchain.

#### Step 1: Install OpenClaw via npm

```bash
npm install -g openclaw@latest
```

This will fail to run, but installs the package files we need.

#### Step 2: Install build dependencies

```bash
apk update
apk add --no-cache rust cargo build-base
```

#### Step 3: Rebuild the clipboard module

```bash
cd $(npm root -g)/openclaw/node_modules/@mariozechner/clipboard
npm install @napi-rs/cli
npm run build
```

This takes approximately 5 minutes on a typical device.

#### Step 4: Verify installation

```bash
openclaw --version
```

You should see the version number (e.g., `2026.1.29`).

## One-liner

After installing Node.js, run this to install and fix OpenClaw in one go:

```bash
apk update && apk add --no-cache rust cargo build-base && \
npm install -g openclaw@latest && \
cd $(npm root -g)/openclaw/node_modules/@mariozechner/clipboard && \
npm install @napi-rs/cli && \
npm run build && \
openclaw --version
```

## Troubleshooting

### nvm bash_completion syntax error

If you see:
```
/bin/sh: /home/user/.nvm/bash_completion: line 13: syntax error: unexpected "("
```

This happens because Alpine's `/bin/sh` is busybox ash, not bash. Fix your `~/.bashrc`:

```bash
# Change this line:
[ -s "$NVM_DIR/bash_completion" ] && . "$NVM_DIR/bash_completion"

# To this:
[ -n "$BASH_VERSION" ] && [ -s "$NVM_DIR/bash_completion" ] && . "$NVM_DIR/bash_completion"
```

### Verifying the fix worked

Check that the rebuilt binary links to musl, not glibc:

```bash
readelf -d $(npm root -g)/openclaw/node_modules/@mariozechner/clipboard/clipboard.linux-arm64-musl.node | grep NEEDED
```

Correct output (musl):
```
 (NEEDED)  Shared library: [libgcc_s.so.1]
 (NEEDED)  Shared library: [libc.musl-aarch64.so.1]
```

Wrong output (glibc - needs rebuild):
```
 (NEEDED)  Shared library: [libc.so.6]
 (NEEDED)  Shared library: [libm.so.6]
```

## Requirements Summary

| Package | Version | Purpose |
|---------|---------|---------|
| Node.js | 22+ | Runtime |
| rust | 1.87+ | Compile native module |
| cargo | 1.87+ | Rust package manager |
| build-base | any | gcc, make, etc. |

## Notes

- Build takes ~5 minutes and ~700MB disk space for Rust toolchain
- You can remove rust/cargo after building if space is tight: `apk del rust cargo`
- After OpenClaw updates, you may need to rebuild the clipboard module again
