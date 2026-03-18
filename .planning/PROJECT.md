# EdgeJS v9 — Browser-Native Node.js Runtime

## What This Is

A Wasm-native Node.js runtime that compiles the Node.js C++ layer (libuv, OpenSSL, nghttp2, ~46 built-in modules) to WebAssembly via Emscripten, while delegating JavaScript execution to the browser's native engine through an N-API bridge. The primary goal is running Claude Code end-to-end in the browser — full conversations with file read/write in a persistent virtual filesystem.

## Core Value

Claude Code runs a full agentic conversation in the browser — reads files, edits files, makes API calls, responds with results — no local install required.

## Requirements

### Validated

- ✓ Build pipeline (Makefile, emscripten-toolchain.cmake, build-emscripten.sh) — existing
- ✓ 44 WASI shim headers (3,875 lines) patching POSIX/V8 embedding API gaps — existing
- ✓ N-API bridge scaffolded (handle table, type marshaling, memory read/write) — existing
- ✓ EdgeJS source integrated as submodule (~46 built-in Node modules) — existing
- ✓ JSPI adapter for async I/O (WebAssembly.Suspending with Asyncify fallback) — existing
- ✓ Browser builtins override layer (crypto, path, url, buffer, process) — existing

### Active

- [ ] Clean Emscripten compilation of all Node.js C++ modules to .wasm
- [ ] V8 embedding API shims complete (v8::Internals for wasm32, sandbox types, platform abstractions)
- [ ] N-API bridge completeness for Claude Code's runtime needs (streams, crypto, http/2 edge cases)
- [ ] HTTPS API calls routed through browser fetch proxy via N-API layer
- [ ] Persistent filesystem via Origin Private File System (OPFS)
- [ ] Terminal UI via xterm.js for interactive Claude Code sessions
- [ ] Child process emulation via Web Workers running additional Wasm instances
- [ ] Embeddable SDK API for developers to integrate into their own web apps
- [ ] Claude Code boots, converses, reads/writes files in browser

### Out of Scope

- RISC-V emulation path (v1) — separate project, already working
- Codex CLI / Gemini CLI full support — v2 after Claude Code works
- Full V8 engine compilation to Wasm — the N-API split deliberately avoids this
- Mobile app — web-first
- Offline mode — API calls require network
- Server-side rendering — this is a client-side runtime

## Context

**Architecture — the N-API split:**

The key insight is that V8 (the JS engine) does NOT compile to Wasm. Only the Node.js C++ runtime layer compiles to Wasm. User JavaScript — including Claude Code's application code and the Anthropic SDK — executes on the browser's native JS engine at full JIT speed. The N-API bridge connects the two worlds via a handle table that maps integer handles to JavaScript values.

```
┌─────────────────────────────────────────┐
│  Compiled to Wasm (Emscripten)          │
│  • ~46 Node.js C++ built-in modules     │
│  • libuv event loop                     │
│  • OpenSSL, nghttp2, c-ares, zlib, ICU  │
│  • musl libc                            │
│  NOT V8. NOT the JS engine.             │
└──────────────┬──────────────────────────┘
               │ N-API calls
               ▼
┌─────────────────────────────────────────┐
│  Browser's native JS engine             │
│  • Full JIT (Ignition → Maglev → TF)   │
│  • Native speed, no emulation tax       │
│  • Executes Claude Code JS application  │
└─────────────────────────────────────────┘
```

**Current compilation state:**

The build pipeline exists and is iterating through V8 embedding API compilation errors. The 44 shim headers represent errors discovered so far. The process is: `emcmake cmake && make` → hit errors → write shims → repeat. Past the "obvious" gaps and into the long tail of V8 embedding API surface area.

**Key compilation challenges:**

- `v8::internal::Internals` class — hardcoded pointer offsets differ on wasm32
- Sandbox/pointer table types — don't exist outside V8's build system
- Platform abstractions (`v8::Platform`, `v8::ArrayBuffer::Allocator`) — need Emscripten-compatible implementations
- Namespace and template instantiation — clang-for-wasm vs clang-for-linux differences
- Bytecode handler mappings — 542-line shim for builtins list

**Existing v1 (RISC-V) context:**

A working RISC-V emulation path exists separately. It runs CLIs today at ~40% native speed with a 29MB checkpoint. The v9 Wasm-native path targets near-native performance by eliminating the emulation layer entirely.

## Constraints

- **Browser compatibility**: JSPI requires Chrome 123+ (Asyncify fallback for others)
- **Binary size**: Wasm module likely 40-50MB raw, ~10-15MB gzipped — must optimize
- **Memory**: 4GB max Wasm linear memory, 128MB initial — must fit Node.js runtime + user workload
- **Threading**: SharedArrayBuffer requires Cross-Origin-Isolation headers (COOP/COEP)
- **Network**: HTTPS via browser fetch proxy — no raw TCP sockets in browsers
- **Upstream dependency**: EdgeJS (aspect-build/aspect-edgejs) as submodule — patches may be needed

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| N-API bridge instead of compiling V8 to Wasm | V8-to-Wasm would be 50-80MB at fraction of native speed; N-API split gives native JS performance | — Pending |
| EdgeJS as base runtime | Provides ~46 Node.js C++ modules already structured for embedding | — Pending |
| Emscripten over wasi-sdk | Better browser integration, JSPI support, mature filesystem/threading | — Pending |
| OPFS for persistent filesystem | Browser-native, no server needed, survives page reloads | — Pending |
| Browser fetch proxy for networking | Intercept at N-API layer, route through browser fetch — simpler than WebSocket tunnel | — Pending |
| Web Workers for child_process | Spawn additional Wasm instances in workers — closest to real subprocess semantics | — Pending |
| Claude Code as v1 target CLI | Most valuable, proves the hardest case (agentic file editing), others follow | — Pending |

---
*Last updated: 2026-03-18 after initialization*
