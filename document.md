# RayV8 Game Toolkit Handbook

RayV8 lets you make Windows games in JavaScript. It runs your code on V8 and
provides raylib for windows, drawing, input, sound, textures, cameras, and more.
This handbook covers the RayV8 game API and the workflow from a new project to
a distributable game.

## Contents

- [Start a project](#start-a-project)
- [Project files](#project-files)
- [The game lifecycle](#the-game-lifecycle)
- [The frame arguments](#the-frame-arguments)
  - [Persistent state](#persistent-state)
  - [Input](#input)
  - [Output queues](#output-queues)
  - [Managed resources](#managed-resources)
- [Drawing sprites](#drawing-sprites)
- [Drawing shapes](#drawing-shapes)
- [Drawing text](#drawing-text)
- [Keyboard and mouse input](#keyboard-and-mouse-input)
- [Textures and sound](#textures-and-sound)
- [Using the raylib API directly](#using-the-raylib-api-directly)
- [Colors, vectors, and rectangles](#colors-vectors-and-rectangles)
- [Window setup](#window-setup)
- [Hot reload](#hot-reload)
- [Project configuration](#project-configuration)
- [Run and build commands](#run-and-build-commands)
- [Complete starter game](#complete-starter-game)
- [Troubleshooting](#troubleshooting)
- [API quick reference](#api-quick-reference)

## Start a project

Extract the RayV8 release ZIP as a complete folder. Open PowerShell in that
folder and run:

```powershell
.\rayv8.exe generate MyGame
.\rayv8.exe run MyGame
```

Edit `MyGame\game\main.js` while the game is running. Saving a valid edit
reloads the game automatically.

RayV8 uses ordinary JavaScript. You do not need Node.js, npm, a browser, or a
web server.

## Project files

A generated project looks like this:

```text
MyGame/
  game/
    main.js
  types/
    raylib.d.ts
  jsconfig.json
  project.json
```

- `game/main.js` is the default game entry point.
- `types/raylib.d.ts` describes all RayV8 and raylib globals to your editor.
- `jsconfig.json` enables JavaScript checking and editor completion.
- `project.json` controls the game name, entry point, output folder, included
  files, and packaging mode.

Keep images, sounds, and other game data inside the project, normally under
`game/assets/`. Add anything needed at runtime to `build.include` in
`project.json`.

## The game lifecycle

RayV8 recognizes these global functions in your entry script:

| Function | When it runs | Typical use |
|---|---|---|
| `configure(args)` | Once, before window creation | Startup window flags |
| `init(args)` | Once at startup | Optional manual window setup |
| `boot(args)` | At startup and after a successful reload | Declare managed assets |
| `tick(args)` | Once per frame | Update the game and queue drawing |
| `shutdown(args)` | Once before exit | Final cleanup |

Most games only need `tick`. RayV8 creates a 1280 by 720 window titled
`RayV8` automatically when `tick` exists and `init` did not create a window.

```js
/** @param {RayV8Args} args */
function tick({ inputs, outputs }) {
  outputs.backgroundColor = BLACK;
  outputs.labels.push({
    text: `Frame ${inputs.frame}`,
    x: 20,
    y: 20,
    color: RAYWHITE
  });
}
```

### The classic loop

Advanced games can omit `tick` and use `init`, `update`, and `draw`. In that
mode, create the window yourself in `init()` and call raylib drawing functions
inside `draw()`:

```js
function init() {
  InitWindow(960, 540, "My Game");
  SetTargetFPS(60);
}

function update() {
  // Update game state.
}

function draw() {
  ClearBackground(BLACK);
  DrawText("Hello!", 30, 30, 30, RAYWHITE);
}
```

Choose either the managed `tick(args)` style or the classic loop for a game.
The managed style is the easiest place to start and provides reload-safe state
and resources.

## The frame arguments

`tick(args)` receives one `RayV8Args` object each frame:

```js
function tick({ state, inputs, outputs, resources }) {
  // state: persistent game data
  // inputs: current frame and input snapshot
  // outputs: drawing commands for this frame
  // resources: managed texture and sound declarations
}
```

The same object is also available as the global `args`, but accepting the
function parameter makes code clearer and gives editors better assistance.

### Persistent state

Put game data that must survive between frames and hot reloads in
`args.state`:

```js
function tick({ state, inputs }) {
  state.score ??= 0;
  state.elapsed ??= 0;
  state.elapsed += inputs.deltaTime;
}
```

State must remain JSON-like: objects, arrays, strings, numbers, booleans, and
`null`. Do not store functions, texture handles, sound handles, cyclic objects,
or other native values in it. Keep resource handles in script-level variables
declared by `boot()`.

### Input

`args.inputs` is a snapshot for the current frame:

| Field | Type | Meaning |
|---|---|---|
| `frame` | `bigint` | Frame number |
| `deltaTime` | `number` | Seconds since the last frame |
| `keyboard.left` | `boolean` | Left arrow held |
| `keyboard.right` | `boolean` | Right arrow held |
| `keyboard.up` | `boolean` | Up arrow held |
| `keyboard.down` | `boolean` | Down arrow held |
| `keyboard.space` | `boolean` | Space held |
| `keyboard.spacePressed` | `boolean` | Space pressed this frame |
| `mouse.x`, `mouse.y` | `number` | Mouse position in window coordinates |
| `mouse.left` | `boolean` | Left mouse button held |
| `mouse.leftPressed` | `boolean` | Left mouse button pressed this frame |

Use `deltaTime` for movement so the game behaves consistently at different
frame rates:

```js
state.x += 240 * inputs.deltaTime;
```

For other keys, mouse buttons, controllers, touch, or gestures, use the direct
raylib input functions described below.

### Output queues

`args.outputs` describes what RayV8 draws after `tick` finishes:

| Field | Purpose |
|---|---|
| `backgroundColor` | Clear color for this frame |
| `sprites` | Textured images |
| `solids` | Rectangles and circles |
| `labels` | Text labels |

The three queues are cleared after every frame. Push every object that should
be visible on the current frame. `backgroundColor` is also a per-frame value.

Objects are drawn in this order: background, sprites, solids, then labels.
Items within a queue are drawn in insertion order.

### Managed resources

Declare textures and sounds in `boot(args)`. Paths are relative to the entry
script, so a path such as `assets/player.png` points to
`game/assets/player.png` when the entry is `game/main.js`.

```js
let playerTexture;
let jumpSound;

function boot({ resources }) {
  playerTexture = resources.texture("player", "assets/player.png");
  jumpSound = resources.sound("jump", "assets/jump.wav");
}
```

Each key identifies one resource. Reusing a key and path keeps the resource
loaded across hot reloads. Changing the path replaces it. Leaving a key out of
the next successful `boot()` unloads it.

Resources may only be declared inside `boot()`. If any declaration or the new
script fails, RayV8 rolls back the new resources and keeps the last working
game running.

## Drawing sprites

Push a sprite into `outputs.sprites`:

```js
outputs.sprites.push({
  texture: playerTexture,
  x: 200,
  y: 140,
  rotation: 0,
  scale: 1,
  tint: WHITE
});
```

| Property | Required | Default | Meaning |
|---|---:|---|---|
| `texture` | Yes | — | A texture returned by `resources.texture()` |
| `x`, `y` | Yes | — | Top-left position |
| `rotation` | No | `0` | Clockwise rotation in degrees |
| `scale` | No | `1` | Uniform size multiplier |
| `tint` | No | `WHITE` | Color multiplied over the texture |

The texture handle includes `width` and `height`, which can help with
centering, collision boxes, and screen limits.

## Drawing shapes

Push rectangles and circles into `outputs.solids`.

Rectangle:

```js
outputs.solids.push({
  shape: "rectangle",
  x: 40,
  y: 80,
  width: 220,
  height: 60,
  color: BLUE
});
```

Circle:

```js
outputs.solids.push({
  shape: "circle",
  x: 400,
  y: 225,
  radius: 24,
  color: RED
});
```

`shape` defaults to `rectangle`. For a rectangle, supply `width` and `height`.
For a circle, `x` and `y` are its center and `radius` sets its size.

## Drawing text

Push text into `outputs.labels`:

```js
outputs.labels.push({
  text: `Score: ${state.score}`,
  x: 24,
  y: 24,
  size: 28,
  color: RAYWHITE
});
```

`text`, `x`, and `y` are required. `size` and `color` are optional.

For font measurement, custom fonts, spacing, or text effects, use raylib
functions such as `MeasureText`, `DrawTextEx`, and `LoadFont` directly.

## Keyboard and mouse input

The input snapshot is convenient for common controls:

```js
function tick({ state, inputs }) {
  state.player ??= { x: 100, y: 100 };
  const step = 300 * inputs.deltaTime;

  if (inputs.keyboard.left) state.player.x -= step;
  if (inputs.keyboard.right) state.player.x += step;
  if (inputs.keyboard.up) state.player.y -= step;
  if (inputs.keyboard.down) state.player.y += step;

  if (inputs.keyboard.spacePressed) {
    state.shots ??= 0;
    state.shots++;
  }
}
```

Use held fields such as `space` for continuous behavior. Use pressed fields
such as `spacePressed` for one action per key press.

The full raylib input API is also global:

```js
if (IsKeyPressed(KeyboardKey.KEY_ENTER)) state.started = true;
if (IsMouseButtonDown(MouseButton.MOUSE_BUTTON_RIGHT)) {
  const mouse = GetMousePosition();
  state.target = { x: mouse.x, y: mouse.y };
}

if (IsGamepadAvailable(0)) {
  const horizontal = GetGamepadAxisMovement(0, GamepadAxis.GAMEPAD_AXIS_LEFT_X);
  state.player.x += horizontal * 300 * GetFrameTime();
}
```

## Textures and sound

Managed resources are reload-safe and are recommended for normal game assets.

```js
let hero;
let pickup;

function boot({ resources }) {
  hero = resources.texture("hero", "assets/hero.png");
  pickup = resources.sound("pickup", "assets/pickup.wav");
}

function tick({ inputs, outputs }) {
  outputs.sprites.push({ texture: hero, x: 100, y: 100 });

  if (inputs.keyboard.spacePressed) {
    pickup.play({ volume: 0.8, pitch: 1, pan: 0.5 });
  }
}
```

A managed sound supports:

| Method | Purpose |
|---|---|
| `play(options?)` | Play with optional `volume`, `pitch`, and `pan` |
| `stop()` | Stop playback |
| `pause()` | Pause playback |
| `resume()` | Resume playback |
| `isPlaying()` | Return whether it is currently playing |
| `setVolume(value)` | Set volume |
| `setPitch(value)` | Set pitch |
| `setPan(value)` | Set stereo pan |

Sound playback happens immediately; sounds are not placed in an output queue.

## Using the raylib API directly

RayV8 exposes raylib functions, constants, enums, and structures as JavaScript
globals. Names follow raylib exactly:

```js
DrawFPS(10, 10);
DrawLine(10, 80, 300, 80, GOLD);
DrawCircle(400, 220, 40, SKYBLUE);

const center = { x: GetScreenWidth() / 2, y: GetScreenHeight() / 2 };
const distance = Vector2Distance(center, GetMousePosition());
```

Useful API groups include:

- Window and timing: `SetWindowTitle`, `SetWindowSize`, `GetScreenWidth`,
  `GetScreenHeight`, `GetFrameTime`, `GetTime`, `GetFPS`
- Input: `IsKeyPressed`, `IsKeyDown`, `GetMousePosition`,
  `IsMouseButtonPressed`, gamepad, touch, and gesture functions
- Shapes: `DrawPixel`, `DrawLine`, `DrawCircle`, `DrawRectangle`, triangles,
  rings, splines, and collision helpers
- Textures and images: image loading and manipulation, texture drawing,
  render textures, filters, and wrapping
- Text: `DrawText`, `DrawTextEx`, `MeasureText`, fonts, and Unicode helpers
- 2D and 3D cameras: `BeginMode2D`, `BeginMode3D`, coordinate conversion, and
  camera controls
- 3D: models, meshes, materials, shaders, rays, bounding boxes, and billboards
- Audio: wave, sound, music, and audio-stream functions
- Math: vectors, matrices, quaternions, interpolation, and collision tests

Your project's `types/raylib.d.ts` is the exact API reference for the bundled
version. In an editor such as VS Code, type a function name or hover over it to
see parameter types and documentation. Search that file for `declare function`
to browse all functions, and `declare const` to browse enums and constants.

Some low-level raylib functions use native pointers represented as `bigint`.
They are intended for advanced use. Prefer managed resources and functions
that accept or return normal JavaScript objects when possible.

### Drawing directly during `tick`

RayV8 starts the drawing frame after `tick()` returns, so the managed output
queues are the normal way to draw in tick-style games. Use the classic
`init`/`update`/`draw` lifecycle when a game relies heavily on direct raylib
drawing calls or nested drawing modes.

## Colors, vectors, and rectangles

Raylib structures are plain JavaScript objects:

```js
const color = { r: 40, g: 120, b: 220, a: 255 };
const point = { x: 100, y: 200 };
const area = { x: 20, y: 30, width: 160, height: 90 };
```

Color channels range from 0 to 255. RayV8 provides raylib's named colors:

`LIGHTGRAY`, `GRAY`, `DARKGRAY`, `YELLOW`, `GOLD`, `ORANGE`, `PINK`, `RED`,
`MAROON`, `GREEN`, `LIME`, `DARKGREEN`, `SKYBLUE`, `BLUE`, `DARKBLUE`,
`PURPLE`, `VIOLET`, `DARKPURPLE`, `BEIGE`, `BROWN`, `DARKBROWN`, `WHITE`,
`BLACK`, `BLANK`, `MAGENTA`, and `RAYWHITE`.

## Window setup

Use `configure()` for flags that raylib requires before window creation:

```js
function configure() {
  SetConfigFlags(
    ConfigFlags.FLAG_WINDOW_RESIZABLE |
    ConfigFlags.FLAG_VSYNC_HINT
  );
}
```

`configure()` runs only at process startup, not during hot reload. Use
`SetWindowState(flags)` and `ClearWindowState(flags)` to change supported
states on an existing window.

To control the initial dimensions and title, create the window in `init()`:

```js
function init() {
  InitWindow(1024, 576, "Star Runner");
  SetTargetFPS(60);
}
```

## Hot reload

While `rayv8 run` is active, saving the entry script starts a reload. On a
successful reload:

- `state` is preserved.
- `boot(args)` runs again.
- Managed resources with unchanged keys and paths are reused.
- Managed resources omitted by the new `boot()` are unloaded.

If the new script does not compile or `boot()` fails, RayV8 prints the error
and keeps the last valid game context, state, and resources running. Fix and
save the file again to retry.

`configure()` is startup-only. Restart the game after changing startup window
flags or other one-time configuration.

## Project configuration

Example `project.json`:

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

| Setting | Meaning |
|---|---|
| `name` | Game and output executable name |
| `entry` | Main script, relative to the project folder |
| `build.mode` | `embed`, `sidecar`, or `loose` |
| `build.output` | Output path inside the project's `build` directory |
| `build.include` | Files and folders copied into the packaged game |

Paths must be relative and stay inside the project. `build.output` must remain
inside the project's `build` directory.

Remember to include asset folders used by the game. The default `"game"`
entry includes everything under `game/`, including `game/assets/`.

## Run and build commands

Generate a project:

```powershell
.\rayv8.exe generate MyGame
```

Run with hot reload:

```powershell
.\rayv8.exe run MyGame
```

Build using the mode in `project.json`:

```powershell
.\rayv8.exe build MyGame
```

Override the packaging mode:

```powershell
.\rayv8.exe build MyGame --embed
.\rayv8.exe build MyGame --sidecar
.\rayv8.exe build MyGame --loose
```

- `embed` creates one self-contained game executable.
- `sidecar` creates an executable and a separate `.game.zip` file.
- `loose` creates an executable beside editable game files.

Packaged game executables do not open a terminal window. Keep the generated
`licenses` directory with redistributed games, or reproduce the applicable
notices in accompanying materials.

Run `.\rayv8.exe --help` for the command summary.

## Complete starter game

This example combines persistent state, movement, a managed texture and sound,
input, shapes, and labels:

```js
let playerTexture;
let jumpSound;

function configure() {
  SetConfigFlags(ConfigFlags.FLAG_VSYNC_HINT);
}

function boot({ resources }) {
  playerTexture = resources.texture("player", "assets/player.png");
  jumpSound = resources.sound("jump", "assets/jump.wav");
}

/** @param {RayV8Args} args */
function tick({ state, inputs, outputs }) {
  state.player ??= { x: 400, y: 300 };
  state.score ??= 0;

  const speed = 260 * inputs.deltaTime;
  if (inputs.keyboard.left) state.player.x -= speed;
  if (inputs.keyboard.right) state.player.x += speed;
  if (inputs.keyboard.up) state.player.y -= speed;
  if (inputs.keyboard.down) state.player.y += speed;

  if (inputs.keyboard.spacePressed) {
    state.score++;
    jumpSound.play({ volume: 0.75 });
  }

  state.player.x = Math.max(0, Math.min(1280 - playerTexture.width, state.player.x));
  state.player.y = Math.max(0, Math.min(720 - playerTexture.height, state.player.y));

  outputs.backgroundColor = { r: 18, g: 24, b: 38, a: 255 };
  outputs.sprites.push({
    texture: playerTexture,
    x: state.player.x,
    y: state.player.y
  });
  outputs.solids.push({
    shape: "rectangle",
    x: 16,
    y: 16,
    width: 190,
    height: 48,
    color: { r: 0, g: 0, b: 0, a: 160 }
  });
  outputs.labels.push({
    text: `Score: ${state.score}`,
    x: 28,
    y: 28,
    size: 24,
    color: RAYWHITE
  });
}
```

Place `player.png` and `jump.wav` under `game/assets/`, then run the project.

## Troubleshooting

### The game does not reload

Check the terminal running `rayv8.exe`. A syntax or runtime error leaves the
last valid version running. Fix the reported error and save again.

### An asset cannot be found

Managed asset paths are relative to the entry script, not the PowerShell
working directory. With `entry` set to `game/main.js`, use
`assets/player.png` for `game/assets/player.png`.

### An asset is missing from the built game

Make sure its file or parent folder appears in `build.include`. Including
`"game"` includes every asset below the game folder.

### State disappears or reload fails

Only store JSON-like data in `state`. Keep textures and sounds in normal
script variables initialized by `boot()`.

### Drawing appears behind another object

Output categories have a fixed order: sprites, then solids, then labels.
Within the same queue, push the background item first and the foreground item
last. Use direct raylib drawing with the classic lifecycle when you need full
control over mixed draw ordering.

### A direct raylib call is unclear

Open `types/raylib.d.ts` and search for the function. Its declaration lists the
accepted JavaScript types, return type, and raylib description.

## API quick reference

```text
Lifecycle
  configure(args)  startup configuration, before the window
  init(args)       optional one-time initialization
  boot(args)       managed resources; reruns after reload
  tick(args)       update and queue output every frame
  shutdown(args)   final cleanup

RayV8Args
  state            persistent JSON-like data
  inputs           frame, deltaTime, keyboard, mouse
  outputs          backgroundColor, sprites, solids, labels
  resources        texture(key, path), sound(key, path)

Sprite
  texture, x, y, rotation?, scale?, tint?

Solid
  shape?, x, y, width?, height?, radius?, color?

Label
  text, x, y, size?, color?

Sound
  play(options?), stop(), pause(), resume(), isPlaying()
  setVolume(value), setPitch(value), setPan(value)

Full raylib surface
  See types/raylib.d.ts for functions, structures, constants, and enums.
```
