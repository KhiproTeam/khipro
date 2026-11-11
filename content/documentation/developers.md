---
title: "khipro-library — Developer Guide"
description: "Integrate the Khipro compositional Bangla input method into any application or platform using khipro-library."
url: "/documentation/developers/"
---

**khipro-library** is a cross-platform C++ library that implements the Khipro
compositional Bangla input method as an embeddable engine. It powers Khipro
integrations on every platform — Linux IMEs, Windows apps, Android keyboards,
and web editors — through one small, stable API.

The four Khipro layout specifications (m17n `.mim` format) are compiled into
generated C++ tables at build time, so the shipped library contains **data
only — no runtime parser**. What you link against is a self-contained engine
with no globals, no I/O, and no external dependencies.

## Bundled layout variants

| Variant | Enum | Intended for | Sourced from |
|---|---|---|---|
| Classic | `KHIPRO_VARIANT_CLASSIC` / `"classic"` | desktop keyboards | [rank-coder/khipro-m17n](https://github.com/rank-coder/khipro-m17n) |
| Classic touchscreen | `KHIPRO_VARIANT_CLASSIC_TOUCH` / `"classic_touch"` | phones, tablets | [KhiproTeam/khipro-mim-touchscreen](https://github.com/KhiproTeam/khipro-mim-touchscreen) |
| Dark | `KHIPRO_VARIANT_DARK` / `"dark"` | desktop keyboards | [KhiproTeam/khipro-dark-m17n](https://github.com/KhiproTeam/khipro-dark-m17n) |
| Dark touchscreen | `KHIPRO_VARIANT_DARK_TOUCH` / `"dark_touch"` | phones, tablets | [KhiproTeam/khipro-dark-mim-touchscreen](https://github.com/KhiproTeam/khipro-dark-mim-touchscreen) |

Each layout tracks its upstream releases **independently** — the library can
ship a newer Classic with an older Dark, exactly as upstream publishes them.

## Get the library

Every [release](https://github.com/khiproteam/library/releases) ships five
prebuilt bundles:

| Bundle | Contents |
|---|---|
| `khipro-<ver>-linux-x86_64.tar.gz` | `.so` + `.a` + `khipro.pc` + headers |
| `khipro-<ver>-macos-universal.tar.gz` | universal `.dylib` (arm64 + x86_64) + `.a` + headers |
| `khipro-<ver>-windows-x86_64.zip` | `khipro.dll` + import libs + headers |
| `khipro-<ver>-android.aar` | per-ABI `.so` (4 ABIs) + Kotlin bindings |
| `khipro-<ver>-wasm.zip` | `khipro.js` + `khipro.wasm` + ESM wrapper + TypeScript types |

Prefer building yourself? See [Building from source](#building-from-source)
below.

## The engine model

```
create → feed_key × N → commit (or reset) → destroy
```

One engine instance represents **one input field's session**:

- `create(variant)` — start a session for a field
- `feed_key(key)` — feed a single keystroke (one UTF-8 character)
- `backspace()` — user pressed backspace inside the composition
- `commit()` — flush remaining preedit into committed text (field blur)
- `reset()` — abandon the composition without committing (input cancel)
- `destroy()` — release the engine

### Reading the result

Every mutating call returns a result with four fields:

| Field | What your UI does |
|---|---|
| `committed_delta` | Append to the document buffer immediately; the delta is one-shot — it is consumed by this call |
| `preedit` | The **full current** inline text — replace the previous rendering with this value each call |
| `committed` | Full session total — ignore unless you need a snapshot |
| `consumed` | If false, forward the raw keystroke to the host (the IME has nothing to do for this key) |

Notes:

- **Thread safety** — a single engine is not thread-safe. Create one per
  input field; different engines in different threads are fine (layout
  tables are shared read-only).
- **Stateless one-shot** — for batch transcription without preedit or
  backspace, skip the engine: `khipro_convert(variant, "ka", out, cap)` →
  `"কা"`.

### Behavior details

These come straight from the engine implementation:

- **Space, Enter, Tab are commit triggers.** Feeding one commits any
  pending preedit *and* appends the trigger character itself to
  `committed_delta` — so when `consumed` is true, do **not** insert the key
  again or you will double it.
- **`feed_key("\b")` delegates to backspace** — you may route the
  backspace keycode directly.
- **Backspace order**: it first erases from the composing text (preedit
  shrinks); only when the composition is empty does it delete from the
  already-committed text — in that case `committed_delta` carries the
  *remaining* committed text (resync your buffer with it). `consumed=false`
  only when both are empty, meaning the host should handle the key.
- **String lifetimes (C ABI)** — the `char*` fields of `khipro_result`
  point into engine-owned buffers and stay valid **until the next call on
  the same engine**. Copy them if you need them longer.
- **`khipro_version()`** returns the *Classic* layout's version string —
  the layout releases are independent per variant, so treat it as the
  bundle's headline layout version.

## API reference

Every platform binding exposes the same nine functions with identical
semantics — field names match modulo language case:

| Function | C ABI | Android Kotlin | Wasm TypeScript |
|---|---|---|---|
| create | `khipro_create(variant)` | `KhiproEngine.create(variant)` | `KhiproEngine.create(mod, variant)` |
| destroy | `khipro_destroy(engine)` | `engine.close()` | `engine.destroy()` |
| feed key | `khipro_feed_key(eng, key)` | `engine.feedKey(key)` | `engine.feedKey(key)` |
| backspace | `khipro_backspace(eng)` | `engine.backspace()` | `engine.backspace()` |
| commit | `khipro_commit(eng)` | `engine.commit()` | `engine.commit()` |
| reset | `khipro_reset(eng)` | `engine.reset()` | `engine.reset()` |
| convert (one-shot) | `khipro_convert(variant, ascii, out, cap)` | `Khipro.convert(ascii, variant)` or `engine.convert(ascii)` | `khiproConvert(mod, ascii, variant)` |
| layout version | `khipro_version()` | `Khipro.layoutVersion` | `Khipro.layoutVersion` |
| library version | `khipro_library_version()` | `Khipro.libraryVersion` | `Khipro.libraryVersion` |

Two host-input wrappers build on top of the core and save you the event
wiring:

- **Android** — `KhiproIme` wraps an `InputConnection` inside an
  `InputMethodService` (a ready-made
  [sample](https://github.com/khiproteam/library/blob/main/bindings/android/samples/KotlinImeExample.kt)
  is in the repo). `KhiproEngine` also exposes live `committed` / `preedit`
  properties mirroring the last result.
- **Web** — `attachKhiproInput(el, engine)` wires a single `<input>` or
  `<textarea>` to DOM events. The top-level `Khipro` version getters are
  populated once the first engine has loaded the module.

### `khipro_convert` buffer contract (C ABI)

The return value tells you what happened:

- `> 0` and it fit — bytes written, excluding the NUL terminator
- the call **didn't fit** — the *required* size (including NUL) is
  returned; nothing is written. Call again with a buffer of at least that
  size.
- `0` — `ascii` was null
- `-1` — internal error

## Quick start

### C (Linux, macOS, Windows)

```c
#include <khipro/khipro.h>
#include <stdio.h>

int main(void) {
    khipro_engine* e = khipro_create(KHIPRO_VARIANT_CLASSIC);

    khipro_result r = khipro_feed_key(e, "k");
    printf("preedit: %s\n", r.preedit);        /* ক */

    r = khipro_feed_key(e, "a");
    printf("delta  : %s\n", r.committed_delta); /* কা */

    khipro_commit(e);
    khipro_destroy(e);
}
```

Build with `gcc app.c $(pkg-config --cflags --libs khipro) -o app`.

### Kotlin (Android)

```kotlin
val engine = KhiproEngine.create(KhiproVariant.CLASSIC_TOUCH)

val r  = engine.feedKey("k")   // r.preedit == "ক"
val r2 = engine.feedKey("a")   // r2.committedDelta == "কা"

engine.commit()
engine.close()
```

Drop the `.aar` into `app/libs/`, add
`implementation(files("libs/khipro-<ver>.aar"))` — no NDK setup, no
`jniLibs` extraction. `minSdk = 21`.

### TypeScript (Web)

```ts
import { loadKhiproModule, KhiproEngine } from "./index.js";

const mod = await loadKhiproModule("./khipro.js");
const engine = await KhiproEngine.create(mod, "classic");

const r = engine.feedKey("k");   // r.preedit === "ক"
```

Ship the five archive files together, or install from npm:
`@khiproteam/khipro`.

## Platform guides

### Linux

Install the release archive system-wide:

```bash
sudo tar -xzf khipro-*-linux-x86_64.tar.gz --strip-components=2 -C /usr/local
sudo ldconfig
pkg-config --modversion khipro   # verify
```

Compile with `gcc app.c $(pkg-config --cflags --libs khipro) -o app`, or in
CMake:

```cmake
find_package(khipro CONFIG REQUIRED)
target_link_libraries(my_app PRIVATE khipro::khipro)
```

Static linking: link `/usr/local/lib/libkhipro.a` with `-lpthread`. Runtime
dependencies are only glibc and libstdc++.

For a desktop IME: in a `GtkIMContext` subclass, feed keys from
`filter_keypress`, show `preedit` with `gtk_im_context_set_preedit_string`,
and emit commits for `committed_delta`. In Qt, the same shape lives inside
`QPlatformInputContext::filterEvent`.

### Windows

Ship `khipro.dll` next to your `.exe` — no system install. MinGW consumers
link directly:

```powershell
gcc app.c -Iwindows/x86_64/include -Lwindows/x86_64/lib -lkhipro -o app.exe
```

MSVC cannot use the MinGW import lib; generate one from the DLL
(`gendef khipro.dll && dlltool -d khipro.def -l khipro.lib -k`), or use
`LoadLibrary` + `GetProcAddress`. Define `KHIPRO_USE_SHARED` before including
the header so declarations become `__declspec(dllimport)`.

For the **static** lib, keep the C++ runtime static (`-static`, or
`find_package(khipro)` / `pkg-config`, which add the flags automatically) —
otherwise the app builds fine but fails to *load* on machines without MinGW
runtime DLLs. Shipped binaries need only standard system libraries
(KERNEL32, UCRT, USER32). The DLL is x86_64-only.

For a real IME, implement `ITfKeyEventSink` (TSF): route `TestKeyDown` /
`KeyDown` to `khipro_feed_key`, open a composition for the preedit span,
and apply `committed_delta` with `ITfRange::SetText`.

### macOS

The release ships a **universal** dylib (arm64 + x86_64; macOS 11+). No
pkg-config — point the compiler at the unpacked `include/` and `lib/`:

```bash
clang app.c -Iinclude -Llib -lkhipro -Wl,-rpath,@loader_path/lib -o app
```

The dylib's install name is `@rpath/libkhipro.dylib`, so your rpath decides
where it lives at runtime. In an `.app` bundle, put it in
`Contents/Frameworks` and link with
`-Wl,-rpath,@loader_path/../Frameworks`; code-sign it with the bundle.
Static: `clang app.c -Iinclude lib/libkhipro.a -lc++` (macOS uses libc++).
A release-downloaded dylib is Gatekeeper-quarantined — clear it once with
`xattr -d com.apple.quarantine libkhipro.dylib`.

Wire into `NSTextInputClient` / `IMKInputController`: feed keys from
`interpretKeyEvents:`, render `preedit` via `setMarkedText:` (clearing the
previous marked text first), insert `committed_delta` via `insertText:`,
and forward the key to the system when `consumed == 0`.

### Android

One AAR contains all four ABIs plus the Kotlin bindings. Drop it into
`app/libs/` and depend on it:

```kotlin
dependencies {
    implementation(files("libs/khipro-<ver>.aar"))
}
```

`minSdk = 21`; the native libs are 16 KB-aligned, so they load on Android
15+ too. No permissions, no NDK setup, ~600 KB per ABI.

Two integration levels:

- **`KhiproEngine`** — the raw engine (`feedKey` / `backspace` / `commit` /
  `reset`, plus live `committed` / `preedit` properties and
  `AutoCloseable`).
- **`KhiproIme`** — wraps an `InputConnection`; `onKey()` and
  `onBackspace()` return `false` when the host should handle the key, and
  results are applied to the connection for you.

Inside an `InputMethodService`, call `finishComposingText()` **before**
applying new state, then `commitText(committedDelta)` and
`setComposingText(preedit)`. Use the `*_TOUCH` variants for soft keyboards;
`CLASSIC` suits ChromeOS/DeX hardware keyboards. Keep one engine per
`InputConnection` — engines are not thread-safe. `Khipro.layoutVersion` /
`Khipro.libraryVersion` are available statically for a debug screen.

### Web (WebAssembly)

The wasm zip has five files (`khipro.js`, `khipro.wasm`, `index.js`,
`index.d.ts`, `package.json`) — keep them co-located, or install
`@khiproteam/khipro` from npm. Loading is async; show a loading state for
the input while `KhiproEngine.create(...)` resolves.

The `attachKhiproInput(element, engine)` helper wires an entire
`<input>`/`<textarea>`:

- printable keys → `feedKey`, render `committed + preedit`
- Backspace → `backspace()`
- Enter → `commit()`, then the browser inserts the newline
- Escape → `reset()` (discard composition)
- blur → `commit()` (nothing is lost when the field loses focus)

Serving notes: `khipro.wasm` is ~500 KB raw, ~180 KB gzipped — serve it
compressed. Call `engine.destroy()` on unmount to free WASM memory. Multiple
engines on one page are safe (no shared preedit state). The top-level
`Khipro.layoutVersion` / `Khipro.libraryVersion` getters return empty
strings until the first engine has loaded. Use `classic_touch` for mobile
browsers, `classic` for physical keyboards.

### Building from source

```bash
git clone https://github.com/khiproteam/library.git khipro && cd khipro
make linux     # native by default
make windows   # container (docker/podman) by default; --native flips it
make android   # container; AAR needs Android SDK/NDK + Gradle
make wasm      # container
make test      # build + all four conformance suites
make sync      # pull upstream layouts, regenerate tables, build + test
```

Linux/macOS/test/sync default to a native toolchain (CMake + C++17);
windows/android/wasm default to containers. On macOS,
`KHIPRO_MACOS_ARCH=universal make macos` reproduces the release artifact.

The golden rule on every platform: **clear the previous preedit before
drawing the new state**, then apply `committed_delta`, then render
`preedit`.

## Versioning and releases

Five independent versions live in
[`khipro.meta`](https://github.com/khiproteam/library/blob/main/khipro.meta):

- `library_version` — the library's own release line (tags `lib-vX.Y.Z`)
- four layout versions — one per variant, each tracking its upstream

Releases are produced by three lines — `main` (finals, "Latest"), `beta`
(prereleases), and `develop` (the union ledger). Every release-line walk
takes **one upstream release per commit** and gates on the full conformance
suites before publishing — a broken upstream layout stops its line instead
of reaching consumers. A release never mixes channels and never moves
backward.

## Conformance testing

Each bundled layout ships a conformance suite
(`resources/khipro-testcases*.tsv`, ~100–116 keystroke-sequence cases per
variant). `make test` builds the library and runs all four suites plus the
engine unit tests. If you regenerate or patch a layout locally, run it —
the same suites gate every official release.

## Project links

- Repository — <https://github.com/khiproteam/library>
- Releases (prebuilt bundles) — <https://github.com/khiproteam/library/releases>
- Layout sources — the four upstream repos listed in the variants table above
- License — MIT
