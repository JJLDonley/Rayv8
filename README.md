# RayV8

RayV8 is a Windows x64 game toolkit for building games in JavaScript with V8
and raylib. It combines a small, focused game lifecycle with direct access to
raylib's graphics, input, audio, window, image, text, 2D, and 3D APIs.

## Documentation

Read the [RayV8 Game Toolkit API Handbook](document.md) for installation,
project structure, the complete `init` and `tick` API, managed resources,
input, drawing, packaging, examples, and the full raylib reference.

## Quick start

Download and extract the latest Windows x64 release. Open PowerShell in the
extracted directory, then generate and run a project:

```powershell
.\rayv8.exe generate MyGame
.\rayv8.exe run MyGame
```

Edit `MyGame\game\main.js` while it runs. RayV8 reloads successful changes
automatically and keeps the last working game active when an edit fails.

## Game structure

RayV8 provides one initializer and one frame function:

```js
init(
  { width: 1280, height: 720, title: "My Game" },
  (args) => {
    // The window exists here. Initialize state and managed resources.
  },
  [ConfigFlags.VSYNC_HINT]
);

/** @param {RayV8Args} args */
function tick(args) {
  const { state, inputs, outputs } = args;
  state.player ??= { x: 400, y: 225 };

  const speed = 240 * inputs.deltaTime;
  if (inputs.keyboard.left) state.player.x -= speed;
  if (inputs.keyboard.right) state.player.x += speed;

  outputs.backgroundColor = BLACK;
  outputs.solids.push({
    shape: "circle",
    x: state.player.x,
    y: state.player.y,
    radius: 24,
    color: SKYBLUE
  });
}
```

`init({}, callback)` uses a 1280 by 720 window titled `RayV8` when window
details are omitted. Its optional flag array is applied before window creation.
The callback receives the same `args` API used by `tick`.

RayV8 also supports raw top-level raylib scripts with no required lifecycle
functions. Games may use direct raylib calls, RayV8's state and output helpers,
or both.

## Build and distribute

Build a self-contained game executable:

```powershell
.\rayv8.exe build MyGame
```

RayV8 also supports sidecar and loose packaging:

```powershell
.\rayv8.exe build MyGame --sidecar
.\rayv8.exe build MyGame --loose
```

Packaged games use the Windows GUI subsystem and do not open a terminal.
Keep the generated `licenses` directory with redistributed games, or reproduce
the applicable notices in accompanying materials.

Run `.\rayv8.exe --help` for the command summary and consult the
[API handbook](document.md) for the complete toolkit reference.
