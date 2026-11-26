# sludgemu

**A compatibility layer that runs Xbox Series X|S titles on Windows desktop.**

Xbox Series X|S ("Scarlett") titles are x64 Windows PE images built against the console flavour of
the Microsoft GDK. Because the CPU side is already x64 Windows code, sludgemu does not emulate a
CPU. It loads the title straight into its own process and supplies the parts of the platform that
only exist on console:

- **Memory:** the console's 64 KiB page-frame memory model, with real aliasing (`xmem.dll`)
- **Graphics:** the console D3D12 runtime (`d3d12_xs.dll`) translated to desktop D3D12 through the
  Agility SDK. Compiled RDNA2 shader binaries are recompiled to DXIL at runtime, and console texture
  layouts are reproduced exactly with AMD addrlib (`xg_xs.dll`).
- **Storage:** DirectStorage on top of desktop DirectStorage, with zlib (libdeflate) and BCPack
  decompression
- **Audio:** the console audio hardware: XApu (XMA and Opus decoding), the XDSP effects path, and
  ACP HAL
- **System services:** the Game Runtime (`xgameruntime.dll`): task queues, async, users, game
  saves, package, store and system APIs
- **Desktop integration:** the title gets a normal resizable window with borderless fullscreen

> [!WARNING]
> **Status: early development.** The loader, memory model, graphics translation, shader recompiler,
> DirectStorage and audio paths are all under active development. Many console APIs are still
> logging stubs, and there is no compatibility list yet. Expect titles to stop at startup, hang or
> crash. Don't expect any title to be playable yet.

> [!IMPORTANT]
> sludgemu contains **no Microsoft GDK code or headers** and **no title code or data**. To use it you
> need your own legally obtained, decrypted title package. Don't open issues or PRs that contain
> GDK material, title files or links to them.

---

## Contents

