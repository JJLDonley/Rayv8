# RayV8

RayV8 v0.1.2 is a Windows x64 game toolkit for building games in JavaScript
with V8. It combines a typed callback-based game loop, working console output,
managed resources, and generated access to the complete raylib, raymath, and
rlgl API surfaces.

## Documentation

Read the [RayV8 Game Toolkit API Handbook](document.md) for installation,
project structure, the complete `init` and `tick` API, managed resources,
input, drawing, packaging, examples, and the full raylib reference.

Editor declarations are split by responsibility: `types/rayv8.d.ts` documents
the lifecycle, `args`, input, managed resources, and drawing queues, while
`types/raylib.d.ts` contains the generated raylib/raymath/rlgl surface. Generated
projects load both automatically through `jsconfig.json`.

## Quick start

Download and extract the latest Windows x64 release. Open PowerShell in the
extracted directory, then generate and run a project:

```powershell
.\rayv8.exe generate MyGame
.\rayv8.exe run MyGame
```

Refresh the toolkit declarations in an existing project after upgrading RayV8:

```powershell
.\rayv8.exe regen MyGame
```

`regen` only replaces the toolkit-owned files in `types/`; game code, assets,
`project.json`, and `jsconfig.json` are left unchanged.

Edit `MyGame\game\main.js` while it runs. RayV8 reloads successful changes
automatically and keeps the last working game active when an edit fails.

## Game structure

RayV8 provides one initializer and one frame function:

```js
init(
  { width: 1280, height: 720, title: "My Game" },
  ({ resource }) => {
    resource.texture("player", "assets/player.png");
    resource.sound("jump", "assets/jump.wav");
  },
  [ConfigFlags.VSYNC_HINT]
);

tick((args) => {
  const { state, keyboard, dt, solids } = args;
  state.player ??= { x: 400, y: 225 };

  const speed = 240 * dt;
  if (keyboard.keyDown("LEFT") || keyboard.keyDown("A")) state.player.x -= speed;
  if (keyboard.keyDown("RIGHT") || keyboard.keyDown("D")) state.player.x += speed;

  args.background = BLACK;
  solids.push({
    shape: "circle",
    x: state.player.x,
    y: state.player.y,
    radius: 24,
    color: SKYBLUE
  });
});
```

`init({}, callback)` uses a 1280 by 720 window titled `RayV8` when window
details are omitted. Its optional flag array is applied before window creation.
The callback receives `state`, `resource`, and `sound` for initialization.
Each frame, `tick` receives the flat RayV8 API: `state`, `frame`, `dt`,
`solids`, `sprites`, `labels`, `background`, `resource`, `sound`, `keyboard`,
`mouse`, and the connected `controller` array.

Managed textures, sounds, models, shaders, fonts, and music live in RayV8's
keyed resource inventory. RayV8 retains unchanged native assets across hot
reloads, rolls back failed replacements, and unloads removed assets. Models,
shaders, fonts, and music are returned for use with direct raylib calls.

RayV8 also supports raw top-level raylib scripts with no required lifecycle
functions. Games may combine direct raylib, raymath, and rlgl calls with
RayV8's state and output helpers. Native pointers cross the API as `bigint`
addresses; callback and variadic APIs require dedicated adapters before they
can safely call into JavaScript.

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
