# RayV8 Development Guide

This guide documents RayV8's architecture, source layout, build pipeline,
packaging process, and repository maintenance procedures.

## Install and use the toolkit

Download `rayv8-0.1.0-windows-x64.zip` from the [latest release](https://github.com/JJLDonley/Rayv8/releases/latest), extract the entire archive, and open PowerShell in the extracted directory. Then generate and run a project:

```powershell
.\rayv8.exe generate MyGame
.\rayv8.exe run MyGame
```

Edit `MyGame\game\main.js` while it runs. Build a self-contained game executable with:

```powershell
.\rayv8.exe build MyGame
```

The release archive includes IntelliSense declarations under `types/` and all applicable third-party license texts under `licenses/`.

The bundled permissive licenses allow commercial and closed-source games.
`rayv8 build` automatically copies the toolkit's `licenses/` directory beside
every generated game executable. Distributions should keep that directory or
reproduce the applicable notices in their installer, documentation, or other
accompanying materials.

## Architecture

The code is split by reason to change:

- `V8Platform` owns the process-wide V8 platform lifecycle.
- `JsRuntime` owns one isolate and transactional script-context replacement.
- `ResourceManager` transactionally owns keyed textures and sounds.
- `ScriptWatcher` only detects source changes.
- `RegisterRaylib` is the JavaScript-to-raylib boundary.
- `Engine` coordinates those pieces and owns the frame lifecycle.

All owning C++ types are non-copyable, V8 scopes stay local, filesystem failures use non-throwing `error_code` paths, and a failed reload leaves the last valid context active.

## Current API

New game scripts define `tick(args)` and may define `boot(args)`. The engine
creates a default 1280x720 window when a tick-style game does not create one in
`init()`. Legacy `init()`, `update()`, `draw()`, and `shutdown()` scripts remain
supported.

The initial lifecycle is `configure(args)` -> `init(args)` -> automatic window
creation if needed -> `boot(args)`. `configure()` is startup-only and is not
called for hot reloads, making it the correct place for `SetConfigFlags`.
Reloads transactionally run only `boot(args)`. Existing-window changes belong
in normal runtime code through `SetWindowState` and `ClearWindowState`.

`args.state` is serialized from the active context and parsed into a candidate
context before reload, so it must remain JSON-like. `args.inputs` is rebuilt
from raylib each frame. `args.outputs` contains fresh sprite, solid, and label
command arrays consumed by the native renderer. Sounds are persistent handles
with playback and parameter-control methods, separate from renderer output.

`boot(args)` declares keyed textures and sounds through `args.resources`.
Declarations form a transaction: unchanged key/path pairs reuse native assets,
changed paths replace them, absent keys are unloaded after a successful boot,
and a thrown boot rolls back candidate assets without disturbing the active
context. An absent `boot()` retains existing resources; use an empty
`boot(args) {}` to remove all declarations.

Raylib functions otherwise keep their original names and argument order. Struct
values use JavaScript objects, for example `{ r: 255, g: 0, b: 0, a: 255 }` for
`Color`.

Bindings are generated from raylib's `raylib_api.json`: 613 functions, 35 structs, 21 grouped enums, and numeric/string defines from the current raylib master checkout. Callback and variadic functions are registered but currently throw an adapter-required error rather than entering V8 unsafely from native threads.

## Build

Requirements: CMake 3.24+, a C++20 compiler, Git, and a prebuilt static V8 SDK. The SDK must contain `include/v8.h` and `lib/v8_monolith` built with pointer compression. CMake prefers the local `.deps/raylib` master checkout and fetches master automatically when it is absent.

After the dependencies under `.deps` have been prepared, use the top-level build command. It builds raylib with the matching runtime, synchronizes RayV8 into V8's GN graph, and publishes artifacts under `build/bin`:

```powershell
.\tools\build.cmd
```

### Rebuilding after renaming the repository folder

CMake and V8's GN output store absolute paths. After renaming or moving the
repository to `C:\RayV8`, run the normal build command from the new repository
root:

```powershell
Set-Location C:\RayV8
.\tools\build.cmd
```

The first build after a move regenerates those paths and may recompile a
substantial part of the V8 GN output once. Later incremental builds reuse the
new output and are faster. Do not rebuild or replace `.deps\v8-sdk` solely
because the repository moved; the existing prebuilt SDK remains valid.

## Repository layout

- `examples/basic/` is the runnable example project, including its `game/`
  directory and `project.json`.
- `sdk/types/` contains the generated TypeScript declarations used when
  assembling the developer SDK.
- `test/` and `tests/` contain generated-project fixtures and runtime tests.

## Distributing the developer SDK

Create a versioned Windows x64 SDK archive with:

```powershell
.\tools\package.cmd
```

The version defaults to the CMake project version. Override it when preparing a
specific prerelease:

```powershell
.\tools\package.cmd -Version 0.2.0-alpha.1
```

The command performs a release build and writes the archive and SHA-256 file to
`build/dist/`. Use `-SkipBuild` only when the current `build/bin` artifacts have
already been verified. The SDK archive contains `rayv8.exe`,
`rayv8_stub.exe`, `types/raylib.d.ts`, the README, and a quick-start guide.
Both executables are statically linked to V8, raylib, and the C++ runtime; their
remaining dependencies are standard Windows system DLLs.

V8 distributions vary. If yours was built without pointer compression or sandboxing, remove the matching compile definition in `CMakeLists.txt` so the host and plugin exactly match that build.

The build publishes two separate programs:

- `rayv8.exe` is the developer CLI.
- `rayv8_stub.exe` is the runtime-only game host used by the packager. It contains no CLI commands.

`rayv8.exe` is linked as a console application for developer diagnostics.
`rayv8_stub.exe` uses the Windows GUI subsystem with `mainCRTStartup`, so games
produced from the stub launch without a terminal window.

Create, run, and package projects with:

```powershell
.\build\bin\rayv8.exe generate MyGame
.\build\bin\rayv8.exe run MyGame
.\build\bin\rayv8.exe build MyGame
```

Every project has a `project.json` manifest:

```json
{
  "name": "MyGame",
  "entry": "game/main.js",
  "build": {
    "mode": "embed",
    "output": "build/bin",
    "include": ["game", "types", "project.json"]
  }
}
```

`build.mode` may be `embed`, `sidecar`, or `loose`. The equivalent `--embed`,
`--sidecar`, and `--loose` command flags override the manifest for one build.
Embedded mode emits one `<name>.exe`; sidecar mode emits the executable and
`<name>.game.zip`; loose mode emits the executable with the configured project
files. Packaged archives retain `game/`, `types/`, and `project.json`, and the
runtime stub reads the manifest to locate the configured entry.

Save the configured entry script while `rayv8 run` is active. A successfully
compiled and booted edit replaces the JS context. A syntax error or thrown
`boot()` is printed and the previous context, state, and resources keep running.

## Next useful layers

- Generate the complete raylib binding from `raylib_api.json`.
- Extend managed resources to music, fonts, shaders, models, and render targets.
- Expand normalized input beyond the initial keyboard and mouse snapshot.
- Add a V8-independent C ABI for native plugins and hot-reload copied DLLs.
- Add debugger/inspector support.