- [Requirements](#requirements)
- [Building](#building)
- [Running](#running)
- [Diagnostics](#diagnostics)
- [Tests and tools](#tests-and-tools)
- [Project layout](#project-layout)
- [Documentation](#documentation)
- [Contributing](#contributing)
- [Third-party code](#third-party-code)

## Requirements

**To build**

| | |
|---|---|
| OS | Windows 10 or 11, x64 |
| Compiler | Visual Studio 2022 with the *Desktop development with C++* workload (MSVC only, since the title ABI is MSVC x64) |
| Build system | CMake 3.24 or newer (Ninja is optional) |
| Network | Needed on the first configure. Dependencies are downloaded then, and nothing is vendored. |

**To run**

- A Direct3D 12 GPU. sludgemu ships its own D3D12 runtime through the Agility SDK (1.620), so the
  OS build doesn't matter, but the GPU driver still has to support the features the title uses.
- Plenty of RAM. sludgemu presents itself as a **Series X** only on hosts with **at least 24 GB of
  RAM and a 10 GB GPU**. Otherwise it presents as a **Series S**, and titles scale their memory
  budget, resolution and quality to match. `--model` overrides this choice.

## Building

```powershell
cmake --preset vs2022
cmake --build --preset vs2022-release     # or vs2022-debug
```

Binaries go to `build/vs2022/bin/<Config>/`. The release preset builds `RelWithDebInfo`.

To build with Ninja instead, run these from an *x64 Native Tools / Developer PowerShell for VS 2022*
prompt:

```powershell
cmake --preset ninja-release              # or ninja-debug
cmake --build --preset ninja-release
```

The build copies the runtime DLLs it needs next to `sludgemu.exe`: DirectStorage, DXC
(`dxcompiler.dll`, `dxil.dll`), and the Agility SDK's `D3D12Core.dll` in a `D3D12\` subfolder.
Keep these files together when you move the build.

<details>
<summary>Build notes for contributors</summary>

- The code is C++20 and compiles **without exceptions or RTTI** (`/EHs-c- /GR-`). It links the
  **static CRT**, so sludgemu's CRT state stays separate from the title's.
- `sludgemu.exe` must not use implicit TLS (no `thread_local`), because the title owns TLS slot 0.
  A startup check enforces this. [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md#tls) explains why.
- Code style is defined in [`.clang-format`](.clang-format).

</details>

## Running

```
sludgemu.exe [options] <package directory | title exe> [-- title arguments]
```

Point sludgemu at an extracted package directory or directly at the title's executable. Any
arguments after `--`, or after the first unrecognised positional argument, are passed to the title.

The title opens its own window like a native game. **F11** or **Alt+Enter** toggles borderless
fullscreen, and closing the window exits.

### Options

| Option | Description |
|---|---|
| `--model=series-x\|series-s` | Console model to present as (default: chosen from host RAM and VRAM, see [Requirements](#requirements)) |
| `--console` | Open a console window that shows the live log |
| `--log-level=LEVEL` | `trace`, `debug`, `info` (default), `warn` or `error` |
| `--gpu-warp` | Render with WARP, the software rasterizer, instead of the GPU |
| `--gpu-debug` | Enable the D3D12 debug layer |
| `--gpu-serialize` | Wait for the GPU after every title submission (for timing diagnostics) |
| `--frame-stats` | Log presents per second and the longest frame interval, once a second |
| `--break-on-stub` | Break into an attached debugger when the title calls an unimplemented import |
| `--dump-threads-after=N` | Log a backtrace of every thread every N seconds (for diagnosing hangs) |
| `--watch-write=SPEC` | Hardware watchpoint that logs every write (value and backtrace) to an 8-byte location: `module.dll+0xRVA`, `0xADDRESS` or `ldrentry:N` |
| `--trace-exec=SPEC[,...]` | Log registers whenever up to three code locations execute |

### Files sludgemu writes

| Path | Contents |
|---|---|
| `sludgemu.log`, next to the exe | Full log of the session |
| `sludgemu_crash.dmp`, next to the exe | Minidump written on a crash |
| `%LOCALAPPDATA%\sludgemu\<title id>\PersistentLocalStorage\` | The title's persistent local storage |

## Diagnostics

These environment variables turn on heavier debugging features:

| Variable | Effect |
|---|---|
| `SLUDGE_SHADER_DUMP=<dir>` | Write each title shader program (raw code and translated HLSL, plus the error when translation fails) to `<dir>` |
| `SLUDGE_CAPTURE_FRAMES=<first>[,<count>]` | Save presented frames, starting at frame `<first>` |
| `SLUDGE_CAPTURE_DIR=<dir>` | Folder for captured frames (default: the current directory) |
| `SLUDGE_RENDERDOC_FRAME=<n>[,<n>...]` | Trigger RenderDoc captures on the given frames, when sludgemu runs under RenderDoc |
| `SLUDGE_TRACE_TEXTURES=<W>x<H>` | Trace the creation and synchronisation of textures with these dimensions |
| `SLUDGE_TRACE_TEXTURES_DUMP=<dir>` | Also dump the traced textures to `<dir>` |
| `SLUDGE_TRACE_SKIP=N` | With `--trace-exec`: skip the first N hits |
| `SLUDGE_TRACE_DUMP=BYTES` | With `--trace-exec`: also dump memory at pointer registers |

Debugging conventions:

- Addresses inside title images are logged as `module+rva [preferred-base address]`, so you can
  paste them straight into a disassembler. The title is mapped at its preferred base whenever
  possible.
- Each call to an unimplemented import logs `module!function called from <address>` and returns
  0, so no failure goes unnoticed.
- Investigate graphics problems with a frame capture (RenderDoc or Nsight Graphics) rather than by
  guessing.

## Tests and tools

**Unit tests** cover the platform-independent parts: the memory model, texture layouts, BCPack,
XMA decoding, XApu and XDSP.

```powershell
ctest --test-dir build/vs2022 -C RelWithDebInfo
```

**`shader_tool`** analyses console shader code, usually taken from a `SLUDGE_SHADER_DUMP` folder:

```
shader_tool disasm <code.bin>                      disassemble raw GPU code
shader_tool stats <dir> [name filter]              instruction statistics; lists undecodable instructions
shader_tool translate <dir> <pipeline> <vs|ps>     translate one stage of a dumped pipeline to HLSL
shader_tool translate-all <dir> <out dir>          translate every unique program and summarise failures
```

## Project layout

| Path | Contents |
|---|---|
| `src/main.cpp` | Entry point and command-line parsing |
| `src/loader/` | PE mapping, import binding, TLS, logging stubs, crash reporting and diagnostics |
| `src/core/` | Title metadata, console memory model, console profile (X or S), CPU topology, virtual file system (`G:\` mapped to the package) |
| `src/hle/` | Console system DLLs implemented by sludgemu, plus host-function overrides |
| `src/hle/d3d12/` | Console D3D12 runtime (`d3d12_xs.dll`) and the BVH builder |
| `src/hle/xg/` | Texture layout library (`xg_xs.dll`) |
| `src/hle/xmem/` | Title memory management (`xmem.dll`) |
| `src/hle/audio/` | XApu, XMA, XDSP, ACP HAL |
| `src/hle/dstorage/` | DirectStorage and the BCPack decoder |
| `src/hle/xgameruntime/` | Game Runtime: task queues, async, users, saves, package, store, system |
| `src/hle/kernel/` | Overrides for host module, memory, file, registry and CPU functions |
| `src/gpu/` | Desktop GPU backend: memory windows, page tables, tile pools, texture untiling, presentation |
| `src/shader/` | RDNA2 decoder and disassembler, and the shader translator to HLSL/DXIL |
| `src/platform/` | Desktop window management |
| `src/common/` | Logging, strings, shared types |
| `tests/` | Unit tests |
| `tools/` | Developer tools (`shader_tool`) |
| `cmake/` | Dependency fetch helpers (NuGet, addrlib) |
| `docs/` | Design notes |

## Documentation

- [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md): process model, loader, TLS, CPU topology,
  memory, graphics, file system, Game Runtime and debugging conventions
- [docs/GPU_MEMORY.md](docs/GPU_MEMORY.md): how the GPU accesses title memory in place through
  windows and page tables, and how coherence works
- [docs/BCPACK.md](docs/BCPACK.md): the BCPack texture compression stream format

## Contributing

Bug reports and patches are welcome. Before you open an issue:

1. Reproduce the problem with `--console --log-level=debug` and attach `sludgemu.log`. Include
   `sludgemu_crash.dmp` if the title crashed.
2. Describe your host: CPU, GPU and driver version, RAM, Windows build, and the console model
   sludgemu picked.
3. Never attach or link title files, decrypted packages or GDK material.

For a pull request, follow `.clang-format`, keep to the no-exceptions/no-RTTI/no-`thread_local`
rules above, and add a unit test in `tests/` when the code is platform-independent.

## Third-party code

The XMA decoder is derived from FFmpeg and keeps its **LGPL-2.1-or-later** license. libdeflate,
Opus, AMD addrlib, the D3D12 Agility SDK, DirectStorage and DXC are downloaded at build time under
their own licenses. [THIRD_PARTY.md](THIRD_PARTY.md) has the full list.

---

*Xbox, Xbox Series X|S, DirectX and DirectStorage are trademarks of Microsoft. This project has no
affiliation with Microsoft and is not endorsed by it.*
