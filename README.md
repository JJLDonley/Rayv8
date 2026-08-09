# RayV8 Toolkit

RayV8 is a Windows x64 game-development toolkit that runs JavaScript on V8
and exposes raylib for graphics, input, audio, and related game APIs.

## Quick start

Extract the entire archive, open PowerShell in the extracted directory, and
generate a project:

```powershell
.\rayv8.exe generate MyGame
.\rayv8.exe run MyGame
```

Edit `MyGame\game\main.js` while the project is running. RayV8 reloads a
successfully compiled edit automatically; if an edit fails, the last valid
game context keeps running and the error is printed to the terminal.

## Game loop

RayV8 calls `tick(args)` every frame. State belongs in `args.state`, which is
preserved across successful hot reloads. Input is a frame snapshot and output
arrays are cleared after every frame:

```js
/** @param {RayV8Args} args */
function tick({ state, inputs, outputs }) {
  state.player ??= { x: 400, y: 225 };
  const speed = 220 * inputs.deltaTime;
  if (inputs.keyboard.left) state.player.x -= speed;
  if (inputs.keyboard.right) state.player.x += speed;

  outputs.backgroundColor = { r: 24, g: 48, b: 100, a: 255 };
  outputs.solids.push({
    shape: "circle", x: state.player.x, y: state.player.y,
    radius: 24, color: RED
  });
}
```

Available output queues are `sprites`, `solids`, and `labels`.
Keyboard input currently includes arrows and space; mouse input includes its
position and left-button state.

Declare managed assets once in `boot(args)`. Paths are relative to the entry
script. The same key and path reuse an asset across hot reloads; changing the
path replaces it. Omitting a previously declared key from a successful
`boot()` unloads it.

```js
let player;
let jump;

function boot(args) {
  player = args.resources.texture("player", "assets/player.png");
  jump = args.resources.sound("jump", "assets/jump.wav");
}

function tick({ outputs }) {
  outputs.sprites.push({ texture: player, x: 100, y: 100 });
  // jump.play({ volume: 0.8, pitch: 1.0, pan: 0.5 });
}
```

If `boot()` fails, RayV8 rolls back newly loaded assets and keeps the previous
game context, state, and resources running. Persistent state must be JSON-like:
objects, arrays, strings, numbers, booleans, and null.

Sound handles provide `play()`, `stop()`, `pause()`, `resume()`, `isPlaying()`,
`setVolume()`, `setPitch()`, and `setPan()`. Sound playback is immediate and is
not part of the renderer output queues.

## Window lifecycle

Use `configure(args)` for startup-only calls that must happen before window
creation, such as `SetConfigFlags`. It runs once when the process starts and is
not rerun during hot reload:

```js
function configure() {
  SetConfigFlags(ConfigFlags.WINDOW_TOPMOST);
}
```

The startup order is `configure()`, optional `init()`, automatic window
creation when needed, then `boot()`. Because `boot()` also runs on hot reload,
it should declare resources rather than set startup-only window flags. To
change an existing window at runtime, use `SetWindowState(flags)` and
`ClearWindowState(flags)`.

## Build a game

The default embedded mode creates one self-contained executable:

```powershell
.\rayv8.exe build MyGame
```

The available packaging modes are:

```powershell
.\rayv8.exe build MyGame --embed
.\rayv8.exe build MyGame --sidecar
.\rayv8.exe build MyGame --loose
```

- `--embed` produces one executable containing the game files.
- `--sidecar` produces an executable and a separate `.game.zip` archive.
- `--loose` produces an executable beside editable game files.

Run `.\rayv8.exe --help` to display the command summary.

## Project layout

A generated project contains:

```text
MyGame/
  game/main.js
  types/raylib.d.ts
  project.json
  jsconfig.json
```

`project.json` selects the entry script, output directory, included files,
and default packaging mode. The declaration file under `types/` provides
editor completion for the RayV8 game loop and JavaScript raylib API.

## Licenses

RayV8 executables include third-party open-source software. The applicable
license texts are distributed in the `licenses/` directory. They must remain
with redistributed copies of the toolkit where their terms require it.

These permissive licenses allow commercial and closed-source games. `rayv8
build` automatically copies the `licenses/` directory beside every generated
game executable. Keep that directory with redistributed games, or reproduce
the applicable notices in their installer, documentation, or other
accompanying materials.
