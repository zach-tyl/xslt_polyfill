# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a WebAssembly-based polyfill for the browser's native `XSLTProcessor` API, implementing XSLT 1.0 transformations. It compiles libxml2 and libxslt to WebAssembly using Emscripten, wrapped with a JavaScript API that matches the browser's native interface.

## Build Commands

### Full Build (requires Emscripten toolchain)
```bash
./build.sh
```
This runs three stages:
1. `./build_libraries.sh` - Compiles libxml2 and libxslt to WASM
2. `./build_wasm.sh` - Builds the WASM XSLT processor (`dist/xslt-wasm.js`)
3. `./combine_and_minify.sh` - Combines WASM with polyfill JS and minifies

### Quick Rebuild (JS changes only)
```bash
./combine_and_minify.sh
```
Use this after modifying `src/xslt-polyfill-src.js` when native code hasn't changed.

### Debug Build
```bash
./build.sh --debug
```
Creates `dist/xslt-wasm-debug.js` with source maps and assertions enabled.

### Generate Test Files
```bash
node test/testcase_generator.js
```
Creates test cases in `test/generated/` for the test suite.

## Build Dependencies

- Emscripten (`emcc`, `emconfigure`, `emmake`)
- `pkg-config`
- `terser` (install via `npm install terser -g`)
- Git submodules must be initialized: `git clone --recursive` or `git submodule update --init`

## Architecture

### Core Components

**`src/transform.c`** - C code compiled to WASM that:
- Wraps libxslt's transformation API
- Uses Emscripten's Asyncify to handle async fetch() calls from C for `<xsl:import>`/`<xsl:include>`
- Implements a custom document loader (`docLoader`) that fetches external stylesheets via JavaScript
- Handles MIME type detection and HTML encoding meta tag adjustment to match Chrome's behavior

**`src/xslt-polyfill-src.js`** - JavaScript polyfill that:
- Provides `XSLTProcessor` class matching the browser API (`importStylesheet`, `transformToDocument`, `transformToFragment`, `setParameter`)
- Handles WASM memory management for passing XML/XSLT strings to C
- Auto-detects XML documents with `<?xml-stylesheet?>` processing instructions and transforms them
- Compiles `<xsl:import>` statements by fetching and inlining them (async pre-processing)
- Patches `document.createElement()` for XHTML compatibility

**`src/libxml2/` and `src/libxslt/`** - Git submodules containing the GNOME XML/XSLT libraries

### Build Outputs

- `dist/xslt-wasm.js` - Compiled WASM module (embedded as base64 via `SINGLE_FILE`)
- `xslt-polyfill.min.js` - Final minified polyfill for production use

### Key Design Decisions

- WASM binary is embedded in JS (Emscripten `SINGLE_FILE`) for simpler deployment
- Asyncify enables synchronous-looking C code to call async JavaScript fetch()
- Security: File/network write operations are forbidden; reads are controlled by browser CORS
- The polyfill only activates when native `XSLTProcessor` is unavailable or `window.xsltUsePolyfillAlways = true`

## Testing

Open `test/test_suite.html` in a browser after running `node test/testcase_generator.js`. Tests run in three modes: native, source (unminified), and minified polyfill.

## npm Package

Published as `xslt-polyfill` on npm. Main entry point is `xslt-polyfill.min.js`.
