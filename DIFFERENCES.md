# Differences from upstream fengari

This is a browser/Obsidian-only fork of [fengari](https://github.com/fengari-lua/fengari) (v0.1.5). All Node.js dependencies have been removed to produce a bundle that works in browser and Electron environments without triggering security scanner warnings.

## Files removed

### `src/liolib.js` — Lua `io` library

Entirely Node.js-dependent. Contains an unconditional `require('fs')` at module load time that cannot be conditionally guarded. The `io` library (`io.open`, `io.read`, `io.write`, `io.lines`, etc.) is not available in this fork.

### `src/loadlib.js` — Lua `package` library / `require()` system

Uses `require('path')`, `require('fs')`, `process.cwd()`, and `(0, eval)('this')` for module resolution. The Lua `require()` function and `package.*` namespace are not available in this fork.

## Files modified

### `src/luaconf.js`

- Removed unconditional `process.env.FENGARICONF` access on line 3 (replaced with `const conf = {}`). This was a crash-on-mobile bug — `process` is undefined in non-Electron browser environments.
- Removed `require('os').platform()` Windows/Linux path detection branch. Collapsed the `if/else if/else` conditional to always use the browser path defaults (`LUA_DIRSEP = "/"`, relative `LUA_LDIR`/`LUA_JSDIR` paths).
- `LUA_EXEC_DIR` export remains but is unused (was only consumed by the removed `loadlib.js`).
- The `FENGARICONF` environment variable for runtime configuration is not supported.

### `src/lbaselib.js`

- Removed `process.stdout.write(Buffer.from(s))` branch for `print()` output. Always uses the browser implementation (`TextDecoder` + `console.log`, or `to_jsstring` + `console.log` fallback).

### `src/lauxlib.js`

- `luaL_loadfilex` stubbed to always return `LUA_ERRFILE` with the message "file loading not supported in this environment". Both the Node.js (`fs`-based) and browser (`XMLHttpRequest`-based) implementations have been removed.
- `luaL_loadfile` and `luaL_dofile` still exist as thin wrappers over the stub, so code referencing them will compile but always fail at runtime.
- `lua_writestringerror` always uses `console.error()` (removed `process.stderr.write()` branch).
- Removed all `require('fs')`, `Buffer`, and `process.stdin.fd` references.

### `src/loslib.js`

Retained browser-safe functions:

| Function | Implementation |
|----------|---------------|
| `os.date` | JavaScript `Date` with custom `strftime` |
| `os.time` | JavaScript `Date` / `Math.floor(date / 1000)` |
| `os.difftime` | Subtraction of two time values |
| `os.clock` | `performance.now() / 1000` |
| `os.setlocale` | Always reports `"C"` locale |

Removed Node.js-only functions:

| Function | Reason |
|----------|--------|
| `os.exit` | Used `process.exit()` |
| `os.getenv` | Used `process.env` |
| `os.remove` | Used `require('fs').unlinkSync` |
| `os.rename` | Used `require('fs').renameSync` |
| `os.tmpname` | Used `require('tmp').tmpNameSync` |
| `os.execute` | Used `require('child_process').execSync` |

### `src/ldblib.js`

- Removed `require('readline-sync')` import and `debug.debug()` interactive REPL function. The `debug.debug()` function is not available in this fork.
- All other debug library functions are retained (`debug.traceback`, `debug.getinfo`, `debug.sethook`, `debug.gethook`, `debug.getlocal`, `debug.setlocal`, `debug.getupvalue`, `debug.setupvalue`, `debug.upvalueid`, `debug.upvaluejoin`, `debug.getuservalue`, `debug.setuservalue`, `debug.getmetatable`, `debug.setmetatable`, `debug.getregistry`). These are all pure JavaScript.

### `src/lualib.js`

- Removed `io` library exports (`LUA_IOLIBNAME`, `luaopen_io`).
- Removed `package` library exports (`LUA_LOADLIBNAME`, `luaopen_package`).

### `src/linit.js`

- Removed `io` and `package` from `loadedlibs` registration.
- Removed `require('./loadlib.js')` and conditional `require('./liolib.js')`.
- `luaL_openlibs` now opens: `_G` (base), `coroutine`, `table`, `os` (safe subset), `string`, `math`, `utf8`, `debug` (minus `debug.debug()`), `fengari`.

## Dependencies removed

| Package | Was used by | Reason |
|---------|------------|--------|
| `readline-sync` | `ldblib.js` (`debug.debug()` interactive input) | Node.js-only CLI package |
| `tmp` | `loslib.js` (`os.tmpname`) | Node.js-only temp file creation |

## Dependencies kept

| Package | Used by | Reason |
|---------|--------|--------|
| `sprintf-js` | `lstrlib.js` (Lua `string.format`) | Pure JavaScript, required for C-style format specifiers |

## Behavioral differences

| Behavior | Upstream fengari | This fork |
|----------|-----------------|-----------|
| `print()` output | `process.stdout.write()` in Node, `console.log` in browser | Always `console.log` |
| `os.clock()` | `process.uptime()` in Node, `performance.now()/1000` in browser | Always `performance.now()/1000` |
| `luaL_loadfilex` | `fs`-based in Node, `XMLHttpRequest`-based in browser | Always returns error |
| `luaL_openlibs` | Opens all 10 libraries (io conditionally) | Opens 9 libraries (no io, no package) |
| `debug.debug()` | `readline-sync` in Node, `window.prompt` in browser | Not available |
| `FENGARICONF` env var | Reads `process.env.FENGARICONF` for runtime config | Not supported |
| Error output | `process.stderr.write()` in Node, `console.error` in browser | Always `console.error` |
| Path defaults | Platform-specific (Windows `\`, Linux `/usr/local/`, browser `./`) | Always browser defaults (`./lua/5.3/`) |

## Coroutine↔Promise bridge validation

The fork's `lua_yieldk`/`lua_resume` implementation (ported from PUC-Rio Lua 5.3) has been validated for use as a coroutine↔Promise bridge — enabling JS-hosted async operations (e.g., file reads via Obsidian's vault API) to be called transparently from Lua code.

Validation tests in `test/coroutine-promise-bridge.test.js` confirm:

| Test | Description |
|------|-------------|
| T1 | C-function yields via `lua_yieldk` with continuation, JS resumes, continuation receives correct status and context |
| T2 | `lua_isyieldable` returns `true` inside a resumed thread, `false` on main state |
| T3 | `pcall` around a yielding function works — yield propagates through `pcall` and resume value is returned |
| T4 | Instruction count hook (`lua_sethook` with `LUA_MASKCOUNT`) fires correctly on thread after yield/resume |
| T5 | Error propagation via continuation — `nil + errmsg` protocol, `luaL_error` from continuation |
| T6 | Error propagation through `pcall` across yield — `pcall` catches continuation error, returns `(false, msg)` |
| T7 | Sequential yields — two async calls in one function, both resume correctly |
| T8 | Lua-level `coroutine.create` captures yield internally — JS host cannot detect it (confirms C-level `lua_newthread` required for bridge) |

Key finding: `lua_newthread` pushes the thread onto the parent's stack. Using `lua_xmove` immediately after `lua_newthread` moves the thread value itself, not the intended function. The correct pattern: compile the chunk directly on the thread via `luaL_loadstring(thread, code)` — the thread shares globals with the main state. Alternatively, use `lua_rawgeti(thread, LUA_REGISTRYINDEX, ref)` to load from the shared registry.

## Inherited limitations from upstream

These are upstream fengari limitations that this fork does not attempt to address:

- `__gc` metamethods don't work (no custom GC; relies on JavaScript garbage collector)
- Weak tables (`__mode`) are not supported
- `lua_gc` / `collectgarbage` are not implemented
- Integer type is 32-bit (JavaScript number limitation)

## Upstream sync policy

Upstream fengari is essentially frozen (last release: v0.1.5). This fork will check upstream quarterly and cherry-pick only security or correctness fixes.
