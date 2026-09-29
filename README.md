# wasmdebug

Step through a `.wat` file in Chrome DevTools' built-in WebAssembly debugger.

## Usage

```
~/wasmdebug/wasmdebug path/to/file.wat
```

It prints a `http://127.0.0.1:<port>/` URL. Open that in Chrome, then open
DevTools (F12) → **Sources** tab → find `module.wasm` under the page's
origin in the file tree. Set a breakpoint by clicking a line number, then
click "Call" on one of the exported functions listed on the page to run
the module and hit it.

Options: `-p PORT` to pick a fixed port, `--open` to auto-launch your
default browser.

The live instance is also available from the DevTools Console as
`wasmdebugInstance` (e.g. `wasmdebugInstance.exports.run(3)`).

## How it works

This script is an HTTP server (Python stdlib) that shells out to your
local `wat2wasm` to do the actual compiling. It serves a page that:

1. Fetches your `.wat` file's raw text from the server, to display it.
2. Fetches `/module.wasm` — on each request, the server runs
   `wat2wasm <file> --output=- --enable-all --debug-names` and streams
   the result back — once to inspect the module's type/export sections
   client-side (see below), and again via
   `WebAssembly.instantiateStreaming(fetch('/module.wasm'))` to actually
   instantiate it.
3. Instantiates the module and renders a "Call" button (with correctly
   typed inputs) for every exported function.

Recompiling on every request means editing the `.wat` file and reloading
the page always gets you the current version — no cache to invalidate,
no server restart needed.

## Requirements

`wat2wasm` on `PATH` (this was set up via `sudo apt install wabt`, which
installs WABT 1.0.34 from Ubuntu's universe repo). The script checks for
it at startup and tells you how to install it if it's missing.

## Design decisions worth knowing about

- **Local compile via `wat2wasm`, not a CDN.** This started out compiling
  client-side in the browser via a CDN-hosted build of `wabt` (there was
  no Node/WABT on this machine initially, and installing either felt like
  more than a "simple tool" should require). Once `wabt` was installed
  locally, the tool switched to shelling out to the real `wat2wasm`
  binary server-side instead — one less moving part (no CDN dependency,
  no in-browser compiler), and you get wat2wasm's native error messages
  (with file:line:col) instead of wabt.js's in-browser ones. The
  trade-off is this now hard-requires `wat2wasm` on `PATH`.

- **`--output=-` needs the `=`.** This build's argument parser rejects
  `-o -` (space-separated) with "option '-o' requires argument" — a bare
  `-` after a space apparently doesn't count as a value. `--output=-`
  (or `-o-`, no space) works. Worth remembering if you ever script
  `wat2wasm` yourself.

- **You'll see a disassembly, not your literal source file.** Chrome's
  Sources panel shows its own reconstruction of the module's text — very
  close to WAT syntax, and with your original `$names` preserved (the
  compiler is asked to keep a debug-names section), but comments and
  exact formatting don't survive the round trip to binary and back.
  Getting Chrome to show the *actual* `.wat` file with byte-exact source
  mapping would need a DWARF/source-map pipeline that doesn't really
  exist for hand-written WAT (that machinery is normally built for
  C/C++/Rust compilers, not wat2wasm). The disassembly view is the
  practical option and is fully interactive: breakpoints, stepping,
  locals in the Scope pane.

- **No hardcoded entry point.** Wasm has no implicit "main" — some
  exported function has to be called from JS. Rather than assuming a
  name like `main`, the page parses the compiled binary's type/export
  sections and generates a form for *every* exported function, with one
  input per parameter labeled by its real type (i32/i64/f32/f64). This
  is also why `sandbox1.wat`'s `run` export shows up as `run(i32)` with
  one input box, ready to call with an argument, rather than the tool
  guessing it takes none.

- **Imports are auto-stubbed.** If your module imports functions (e.g. a
  `env.log`), the tool doesn't know what a real host should do, so it
  synthesizes a stub that logs the call to the console and returns
  zeroed results (memories/tables/globals get a default instance
  matching the shape your module declared). This is enough to get you to
  a breakpoint; it won't reproduce real host behavior.

- **Why `/module.wasm` is fetched twice.** The import-stub logic needs to
  know each import's type *before* instantiation (it's passed as the
  `importObject` argument), so the module bytes are fetched once as a
  plain `ArrayBuffer` for inspection. Instantiation itself then uses a
  second, streaming fetch of the same URL — `instantiateStreaming` is
  what makes Chrome show a clean `module.wasm` entry in Sources instead
  of an anonymous blob; a buffer-based `WebAssembly.instantiate(bytes)`
  doesn't get that treatment. Since `wat2wasm` is fast and this is a
  local single-user tool, compiling twice per page load isn't worth
  optimizing away.

## Testing performed

No Node/Playwright/Puppeteer was available in the environment this was
built in, so the button-click interaction wasn't scripted end-to-end.
What *was* verified with real headless Chrome, against the local-compile
version:

- `sandbox1.wat` (table, `call_indirect`, mutable global, memory,
  no imports) compiles, links, and renders `run(i32)` correctly.
- A synthetic module with a function import, a memory export, a global
  export, a 2-arg function, and a 0-arg function all rendered with the
  right input types and the right "other exports" listing, and the
  import stub let instantiation succeed.
- A deliberately broken `.wat` file surfaces wat2wasm's real compile
  error (e.g. `bad.wat:1:43: error: unexpected token ...`) in the status
  area instead of failing silently.

Worth a quick manual check on your end: that clicking "Call" actually
pauses at a breakpoint you set in the Sources panel.
