# RayV8 Game Toolkit API Handbook

This is the complete user-facing reference for the RayV8 toolkit bundled with
this release. RayV8 runs JavaScript on V8 and exposes a managed game loop plus
the raylib API as JavaScript globals.

> This handbook is generated from RayV8's bundled `raylib_api.json`. The API
> inventory at the end covers every exposed structure, enum, constant, and
> function rather than a hand-selected subset.

## Contents

- [Quick start](#quick-start)
- [The two API layers](#the-two-api-layers)
- [Lifecycle](#lifecycle)
- [RayV8 frame API](#rayv8-frame-api)
  - [State](#state)
  - [Frame and timing](#frame-and-timing)
  - [Keyboard](#keyboard)
  - [Mouse](#mouse)
  - [Controllers](#controllers)
  - [Drawing fields](#drawing-fields)
  - [Managed resources](#managed-resources)
- [Direct raylib API](#direct-raylib-api)
- [Project configuration](#project-configuration)
- [Commands and distribution](#commands-and-distribution)
- [Patterns and examples](#patterns-and-examples)
- [Complete API inventory](#complete-api-inventory)
  - [Structures](#structures)
  - [Callbacks](#callbacks)
  - [Enums](#enums)
  - [Constants](#constants)
  - [Functions by category](#functions-by-category)

## Quick start

```powershell
.\rayv8.exe generate MyGame
.\rayv8.exe run MyGame
```

Edit `MyGame\game\main.js` while it runs. A valid save reloads automatically.
RayV8 does not require Node.js, npm, a browser, or a web server.

```js
/** @param {RayV8Args} args */
function tick(args) {
  const { state, keyboard, dt, solids } = args;
  state.player ??= { x: 400, y: 225 };
  const speed = 260 * dt;

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
}
```

## The two API layers

RayV8 deliberately provides two complementary layers:

1. **The RayV8 frame API** supplies flat access to reload-safe state, timing,
   drawing queues, managed resources, sound operations, and structured input.
2. **The direct raylib API** supplies the complete keyboard, mouse, gamepad,
   touch, gesture, drawing, image, texture, text, 2D, 3D, audio, file, and
   window surface.

The structured input layer covers every raylib keyboard key, mouse button,
gamepad button, and gamepad axis by its enum member name. The exhaustive `KeyboardKey`,
`MouseButton`, `GamepadButton`, `GamepadAxis`, and `Gesture` tables appear in
[Enums](#enums), and all related calls appear in
[Functions by category](#functions-by-category).

## Lifecycle

| Function | Called | Purpose |
|---|---|---|
| `init(options, callback, flags?)` | Once at startup | Apply pre-window flags, create the window, then run initialization |
| `tick(args)` | Every frame | Read input, update state, draw directly, and/or fill output queues |

Calling `init` always creates a window. Omitted window fields default to 1280
by 720 with the title `RayV8`. Flags in the optional third array are combined
and applied before window creation. The callback runs afterward with `args`:

```text
pre-window flags -> window creation -> init callback -> tick each frame -> automatic cleanup
```

```js
init(
  { width: 960, height: 540, title: "My Game" },
  (args) => {
    // The window exists here. Initialize assets with args.resource.
  },
  [ConfigFlags.VSYNC_HINT, ConfigFlags.WINDOW_RESIZABLE]
);
```

`tick` is declared normally and receives `args` each frame:

```js
function tick(args) {
  DrawText("Hello", 24, 24, 32, RAYWHITE);
}
```

RayV8 opens and closes the drawing frame around `tick`, so direct raylib draw
calls and managed output queues can be used in the same function. Cleanup is
automatic. A raw top-level raylib script may instead own its window and loop
without declaring `init` or `tick`.

## RayV8 frame API

`tick(args)` receives this object:

```ts
interface RayV8Args {
  state: Record<string, any>;
  frame: bigint;
  dt: number;
  solids: RayV8Solid[];
  sprites: RayV8Sprite[];
  labels: RayV8Label[];
  background: Color;
  resource: RayV8Resource;
  sound: RayV8SoundController;
  keyboard: RayV8Keyboard;
  mouse: RayV8Mouse;
  controller: RayV8Controller[];
}
```

It is also exposed as global `args`, although accepting the parameter is
clearer and works better with editor type checking.

### State

`args.state` persists between frames and successful hot reloads:

```js
state.score ??= 0;
state.player ??= { x: 100, y: 100, inventory: [] };
```

State must be JSON-like: objects, arrays, strings, numbers, booleans, and
`null`. Do not put functions, cyclic references, native raylib objects,
textures, or sounds in state. Store managed handles in script-level variables
assigned by the `init` callback.

### Frame and timing

`args.frame` is an increasing `bigint` frame counter. `args.dt` is the number
of seconds since the previous frame. Multiply continuous movement by `dt` so
it remains stable at different frame rates:

```js
if (args.keyboard.keyDown("RIGHT")) state.x += 300 * args.dt;
```

### Keyboard

Every [KeyboardKey](#keyboardkey) member is accepted by name:

```js
if (args.keyboard.keyPressed("ENTER")) state.started = true;
if (args.keyboard.keyDown("LEFT_SHIFT")) state.speed = 500;
if (args.keyboard.keyReleased("ESCAPE")) state.menu = true;
if (args.keyboard.keyUp("SPACE")) state.canJump = true;
```

| Method | Meaning |
|---|---|
| `keyPressed(key)` | Key changed from up to down this frame |
| `keyPressedRepeat(key)` | OS key-repeat event occurred |
| `keyDown(key)` | Key is currently held |
| `keyReleased(key)` | Key changed from down to up this frame |
| `keyUp(key)` | Key is currently not held |
| `pressed()` | Consume and return the next pressed key code, or `0` |
| `characterPressed()` | Consume and return the next typed Unicode codepoint, or `0` |
| `keyName(key)` | Return the platform name for a key |

String names are checked. An unknown member such as `"NOT_A_KEY"` throws a
clear range error. Numeric raylib key values are also accepted for advanced use.

### Mouse

`args.mouse` contains `x`, `y`, a `delta` vector, and a two-axis `wheel`
vector. Every [MouseButton](#mousebutton) member is supported:

```js
if (args.mouse.buttonPressed("LEFT")) {
  state.target = { x: args.mouse.x, y: args.mouse.y };
}
if (args.mouse.buttonDown("RIGHT")) { /* held */ }
if (args.mouse.buttonReleased("MIDDLE")) { /* released this frame */ }
```

The button methods are `buttonPressed`, `buttonDown`, `buttonReleased`, and
`buttonUp`. Direct raylib calls remain available for cursor shape, position,
offset, and scale.

### Controllers

`args.controller` is a dense array of currently connected gamepads. Each item
contains its raylib `index`, `name`, `axisCount`, button methods, axis access,
and vibration:

```js
for (const gamepad of args.controller) {
  if (gamepad.buttonPressed("RIGHT_FACE_DOWN")) state.jump = true;
  const x = gamepad.axis("LEFT_X");
  const y = gamepad.axis("LEFT_Y");
  state.player.x += x * 300 * args.dt;
  state.player.y += y * 300 * args.dt;
  // gamepad.vibrate(0.5, 0.5, 0.1);
}
```

Button names come from [GamepadButton](#gamepadbutton); axis names come from
[GamepadAxis](#gamepadaxis). Button methods follow the same pressed, down,
released, and up semantics as the mouse.

Touch and gestures remain available through the direct raylib API.

### Drawing fields

RayV8 creates fresh `sprites`, `solids`, and `labels` arrays before every
`tick`. Re-add everything that should appear that frame. Rendering order is
background -> sprites -> solids -> labels; insertion order is preserved within
each queue.

#### `background`

```ts
background: Color
```

Assign the clear color directly. It defaults to `BLACK` each frame:

```js
args.background = { r: 18, g: 24, b: 38, a: 255 };
```

#### `sprites`

```ts
interface RayV8Sprite {
  texture: string;      // registered resource key
  x: number;
  y: number;
  rotation?: number; // default 0 degrees
  scale?: number;    // default 1
  tint?: Color;      // default WHITE
}
```

`texture` is the key registered with `resource.texture(key, path)`. RayV8
resolves and owns the native texture internally. `x` and `y` locate its
top-left drawing origin. Rotation is in degrees.

#### `solids`

```ts
interface RayV8Solid {
  shape?: "rectangle" | "circle"; // default rectangle
  x: number;
  y: number;
  width?: number;
  height?: number;
  radius?: number;
  color?: Color;                   // default WHITE
}
```

Rectangles use `x`, `y`, `width`, and `height`. Circles use center `x`, center
`y`, and `radius`.

#### `labels`

```ts
interface RayV8Label {
  text: string;
  x: number;
  y: number;
  size?: number;  // default 20
  color?: Color;  // default WHITE
}
```

Labels use raylib's default font. Use direct text functions in `tick` for
custom fonts, spacing, rotation, or mixed draw ordering.

### Managed resources

Textures and sounds declared in the `init` callback are transactionally
managed across hot reloads:

```js
init({}, ({ resource }) => {
  resource.texture("hero", "assets/hero.png");
  resource.sound("jump", "assets/jump.wav");
  const story = resource.file("assets/story.txt");
  const levels = resource.data("assets/levels.json");
});
```

Paths are relative to the entry script. If the entry is `game/main.js`,
`assets/hero.png` resolves to `game/assets/hero.png`.

| API | Returns | Meaning |
|---|---|---|
| `resource.texture(key, path)` | `void` | Register, initialize, or reuse a texture |
| `resource.sound(key, path)` | `void` | Register, initialize, or reuse a sound |
| `resource.file(path)` | `string` | Load a UTF-8 text file |
| `resource.data(path)` | `unknown` | Load and parse a JSON file |

Keys identify managed native assets in game code. Repeating the same key and
normalized path reuses the texture or sound across hot reloads. Changing a
key's path replaces its asset. Keys omitted from the next successful
initialization are unloaded. Resources can only be initialized inside the
`init` callback.
If loading fails, the transaction rolls back and the last working context and
resources continue.

All sound behavior belongs to `args.sound` and addresses the registered key:

| Method | Meaning |
|---|---|
| `sound.play(key, { volume?, pitch?, pan? }?)` | Apply options and start playback |
| `sound.stop(key)` | Stop and rewind |
| `sound.pause(key)` | Pause |
| `sound.resume(key)` | Resume |
| `sound.isPlaying(key)` | Return current playback state |
| `sound.setVolume(key, value)` | Set volume |
| `sound.setPitch(key, value)` | Set pitch |
| `sound.setPan(key, value)` | Set stereo pan |

Sound calls act immediately and are not output-queue entries.

## Direct raylib API

Raylib structures are ordinary JavaScript objects:

```js
const position = { x: 100, y: 80 };                         // Vector2
const bounds = { x: 20, y: 30, width: 200, height: 100 };  // Rectangle
const color = { r: 40, g: 120, b: 220, a: 255 };           // Color
```

Enums are read-only objects. Use their JavaScript member names exactly as
listed in [Enums](#enums):

```js
SetConfigFlags(ConfigFlags.WINDOW_RESIZABLE | ConfigFlags.VSYNC_HINT);
IsKeyDown(KeyboardKey.A);
SetTextureFilter(texture, TextureFilter.BILINEAR);
```

RayV8 binds native pointers as `bigint | null`. Pointer-heavy functions are
low-level and often require memory supplied by another native API. Prefer
managed resources and object-based calls unless you understand the native
contract. Functions taking variadic arguments or native callback types are
registered but intentionally throw because they require a native adapter;
they are marked **Not callable** in the generated function tables.

RayV8 calls `tick` inside the drawing phase. Games may use the `sprites`,
`solids`, and `labels` arrays, call direct drawing functions, or use paired modes such as
`BeginMode2D`/`EndMode2D` directly inside `tick`.

## Project configuration

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

| Property | Meaning |
|---|---|
| `name` | Game/output executable name; defaults to project folder name |
| `entry` | Entry script relative to project root |
| `build.mode` | Default `embed`, `sidecar`, or `loose` mode |
| `build.output` | Relative output path inside the project's `build` folder |
| `build.include` | Relative files/folders copied into the game package |

All paths must stay inside the project. Include every runtime asset. The
default `"game"` includes everything under `game/`, including assets.

## Commands and distribution

```powershell
.\rayv8.exe generate MyGame
.\rayv8.exe run MyGame
.\rayv8.exe build MyGame
.\rayv8.exe build MyGame --embed
.\rayv8.exe build MyGame --sidecar
.\rayv8.exe build MyGame --loose
.\rayv8.exe --help
```

| Mode | Output |
|---|---|
| `embed` | One executable containing the game files |
| `sidecar` | Executable plus a separate `.game.zip` |
| `loose` | Executable beside editable game files |

Packaged games use the Windows GUI subsystem and do not open a console.
Distribute the generated `licenses` directory with the game, or reproduce the
applicable notices in accompanying materials.

## Patterns and examples

### Movement with any WASD or arrow key

```js
function tick(args) {
  const { state, keyboard, dt, solids } = args;
  state.player ??= { x: 400, y: 225 };
  const speed = keyboard.keyDown("LEFT_SHIFT") ? 480 : 240;

  const left = keyboard.keyDown("LEFT") || keyboard.keyDown("A");
  const right = keyboard.keyDown("RIGHT") || keyboard.keyDown("D");
  const up = keyboard.keyDown("UP") || keyboard.keyDown("W");
  const down = keyboard.keyDown("DOWN") || keyboard.keyDown("S");

  if (left) state.player.x -= speed * dt;
  if (right) state.player.x += speed * dt;
  if (up) state.player.y -= speed * dt;
  if (down) state.player.y += speed * dt;

  args.background = BLACK;
  solids.push({ shape: "circle", ...state.player, radius: 20, color: GOLD });
}
```

### Click-to-move

```js
function tick({ state, mouse, solids }) {
  state.target ??= { x: 100, y: 100 };
  if (mouse.buttonPressed("LEFT")) {
    state.target = { x: mouse.x, y: mouse.y };
  }
  solids.push({ shape: "circle", ...state.target, radius: 12, color: RED });
}
```

### Hot-reload-safe asset use

```js
init({}, ({ resource }) => {
  resource.texture("player", "assets/player.png");
  resource.sound("pickup", "assets/pickup.wav");
});

function tick({ state, keyboard, sprites, sound }) {
  state.position ??= { x: 100, y: 100 };
  if (keyboard.keyPressed("SPACE")) sound.play("pickup", { volume: 0.8 });
  sprites.push({ texture: "player", ...state.position });
}
```

## Complete API inventory

The following inventory is generated from the exact raylib definition bundled
with this RayV8 checkout.

## Structures

### Vector2

Vector2, 2 components

| Field | Type | Meaning |
|---|---|---|
| `x` | `number` | Vector x component |
| `y` | `number` | Vector y component |

### Vector3

Vector3, 3 components

| Field | Type | Meaning |
|---|---|---|
| `x` | `number` | Vector x component |
| `y` | `number` | Vector y component |
| `z` | `number` | Vector z component |

### Vector4

Vector4, 4 components

| Field | Type | Meaning |
|---|---|---|
| `x` | `number` | Vector x component |
| `y` | `number` | Vector y component |
| `z` | `number` | Vector z component |
| `w` | `number` | Vector w component |

### Matrix

Matrix, 4x4 components, column major, OpenGL style, right-handed

| Field | Type | Meaning |
|---|---|---|
| `m0` | `number` | Matrix first row (4 components) |
| `m4` | `number` | Matrix first row (4 components) |
| `m8` | `number` | Matrix first row (4 components) |
| `m12` | `number` | Matrix first row (4 components) |
| `m1` | `number` | Matrix second row (4 components) |
| `m5` | `number` | Matrix second row (4 components) |
| `m9` | `number` | Matrix second row (4 components) |
| `m13` | `number` | Matrix second row (4 components) |
| `m2` | `number` | Matrix third row (4 components) |
| `m6` | `number` | Matrix third row (4 components) |
| `m10` | `number` | Matrix third row (4 components) |
| `m14` | `number` | Matrix third row (4 components) |
| `m3` | `number` | Matrix fourth row (4 components) |
| `m7` | `number` | Matrix fourth row (4 components) |
| `m11` | `number` | Matrix fourth row (4 components) |
| `m15` | `number` | Matrix fourth row (4 components) |

### Color

Color, 4 components, R8G8B8A8 (32bit)

| Field | Type | Meaning |
|---|---|---|
| `r` | `number` | Color red value |
| `g` | `number` | Color green value |
| `b` | `number` | Color blue value |
| `a` | `number` | Color alpha value |

### Rectangle

Rectangle, 4 components

| Field | Type | Meaning |
|---|---|---|
| `x` | `number` | Rectangle top-left corner position x |
| `y` | `number` | Rectangle top-left corner position y |
| `width` | `number` | Rectangle width |
| `height` | `number` | Rectangle height |

### Image

Image, pixel data stored in CPU memory (RAM)

| Field | Type | Meaning |
|---|---|---|
| `data` | `bigint | null` | Image raw data |
| `width` | `number` | Image base width |
| `height` | `number` | Image base height |
| `mipmaps` | `number` | Mipmap levels, 1 by default |
| `format` | `number` | Data format (PixelFormat type) |

### Texture

Texture, tex data stored in GPU memory (VRAM)

| Field | Type | Meaning |
|---|---|---|
| `id` | `number` | OpenGL texture id |
| `width` | `number` | Texture base width |
| `height` | `number` | Texture base height |
| `mipmaps` | `number` | Mipmap levels, 1 by default |
| `format` | `number` | Data format (PixelFormat type) |

### RenderTexture

RenderTexture, fbo for texture rendering

| Field | Type | Meaning |
|---|---|---|
| `id` | `number` | OpenGL framebuffer object id |
| `texture` | `Texture` | Color buffer attachment texture |
| `depth` | `Texture` | Depth buffer attachment texture |

### NPatchInfo

NPatchInfo, n-patch layout info

| Field | Type | Meaning |
|---|---|---|
| `source` | `Rectangle` | Texture source rectangle |
| `left` | `number` | Left border offset |
| `top` | `number` | Top border offset |
| `right` | `number` | Right border offset |
| `bottom` | `number` | Bottom border offset |
| `layout` | `number` | Layout of the n-patch: 3x3, 1x3 or 3x1 |

### GlyphInfo

GlyphInfo, font characters glyphs info

| Field | Type | Meaning |
|---|---|---|
| `value` | `number` | Character value (Unicode) |
| `offsetX` | `number` | Character offset X when drawing |
| `offsetY` | `number` | Character offset Y when drawing |
| `advanceX` | `number` | Character advance position X |
| `image` | `Image` | Character image data |

### Font

Font, font texture and GlyphInfo array data

| Field | Type | Meaning |
|---|---|---|
| `baseSize` | `number` | Base size (default chars height) |
| `glyphCount` | `number` | Number of glyph characters |
| `glyphPadding` | `number` | Padding around the glyph characters |
| `texture` | `Texture2D` | Texture atlas containing the glyphs |
| `recs` | `bigint | null` | Rectangles in texture for the glyphs |
| `glyphs` | `bigint | null` | Glyphs info data |

### Camera3D

Camera, defines position/orientation in 3d space

| Field | Type | Meaning |
|---|---|---|
| `position` | `Vector3` | Camera position |
| `target` | `Vector3` | Camera target it looks-at |
| `up` | `Vector3` | Camera up vector (rotation over its axis) |
| `fovy` | `number` | Camera field-of-view aperture in Y (degrees) in perspective, used as near plane height in world units in orthographic |
| `projection` | `number` | Camera projection: CAMERA_PERSPECTIVE or CAMERA_ORTHOGRAPHIC |

### Camera2D

Camera2D, defines position/orientation in 2d space

| Field | Type | Meaning |
|---|---|---|
| `offset` | `Vector2` | Camera offset (screen space offset from window origin) |
| `target` | `Vector2` | Camera target (world space target point that is mapped to screen space offset) |
| `rotation` | `number` | Camera rotation in degrees (pivots around target) |
| `zoom` | `number` | Camera zoom (scaling around target), must not be set to 0, set to 1.0f for no scale |

### Mesh

Mesh, vertex data and vao/vbo

| Field | Type | Meaning |
|---|---|---|
| `vertexCount` | `number` | Number of vertices stored in arrays |
| `triangleCount` | `number` | Number of triangles stored (indexed or not) |
| `vertices` | `bigint | null` | Vertex position (XYZ - 3 components per vertex) (shader-location = 0) |
| `texcoords` | `bigint | null` | Vertex texture coordinates (UV - 2 components per vertex) (shader-location = 1) |
| `texcoords2` | `bigint | null` | Vertex texture second coordinates (UV - 2 components per vertex) (shader-location = 5) |
| `normals` | `bigint | null` | Vertex normals (XYZ - 3 components per vertex) (shader-location = 2) |
| `tangents` | `bigint | null` | Vertex tangents (XYZW - 4 components per vertex) (shader-location = 4) |
| `colors` | `bigint | null` | Vertex colors (RGBA - 4 components per vertex) (shader-location = 3) |
| `indices` | `bigint | null` | Vertex indices (in case vertex data comes indexed) |
| `boneCount` | `number` | Number of bones (MAX: 256 bones) |
| `boneIndices` | `bigint | null` | Vertex bone indices, up to 4 bones influence by vertex (skinning) (shader-location = 6) |
| `boneWeights` | `bigint | null` | Vertex bone weight, up to 4 bones influence by vertex (skinning) (shader-location = 7) |
| `animVertices` | `bigint | null` | Animated vertex positions (after bones transformations) |
| `animNormals` | `bigint | null` | Animated normals (after bones transformations) |
| `vaoId` | `number` | OpenGL Vertex Array Object id |
| `vboId` | `bigint | null` | OpenGL Vertex Buffer Objects id (default vertex data) |

### Shader

Shader

| Field | Type | Meaning |
|---|---|---|
| `id` | `number` | Shader program id |
| `locs` | `bigint | null` | Shader locations array (RL_MAX_SHADER_LOCATIONS) |

### MaterialMap

MaterialMap

| Field | Type | Meaning |
|---|---|---|
| `texture` | `Texture2D` | Material map texture |
| `color` | `Color` | Material map color |
| `value` | `number` | Material map value |

### Material

Material, includes shader and maps

| Field | Type | Meaning |
|---|---|---|
| `shader` | `Shader` | Material shader |
| `maps` | `bigint | null` | Material maps array (MAX_MATERIAL_MAPS) |
| `params` | `Array<number>` | Material generic parameters (if required) |

### Transform

Transform, vertex transformation data

| Field | Type | Meaning |
|---|---|---|
| `translation` | `Vector3` | Translation |
| `rotation` | `Quaternion` | Rotation |
| `scale` | `Vector3` | Scale |

### BoneInfo

Bone, skeletal animation bone

| Field | Type | Meaning |
|---|---|---|
| `name` | `Array<number>` | Bone name |
| `parent` | `number` | Bone parent |

### ModelSkeleton

Skeleton, animation bones hierarchy

| Field | Type | Meaning |
|---|---|---|
| `boneCount` | `number` | Number of bones |
| `bones` | `bigint | null` | Bones information (skeleton) |
| `bindPose` | `ModelAnimPose` | Bones base transformation (Transform[]) |

### Model

Model, meshes, materials and animation data

| Field | Type | Meaning |
|---|---|---|
| `transform` | `Matrix` | Local transform matrix |
| `meshCount` | `number` | Number of meshes |
| `materialCount` | `number` | Number of materials |
| `meshes` | `bigint | null` | Meshes array |
| `materials` | `bigint | null` | Materials array |
| `meshMaterial` | `bigint | null` | Mesh material number |
| `skeleton` | `ModelSkeleton` | Skeleton for animation |
| `currentPose` | `ModelAnimPose` | Current animation pose (Transform[]) |
| `boneMatrices` | `bigint | null` | Bones animated transformation matrices |

### ModelAnimation

ModelAnimation, contains a full animation sequence

| Field | Type | Meaning |
|---|---|---|
| `name` | `Array<number>` | Animation name |
| `boneCount` | `number` | Number of bones (per pose) |
| `keyframeCount` | `number` | Number of animation key frames |
| `keyframePoses` | `bigint | null` | Animation sequence keyframe poses [keyframe][pose] |

### Ray

Ray, ray for raycasting

| Field | Type | Meaning |
|---|---|---|
| `position` | `Vector3` | Ray position (origin) |
| `direction` | `Vector3` | Ray direction (normalized) |

### RayCollision

RayCollision, ray hit information

| Field | Type | Meaning |
|---|---|---|
| `hit` | `boolean` | Did the ray hit something? |
| `distance` | `number` | Distance to the nearest hit |
| `point` | `Vector3` | Point of the nearest hit |
| `normal` | `Vector3` | Surface normal of hit |

### BoundingBox

BoundingBox

| Field | Type | Meaning |
|---|---|---|
| `min` | `Vector3` | Minimum vertex box-corner |
| `max` | `Vector3` | Maximum vertex box-corner |

### Wave

Wave, audio wave data

| Field | Type | Meaning |
|---|---|---|
| `frameCount` | `number` | Total number of frames (considering channels) |
| `sampleRate` | `number` | Frequency (samples per second) |
| `sampleSize` | `number` | Bit depth (bits per sample): 8, 16, 32 (24 not supported) |
| `channels` | `number` | Number of channels (1-mono, 2-stereo, ...) |
| `data` | `bigint | null` | Buffer data pointer |

### AudioStream

AudioStream, custom audio stream

| Field | Type | Meaning |
|---|---|---|
| `buffer` | `bigint | null` | Pointer to internal data used by the audio system |
| `processor` | `bigint | null` | Pointer to internal data processor, useful for audio effects |
| `sampleRate` | `number` | Frequency (samples per second) |
| `sampleSize` | `number` | Bit depth (bits per sample): 8, 16, 32 (24 not supported) |
| `channels` | `number` | Number of channels (1-mono, 2-stereo, ...) |

### Sound

Sound

| Field | Type | Meaning |
|---|---|---|
| `stream` | `AudioStream` | Audio stream |
| `frameCount` | `number` | Total number of frames (considering channels) |

### Music

Music, audio stream, anything longer than ~10 seconds should be streamed

| Field | Type | Meaning |
|---|---|---|
| `stream` | `AudioStream` | Audio stream |
| `frameCount` | `number` | Total number of frames (considering channels) |
| `looping` | `boolean` | Music looping enable |
| `ctxType` | `number` | Type of music context (audio filetype) |
| `ctxData` | `bigint | null` | Audio context data, depends on type |

### VrDeviceInfo

VrDeviceInfo, Head-Mounted-Display device parameters

| Field | Type | Meaning |
|---|---|---|
| `hResolution` | `number` | Horizontal resolution in pixels |
| `vResolution` | `number` | Vertical resolution in pixels |
| `hScreenSize` | `number` | Horizontal size in meters |
| `vScreenSize` | `number` | Vertical size in meters |
| `eyeToScreenDistance` | `number` | Distance between eye and display in meters |
| `lensSeparationDistance` | `number` | Lens separation distance in meters |
| `interpupillaryDistance` | `number` | IPD (distance between pupils) in meters |
| `lensDistortionValues` | `Array<number>` | Lens distortion constant parameters |
| `chromaAbCorrection` | `Array<number>` | Chromatic aberration correction parameters |

### VrStereoConfig

VrStereoConfig, VR stereo rendering configuration for simulator

| Field | Type | Meaning |
|---|---|---|
| `projection` | `Array<Matrix>` | VR projection matrices (per eye) |
| `viewOffset` | `Array<Matrix>` | VR view offset matrices (per eye) |
| `leftLensCenter` | `Array<number>` | VR left lens center |
| `rightLensCenter` | `Array<number>` | VR right lens center |
| `leftScreenCenter` | `Array<number>` | VR left screen center |
| `rightScreenCenter` | `Array<number>` | VR right screen center |
| `scale` | `Array<number>` | VR distortion scale |
| `scaleIn` | `Array<number>` | VR distortion scale in |

### FilePathList

File path list

| Field | Type | Meaning |
|---|---|---|
| `count` | `number` | Filepaths entries count |
| `paths` | `bigint | null` | Filepaths entries |

### AutomationEvent

Automation event

| Field | Type | Meaning |
|---|---|---|
| `frame` | `number` | Event frame |
| `type` | `number` | Event type (AutomationEventType) |
| `params` | `Array<number>` | Event parameters (if required) |

### AutomationEventList

Automation event list

| Field | Type | Meaning |
|---|---|---|
| `capacity` | `number` | Events max entries (MAX_AUTOMATION_EVENTS) |
| `count` | `number` | Events entries count |
| `events` | `bigint | null` | Events entries |

## Callbacks

Callbacks are listed for type completeness. APIs requiring native callbacks are not callable without a native adapter.

- `TraceLogCallback = (logLevel: number, text: string, args: va_list) => void` - Logging: Redirect trace log messages
- `LoadFileDataCallback = (fileName: string, dataSize: bigint | null) => bigint | null` - FileIO: Load binary data
- `SaveFileDataCallback = (fileName: string, data: bigint | null, dataSize: number) => boolean` - FileIO: Save binary data
- `LoadFileTextCallback = (fileName: string) => string` - FileIO: Load text data
- `SaveFileTextCallback = (fileName: string, text: string) => boolean` - FileIO: Save text data
- `AudioCallback = (bufferData: bigint | null, frames: number) => void` - Not documented upstream

## Enums

Enum objects and their members are global and read-only. Values can be combined with `|` where raylib treats an enum as flags.

### ConfigFlags

System/Window config flags

| JavaScript member | Native value | Meaning |
|---|---:|---|
| `ConfigFlags.VSYNC_HINT` | 64 | Set to try enabling V-Sync on GPU |
| `ConfigFlags.FULLSCREEN_MODE` | 2 | Set to run program in fullscreen |
| `ConfigFlags.WINDOW_RESIZABLE` | 4 | Set to allow resizable window |
| `ConfigFlags.WINDOW_UNDECORATED` | 8 | Set to disable window decoration (frame and buttons) |
| `ConfigFlags.WINDOW_HIDDEN` | 128 | Set to hide window |
| `ConfigFlags.WINDOW_MINIMIZED` | 512 | Set to minimize window (iconify) |
| `ConfigFlags.WINDOW_MAXIMIZED` | 1024 | Set to maximize window (expanded to monitor) |
| `ConfigFlags.WINDOW_UNFOCUSED` | 2048 | Set to window non focused |
| `ConfigFlags.WINDOW_TOPMOST` | 4096 | Set to window always on top |
| `ConfigFlags.WINDOW_ALWAYS_RUN` | 256 | Set to allow windows running while minimized |
| `ConfigFlags.WINDOW_TRANSPARENT` | 16 | Set to allow transparent framebuffer |
| `ConfigFlags.WINDOW_HIGHDPI` | 8192 | Set to support HighDPI |
| `ConfigFlags.WINDOW_MOUSE_PASSTHROUGH` | 16384 | Set to support mouse passthrough, only supported when FLAG_WINDOW_UNDECORATED |
| `ConfigFlags.BORDERLESS_WINDOWED_MODE` | 32768 | Set to run program in borderless windowed mode |
| `ConfigFlags.MSAA_4X_HINT` | 32 | Set to try enabling MSAA 4X |
| `ConfigFlags.INTERLACED_HINT` | 65536 | Set to try enabling interlaced video format (for V3D) |

### TraceLogLevel

Trace log level

| JavaScript member | Native value | Meaning |
|---|---:|---|
| `TraceLogLevel.ALL` | 0 | Display all logs |
| `TraceLogLevel.TRACE` | 1 | Trace logging, intended for internal use only |
| `TraceLogLevel.DEBUG` | 2 | Debug logging, used for internal debugging, it should be disabled on release builds |
| `TraceLogLevel.INFO` | 3 | Info logging, used for program execution info |
| `TraceLogLevel.WARNING` | 4 | Warning logging, used on recoverable failures |
| `TraceLogLevel.ERROR` | 5 | Error logging, used on unrecoverable failures |
| `TraceLogLevel.FATAL` | 6 | Fatal logging, used to abort program: exit(EXIT_FAILURE) |
| `TraceLogLevel.NONE` | 7 | Disable logging |

### KeyboardKey

Keyboard keys (US keyboard layout)

| JavaScript member | Native value | Meaning |
|---|---:|---|
| `KeyboardKey.NULL` | 0 | Key: NULL, used for no key pressed |
| `KeyboardKey.APOSTROPHE` | 39 | Key: ' |
| `KeyboardKey.COMMA` | 44 | Key: , |
| `KeyboardKey.MINUS` | 45 | Key: - |
| `KeyboardKey.PERIOD` | 46 | Key: . |
| `KeyboardKey.SLASH` | 47 | Key: / |
| `KeyboardKey.ZERO` | 48 | Key: 0 |
| `KeyboardKey.ONE` | 49 | Key: 1 |
| `KeyboardKey.TWO` | 50 | Key: 2 |
| `KeyboardKey.THREE` | 51 | Key: 3 |
| `KeyboardKey.FOUR` | 52 | Key: 4 |
| `KeyboardKey.FIVE` | 53 | Key: 5 |
| `KeyboardKey.SIX` | 54 | Key: 6 |
| `KeyboardKey.SEVEN` | 55 | Key: 7 |
| `KeyboardKey.EIGHT` | 56 | Key: 8 |
| `KeyboardKey.NINE` | 57 | Key: 9 |
| `KeyboardKey.SEMICOLON` | 59 | Key: ; |
| `KeyboardKey.EQUAL` | 61 | Key: = |
| `KeyboardKey.A` | 65 | Key: A \| a |
| `KeyboardKey.B` | 66 | Key: B \| b |
| `KeyboardKey.C` | 67 | Key: C \| c |
| `KeyboardKey.D` | 68 | Key: D \| d |
| `KeyboardKey.E` | 69 | Key: E \| e |
| `KeyboardKey.F` | 70 | Key: F \| f |
| `KeyboardKey.G` | 71 | Key: G \| g |
| `KeyboardKey.H` | 72 | Key: H \| h |
| `KeyboardKey.I` | 73 | Key: I \| i |
| `KeyboardKey.J` | 74 | Key: J \| j |
| `KeyboardKey.K` | 75 | Key: K \| k |
| `KeyboardKey.L` | 76 | Key: L \| l |
| `KeyboardKey.M` | 77 | Key: M \| m |
| `KeyboardKey.N` | 78 | Key: N \| n |
| `KeyboardKey.O` | 79 | Key: O \| o |
| `KeyboardKey.P` | 80 | Key: P \| p |
| `KeyboardKey.Q` | 81 | Key: Q \| q |
| `KeyboardKey.R` | 82 | Key: R \| r |
| `KeyboardKey.S` | 83 | Key: S \| s |
| `KeyboardKey.T` | 84 | Key: T \| t |
| `KeyboardKey.U` | 85 | Key: U \| u |
| `KeyboardKey.V` | 86 | Key: V \| v |
| `KeyboardKey.W` | 87 | Key: W \| w |
| `KeyboardKey.X` | 88 | Key: X \| x |
| `KeyboardKey.Y` | 89 | Key: Y \| y |
| `KeyboardKey.Z` | 90 | Key: Z \| z |
| `KeyboardKey.LEFT_BRACKET` | 91 | Key: [ |
| `KeyboardKey.BACKSLASH` | 92 | Key: '\' |
| `KeyboardKey.RIGHT_BRACKET` | 93 | Key: ] |
| `KeyboardKey.GRAVE` | 96 | Key: ` |
| `KeyboardKey.SPACE` | 32 | Key: Space |
| `KeyboardKey.ESCAPE` | 256 | Key: Esc |
| `KeyboardKey.ENTER` | 257 | Key: Enter |
| `KeyboardKey.TAB` | 258 | Key: Tab |
| `KeyboardKey.BACKSPACE` | 259 | Key: Backspace |
| `KeyboardKey.INSERT` | 260 | Key: Ins |
| `KeyboardKey.DELETE` | 261 | Key: Del |
| `KeyboardKey.RIGHT` | 262 | Key: Cursor right |
| `KeyboardKey.LEFT` | 263 | Key: Cursor left |
| `KeyboardKey.DOWN` | 264 | Key: Cursor down |
| `KeyboardKey.UP` | 265 | Key: Cursor up |
| `KeyboardKey.PAGE_UP` | 266 | Key: Page up |
| `KeyboardKey.PAGE_DOWN` | 267 | Key: Page down |
| `KeyboardKey.HOME` | 268 | Key: Home |
| `KeyboardKey.END` | 269 | Key: End |
| `KeyboardKey.CAPS_LOCK` | 280 | Key: Caps lock |
| `KeyboardKey.SCROLL_LOCK` | 281 | Key: Scroll down |
| `KeyboardKey.NUM_LOCK` | 282 | Key: Num lock |
| `KeyboardKey.PRINT_SCREEN` | 283 | Key: Print screen |
| `KeyboardKey.PAUSE` | 284 | Key: Pause |
| `KeyboardKey.F1` | 290 | Key: F1 |
| `KeyboardKey.F2` | 291 | Key: F2 |
| `KeyboardKey.F3` | 292 | Key: F3 |
| `KeyboardKey.F4` | 293 | Key: F4 |
| `KeyboardKey.F5` | 294 | Key: F5 |
| `KeyboardKey.F6` | 295 | Key: F6 |
| `KeyboardKey.F7` | 296 | Key: F7 |
| `KeyboardKey.F8` | 297 | Key: F8 |
| `KeyboardKey.F9` | 298 | Key: F9 |
| `KeyboardKey.F10` | 299 | Key: F10 |
| `KeyboardKey.F11` | 300 | Key: F11 |
| `KeyboardKey.F12` | 301 | Key: F12 |
| `KeyboardKey.LEFT_SHIFT` | 340 | Key: Shift left |
| `KeyboardKey.LEFT_CONTROL` | 341 | Key: Control left |
| `KeyboardKey.LEFT_ALT` | 342 | Key: Alt left |
| `KeyboardKey.LEFT_SUPER` | 343 | Key: Super left |
| `KeyboardKey.RIGHT_SHIFT` | 344 | Key: Shift right |
| `KeyboardKey.RIGHT_CONTROL` | 345 | Key: Control right |
| `KeyboardKey.RIGHT_ALT` | 346 | Key: Alt right |
| `KeyboardKey.RIGHT_SUPER` | 347 | Key: Super right |
| `KeyboardKey.KB_MENU` | 348 | Key: KB menu |
| `KeyboardKey.KP_0` | 320 | Key: Keypad 0 |
| `KeyboardKey.KP_1` | 321 | Key: Keypad 1 |
| `KeyboardKey.KP_2` | 322 | Key: Keypad 2 |
| `KeyboardKey.KP_3` | 323 | Key: Keypad 3 |
| `KeyboardKey.KP_4` | 324 | Key: Keypad 4 |
| `KeyboardKey.KP_5` | 325 | Key: Keypad 5 |
| `KeyboardKey.KP_6` | 326 | Key: Keypad 6 |
| `KeyboardKey.KP_7` | 327 | Key: Keypad 7 |
| `KeyboardKey.KP_8` | 328 | Key: Keypad 8 |
| `KeyboardKey.KP_9` | 329 | Key: Keypad 9 |
| `KeyboardKey.KP_DECIMAL` | 330 | Key: Keypad . |
| `KeyboardKey.KP_DIVIDE` | 331 | Key: Keypad / |
| `KeyboardKey.KP_MULTIPLY` | 332 | Key: Keypad * |
| `KeyboardKey.KP_SUBTRACT` | 333 | Key: Keypad - |
| `KeyboardKey.KP_ADD` | 334 | Key: Keypad + |
| `KeyboardKey.KP_ENTER` | 335 | Key: Keypad Enter |
| `KeyboardKey.KP_EQUAL` | 336 | Key: Keypad = |
| `KeyboardKey.BACK` | 4 | Key: Android back button |
| `KeyboardKey.MENU` | 5 | Key: Android menu button |
| `KeyboardKey.VOLUME_UP` | 24 | Key: Android volume up button |
| `KeyboardKey.VOLUME_DOWN` | 25 | Key: Android volume down button |

### MouseButton

Mouse buttons

| JavaScript member | Native value | Meaning |
|---|---:|---|
| `MouseButton.LEFT` | 0 | Mouse button left |
| `MouseButton.RIGHT` | 1 | Mouse button right |
| `MouseButton.MIDDLE` | 2 | Mouse button middle (pressed wheel) |
| `MouseButton.SIDE` | 3 | Mouse button side (advanced mouse device) |
| `MouseButton.EXTRA` | 4 | Mouse button extra (advanced mouse device) |
| `MouseButton.FORWARD` | 5 | Mouse button forward (advanced mouse device) |
| `MouseButton.BACK` | 6 | Mouse button back (advanced mouse device) |

### MouseCursor

Mouse cursor

| JavaScript member | Native value | Meaning |
|---|---:|---|
| `MouseCursor.DEFAULT` | 0 | Default pointer shape |
| `MouseCursor.ARROW` | 1 | Arrow shape |
| `MouseCursor.IBEAM` | 2 | Text writing cursor shape |
| `MouseCursor.CROSSHAIR` | 3 | Cross shape |
| `MouseCursor.POINTING_HAND` | 4 | Pointing hand cursor |
| `MouseCursor.RESIZE_EW` | 5 | Horizontal resize/move arrow shape |
| `MouseCursor.RESIZE_NS` | 6 | Vertical resize/move arrow shape |
| `MouseCursor.RESIZE_NWSE` | 7 | Top-left to bottom-right diagonal resize/move arrow shape |
| `MouseCursor.RESIZE_NESW` | 8 | The top-right to bottom-left diagonal resize/move arrow shape |
| `MouseCursor.RESIZE_ALL` | 9 | The omnidirectional resize/move cursor shape |
| `MouseCursor.NOT_ALLOWED` | 10 | The operation-not-allowed shape |

### GamepadButton

Gamepad buttons

| JavaScript member | Native value | Meaning |
|---|---:|---|
| `GamepadButton.UNKNOWN` | 0 | Unknown button, for error checking |
| `GamepadButton.LEFT_FACE_UP` | 1 | Gamepad left DPAD up button |
| `GamepadButton.LEFT_FACE_RIGHT` | 2 | Gamepad left DPAD right button |
| `GamepadButton.LEFT_FACE_DOWN` | 3 | Gamepad left DPAD down button |
| `GamepadButton.LEFT_FACE_LEFT` | 4 | Gamepad left DPAD left button |
| `GamepadButton.RIGHT_FACE_UP` | 5 | Gamepad right button up (i.e. PS3: Triangle, Xbox: Y) |
| `GamepadButton.RIGHT_FACE_RIGHT` | 6 | Gamepad right button right (i.e. PS3: Circle, Xbox: B) |
| `GamepadButton.RIGHT_FACE_DOWN` | 7 | Gamepad right button down (i.e. PS3: Cross, Xbox: A) |
| `GamepadButton.RIGHT_FACE_LEFT` | 8 | Gamepad right button left (i.e. PS3: Square, Xbox: X) |
| `GamepadButton.LEFT_TRIGGER_1` | 9 | Gamepad top/back trigger left (first), it could be a trailing button |
| `GamepadButton.LEFT_TRIGGER_2` | 10 | Gamepad top/back trigger left (second), it could be a trailing button |
| `GamepadButton.RIGHT_TRIGGER_1` | 11 | Gamepad top/back trigger right (first), it could be a trailing button |
| `GamepadButton.RIGHT_TRIGGER_2` | 12 | Gamepad top/back trigger right (second), it could be a trailing button |
| `GamepadButton.MIDDLE_LEFT` | 13 | Gamepad center buttons, left one (i.e. PS3: Select) |
| `GamepadButton.MIDDLE` | 14 | Gamepad center buttons, middle one (i.e. PS3: PS, Xbox: XBOX) |
| `GamepadButton.MIDDLE_RIGHT` | 15 | Gamepad center buttons, right one (i.e. PS3: Start) |
| `GamepadButton.LEFT_THUMB` | 16 | Gamepad joystick pressed button left |
| `GamepadButton.RIGHT_THUMB` | 17 | Gamepad joystick pressed button right |

### GamepadAxis

Gamepad axes

| JavaScript member | Native value | Meaning |
|---|---:|---|
| `GamepadAxis.LEFT_X` | 0 | Gamepad left stick X axis |
| `GamepadAxis.LEFT_Y` | 1 | Gamepad left stick Y axis |
| `GamepadAxis.RIGHT_X` | 2 | Gamepad right stick X axis |
| `GamepadAxis.RIGHT_Y` | 3 | Gamepad right stick Y axis |
| `GamepadAxis.LEFT_TRIGGER` | 4 | Gamepad back trigger left, pressure level: [1..-1] |
| `GamepadAxis.RIGHT_TRIGGER` | 5 | Gamepad back trigger right, pressure level: [1..-1] |

### MaterialMapIndex

Material map index

| JavaScript member | Native value | Meaning |
|---|---:|---|
| `MaterialMapIndex.ALBEDO` | 0 | Albedo material (same as: MATERIAL_MAP_DIFFUSE) |
| `MaterialMapIndex.METALNESS` | 1 | Metalness material (same as: MATERIAL_MAP_SPECULAR) |
| `MaterialMapIndex.NORMAL` | 2 | Normal material |
| `MaterialMapIndex.ROUGHNESS` | 3 | Roughness material |
| `MaterialMapIndex.OCCLUSION` | 4 | Ambient occlusion material |
| `MaterialMapIndex.EMISSION` | 5 | Emission material |
| `MaterialMapIndex.HEIGHT` | 6 | Heightmap material |
| `MaterialMapIndex.CUBEMAP` | 7 | Cubemap material (NOTE: Uses GL_TEXTURE_CUBE_MAP) |
| `MaterialMapIndex.IRRADIANCE` | 8 | Irradiance material (NOTE: Uses GL_TEXTURE_CUBE_MAP) |
| `MaterialMapIndex.PREFILTER` | 9 | Prefilter material (NOTE: Uses GL_TEXTURE_CUBE_MAP) |
| `MaterialMapIndex.BRDF` | 10 | Brdf material |

### ShaderLocationIndex

Shader location index

| JavaScript member | Native value | Meaning |
|---|---:|---|
| `ShaderLocationIndex.VERTEX_POSITION` | 0 | Shader location: vertex attribute: position |
| `ShaderLocationIndex.VERTEX_TEXCOORD01` | 1 | Shader location: vertex attribute: texcoord01 |
| `ShaderLocationIndex.VERTEX_TEXCOORD02` | 2 | Shader location: vertex attribute: texcoord02 |
| `ShaderLocationIndex.VERTEX_NORMAL` | 3 | Shader location: vertex attribute: normal |
| `ShaderLocationIndex.VERTEX_TANGENT` | 4 | Shader location: vertex attribute: tangent |
| `ShaderLocationIndex.VERTEX_COLOR` | 5 | Shader location: vertex attribute: color |
| `ShaderLocationIndex.MATRIX_MVP` | 6 | Shader location: matrix uniform: model-view-projection |
| `ShaderLocationIndex.MATRIX_VIEW` | 7 | Shader location: matrix uniform: view (camera transform) |
| `ShaderLocationIndex.MATRIX_PROJECTION` | 8 | Shader location: matrix uniform: projection |
| `ShaderLocationIndex.MATRIX_MODEL` | 9 | Shader location: matrix uniform: model (transform) |
| `ShaderLocationIndex.MATRIX_NORMAL` | 10 | Shader location: matrix uniform: normal |
| `ShaderLocationIndex.VECTOR_VIEW` | 11 | Shader location: vector uniform: view |
| `ShaderLocationIndex.COLOR_DIFFUSE` | 12 | Shader location: vector uniform: diffuse color |
| `ShaderLocationIndex.COLOR_SPECULAR` | 13 | Shader location: vector uniform: specular color |
| `ShaderLocationIndex.COLOR_AMBIENT` | 14 | Shader location: vector uniform: ambient color |
| `ShaderLocationIndex.MAP_ALBEDO` | 15 | Shader location: sampler2d texture: albedo (same as: SHADER_LOC_MAP_DIFFUSE) |
| `ShaderLocationIndex.MAP_METALNESS` | 16 | Shader location: sampler2d texture: metalness (same as: SHADER_LOC_MAP_SPECULAR) |
| `ShaderLocationIndex.MAP_NORMAL` | 17 | Shader location: sampler2d texture: normal |
| `ShaderLocationIndex.MAP_ROUGHNESS` | 18 | Shader location: sampler2d texture: roughness |
| `ShaderLocationIndex.MAP_OCCLUSION` | 19 | Shader location: sampler2d texture: occlusion |
| `ShaderLocationIndex.MAP_EMISSION` | 20 | Shader location: sampler2d texture: emission |
| `ShaderLocationIndex.MAP_HEIGHT` | 21 | Shader location: sampler2d texture: heightmap |
| `ShaderLocationIndex.MAP_CUBEMAP` | 22 | Shader location: samplerCube texture: cubemap |
| `ShaderLocationIndex.MAP_IRRADIANCE` | 23 | Shader location: samplerCube texture: irradiance |
| `ShaderLocationIndex.MAP_PREFILTER` | 24 | Shader location: samplerCube texture: prefilter |
| `ShaderLocationIndex.MAP_BRDF` | 25 | Shader location: sampler2d texture: brdf |
| `ShaderLocationIndex.VERTEX_BONEIDS` | 26 | Shader location: vertex attribute: bone indices |
| `ShaderLocationIndex.VERTEX_BONEWEIGHTS` | 27 | Shader location: vertex attribute: bone weights |
| `ShaderLocationIndex.MATRIX_BONETRANSFORMS` | 28 | Shader location: matrix attribute: bone transforms (animation) |
| `ShaderLocationIndex.VERTEX_INSTANCETRANSFORM` | 29 | Shader location: vertex attribute: instance transforms |

### ShaderUniformDataType

Shader uniform data type

| JavaScript member | Native value | Meaning |
|---|---:|---|
| `ShaderUniformDataType.FLOAT` | 0 | Shader uniform type: float |
| `ShaderUniformDataType.VEC2` | 1 | Shader uniform type: vec2 (2 float) |
| `ShaderUniformDataType.VEC3` | 2 | Shader uniform type: vec3 (3 float) |
| `ShaderUniformDataType.VEC4` | 3 | Shader uniform type: vec4 (4 float) |
| `ShaderUniformDataType.INT` | 4 | Shader uniform type: int |
| `ShaderUniformDataType.IVEC2` | 5 | Shader uniform type: ivec2 (2 int) |
| `ShaderUniformDataType.IVEC3` | 6 | Shader uniform type: ivec3 (3 int) |
| `ShaderUniformDataType.IVEC4` | 7 | Shader uniform type: ivec4 (4 int) |
| `ShaderUniformDataType.UINT` | 8 | Shader uniform type: unsigned int |
| `ShaderUniformDataType.UIVEC2` | 9 | Shader uniform type: uivec2 (2 unsigned int) |
| `ShaderUniformDataType.UIVEC3` | 10 | Shader uniform type: uivec3 (3 unsigned int) |
| `ShaderUniformDataType.UIVEC4` | 11 | Shader uniform type: uivec4 (4 unsigned int) |
| `ShaderUniformDataType.SAMPLER2D` | 12 | Shader uniform type: sampler2d |

### ShaderAttributeDataType

Shader attribute data types

| JavaScript member | Native value | Meaning |
|---|---:|---|
| `ShaderAttributeDataType.FLOAT` | 0 | Shader attribute type: float |
| `ShaderAttributeDataType.VEC2` | 1 | Shader attribute type: vec2 (2 float) |
| `ShaderAttributeDataType.VEC3` | 2 | Shader attribute type: vec3 (3 float) |
| `ShaderAttributeDataType.VEC4` | 3 | Shader attribute type: vec4 (4 float) |

### PixelFormat

Pixel formats

| JavaScript member | Native value | Meaning |
|---|---:|---|
| `PixelFormat.UNCOMPRESSED_GRAYSCALE` | 1 | 8 bit per pixel (no alpha) |
| `PixelFormat.UNCOMPRESSED_GRAY_ALPHA` | 2 | 8*2 bpp (2 channels) |
| `PixelFormat.UNCOMPRESSED_R5G6B5` | 3 | 16 bpp |
| `PixelFormat.UNCOMPRESSED_R8G8B8` | 4 | 24 bpp |
| `PixelFormat.UNCOMPRESSED_R5G5B5A1` | 5 | 16 bpp (1 bit alpha) |
| `PixelFormat.UNCOMPRESSED_R4G4B4A4` | 6 | 16 bpp (4 bit alpha) |
| `PixelFormat.UNCOMPRESSED_R8G8B8A8` | 7 | 32 bpp |
| `PixelFormat.UNCOMPRESSED_R32` | 8 | 32 bpp (1 channel - float) |
| `PixelFormat.UNCOMPRESSED_R32G32B32` | 9 | 32*3 bpp (3 channels - float) |
| `PixelFormat.UNCOMPRESSED_R32G32B32A32` | 10 | 32*4 bpp (4 channels - float) |
| `PixelFormat.UNCOMPRESSED_R16` | 11 | 16 bpp (1 channel - half float) |
| `PixelFormat.UNCOMPRESSED_R16G16B16` | 12 | 16*3 bpp (3 channels - half float) |
| `PixelFormat.UNCOMPRESSED_R16G16B16A16` | 13 | 16*4 bpp (4 channels - half float) |
| `PixelFormat.COMPRESSED_DXT1_RGB` | 14 | 4 bpp (no alpha) |
| `PixelFormat.COMPRESSED_DXT1_RGBA` | 15 | 4 bpp (1 bit alpha) |
| `PixelFormat.COMPRESSED_DXT3_RGBA` | 16 | 8 bpp |
| `PixelFormat.COMPRESSED_DXT5_RGBA` | 17 | 8 bpp |
| `PixelFormat.COMPRESSED_ETC1_RGB` | 18 | 4 bpp |
| `PixelFormat.COMPRESSED_ETC2_RGB` | 19 | 4 bpp |
| `PixelFormat.COMPRESSED_ETC2_EAC_RGBA` | 20 | 8 bpp |
| `PixelFormat.COMPRESSED_PVRT_RGB` | 21 | 4 bpp |
| `PixelFormat.COMPRESSED_PVRT_RGBA` | 22 | 4 bpp |
| `PixelFormat.COMPRESSED_ASTC_4x4_RGBA` | 23 | 8 bpp |
| `PixelFormat.COMPRESSED_ASTC_8x8_RGBA` | 24 | 2 bpp |

### TextureFilter

Texture parameters: filter mode

| JavaScript member | Native value | Meaning |
|---|---:|---|
| `TextureFilter.POINT` | 0 | No filter, pixel approximation |
| `TextureFilter.BILINEAR` | 1 | Linear filtering |
| `TextureFilter.TRILINEAR` | 2 | Trilinear filtering (linear with mipmaps) |
| `TextureFilter.ANISOTROPIC_4X` | 3 | Anisotropic filtering 4x |
| `TextureFilter.ANISOTROPIC_8X` | 4 | Anisotropic filtering 8x |
| `TextureFilter.ANISOTROPIC_16X` | 5 | Anisotropic filtering 16x |

### TextureWrap

Texture parameters: wrap mode

| JavaScript member | Native value | Meaning |
|---|---:|---|
| `TextureWrap.REPEAT` | 0 | Repeats texture in tiled mode |
| `TextureWrap.CLAMP` | 1 | Clamps texture to edge pixel in tiled mode |
| `TextureWrap.MIRROR_REPEAT` | 2 | Mirrors and repeats the texture in tiled mode |
| `TextureWrap.MIRROR_CLAMP` | 3 | Mirrors and clamps to border the texture in tiled mode |

### CubemapLayout

Cubemap layouts

| JavaScript member | Native value | Meaning |
|---|---:|---|
| `CubemapLayout.AUTO_DETECT` | 0 | Automatically detect layout type |
| `CubemapLayout.LINE_VERTICAL` | 1 | Layout is defined by a vertical line with faces |
| `CubemapLayout.LINE_HORIZONTAL` | 2 | Layout is defined by a horizontal line with faces |
| `CubemapLayout.CROSS_THREE_BY_FOUR` | 3 | Layout is defined by a 3x4 cross with cubemap faces |
| `CubemapLayout.CROSS_FOUR_BY_THREE` | 4 | Layout is defined by a 4x3 cross with cubemap faces |

### FontType

Font type, defines generation method

| JavaScript member | Native value | Meaning |
|---|---:|---|
| `FontType.DEFAULT` | 0 | Default font generation, anti-aliased |
| `FontType.BITMAP` | 1 | Bitmap font generation, no anti-aliasing |
| `FontType.SDF` | 2 | SDF font generation, requires external shader |

### BlendMode

Color blending modes (pre-defined)

| JavaScript member | Native value | Meaning |
|---|---:|---|
| `BlendMode.ALPHA` | 0 | Blend textures considering alpha (default) |
| `BlendMode.ADDITIVE` | 1 | Blend textures adding colors |
| `BlendMode.MULTIPLIED` | 2 | Blend textures multiplying colors |
| `BlendMode.ADD_COLORS` | 3 | Blend textures adding colors (alternative) |
| `BlendMode.SUBTRACT_COLORS` | 4 | Blend textures subtracting colors (alternative) |
| `BlendMode.ALPHA_PREMULTIPLY` | 5 | Blend premultiplied textures considering alpha |
| `BlendMode.CUSTOM` | 6 | Blend textures using custom src/dst factors (use rlSetBlendFactors()) |
| `BlendMode.CUSTOM_SEPARATE` | 7 | Blend textures using custom rgb/alpha separate src/dst factors (use rlSetBlendFactorsSeparate()) |

### Gesture

Gesture

| JavaScript member | Native value | Meaning |
|---|---:|---|
| `Gesture.NONE` | 0 | No gesture |
| `Gesture.TAP` | 1 | Tap gesture |
| `Gesture.DOUBLETAP` | 2 | Double tap gesture |
| `Gesture.HOLD` | 4 | Hold gesture |
| `Gesture.DRAG` | 8 | Drag gesture |
| `Gesture.SWIPE_RIGHT` | 16 | Swipe right gesture |
| `Gesture.SWIPE_LEFT` | 32 | Swipe left gesture |
| `Gesture.SWIPE_UP` | 64 | Swipe up gesture |
| `Gesture.SWIPE_DOWN` | 128 | Swipe down gesture |
| `Gesture.PINCH_IN` | 256 | Pinch in gesture |
| `Gesture.PINCH_OUT` | 512 | Pinch out gesture |

### CameraMode

Camera system modes

| JavaScript member | Native value | Meaning |
|---|---:|---|
| `CameraMode.CUSTOM` | 0 | Camera custom, controlled by user (UpdateCamera() does nothing) |
| `CameraMode.FREE` | 1 | Camera free mode |
| `CameraMode.ORBITAL` | 2 | Camera orbital, around target, zoom supported |
| `CameraMode.FIRST_PERSON` | 3 | Camera first person |
| `CameraMode.THIRD_PERSON` | 4 | Camera third person |

### CameraProjection

Camera projection

| JavaScript member | Native value | Meaning |
|---|---:|---|
| `CameraProjection.PERSPECTIVE` | 0 | Perspective projection |
| `CameraProjection.ORTHOGRAPHIC` | 1 | Orthographic projection |

### NPatchLayout

N-patch layout

| JavaScript member | Native value | Meaning |
|---|---:|---|
| `NPatchLayout.NINE_PATCH` | 0 | Npatch layout: 3x3 tiles |
| `NPatchLayout.THREE_PATCH_VERTICAL` | 1 | Npatch layout: 1x3 tiles |
| `NPatchLayout.THREE_PATCH_HORIZONTAL` | 2 | Npatch layout: 3x1 tiles |

## Constants

| Constant | Type | Value | Meaning |
|---|---|---|---|
| `RAYLIB_VERSION_MAJOR` | `number` | `6` | Not documented upstream |
| `RAYLIB_VERSION_MINOR` | `number` | `1` | Not documented upstream |
| `RAYLIB_VERSION_PATCH` | `number` | `0` | Not documented upstream |
| `RAYLIB_VERSION` | `string` | `6.1-dev` | Not documented upstream |
| `PI` | `number` | `3.14159265358979323846` | Not documented upstream |
| `DEG2RAD` | `number` | `(PI/180.0f)` | Not documented upstream |
| `RAD2DEG` | `number` | `(180.0f/PI)` | Not documented upstream |
| `LIGHTGRAY` | `Color` | `CLITERAL(Color){ 200, 200, 200, 255 }` | Light Gray |
| `GRAY` | `Color` | `CLITERAL(Color){ 130, 130, 130, 255 }` | Gray |
| `DARKGRAY` | `Color` | `CLITERAL(Color){ 80, 80, 80, 255 }` | Dark Gray |
| `YELLOW` | `Color` | `CLITERAL(Color){ 253, 249, 0, 255 }` | Yellow |
| `GOLD` | `Color` | `CLITERAL(Color){ 255, 203, 0, 255 }` | Gold |
| `ORANGE` | `Color` | `CLITERAL(Color){ 255, 161, 0, 255 }` | Orange |
| `PINK` | `Color` | `CLITERAL(Color){ 255, 109, 194, 255 }` | Pink |
| `RED` | `Color` | `CLITERAL(Color){ 230, 41, 55, 255 }` | Red |
| `MAROON` | `Color` | `CLITERAL(Color){ 190, 33, 55, 255 }` | Maroon |
| `GREEN` | `Color` | `CLITERAL(Color){ 0, 228, 48, 255 }` | Green |
| `LIME` | `Color` | `CLITERAL(Color){ 0, 158, 47, 255 }` | Lime |
| `DARKGREEN` | `Color` | `CLITERAL(Color){ 0, 117, 44, 255 }` | Dark Green |
| `SKYBLUE` | `Color` | `CLITERAL(Color){ 102, 191, 255, 255 }` | Sky Blue |
| `BLUE` | `Color` | `CLITERAL(Color){ 0, 121, 241, 255 }` | Blue |
| `DARKBLUE` | `Color` | `CLITERAL(Color){ 0, 82, 172, 255 }` | Dark Blue |
| `PURPLE` | `Color` | `CLITERAL(Color){ 200, 122, 255, 255 }` | Purple |
| `VIOLET` | `Color` | `CLITERAL(Color){ 135, 60, 190, 255 }` | Violet |
| `DARKPURPLE` | `Color` | `CLITERAL(Color){ 112, 31, 126, 255 }` | Dark Purple |
| `BEIGE` | `Color` | `CLITERAL(Color){ 211, 176, 131, 255 }` | Beige |
| `BROWN` | `Color` | `CLITERAL(Color){ 127, 106, 79, 255 }` | Brown |
| `DARKBROWN` | `Color` | `CLITERAL(Color){ 76, 63, 47, 255 }` | Dark Brown |
| `WHITE` | `Color` | `CLITERAL(Color){ 255, 255, 255, 255 }` | White |
| `BLACK` | `Color` | `CLITERAL(Color){ 0, 0, 0, 255 }` | Black |
| `BLANK` | `Color` | `CLITERAL(Color){ 0, 0, 0, 0 }` | Blank (Transparent) |
| `MAGENTA` | `Color` | `CLITERAL(Color){ 255, 0, 255, 255 }` | Magenta |
| `RAYWHITE` | `Color` | `CLITERAL(Color){ 245, 245, 245, 255 }` | My own White (raylib logo) |

## Functions by category

Signatures use the JavaScript types exposed by RayV8. `bigint | null` represents a native pointer.

### Window and monitor

| Function | Returns | Availability | Meaning |
|---|---|---|---|
| `InitWindow(width: number, height: number, title: string)` | `void` | Callable | Initialize window and OpenGL context |
| `CloseWindow()` | `void` | Callable | Close window and unload OpenGL context |
| `WindowShouldClose()` | `boolean` | Callable | Check if application should close (KEY_ESCAPE pressed or windows close icon clicked) |
| `IsWindowReady()` | `boolean` | Callable | Check if window has been initialized successfully |
| `IsWindowFullscreen()` | `boolean` | Callable | Check if window is currently fullscreen |
| `IsWindowHidden()` | `boolean` | Callable | Check if window is currently hidden |
| `IsWindowMinimized()` | `boolean` | Callable | Check if window is currently minimized |
| `IsWindowMaximized()` | `boolean` | Callable | Check if window is currently maximized |
| `IsWindowFocused()` | `boolean` | Callable | Check if window is currently focused |
| `IsWindowResized()` | `boolean` | Callable | Check if window has been resized last frame |
| `IsWindowState(flag: number)` | `boolean` | Callable | Check if one specific window flag is enabled |
| `SetWindowState(flags: number)` | `void` | Callable | Set window configuration state using flags |
| `ClearWindowState(flags: number)` | `void` | Callable | Clear window configuration state flags |
| `ToggleFullscreen()` | `void` | Callable | Toggle window state: fullscreen/windowed, resizes monitor to match window resolution |
| `ToggleBorderlessWindowed()` | `void` | Callable | Toggle window state: borderless windowed, resizes window to match monitor resolution |
| `MaximizeWindow()` | `void` | Callable | Set window state: maximized, if resizable |
| `MinimizeWindow()` | `void` | Callable | Set window state: minimized, if resizable |
| `RestoreWindow()` | `void` | Callable | Restore window from being minimized/maximized |
| `SetWindowIcon(image: Image)` | `void` | Callable | Set icon for window (single image, RGBA 32bit) |
| `SetWindowIcons(images: bigint | null, count: number)` | `void` | Callable | Set icon for window (multiple images, RGBA 32bit) |
| `SetWindowTitle(title: string)` | `void` | Callable | Set title for window |
| `SetWindowPosition(x: number, y: number)` | `void` | Callable | Set window position on screen |
| `SetWindowMonitor(monitor: number)` | `void` | Callable | Set monitor for the current window |
| `SetWindowMinSize(width: number, height: number)` | `void` | Callable | Set window minimum dimensions (for FLAG_WINDOW_RESIZABLE) |
| `SetWindowMaxSize(width: number, height: number)` | `void` | Callable | Set window maximum dimensions (for FLAG_WINDOW_RESIZABLE) |
| `SetWindowSize(width: number, height: number)` | `void` | Callable | Set window dimensions |
| `SetWindowOpacity(opacity: number)` | `void` | Callable | Set window opacity [0.0f..1.0f] |
| `SetWindowFocused()` | `void` | Callable | Set window focused |
| `GetWindowHandle()` | `bigint | null` | Callable | Get native window handle |
| `GetScreenWidth()` | `number` | Callable | Get current screen width |
| `GetScreenHeight()` | `number` | Callable | Get current screen height |
| `GetRenderWidth()` | `number` | Callable | Get current render width (it considers HiDPI) |
| `GetRenderHeight()` | `number` | Callable | Get current render height (it considers HiDPI) |
| `GetMonitorCount()` | `number` | Callable | Get number of connected monitors |
| `GetCurrentMonitor()` | `number` | Callable | Get current monitor where window is placed |
| `GetMonitorPosition(monitor: number)` | `Vector2` | Callable | Get specified monitor position |
| `GetMonitorWidth(monitor: number)` | `number` | Callable | Get specified monitor width (current video mode used by monitor) |
| `GetMonitorHeight(monitor: number)` | `number` | Callable | Get specified monitor height (current video mode used by monitor) |
| `GetMonitorPhysicalWidth(monitor: number)` | `number` | Callable | Get specified monitor physical width in millimetres |
| `GetMonitorPhysicalHeight(monitor: number)` | `number` | Callable | Get specified monitor physical height in millimetres |
| `GetMonitorRefreshRate(monitor: number)` | `number` | Callable | Get specified monitor refresh rate |
| `GetWindowPosition()` | `Vector2` | Callable | Get window position XY on monitor |
| `GetWindowScaleDPI()` | `Vector2` | Callable | Get window scale DPI factor |
| `GetMonitorName(monitor: number)` | `string` | Callable | Get the human-readable, UTF-8 encoded name of the specified monitor |
| `SetClipboardText(text: string)` | `void` | Callable | Set clipboard text content |
| `GetClipboardText()` | `string` | Callable | Get clipboard text content |
| `GetClipboardImage()` | `Image` | Callable | Get clipboard image content |
| `EnableEventWaiting()` | `void` | Callable | Enable waiting for events on EndDrawing(), no automatic event polling |
| `DisableEventWaiting()` | `void` | Callable | Disable waiting for events on EndDrawing(), automatic events polling |

### Cursor

| Function | Returns | Availability | Meaning |
|---|---|---|---|
| `ShowCursor()` | `void` | Callable | Show cursor |
| `HideCursor()` | `void` | Callable | Hide cursor |
| `IsCursorHidden()` | `boolean` | Callable | Check if cursor is not visible |
| `EnableCursor()` | `void` | Callable | Enable cursor (unlock cursor) |
| `DisableCursor()` | `void` | Callable | Disable cursor (lock cursor) |
| `IsCursorOnScreen()` | `boolean` | Callable | Check if cursor is on the screen |

### Drawing context, cameras, shaders, and VR

| Function | Returns | Availability | Meaning |
|---|---|---|---|
| `ClearBackground(color: Color)` | `void` | Callable | Clear background (framebuffer) to color |
| `BeginDrawing()` | `void` | Callable | Begin canvas (framebuffer) drawing |
| `EndDrawing()` | `void` | Callable | End canvas (framebuffer) drawing and swap buffers (double buffering) |
| `BeginMode2D(camera: Camera2D)` | `void` | Callable | Begin 2D mode with custom camera (2D) |
| `EndMode2D()` | `void` | Callable | End 2D mode with custom camera |
| `BeginMode3D(camera: Camera3D)` | `void` | Callable | Begin 3D mode with custom camera (3D) |
| `EndMode3D()` | `void` | Callable | End 3D mode and returns to default 2D orthographic mode |
| `BeginTextureMode(target: RenderTexture2D)` | `void` | Callable | Begin drawing to render texture |
| `EndTextureMode()` | `void` | Callable | End drawing to render texture |
| `BeginShaderMode(shader: Shader)` | `void` | Callable | Begin custom shader drawing |
| `EndShaderMode()` | `void` | Callable | End custom shader drawing (use default shader) |
| `BeginBlendMode(mode: number)` | `void` | Callable | Begin blending mode (alpha, additive, multiplied, subtract, custom) |
| `EndBlendMode()` | `void` | Callable | End blending mode (reset to default: alpha blending) |
| `BeginScissorMode(x: number, y: number, width: number, height: number)` | `void` | Callable | Begin scissor mode (define screen area for following drawing) |
| `EndScissorMode()` | `void` | Callable | End scissor mode |
| `BeginVrStereoMode(config: VrStereoConfig)` | `void` | Callable | Begin stereo rendering (requires VR simulator) |
| `EndVrStereoMode()` | `void` | Callable | End stereo rendering (requires VR simulator) |
| `LoadVrStereoConfig(device: VrDeviceInfo)` | `VrStereoConfig` | Callable | Load VR stereo config for VR simulator device parameters |
| `UnloadVrStereoConfig(config: VrStereoConfig)` | `void` | Callable | Unload VR stereo config |
| `LoadShader(vsFileName: string, fsFileName: string)` | `Shader` | Callable | Load shader from files and bind default locations |
| `LoadShaderFromMemory(vsCode: string, fsCode: string)` | `Shader` | Callable | Load shader from code strings and bind default locations |
| `IsShaderValid(shader: Shader)` | `boolean` | Callable | Check if shader is valid (loaded on GPU) |
| `GetShaderLocation(shader: Shader, uniformName: string)` | `number` | Callable | Get shader uniform location |
| `GetShaderLocationAttrib(shader: Shader, attribName: string)` | `number` | Callable | Get shader attribute location |
| `SetShaderValue(shader: Shader, locIndex: number, value: bigint | null, uniformType: number)` | `void` | Callable | Set shader uniform value |
| `SetShaderValueV(shader: Shader, locIndex: number, value: bigint | null, uniformType: number, count: number)` | `void` | Callable | Set shader uniform value vector |
| `SetShaderValueMatrix(shader: Shader, locIndex: number, mat: Matrix)` | `void` | Callable | Set shader uniform value (matrix 4x4) |
| `SetShaderValueTexture(shader: Shader, locIndex: number, texture: Texture2D)` | `void` | Callable | Set shader uniform value and bind the texture (sampler2d) |
| `UnloadShader(shader: Shader)` | `void` | Callable | Unload shader from GPU memory (VRAM) |
| `GetScreenToWorldRay(position: Vector2, camera: Camera)` | `Ray` | Callable | Get a ray trace from screen position (i.e mouse) |
| `GetScreenToWorldRayEx(position: Vector2, camera: Camera, width: number, height: number)` | `Ray` | Callable | Get a ray trace from screen position (i.e mouse) in a viewport |
| `GetWorldToScreen(position: Vector3, camera: Camera)` | `Vector2` | Callable | Get screen space position for a 3d world space position |
| `GetWorldToScreenEx(position: Vector3, camera: Camera, width: number, height: number)` | `Vector2` | Callable | Get sized screen space position for a 3d world space position |
| `GetWorldToScreen2D(position: Vector2, camera: Camera2D)` | `Vector2` | Callable | Get screen space position for a 2d camera world space position |
| `GetScreenToWorld2D(position: Vector2, camera: Camera2D)` | `Vector2` | Callable | Get world space position for a 2d camera screen space position |
| `GetCameraMatrix(camera: Camera)` | `Matrix` | Callable | Get camera transform matrix (view matrix) |
| `GetCameraMatrix2D(camera: Camera2D)` | `Matrix` | Callable | Get camera 2d transform matrix |

### Timing and random values

| Function | Returns | Availability | Meaning |
|---|---|---|---|
| `SetTargetFPS(fps: number)` | `void` | Callable | Set target FPS (maximum) |
| `GetFrameTime()` | `number` | Callable | Get time in seconds for last frame drawn (delta time) |
| `GetTime()` | `number` | Callable | Get elapsed time in seconds since InitWindow() |
| `GetFPS()` | `number` | Callable | Get current FPS |
| `SwapScreenBuffer()` | `void` | Callable | Swap back buffer with front buffer (screen drawing) |
| `PollInputEvents()` | `void` | Callable | Register all input events |
| `WaitTime(seconds: number)` | `void` | Callable | Wait for some time (halt program execution) |
| `SetRandomSeed(seed: number)` | `void` | Callable | Set the seed for the random number generator |
| `GetRandomValue(min: number, max: number)` | `number` | Callable | Get a random value between min and max (both included) |
| `LoadRandomSequence(count: number, min: number, max: number)` | `bigint | null` | Callable | Load random values sequence, no values repeated |
| `UnloadRandomSequence(sequence: bigint | null)` | `void` | Callable | Unload random values sequence |

### Logging and configuration

| Function | Returns | Availability | Meaning |
|---|---|---|---|
| `TakeScreenshot(fileName: string)` | `void` | Callable | Takes a screenshot of current screen (filename extension defines format) |
| `SetConfigFlags(flags: number)` | `void` | Callable | Set up init configuration flags (view FLAGS) |
| `OpenURL(url: string)` | `void` | Callable | Open URL with default system browser (if available) |
| `SetTraceLogLevel(logLevel: number)` | `void` | Callable | Set the current threshold (minimum) log level |
| `TraceLog(logLevel: number, text: string, ...args: unknown[])` | `void` | **Not callable** | Show trace log messages (LOG_DEBUG, LOG_INFO, LOG_WARNING, LOG_ERROR...) |
| `SetTraceLogCallback(callback: TraceLogCallback)` | `void` | **Not callable** | Set custom trace log |

### Memory and file data

| Function | Returns | Availability | Meaning |
|---|---|---|---|
| `MemAlloc(size: number)` | `bigint | null` | Callable | Internal memory allocator |
| `MemRealloc(ptr: bigint | null, size: number)` | `bigint | null` | Callable | Internal memory reallocator |
| `MemFree(ptr: bigint | null)` | `void` | Callable | Internal memory free |
| `LoadFileData(fileName: string, dataSize: bigint | null)` | `bigint | null` | Callable | Load file data as byte array (read) |
| `UnloadFileData(data: bigint | null)` | `void` | Callable | Unload file data allocated by LoadFileData() |
| `SaveFileData(fileName: string, data: bigint | null, dataSize: number)` | `boolean` | Callable | Save data to file from byte array (write), returns true on success |
| `ExportDataAsCode(data: bigint | null, dataSize: number, fileName: string)` | `boolean` | Callable | Export data to code (.h), returns true on success |
| `LoadFileText(fileName: string)` | `string` | Callable | Load text data from file (read), returns a '\0' terminated string |
| `UnloadFileText(text: string)` | `void` | Callable | Unload file text data allocated by LoadFileText() |
| `SaveFileText(fileName: string, text: string)` | `boolean` | Callable | Save text data to file (write), string must be '\0' terminated, returns true on success |
| `SetLoadFileDataCallback(callback: LoadFileDataCallback)` | `void` | **Not callable** | Set custom file binary data loader |
| `SetSaveFileDataCallback(callback: SaveFileDataCallback)` | `void` | **Not callable** | Set custom file binary data saver |
| `SetLoadFileTextCallback(callback: LoadFileTextCallback)` | `void` | **Not callable** | Set custom file text data loader |
| `SetSaveFileTextCallback(callback: SaveFileTextCallback)` | `void` | **Not callable** | Set custom file text data saver |

### Paths and directories

| Function | Returns | Availability | Meaning |
|---|---|---|---|
| `FileRename(fileName: string, fileRename: string)` | `number` | Callable | Rename file (if exists), returns 0 on success |
| `FileRemove(fileName: string)` | `number` | Callable | Remove file (if exists), returns 0 on success |
| `FileCopy(srcPath: string, dstPath: string)` | `number` | Callable | Copy file from one path to another, dstPath created if it doesn't exist, returns 0 on success |
| `FileMove(srcPath: string, dstPath: string)` | `number` | Callable | Move file from one directory to another, dstPath created if it doesn't exist, returns 0 on success |
| `FileTextReplace(fileName: string, search: string, replacement: string)` | `number` | Callable | Replace text in an existing file, returns 0 on success |
| `FileTextFindIndex(fileName: string, search: string)` | `number` | Callable | Find text in existing file, returns -1 if index not found or index otherwise |
| `FileExists(fileName: string)` | `boolean` | Callable | Check if file exists |
| `DirectoryExists(dirPath: string)` | `boolean` | Callable | Check if directory path exists |
| `IsFileExtension(fileName: string, ext: string)` | `boolean` | Callable | Check file extension (recommended include point: .png, .wav) |
| `GetFileLength(fileName: string)` | `number` | Callable | Get file length in bytes (NOTE: GetFileSize() conflicts with windows.h) |
| `GetFileModTime(fileName: string)` | `number` | Callable | Get file modification time (last write time) |
| `GetFileExtension(fileName: string)` | `string` | Callable | Get pointer to extension for a filename string (includes dot: '.png') |
| `GetFileName(filePath: string)` | `string` | Callable | Get pointer to filename for a path string |
| `GetFileNameWithoutExt(filePath: string)` | `string` | Callable | Get filename string without extension (uses static string) |
| `GetDirectoryPath(filePath: string)` | `string` | Callable | Get full path for a provided fileName with path (uses static string) |
| `GetPrevDirectoryPath(dirPath: string)` | `string` | Callable | Get previous directory path for a provided path (uses static string) |
| `GetWorkingDirectory()` | `string` | Callable | Get current working directory (uses static string) |
| `GetApplicationDirectory()` | `string` | Callable | Get the directory of the running application (uses static string) |
| `MakeDirectory(dirPath: string)` | `number` | Callable | Create directories (including full path requested), returns 0 on success |
| `ChangeDirectory(dirPath: string)` | `number` | Callable | Change working directory, returns 0 on success |
| `IsPathFile(path: string)` | `boolean` | Callable | Check if provided path points to a file |
| `IsPathDirectory(path: string)` | `boolean` | Callable | Check if provided path points to a directory |
| `IsPathAbsolute(path: string)` | `boolean` | Callable | Check if provided path is an absolute path |
| `IsFileNameValid(fileName: string)` | `boolean` | Callable | Check if fileName is valid for the platform/OS |
| `LoadDirectoryFiles(dirPath: string)` | `FilePathList` | Callable | Load directory filepaths, files and directories, no subdirs scan |
| `LoadDirectoryFilesEx(basePath: string, filter: string, scanSubdirs: boolean)` | `FilePathList` | Callable | Load directory filepaths with extension filtering and subdir scan; some filters available: '*.*','FILES*','DIRS*' |
| `UnloadDirectoryFiles(files: FilePathList)` | `void` | Callable | Unload filepaths |
| `IsFileDropped()` | `boolean` | Callable | Check if file has been dropped into window |
| `LoadDroppedFiles()` | `FilePathList` | Callable | Load dropped filepaths |
| `UnloadDroppedFiles(files: FilePathList)` | `void` | Callable | Unload dropped filepaths |
| `GetDirectoryFileCount(dirPath: string)` | `number` | Callable | Get the file count in a directory |
| `GetDirectoryFileCountEx(basePath: string, filter: string, scanSubdirs: boolean)` | `number` | Callable | Get the file count in a directory with extension filtering and recursive directory scan. Use 'DIR' in the filter string to include directories in the result |

### Compression, hashes, and automation

| Function | Returns | Availability | Meaning |
|---|---|---|---|
| `CompressData(data: bigint | null, dataSize: number, compDataSize: bigint | null)` | `bigint | null` | Callable | Compress data (DEFLATE algorithm), memory must be MemFree() |
| `DecompressData(compData: bigint | null, compDataSize: number, dataSize: bigint | null)` | `bigint | null` | Callable | Decompress data (DEFLATE algorithm), memory must be MemFree() |
| `EncodeDataBase64(data: bigint | null, dataSize: number, outputSize: bigint | null)` | `string` | Callable | Encode data to Base64 string (includes NULL terminator), memory must be MemFree() |
| `DecodeDataBase64(text: string, outputSize: bigint | null)` | `bigint | null` | Callable | Decode Base64 string (expected NULL terminated), memory must be MemFree() |
| `ComputeCRC32(data: bigint | null, dataSize: number)` | `number` | Callable | Compute CRC32 hash code |
| `ComputeMD5(data: bigint | null, dataSize: number)` | `bigint | null` | Callable | Compute MD5 hash code, returns static int[4] (16 bytes) |
| `ComputeSHA1(data: bigint | null, dataSize: number)` | `bigint | null` | Callable | Compute SHA1 hash code, returns static int[5] (20 bytes) |
| `ComputeSHA256(data: bigint | null, dataSize: number)` | `bigint | null` | Callable | Compute SHA256 hash code, returns static int[8] (32 bytes) |
| `LoadAutomationEventList(fileName: string)` | `AutomationEventList` | Callable | Load automation events list from file, NULL for empty list, capacity = MAX_AUTOMATION_EVENTS |
| `UnloadAutomationEventList(list: AutomationEventList)` | `void` | Callable | Unload automation events list from file |
| `ExportAutomationEventList(list: AutomationEventList, fileName: string)` | `boolean` | Callable | Export automation events list as text file |
| `SetAutomationEventList(list: bigint | null)` | `void` | Callable | Set automation event list to record to |
| `SetAutomationEventBaseFrame(frame: number)` | `void` | Callable | Set automation event internal base frame to start recording |
| `StartAutomationEventRecording()` | `void` | Callable | Start recording automation events (AutomationEventList must be set) |
| `StopAutomationEventRecording()` | `void` | Callable | Stop recording automation events |
| `PlayAutomationEvent(event: AutomationEvent)` | `void` | Callable | Play a recorded automation event |

### Input

| Function | Returns | Availability | Meaning |
|---|---|---|---|
| `IsKeyPressed(key: number)` | `boolean` | Callable | Check if key has been pressed once |
| `IsKeyPressedRepeat(key: number)` | `boolean` | Callable | Check if key has been pressed again |
| `IsKeyDown(key: number)` | `boolean` | Callable | Check if key is being pressed |
| `IsKeyReleased(key: number)` | `boolean` | Callable | Check if key has been released once |
| `IsKeyUp(key: number)` | `boolean` | Callable | Check if key is NOT being pressed |
| `GetKeyPressed()` | `number` | Callable | Get key pressed (keycode), call it multiple times for keys queued, returns 0 when the queue is empty |
| `GetCharPressed()` | `number` | Callable | Get char pressed (unicode), call it multiple times for chars queued, returns 0 when the queue is empty |
| `GetKeyName(key: number)` | `string` | Callable | Get name of a QWERTY key on the current keyboard layout (eg returns string 'q' for KEY_A on an AZERTY keyboard) |
| `SetExitKey(key: number)` | `void` | Callable | Set a custom key to exit program (default is ESC) |
| `IsGamepadAvailable(gamepad: number)` | `boolean` | Callable | Check if gamepad is available |
| `GetGamepadName(gamepad: number)` | `string` | Callable | Get gamepad internal name id |
| `IsGamepadButtonPressed(gamepad: number, button: number)` | `boolean` | Callable | Check if gamepad button has been pressed once |
| `IsGamepadButtonDown(gamepad: number, button: number)` | `boolean` | Callable | Check if gamepad button is being pressed |
| `IsGamepadButtonReleased(gamepad: number, button: number)` | `boolean` | Callable | Check if gamepad button has been released once |
| `IsGamepadButtonUp(gamepad: number, button: number)` | `boolean` | Callable | Check if gamepad button is NOT being pressed |
| `GetGamepadButtonPressed()` | `number` | Callable | Get the last gamepad button pressed |
| `GetGamepadAxisCount(gamepad: number)` | `number` | Callable | Get axis count for a gamepad |
| `GetGamepadAxisMovement(gamepad: number, axis: number)` | `number` | Callable | Get movement value for a gamepad axis |
| `SetGamepadMappings(mappings: string)` | `number` | Callable | Set internal gamepad mappings (SDL_GameControllerDB) |
| `SetGamepadVibration(gamepad: number, leftMotor: number, rightMotor: number, duration: number)` | `void` | Callable | Set gamepad vibration for both motors (duration in seconds) |
| `IsMouseButtonPressed(button: number)` | `boolean` | Callable | Check if mouse button has been pressed once |
| `IsMouseButtonDown(button: number)` | `boolean` | Callable | Check if mouse button is being pressed |
| `IsMouseButtonReleased(button: number)` | `boolean` | Callable | Check if mouse button has been released once |
| `IsMouseButtonUp(button: number)` | `boolean` | Callable | Check if mouse button is NOT being pressed |
| `GetMouseX()` | `number` | Callable | Get mouse position X |
| `GetMouseY()` | `number` | Callable | Get mouse position Y |
| `GetMousePosition()` | `Vector2` | Callable | Get mouse position XY |
| `GetMouseDelta()` | `Vector2` | Callable | Get mouse delta between frames |
| `SetMousePosition(x: number, y: number)` | `void` | Callable | Set mouse position XY |
| `SetMouseOffset(offsetX: number, offsetY: number)` | `void` | Callable | Set mouse offset |
| `SetMouseScale(scaleX: number, scaleY: number)` | `void` | Callable | Set mouse scaling |
| `GetMouseWheelMove()` | `number` | Callable | Get mouse wheel movement for X or Y, whichever is larger |
| `GetMouseWheelMoveV()` | `Vector2` | Callable | Get mouse wheel movement for both X and Y |
| `SetMouseCursor(cursor: number)` | `void` | Callable | Set mouse cursor |
| `GetTouchX()` | `number` | Callable | Get touch position X for touch point 0 (relative to screen size) |
| `GetTouchY()` | `number` | Callable | Get touch position Y for touch point 0 (relative to screen size) |
| `GetTouchPosition(index: number)` | `Vector2` | Callable | Get touch position XY for a touch point index (relative to screen size) |
| `GetTouchPointId(index: number)` | `number` | Callable | Get touch point identifier for provided index |
| `GetTouchPointCount()` | `number` | Callable | Get number of touch points |
| `SetGesturesEnabled(flags: number)` | `void` | Callable | Enable a set of gestures using flags |
| `IsGestureDetected(gesture: number)` | `boolean` | Callable | Check if gesture has been detected |
| `GetGestureDetected()` | `number` | Callable | Get latest detected gesture |
| `GetGestureHoldDuration()` | `number` | Callable | Get gesture hold time in seconds |
| `GetGestureDragVector()` | `Vector2` | Callable | Get gesture drag vector |
| `GetGestureDragAngle()` | `number` | Callable | Get gesture drag angle |
| `GetGesturePinchVector()` | `Vector2` | Callable | Get gesture pinch delta |
| `GetGesturePinchAngle()` | `number` | Callable | Get gesture pinch angle |

### Camera updates

| Function | Returns | Availability | Meaning |
|---|---|---|---|
| `UpdateCamera(camera: bigint | null, mode: number)` | `void` | Callable | Update camera position for selected mode |
| `UpdateCameraPro(camera: bigint | null, movement: Vector3, rotation: Vector3, zoom: number)` | `void` | Callable | Update camera movement/rotation |

### Shapes and 2D collision

| Function | Returns | Availability | Meaning |
|---|---|---|---|
| `SetShapesTexture(texture: Texture2D, rec: Rectangle)` | `void` | Callable | Set texture and rectangle to be used on shapes drawing |
| `GetShapesTexture()` | `Texture2D` | Callable | Get texture that is used for shapes drawing |
| `GetShapesTextureRectangle()` | `Rectangle` | Callable | Get texture source rectangle that is used for shapes drawing |
| `DrawPixel(posX: number, posY: number, color: Color)` | `void` | Callable | Draw a pixel using geometry [Can be slow, use with care] |
| `DrawPixelV(position: Vector2, color: Color)` | `void` | Callable | Draw a pixel using geometry (Vector version) [Can be slow, use with care] |
| `DrawLine(startPosX: number, startPosY: number, endPosX: number, endPosY: number, color: Color)` | `void` | Callable | Draw a line |
| `DrawLineV(startPos: Vector2, endPos: Vector2, color: Color)` | `void` | Callable | Draw a line (using gl lines) |
| `DrawLineEx(startPos: Vector2, endPos: Vector2, thick: number, color: Color)` | `void` | Callable | Draw a line (using triangles/quads) |
| `DrawLineStrip(points: bigint | null, pointCount: number, color: Color)` | `void` | Callable | Draw lines sequence (using gl lines) |
| `DrawLineBezier(startPos: Vector2, endPos: Vector2, thick: number, color: Color)` | `void` | Callable | Draw line segment cubic-bezier in-out interpolation |
| `DrawLineDashed(startPos: Vector2, endPos: Vector2, dashSize: number, spaceSize: number, color: Color)` | `void` | Callable | Draw a dashed line |
| `DrawTriangle(v1: Vector2, v2: Vector2, v3: Vector2, color: Color)` | `void` | Callable | Draw a color-filled triangle, counter-clockwise vertex order |
| `DrawTriangleGradient(v1: Vector2, v2: Vector2, v3: Vector2, c1: Color, c2: Color, c3: Color)` | `void` | Callable | Draw triangle with interpolated colors, counter-clockwise vertex/color order |
| `DrawTriangleLines(v1: Vector2, v2: Vector2, v3: Vector2, color: Color)` | `void` | Callable | Draw triangle outline, counter-clockwise vertex order |
| `DrawTriangleFan(points: bigint | null, pointCount: number, color: Color)` | `void` | Callable | Draw a triangle fan defined by points (first vertex is the center) |
| `DrawTriangleStrip(points: bigint | null, pointCount: number, color: Color)` | `void` | Callable | Draw a triangle strip defined by points |
| `DrawRectangle(posX: number, posY: number, width: number, height: number, color: Color)` | `void` | Callable | Draw a color-filled rectangle |
| `DrawRectangleV(position: Vector2, size: Vector2, color: Color)` | `void` | Callable | Draw a color-filled rectangle (Vector version) |
| `DrawRectangleRec(rec: Rectangle, color: Color)` | `void` | Callable | Draw a color-filled rectangle |
| `DrawRectanglePro(rec: Rectangle, origin: Vector2, rotation: number, color: Color)` | `void` | Callable | Draw a color-filled rectangle with pro parameters |
| `DrawRectangleGradientV(posX: number, posY: number, width: number, height: number, top: Color, bottom: Color)` | `void` | Callable | Draw a vertical-gradient-filled rectangle |
| `DrawRectangleGradientH(posX: number, posY: number, width: number, height: number, left: Color, right: Color)` | `void` | Callable | Draw a horizontal-gradient-filled rectangle |
| `DrawRectangleGradientEx(rec: Rectangle, col1: Color, col2: Color, col3: Color, col4: Color)` | `void` | Callable | Draw a gradient-filled rectangle with custom vertex colors, counter-clockwise color order |
| `DrawRectangleLines(posX: number, posY: number, width: number, height: number, color: Color)` | `void` | Callable | Draw rectangle outline |
| `DrawRectangleLinesEx(rec: Rectangle, thick: number, color: Color)` | `void` | Callable | Draw rectangle outline with line thickness |
| `DrawRectangleRounded(rec: Rectangle, roundness: number, segments: number, color: Color)` | `void` | Callable | Draw rectangle with rounded edges |
| `DrawRectangleRoundedLines(rec: Rectangle, roundness: number, segments: number, color: Color)` | `void` | Callable | Draw rectangle lines with rounded edges |
| `DrawRectangleRoundedLinesEx(rec: Rectangle, roundness: number, segments: number, thick: number, color: Color)` | `void` | Callable | Draw rectangle lines with rounded edges outline and line thickness |
| `DrawPoly(center: Vector2, sides: number, radius: number, rotation: number, color: Color)` | `void` | Callable | Draw a polygon of n sides |
| `DrawPolyLines(center: Vector2, sides: number, radius: number, rotation: number, color: Color)` | `void` | Callable | Draw a polygon outline of n sides |
| `DrawPolyLinesEx(center: Vector2, sides: number, radius: number, rotation: number, thick: number, color: Color)` | `void` | Callable | Draw a polygon outline of n sides with line thickness |
| `DrawCircle(centerX: number, centerY: number, radius: number, color: Color)` | `void` | Callable | Draw a color-filled circle |
| `DrawCircleV(center: Vector2, radius: number, color: Color)` | `void` | Callable | Draw a color-filled circle (Vector version) |
| `DrawCircleGradient(center: Vector2, radius: number, inner: Color, outer: Color)` | `void` | Callable | Draw a gradient-filled circle |
| `DrawCircleSector(center: Vector2, radius: number, startAngle: number, endAngle: number, segments: number, color: Color)` | `void` | Callable | Draw a piece of a circle |
| `DrawCircleSectorLines(center: Vector2, radius: number, startAngle: number, endAngle: number, segments: number, color: Color)` | `void` | Callable | Draw circle sector outline |
| `DrawCircleLines(centerX: number, centerY: number, radius: number, color: Color)` | `void` | Callable | Draw circle outline |
| `DrawCircleLinesV(center: Vector2, radius: number, color: Color)` | `void` | Callable | Draw circle outline (Vector version) |
| `DrawCircleLinesEx(center: Vector2, radius: number, thick: number, color: Color)` | `void` | Callable | Draw circle outline with line thickness |
| `DrawEllipse(centerX: number, centerY: number, radiusH: number, radiusV: number, color: Color)` | `void` | Callable | Draw ellipse |
| `DrawEllipseV(center: Vector2, radiusH: number, radiusV: number, color: Color)` | `void` | Callable | Draw ellipse (Vector version) |
| `DrawEllipseLines(centerX: number, centerY: number, radiusH: number, radiusV: number, color: Color)` | `void` | Callable | Draw ellipse outline |
| `DrawEllipseLinesV(center: Vector2, radiusH: number, radiusV: number, color: Color)` | `void` | Callable | Draw ellipse outline (Vector version) |
| `DrawRing(center: Vector2, innerRadius: number, outerRadius: number, startAngle: number, endAngle: number, segments: number, color: Color)` | `void` | Callable | Draw ring |
| `DrawRingLines(center: Vector2, innerRadius: number, outerRadius: number, startAngle: number, endAngle: number, segments: number, color: Color)` | `void` | Callable | Draw ring outline |
| `DrawSplineLinear(points: bigint | null, pointCount: number, thick: number, color: Color)` | `void` | Callable | Draw spline: Linear, minimum 2 points |
| `DrawSplineBasis(points: bigint | null, pointCount: number, thick: number, color: Color)` | `void` | Callable | Draw spline: B-Spline, minimum 4 points |
| `DrawSplineCatmullRom(points: bigint | null, pointCount: number, thick: number, color: Color)` | `void` | Callable | Draw spline: Catmull-Rom, minimum 4 points |
| `DrawSplineBezierQuadratic(points: bigint | null, pointCount: number, thick: number, color: Color)` | `void` | Callable | Draw spline: Quadratic Bezier, minimum 3 points (1 control point): [p1, c2, p3, c4...] |
| `DrawSplineBezierCubic(points: bigint | null, pointCount: number, thick: number, color: Color)` | `void` | Callable | Draw spline: Cubic Bezier, minimum 4 points (2 control points): [p1, c2, c3, p4, c5, c6...] |
| `DrawSplineSegmentLinear(p1: Vector2, p2: Vector2, thick: number, color: Color)` | `void` | Callable | Draw spline segment: Linear, 2 points |
| `DrawSplineSegmentBasis(p1: Vector2, p2: Vector2, p3: Vector2, p4: Vector2, thick: number, color: Color)` | `void` | Callable | Draw spline segment: B-Spline, 4 points |
| `DrawSplineSegmentCatmullRom(p1: Vector2, p2: Vector2, p3: Vector2, p4: Vector2, thick: number, color: Color)` | `void` | Callable | Draw spline segment: Catmull-Rom, 4 points |
| `DrawSplineSegmentBezierQuadratic(p1: Vector2, c2: Vector2, p3: Vector2, thick: number, color: Color)` | `void` | Callable | Draw spline segment: Quadratic Bezier, 2 points, 1 control point |
| `DrawSplineSegmentBezierCubic(p1: Vector2, c2: Vector2, c3: Vector2, p4: Vector2, thick: number, color: Color)` | `void` | Callable | Draw spline segment: Cubic Bezier, 2 points, 2 control points |
| `GetSplinePointLinear(startPos: Vector2, endPos: Vector2, t: number)` | `Vector2` | Callable | Get (evaluate) spline point: Linear |
| `GetSplinePointBasis(p1: Vector2, p2: Vector2, p3: Vector2, p4: Vector2, t: number)` | `Vector2` | Callable | Get (evaluate) spline point: B-Spline |
| `GetSplinePointCatmullRom(p1: Vector2, p2: Vector2, p3: Vector2, p4: Vector2, t: number)` | `Vector2` | Callable | Get (evaluate) spline point: Catmull-Rom |
| `GetSplinePointBezierQuadratic(p1: Vector2, c2: Vector2, p3: Vector2, t: number)` | `Vector2` | Callable | Get (evaluate) spline point: Quadratic Bezier |
| `GetSplinePointBezierCubic(p1: Vector2, c2: Vector2, c3: Vector2, p4: Vector2, t: number)` | `Vector2` | Callable | Get (evaluate) spline point: Cubic Bezier |
| `CheckCollisionRecs(rec1: Rectangle, rec2: Rectangle)` | `boolean` | Callable | Check collision between two rectangles |
| `CheckCollisionCircles(center1: Vector2, radius1: number, center2: Vector2, radius2: number)` | `boolean` | Callable | Check collision between two circles |
| `CheckCollisionCircleRec(center: Vector2, radius: number, rec: Rectangle)` | `boolean` | Callable | Check collision between circle and rectangle |
| `CheckCollisionCircleLine(center: Vector2, radius: number, p1: Vector2, p2: Vector2)` | `boolean` | Callable | Check if circle collides with a line created between two points [p1] and [p2] |
| `CheckCollisionPointRec(point: Vector2, rec: Rectangle)` | `boolean` | Callable | Check if point is inside rectangle |
| `CheckCollisionPointCircle(point: Vector2, center: Vector2, radius: number)` | `boolean` | Callable | Check if point is inside circle |
| `CheckCollisionPointTriangle(point: Vector2, p1: Vector2, p2: Vector2, p3: Vector2)` | `boolean` | Callable | Check if point is inside a triangle |
| `CheckCollisionPointLine(point: Vector2, p1: Vector2, p2: Vector2, threshold: number)` | `boolean` | Callable | Check if point belongs to line created between two points [p1] and [p2] with defined margin in pixels [threshold] |
| `CheckCollisionPointPoly(point: Vector2, points: bigint | null, pointCount: number)` | `boolean` | Callable | Check if point is within a polygon described by array of vertices |
| `CheckCollisionLines(startPos1: Vector2, endPos1: Vector2, startPos2: Vector2, endPos2: Vector2, collisionPoint: bigint | null)` | `boolean` | Callable | Check the collision between two lines defined by two points each, returns collision point by reference |
| `GetCollisionRec(rec1: Rectangle, rec2: Rectangle)` | `Rectangle` | Callable | Get collision rectangle for two rectangles collision |

### Images

| Function | Returns | Availability | Meaning |
|---|---|---|---|
| `LoadImage(fileName: string)` | `Image` | Callable | Load image from file into CPU memory (RAM) |
| `LoadImageRaw(fileName: string, width: number, height: number, format: number, headerSize: number)` | `Image` | Callable | Load image from RAW file data |
| `LoadImageAnim(fileName: string, frames: bigint | null)` | `Image` | Callable | Load image sequence from file (frames appended to image.data) |
| `LoadImageAnimFromMemory(fileType: string, fileData: bigint | null, dataSize: number, frames: bigint | null)` | `Image` | Callable | Load image sequence from memory buffer |
| `LoadImageFromMemory(fileType: string, fileData: bigint | null, dataSize: number)` | `Image` | Callable | Load image from memory buffer, fileType refers to extension: i.e. '.png' |
| `LoadImageFromTexture(texture: Texture2D)` | `Image` | Callable | Load image from GPU texture data |
| `LoadImageFromScreen()` | `Image` | Callable | Load image from screen buffer (screenshot) |
| `IsImageValid(image: Image)` | `boolean` | Callable | Check if an image is valid (data and parameters) |
| `UnloadImage(image: Image)` | `void` | Callable | Unload image from CPU memory (RAM) |
| `ExportImage(image: Image, fileName: string)` | `boolean` | Callable | Export image data to file, returns true on success |
| `ExportImageToMemory(image: Image, fileType: string, fileSize: bigint | null)` | `bigint | null` | Callable | Export image to memory buffer, memory must be MemFree() |
| `ExportImageAsCode(image: Image, fileName: string)` | `boolean` | Callable | Export image as code file defining an array of bytes, returns true on success |
| `GenImageColor(width: number, height: number, color: Color)` | `Image` | Callable | Generate image: plain color |
| `GenImageGradientLinear(width: number, height: number, direction: number, start: Color, end: Color)` | `Image` | Callable | Generate image: linear gradient, direction in degrees [0..360], 0=Vertical gradient |
| `GenImageGradientRadial(width: number, height: number, density: number, inner: Color, outer: Color)` | `Image` | Callable | Generate image: radial gradient |
| `GenImageGradientSquare(width: number, height: number, density: number, inner: Color, outer: Color)` | `Image` | Callable | Generate image: square gradient |
| `GenImageChecked(width: number, height: number, checksX: number, checksY: number, col1: Color, col2: Color)` | `Image` | Callable | Generate image: checked |
| `GenImageWhiteNoise(width: number, height: number, factor: number)` | `Image` | Callable | Generate image: white noise |
| `GenImagePerlinNoise(width: number, height: number, offsetX: number, offsetY: number, scale: number)` | `Image` | Callable | Generate image: perlin noise |
| `GenImageCellular(width: number, height: number, tileSize: number)` | `Image` | Callable | Generate image: cellular algorithm, bigger tileSize means bigger cells |
| `GenImageText(width: number, height: number, text: string)` | `Image` | Callable | Generate image: grayscale image from text data |
| `ImageCopy(image: Image)` | `Image` | Callable | Create an image duplicate (useful for transformations) |
| `ImageFromImage(image: Image, rec: Rectangle)` | `Image` | Callable | Create an image from another image piece |
| `ImageFromChannel(image: Image, selectedChannel: number)` | `Image` | Callable | Create an image from a selected channel of another image (GRAYSCALE) |
| `ImageText(text: string, fontSize: number, color: Color)` | `Image` | Callable | Create an image from text (default font) |
| `ImageTextEx(font: Font, text: string, fontSize: number, spacing: number, tint: Color)` | `Image` | Callable | Create an image from text (custom sprite font) |
| `ImageFormat(image: bigint | null, newFormat: number)` | `void` | Callable | Convert image data to desired format |
| `ImageToPOT(image: bigint | null, fill: Color)` | `void` | Callable | Convert image to POT (power-of-two) |
| `ImageCrop(image: bigint | null, crop: Rectangle)` | `void` | Callable | Crop an image to a defined rectangle |
| `ImageAlphaCrop(image: bigint | null, threshold: number)` | `void` | Callable | Crop image depending on alpha value |
| `ImageAlphaClear(image: bigint | null, color: Color, threshold: number)` | `void` | Callable | Clear alpha channel to desired color |
| `ImageAlphaMask(image: bigint | null, alphaMask: Image)` | `void` | Callable | Apply alpha mask to image |
| `ImageAlphaPremultiply(image: bigint | null)` | `void` | Callable | Premultiply alpha channel |
| `ImageBlurGaussian(image: bigint | null, blurSize: number)` | `void` | Callable | Apply Gaussian blur using a box blur approximation |
| `ImageKernelConvolution(image: bigint | null, kernel: bigint | null, kernelSize: number)` | `void` | Callable | Apply custom square convolution kernel to image |
| `ImageResize(image: bigint | null, newWidth: number, newHeight: number)` | `void` | Callable | Resize image (Bicubic scaling algorithm) |
| `ImageResizeNN(image: bigint | null, newWidth: number, newHeight: number)` | `void` | Callable | Resize image (Nearest-Neighbor scaling algorithm) |
| `ImageResizeCanvas(image: bigint | null, newWidth: number, newHeight: number, offsetX: number, offsetY: number, fill: Color)` | `void` | Callable | Resize canvas and fill with color |
| `ImageMipmaps(image: bigint | null)` | `void` | Callable | Compute all mipmap levels for a provided image |
| `ImageDither(image: bigint | null, rBpp: number, gBpp: number, bBpp: number, aBpp: number)` | `void` | Callable | Dither image data to 16bpp or lower (Floyd-Steinberg dithering) |
| `ImageFlipVertical(image: bigint | null)` | `void` | Callable | Flip image vertically |
| `ImageFlipHorizontal(image: bigint | null)` | `void` | Callable | Flip image horizontally |
| `ImageRotate(image: bigint | null, degrees: number)` | `void` | Callable | Rotate image by input angle in degrees (-359 to 359) |
| `ImageRotateCW(image: bigint | null)` | `void` | Callable | Rotate image clockwise 90deg |
| `ImageRotateCCW(image: bigint | null)` | `void` | Callable | Rotate image counter-clockwise 90deg |
| `ImageColorTint(image: bigint | null, color: Color)` | `void` | Callable | Modify image color: tint |
| `ImageColorInvert(image: bigint | null)` | `void` | Callable | Modify image color: invert |
| `ImageColorGrayscale(image: bigint | null)` | `void` | Callable | Modify image color: grayscale |
| `ImageColorContrast(image: bigint | null, contrast: number)` | `void` | Callable | Modify image color: contrast (-100 to 100) |
| `ImageColorBrightness(image: bigint | null, brightness: number)` | `void` | Callable | Modify image color: brightness (-255 to 255) |
| `ImageColorReplace(image: bigint | null, color: Color, replace: Color)` | `void` | Callable | Modify image color: replace color |
| `LoadImageColors(image: Image)` | `bigint | null` | Callable | Load color data from image as a Color array (RGBA - 32bit) |
| `LoadImagePalette(image: Image, maxPaletteSize: number, colorCount: bigint | null)` | `bigint | null` | Callable | Load colors palette from image as a Color array (RGBA - 32bit) |
| `UnloadImageColors(colors: bigint | null)` | `void` | Callable | Unload color data loaded with LoadImageColors() |
| `UnloadImagePalette(colors: bigint | null)` | `void` | Callable | Unload colors palette loaded with LoadImagePalette() |
| `GetImageAlphaBorder(image: Image, threshold: number)` | `Rectangle` | Callable | Get image alpha border rectangle |
| `GetImageColor(image: Image, x: number, y: number)` | `Color` | Callable | Get image pixel color at (x, y) position |
| `ImageClearBackground(dst: bigint | null, color: Color)` | `void` | Callable | Clear image background with provided color |
| `ImageDrawPixel(dst: bigint | null, posX: number, posY: number, color: Color)` | `void` | Callable | Draw pixel within an image |
| `ImageDrawPixelV(dst: bigint | null, position: Vector2, color: Color)` | `void` | Callable | Draw pixel within an image (Vector version) |
| `ImageDrawLine(dst: bigint | null, startPosX: number, startPosY: number, endPosX: number, endPosY: number, color: Color)` | `void` | Callable | Draw line within an image |
| `ImageDrawLineV(dst: bigint | null, start: Vector2, end: Vector2, color: Color)` | `void` | Callable | Draw line within an image (Vector version) |
| `ImageDrawLineEx(dst: bigint | null, start: Vector2, end: Vector2, thick: number, color: Color)` | `void` | Callable | Draw a line defining thickness within an image |
| `ImageDrawLineStrip(dst: bigint | null, points: bigint | null, pointCount: number, color: Color)` | `void` | Callable | Draw a lines sequence within an image |
| `ImageDrawTriangle(dst: bigint | null, v1: Vector2, v2: Vector2, v3: Vector2, color: Color)` | `void` | Callable | Draw triangle within an image |
| `ImageDrawTriangleGradient(dst: bigint | null, v1: Vector2, v2: Vector2, v3: Vector2, c1: Color, c2: Color, c3: Color)` | `void` | Callable | Draw triangle with interpolated colors within an image |
| `ImageDrawTriangleLines(dst: bigint | null, v1: Vector2, v2: Vector2, v3: Vector2, color: Color)` | `void` | Callable | Draw triangle outline within an image |
| `ImageDrawTriangleFan(dst: bigint | null, points: bigint | null, pointCount: number, color: Color)` | `void` | Callable | Draw a triangle fan defined by points within an image (first vertex is the center) |
| `ImageDrawTriangleStrip(dst: bigint | null, points: bigint | null, pointCount: number, color: Color)` | `void` | Callable | Draw a triangle strip defined by points within an image |
| `ImageDrawRectangle(dst: bigint | null, posX: number, posY: number, width: number, height: number, color: Color)` | `void` | Callable | Draw rectangle within an image |
| `ImageDrawRectangleV(dst: bigint | null, position: Vector2, size: Vector2, color: Color)` | `void` | Callable | Draw rectangle within an image (Vector version) |
| `ImageDrawRectangleRec(dst: bigint | null, rec: Rectangle, color: Color)` | `void` | Callable | Draw rectangle within an image |
| `ImageDrawRectanglePro(dst: bigint | null, rec: Rectangle, origin: Vector2, rotation: number, color: Color)` | `void` | Callable | Draw a color-filled rectangle with pro parameters within and image |
| `ImageDrawRectangleLines(dst: bigint | null, posX: number, posY: number, width: number, height: number, color: Color)` | `void` | Callable | Draw rectangle lines within an image |
| `ImageDrawRectangleLinesEx(dst: bigint | null, rec: Rectangle, thick: number, color: Color)` | `void` | Callable | Draw rectangle lines within an image with line thickness |
| `ImageDrawRectangleGradientEx(dst: bigint | null, rec: Rectangle, col1: Color, col2: Color, col3: Color, col4: Color)` | `void` | Callable | Draw rectangle with gradient colors within an image, counter-clockwise color order |
| `ImageDrawCircle(dst: bigint | null, centerX: number, centerY: number, radius: number, color: Color)` | `void` | Callable | Draw a filled circle within an image |
| `ImageDrawCircleV(dst: bigint | null, center: Vector2, radius: number, color: Color)` | `void` | Callable | Draw a filled circle within an image (Vector version) |
| `ImageDrawCircleLines(dst: bigint | null, centerX: number, centerY: number, radius: number, color: Color)` | `void` | Callable | Draw circle outline within an image |
| `ImageDrawCircleLinesV(dst: bigint | null, center: Vector2, radius: number, color: Color)` | `void` | Callable | Draw circle outline within an image (Vector version) |
| `ImageDrawCircleGradient(dst: bigint | null, center: Vector2, radius: number, inner: Color, outer: Color)` | `void` | Callable | Draw a gradient-filled circle within an image |
| `ImageDrawImage(dst: bigint | null, src: Image, posX: number, posY: number, tint: Color)` | `void` | Callable | Draw an image within an image |
| `ImageDrawImageEx(dst: bigint | null, src: Image, position: Vector2, rotation: number, scale: number, tint: Color)` | `void` | Callable | Draw an image with scaling and rotation within an image |
| `ImageDrawImageRec(dst: bigint | null, src: Image, srcRec: Rectangle, position: Vector2, tint: Color)` | `void` | Callable | Draw a part of an image defined by a rectangle within an image |
| `ImageDrawImagePro(dst: bigint | null, src: Image, srcRec: Rectangle, dstRec: Rectangle, origin: Vector2, rotation: number, tint: Color)` | `void` | Callable | Draw a part of an image defined by a rectangle into destination rectangle, with scaling and rotation, within an image |
| `ImageDrawText(dst: bigint | null, text: string, posX: number, posY: number, fontSize: number, color: Color)` | `void` | Callable | Draw text (using default font) within an image (destination) |
| `ImageDrawTextEx(dst: bigint | null, font: Font, text: string, position: Vector2, fontSize: number, spacing: number, tint: Color)` | `void` | Callable | Draw text (custom sprite font) within an image (destination) |
| `ImageDrawTextPro(dst: bigint | null, font: Font, text: string, position: Vector2, origin: Vector2, rotation: number, fontSize: number, spacing: number, tint: Color)` | `void` | Callable | Draw text using Font and pro parameters (rotation) |

### Textures and color

| Function | Returns | Availability | Meaning |
|---|---|---|---|
| `LoadTexture(fileName: string)` | `Texture2D` | Callable | Load texture from file into GPU memory (VRAM) |
| `LoadTextureFromImage(image: Image)` | `Texture2D` | Callable | Load texture from image data |
| `LoadTextureCubemap(image: Image, layout: number)` | `TextureCubemap` | Callable | Load cubemap from image, multiple image cubemap layouts supported |
| `LoadRenderTexture(width: number, height: number)` | `RenderTexture2D` | Callable | Load texture for rendering (framebuffer) |
| `IsTextureValid(texture: Texture2D)` | `boolean` | Callable | Check if texture is valid (loaded in GPU) |
| `UnloadTexture(texture: Texture2D)` | `void` | Callable | Unload texture from GPU memory (VRAM) |
| `IsRenderTextureValid(target: RenderTexture2D)` | `boolean` | Callable | Check if render texture is valid (loaded in GPU) |
| `UnloadRenderTexture(target: RenderTexture2D)` | `void` | Callable | Unload render texture from GPU memory (VRAM) |
| `UpdateTexture(texture: Texture2D, pixels: bigint | null)` | `void` | Callable | Update GPU texture with new data (pixels should be able to fill texture) |
| `UpdateTextureRec(texture: Texture2D, rec: Rectangle, pixels: bigint | null)` | `void` | Callable | Update GPU texture rectangle with new data (pixels and rec should fit in texture) |
| `GenTextureMipmaps(texture: bigint | null)` | `void` | Callable | Generate GPU mipmaps for a texture |
| `SetTextureFilter(texture: Texture2D, filter: number)` | `void` | Callable | Set texture scaling filter mode |
| `SetTextureWrap(texture: Texture2D, wrap: number)` | `void` | Callable | Set texture wrapping mode |
| `DrawTexture(texture: Texture2D, posX: number, posY: number, tint: Color)` | `void` | Callable | Draw a Texture2D |
| `DrawTextureV(texture: Texture2D, position: Vector2, tint: Color)` | `void` | Callable | Draw a Texture2D with position defined as Vector2 |
| `DrawTextureEx(texture: Texture2D, position: Vector2, rotation: number, scale: number, tint: Color)` | `void` | Callable | Draw a Texture2D with rotation and scale |
| `DrawTextureRec(texture: Texture2D, rec: Rectangle, position: Vector2, tint: Color)` | `void` | Callable | Draw a part of a texture defined by a rectangle |
| `DrawTexturePro(texture: Texture2D, srcrec: Rectangle, dstrec: Rectangle, origin: Vector2, rotation: number, tint: Color)` | `void` | Callable | Draw a part of a texture defined by a source rectangle to destination rectangle, with scaling and rotation |
| `DrawTextureNPatch(texture: Texture2D, nPatchInfo: NPatchInfo, dstrec: Rectangle, origin: Vector2, rotation: number, tint: Color)` | `void` | Callable | Draw a texture (or part of it) that stretches or shrinks nicely |
| `ColorIsEqual(col1: Color, col2: Color)` | `boolean` | Callable | Check if two colors are equal |
| `Fade(color: Color, alpha: number)` | `Color` | Callable | Get color with alpha applied, alpha goes from 0.0f to 1.0f |
| `ColorToInt(color: Color)` | `number` | Callable | Get hexadecimal value for a Color (0xRRGGBBAA) |
| `ColorNormalize(color: Color)` | `Vector4` | Callable | Get Color normalized as float [0..1] |
| `ColorFromNormalized(normalized: Vector4)` | `Color` | Callable | Get Color from normalized values [0..1] |
| `ColorToHSV(color: Color)` | `Vector3` | Callable | Get HSV values for a Color, hue [0..360], saturation/value [0..1] |
| `ColorFromHSV(hue: number, saturation: number, value: number)` | `Color` | Callable | Get a Color from HSV values, hue [0..360], saturation/value [0..1] |
| `ColorTint(color: Color, tint: Color)` | `Color` | Callable | Get color multiplied with another color |
| `ColorBrightness(color: Color, factor: number)` | `Color` | Callable | Get color with brightness correction, brightness factor goes from -1.0f to 1.0f |
| `ColorContrast(color: Color, contrast: number)` | `Color` | Callable | Get color with contrast correction, contrast values between -1.0f and 1.0f |
| `ColorAlpha(color: Color, alpha: number)` | `Color` | Callable | Get color with alpha applied, alpha goes from 0.0f to 1.0f |
| `ColorAlphaBlend(dst: Color, src: Color, tint: Color)` | `Color` | Callable | Get src alpha-blended into dst color with tint |
| `ColorLerp(color1: Color, color2: Color, factor: number)` | `Color` | Callable | Get color lerp interpolation between two colors, factor [0.0f..1.0f] |
| `GetColor(hexValue: number)` | `Color` | Callable | Get Color structure from hexadecimal value |
| `GetPixelColor(srcPtr: bigint | null, format: number)` | `Color` | Callable | Get Color from a source pixel pointer of certain format |
| `SetPixelColor(dstPtr: bigint | null, color: Color, format: number)` | `void` | Callable | Set color formatted into destination pixel pointer |
| `GetPixelDataSize(width: number, height: number, format: number)` | `number` | Callable | Get pixel data size in bytes for certain format |

### Fonts and text

| Function | Returns | Availability | Meaning |
|---|---|---|---|
| `GetFontDefault()` | `Font` | Callable | Get the default Font |
| `LoadFont(fileName: string)` | `Font` | Callable | Load font from file into GPU memory (VRAM) |
| `LoadFontEx(fileName: string, fontSize: number, codepoints: bigint | null, codepointCount: number)` | `Font` | Callable | Load font from file with defined codepoints and generation size, use NULL for codepoints and 0 for codepointCount to load the default character set, font size is provided in pixels height |
| `LoadFontFromImage(image: Image, key: Color, firstChar: number)` | `Font` | Callable | Load font from Image (XNA style) |
| `LoadFontFromMemory(fileType: string, fileData: bigint | null, dataSize: number, fontSize: number, codepoints: bigint | null, codepointCount: number)` | `Font` | Callable | Load font from memory buffer, fileType refers to extension: i.e. '.ttf' |
| `IsFontValid(font: Font)` | `boolean` | Callable | Check if font is valid (font data loaded, WARNING: GPU texture not checked) |
| `LoadFontData(fileData: bigint | null, dataSize: number, fontSize: number, codepoints: bigint | null, codepointCount: number, type: number, glyphCount: bigint | null)` | `bigint | null` | Callable | Load font data for further use |
| `GenImageFontAtlas(glyphs: bigint | null, glyphRecs: bigint | null, glyphCount: number, fontSize: number, padding: number, packMethod: number)` | `Image` | Callable | Generate image font atlas using chars info |
| `UnloadFontData(glyphs: bigint | null, glyphCount: number)` | `void` | Callable | Unload font chars info data (RAM) |
| `UnloadFont(font: Font)` | `void` | Callable | Unload font from GPU memory (VRAM) |
| `ExportFontAsCode(font: Font, fileName: string)` | `boolean` | Callable | Export font as code file, returns true on success |
| `DrawFPS(posX: number, posY: number)` | `void` | Callable | Draw current FPS |
| `DrawText(text: string, posX: number, posY: number, fontSize: number, color: Color)` | `void` | Callable | Draw text (using default font) |
| `DrawTextEx(font: Font, text: string, position: Vector2, fontSize: number, spacing: number, tint: Color)` | `void` | Callable | Draw text using font and additional parameters |
| `DrawTextPro(font: Font, text: string, position: Vector2, origin: Vector2, rotation: number, fontSize: number, spacing: number, tint: Color)` | `void` | Callable | Draw text using Font and pro parameters (rotation) |
| `DrawTextCodepoint(font: Font, codepoint: number, position: Vector2, fontSize: number, tint: Color)` | `void` | Callable | Draw one character (codepoint) |
| `DrawTextCodepoints(font: Font, codepoints: bigint | null, codepointCount: number, position: Vector2, fontSize: number, spacing: number, tint: Color)` | `void` | Callable | Draw multiple characters (codepoint) |
| `SetTextLineSpacing(spacing: number)` | `void` | Callable | Set vertical line spacing when drawing with line-breaks |
| `MeasureText(text: string, fontSize: number)` | `number` | Callable | Measure string width for default font |
| `MeasureTextEx(font: Font, text: string, fontSize: number, spacing: number)` | `Vector2` | Callable | Measure string size for Font |
| `MeasureTextCodepoints(font: Font, codepoints: bigint | null, length: number, fontSize: number, spacing: number)` | `Vector2` | Callable | Measure string size for an existing array of codepoints for Font |
| `GetGlyphIndex(font: Font, codepoint: number)` | `number` | Callable | Get glyph index position in font for a codepoint (unicode character), fallback to '?' if not found |
| `GetGlyphInfo(font: Font, codepoint: number)` | `GlyphInfo` | Callable | Get glyph font info data for a codepoint (unicode character), fallback to '?' if not found |
| `GetGlyphAtlasRec(font: Font, codepoint: number)` | `Rectangle` | Callable | Get glyph rectangle in font atlas for a codepoint (unicode character), fallback to '?' if not found |
| `LoadUTF8(codepoints: bigint | null, length: number)` | `string` | Callable | Load UTF-8 text encoded from codepoints array |
| `UnloadUTF8(text: string)` | `void` | Callable | Unload UTF-8 text encoded from codepoints array |
| `LoadCodepoints(text: string, count: bigint | null)` | `bigint | null` | Callable | Load all codepoints from a UTF-8 text string, codepoints count returned by parameter |
| `UnloadCodepoints(codepoints: bigint | null)` | `void` | Callable | Unload codepoints data from memory |
| `GetCodepointCount(text: string)` | `number` | Callable | Get total number of codepoints in a UTF-8 encoded string |
| `GetCodepoint(text: string, codepointSize: bigint | null)` | `number` | Callable | Get next codepoint in a UTF-8 encoded string, 0x3f('?') is returned on failure |
| `GetCodepointNext(text: string, codepointSize: bigint | null)` | `number` | Callable | Get next codepoint in a UTF-8 encoded string, 0x3f('?') is returned on failure |
| `GetCodepointPrevious(text: string, codepointSize: bigint | null)` | `number` | Callable | Get previous codepoint in a UTF-8 encoded string, 0x3f('?') is returned on failure |
| `CodepointToUTF8(codepoint: number, utf8Size: bigint | null)` | `string` | Callable | Encode one codepoint into UTF-8 byte array (array length returned as parameter) |
| `LoadTextLines(text: string, count: bigint | null)` | `bigint | null` | Callable | Load text as separate lines ('\n') |
| `UnloadTextLines(text: bigint | null, lineCount: number)` | `void` | Callable | Unload text lines |
| `TextCopy(dst: string, src: string)` | `number` | Callable | Copy one string to another, returns bytes copied |
| `TextIsEqual(text1: string, text2: string)` | `boolean` | Callable | Check if two text strings are equal |
| `TextLength(text: string)` | `number` | Callable | Get text length, checks for '\0' ending |
| `TextFormat(text: string, ...args: unknown[])` | `string` | **Not callable** | Text formatting with variables (sprintf() style) |
| `TextSubtext(text: string, position: number, length: number)` | `string` | Callable | Get a piece of a text string |
| `TextRemoveSpaces(text: string)` | `string` | Callable | Remove text spaces, concat words |
| `GetTextBetween(text: string, begin: string, end: string)` | `string` | Callable | Get text between two strings |
| `TextReplace(text: string, search: string, replacement: string)` | `string` | Callable | Replace text string with new string |
| `TextReplaceAlloc(text: string, search: string, replacement: string)` | `string` | Callable | Replace text string with new string, memory must be MemFree() |
| `TextReplaceBetween(text: string, begin: string, end: string, replacement: string)` | `string` | Callable | Replace text between two specific strings |
| `TextReplaceBetweenAlloc(text: string, begin: string, end: string, replacement: string)` | `string` | Callable | Replace text between two specific strings, memory must be MemFree() |
| `TextInsert(text: string, insert: string, position: number)` | `string` | Callable | Insert text in a defined byte position |
| `TextInsertAlloc(text: string, insert: string, position: number)` | `string` | Callable | Insert text in a defined byte position, memory must be MemFree() |
| `TextJoin(textList: bigint | null, count: number, delimiter: string)` | `string` | Callable | Join text strings with delimiter |
| `TextSplit(text: string, delimiter: number, count: bigint | null)` | `bigint | null` | Callable | Split text into multiple strings, using MAX_TEXTSPLIT_COUNT static strings |
| `TextAppend(text: string, append: string, position: bigint | null)` | `void` | Callable | Append text at specific position and move cursor |
| `TextFindIndex(text: string, search: string)` | `number` | Callable | Find first text occurrence within a string, -1 if not found |
| `TextToUpper(text: string)` | `string` | Callable | Get upper case version of provided string |
| `TextToLower(text: string)` | `string` | Callable | Get lower case version of provided string |
| `TextToPascal(text: string)` | `string` | Callable | Get Pascal case notation version of provided string |
| `TextToSnake(text: string)` | `string` | Callable | Get Snake case notation version of provided string |
| `TextToCamel(text: string)` | `string` | Callable | Get Camel case notation version of provided string |
| `TextToInteger(text: string)` | `number` | Callable | Get integer value from text |
| `TextToFloat(text: string)` | `number` | Callable | Get float value from text |

### 3D shapes, models, meshes, and collision

| Function | Returns | Availability | Meaning |
|---|---|---|---|
| `DrawLine3D(startPos: Vector3, endPos: Vector3, color: Color)` | `void` | Callable | Draw a line in 3D world space |
| `DrawPoint3D(position: Vector3, color: Color)` | `void` | Callable | Draw a point in 3D space, actually a small line |
| `DrawCircle3D(center: Vector3, radius: number, rotationAxis: Vector3, rotationAngle: number, color: Color)` | `void` | Callable | Draw a circle in 3D world space |
| `DrawTriangle3D(v1: Vector3, v2: Vector3, v3: Vector3, color: Color)` | `void` | Callable | Draw a color-filled triangle, counter-clockwise vertex order |
| `DrawTriangleStrip3D(points: bigint | null, pointCount: number, color: Color)` | `void` | Callable | Draw a triangle strip defined by points |
| `DrawCube(position: Vector3, width: number, height: number, length: number, color: Color)` | `void` | Callable | Draw cube |
| `DrawCubeV(position: Vector3, size: Vector3, color: Color)` | `void` | Callable | Draw cube (Vector version) |
| `DrawCubeWires(position: Vector3, width: number, height: number, length: number, color: Color)` | `void` | Callable | Draw cube wires |
| `DrawCubeWiresV(position: Vector3, size: Vector3, color: Color)` | `void` | Callable | Draw cube wires (Vector version) |
| `DrawSphere(centerPos: Vector3, radius: number, color: Color)` | `void` | Callable | Draw sphere |
| `DrawSphereEx(centerPos: Vector3, radius: number, rings: number, slices: number, color: Color)` | `void` | Callable | Draw sphere with defined rings and slices |
| `DrawSphereWires(centerPos: Vector3, radius: number, rings: number, slices: number, color: Color)` | `void` | Callable | Draw sphere wires |
| `DrawCylinder(position: Vector3, radiusTop: number, radiusBottom: number, height: number, sides: number, color: Color)` | `void` | Callable | Draw a cylinder/cone |
| `DrawCylinderEx(startPos: Vector3, endPos: Vector3, startRadius: number, endRadius: number, sides: number, color: Color)` | `void` | Callable | Draw a cylinder with base at startPos and top at endPos |
| `DrawCylinderWires(position: Vector3, radiusTop: number, radiusBottom: number, height: number, sides: number, color: Color)` | `void` | Callable | Draw a cylinder/cone wires |
| `DrawCylinderWiresEx(startPos: Vector3, endPos: Vector3, startRadius: number, endRadius: number, sides: number, color: Color)` | `void` | Callable | Draw a cylinder wires with base at startPos and top at endPos |
| `DrawCapsule(startPos: Vector3, endPos: Vector3, radius: number, rings: number, slices: number, color: Color)` | `void` | Callable | Draw a capsule with the center of its sphere caps at startPos and endPos |
| `DrawCapsuleWires(startPos: Vector3, endPos: Vector3, radius: number, rings: number, slices: number, color: Color)` | `void` | Callable | Draw capsule wireframe with the center of its sphere caps at startPos and endPos |
| `DrawPlane(centerPos: Vector3, size: Vector2, color: Color)` | `void` | Callable | Draw a plane XZ |
| `DrawRay(ray: Ray, color: Color)` | `void` | Callable | Draw a ray line |
| `DrawGrid(slices: number, spacing: number)` | `void` | Callable | Draw a grid (centered at (0, 0, 0)) |
| `LoadModel(fileName: string)` | `Model` | Callable | Load model from files (meshes and materials) |
| `LoadModelFromMesh(mesh: Mesh)` | `Model` | Callable | Load model from generated mesh (default material) |
| `IsModelValid(model: Model)` | `boolean` | Callable | Check if model is valid (loaded in GPU, VAO/VBOs) |
| `UnloadModel(model: Model)` | `void` | Callable | Unload model (including meshes) from memory (RAM and/or VRAM) |
| `GetModelBoundingBox(model: Model)` | `BoundingBox` | Callable | Compute model bounding box limits (considers all meshes) |
| `DrawModel(model: Model, position: Vector3, scale: number, tint: Color)` | `void` | Callable | Draw a model (with texture if set) |
| `DrawModelEx(model: Model, position: Vector3, rotationAxis: Vector3, rotationAngle: number, scale: Vector3, tint: Color)` | `void` | Callable | Draw a model with custom transform |
| `DrawModelWires(model: Model, position: Vector3, scale: number, tint: Color)` | `void` | Callable | Draw a model wires (with texture if set) |
| `DrawModelWiresEx(model: Model, position: Vector3, rotationAxis: Vector3, rotationAngle: number, scale: Vector3, tint: Color)` | `void` | Callable | Draw a model wires with custom transform |
| `DrawBoundingBox(box: BoundingBox, color: Color)` | `void` | Callable | Draw bounding box (wires) |
| `DrawBillboard(camera: Camera, texture: Texture2D, position: Vector3, scale: number, tint: Color)` | `void` | Callable | Draw a billboard texture |
| `DrawBillboardRec(camera: Camera, texture: Texture2D, rec: Rectangle, position: Vector3, size: Vector2, tint: Color)` | `void` | Callable | Draw a billboard texture defined by rectangle |
| `DrawBillboardPro(camera: Camera, texture: Texture2D, rec: Rectangle, position: Vector3, up: Vector3, size: Vector2, origin: Vector2, rotation: number, tint: Color)` | `void` | Callable | Draw a billboard texture defined by source rectangle with scaling and rotation |
| `UploadMesh(mesh: bigint | null, dynamic: boolean)` | `void` | Callable | Upload mesh vertex data in GPU and provide VAO/VBO ids |
| `UpdateMeshBuffer(mesh: Mesh, index: number, data: bigint | null, dataSize: number, offset: number)` | `void` | Callable | Update mesh vertex data in GPU for a specific buffer index |
| `UnloadMesh(mesh: Mesh)` | `void` | Callable | Unload mesh data from CPU and GPU |
| `DrawMesh(mesh: Mesh, material: Material, transform: Matrix)` | `void` | Callable | Draw a 3d mesh with material and transform |
| `DrawMeshInstanced(mesh: Mesh, material: Material, transforms: bigint | null, instances: number)` | `void` | Callable | Draw multiple mesh instances with material and different transforms |
| `GetMeshBoundingBox(mesh: Mesh)` | `BoundingBox` | Callable | Compute mesh bounding box limits |
| `GenMeshTangents(mesh: bigint | null)` | `void` | Callable | Compute mesh tangents |
| `ExportMesh(mesh: Mesh, fileName: string)` | `boolean` | Callable | Export mesh data to file, returns true on success |
| `ExportMeshAsCode(mesh: Mesh, fileName: string)` | `boolean` | Callable | Export mesh as code file (.h) defining multiple arrays of vertex attributes |
| `GenMeshPoly(sides: number, radius: number)` | `Mesh` | Callable | Generate polygonal mesh |
| `GenMeshPlane(width: number, length: number, resX: number, resZ: number)` | `Mesh` | Callable | Generate plane mesh (with subdivisions) |
| `GenMeshCube(width: number, height: number, length: number)` | `Mesh` | Callable | Generate cuboid mesh |
| `GenMeshSphere(radius: number, rings: number, slices: number)` | `Mesh` | Callable | Generate sphere mesh (standard sphere) |
| `GenMeshHemiSphere(radius: number, rings: number, slices: number)` | `Mesh` | Callable | Generate half-sphere mesh (no bottom cap) |
| `GenMeshCylinder(radius: number, height: number, slices: number)` | `Mesh` | Callable | Generate cylinder mesh |
| `GenMeshCone(radius: number, height: number, slices: number)` | `Mesh` | Callable | Generate cone/pyramid mesh |
| `GenMeshTorus(radius: number, size: number, radSeg: number, sides: number)` | `Mesh` | Callable | Generate torus mesh |
| `GenMeshKnot(radius: number, size: number, radSeg: number, sides: number)` | `Mesh` | Callable | Generate trefoil knot mesh |
| `GenMeshHeightmap(heightmap: Image, size: Vector3)` | `Mesh` | Callable | Generate heightmap mesh from image data |
| `GenMeshCubicmap(cubicmap: Image, cubeSize: Vector3)` | `Mesh` | Callable | Generate cubes-based map mesh from image data |
| `LoadMaterials(fileName: string, materialCount: bigint | null)` | `bigint | null` | Callable | Load materials from model file |
| `LoadMaterialDefault()` | `Material` | Callable | Load default material (Supports: DIFFUSE, SPECULAR, NORMAL maps) |
| `IsMaterialValid(material: Material)` | `boolean` | Callable | Check if material is valid (shader assigned, map textures loaded in GPU) |
| `UnloadMaterial(material: Material)` | `void` | Callable | Unload material from GPU memory (VRAM) |
| `SetMaterialTexture(material: bigint | null, mapType: number, texture: Texture2D)` | `void` | Callable | Set texture for a material map type (MATERIAL_MAP_DIFFUSE, MATERIAL_MAP_SPECULAR...) |
| `SetModelMeshMaterial(model: bigint | null, meshId: number, materialId: number)` | `void` | Callable | Set material for a mesh |
| `LoadModelAnimations(fileName: string, animCount: bigint | null)` | `bigint | null` | Callable | Load model animations from file |
| `UpdateModelAnimation(model: Model, anim: ModelAnimation, frame: number)` | `void` | Callable | Update model animation pose (vertex buffers and bone matrices) |
| `UpdateModelAnimationEx(model: Model, animA: ModelAnimation, frameA: number, animB: ModelAnimation, frameB: number, blend: number)` | `void` | Callable | Update model animation pose, blending two animations |
| `UnloadModelAnimations(animations: bigint | null, animCount: number)` | `void` | Callable | Unload animation array data |
| `IsModelAnimationValid(model: Model, anim: ModelAnimation)` | `boolean` | Callable | Check model animation skeleton match |
| `CheckCollisionSpheres(center1: Vector3, radius1: number, center2: Vector3, radius2: number)` | `boolean` | Callable | Check collision between two spheres |
| `CheckCollisionBoxes(box1: BoundingBox, box2: BoundingBox)` | `boolean` | Callable | Check collision between two bounding boxes |
| `CheckCollisionBoxSphere(box: BoundingBox, center: Vector3, radius: number)` | `boolean` | Callable | Check collision between box and sphere |
| `GetRayCollisionSphere(ray: Ray, center: Vector3, radius: number)` | `RayCollision` | Callable | Get collision info between ray and sphere |
| `GetRayCollisionBox(ray: Ray, box: BoundingBox)` | `RayCollision` | Callable | Get collision info between ray and box |
| `GetRayCollisionMesh(ray: Ray, mesh: Mesh, transform: Matrix)` | `RayCollision` | Callable | Get collision info between ray and mesh |
| `GetRayCollisionTriangle(ray: Ray, p1: Vector3, p2: Vector3, p3: Vector3)` | `RayCollision` | Callable | Get collision info between ray and triangle |
| `GetRayCollisionQuad(ray: Ray, p1: Vector3, p2: Vector3, p3: Vector3, p4: Vector3)` | `RayCollision` | Callable | Get collision info between ray and quad |
| `InitAudioDevice()` | `void` | Callable | Initialize audio device and context |
| `CloseAudioDevice()` | `void` | Callable | Close the audio device and context |
| `IsAudioDeviceReady()` | `boolean` | Callable | Check if audio device has been initialized successfully |
| `SetMasterVolume(volume: number)` | `void` | Callable | Set master volume (listener) |
| `GetMasterVolume()` | `number` | Callable | Get master volume (listener) |

### Audio

| Function | Returns | Availability | Meaning |
|---|---|---|---|
| `LoadWave(fileName: string)` | `Wave` | Callable | Load wave data from file |
| `LoadWaveFromMemory(fileType: string, fileData: bigint | null, dataSize: number)` | `Wave` | Callable | Load wave from memory buffer, fileType refers to extension: i.e. '.wav' |
| `IsWaveValid(wave: Wave)` | `boolean` | Callable | Check if wave data is valid (data loaded and parameters) |
| `LoadSound(fileName: string)` | `Sound` | Callable | Load sound from file |
| `LoadSoundFromWave(wave: Wave)` | `Sound` | Callable | Load sound from wave data |
| `LoadSoundAlias(source: Sound)` | `Sound` | Callable | Load sound alias, new sound that shares the same sample data as the source sound, does not own the sound data |
| `IsSoundValid(sound: Sound)` | `boolean` | Callable | Check if sound is valid (data loaded and buffers initialized) |
| `UpdateSound(sound: Sound, data: bigint | null, frameCount: number)` | `void` | Callable | Update sound buffer with new data (default data format: 32 bit float, stereo) |
| `UnloadWave(wave: Wave)` | `void` | Callable | Unload wave data |
| `UnloadSound(sound: Sound)` | `void` | Callable | Unload sound |
| `UnloadSoundAlias(alias: Sound)` | `void` | Callable | Unload sound alias (does not deallocate sample data) |
| `ExportWave(wave: Wave, fileName: string)` | `boolean` | Callable | Export wave data to file, returns true on success |
| `ExportWaveAsCode(wave: Wave, fileName: string)` | `boolean` | Callable | Export wave sample data to code (.h), returns true on success |
| `PlaySound(sound: Sound)` | `void` | Callable | Play a sound |
| `StopSound(sound: Sound)` | `void` | Callable | Stop playing a sound |
| `PauseSound(sound: Sound)` | `void` | Callable | Pause a sound |
| `ResumeSound(sound: Sound)` | `void` | Callable | Resume a paused sound |
| `IsSoundPlaying(sound: Sound)` | `boolean` | Callable | Check if sound is currently playing |
| `SetSoundVolume(sound: Sound, volume: number)` | `void` | Callable | Set volume for a sound (1.0 is max level) |
| `SetSoundPitch(sound: Sound, pitch: number)` | `void` | Callable | Set pitch for a sound (1.0 is base level) |
| `SetSoundPan(sound: Sound, pan: number)` | `void` | Callable | Set pan for a sound (-1.0 left, 0.0 center, 1.0 right) |
| `WaveCopy(wave: Wave)` | `Wave` | Callable | Copy a wave to a new wave |
| `WaveCrop(wave: bigint | null, initFrame: number, finalFrame: number)` | `void` | Callable | Crop a wave to defined frames range |
| `WaveFormat(wave: bigint | null, sampleRate: number, sampleSize: number, channels: number)` | `void` | Callable | Convert wave data to desired format |
| `LoadWaveSamples(wave: Wave)` | `bigint | null` | Callable | Load samples data from wave as a 32bit float data array |
| `UnloadWaveSamples(samples: bigint | null)` | `void` | Callable | Unload samples data loaded with LoadWaveSamples() |
| `LoadMusicStream(fileName: string)` | `Music` | Callable | Load music stream from file |
| `LoadMusicStreamFromMemory(fileType: string, data: bigint | null, dataSize: number)` | `Music` | Callable | Load music stream from data |
| `IsMusicValid(music: Music)` | `boolean` | Callable | Check if music stream is valid (context and buffers initialized) |
| `UnloadMusicStream(music: Music)` | `void` | Callable | Unload music stream |
| `PlayMusicStream(music: Music)` | `void` | Callable | Start music playing |
| `IsMusicStreamPlaying(music: Music)` | `boolean` | Callable | Check if music is playing |
| `UpdateMusicStream(music: Music)` | `void` | Callable | Update buffers for music streaming |
| `StopMusicStream(music: Music)` | `void` | Callable | Stop music playing |
| `PauseMusicStream(music: Music)` | `void` | Callable | Pause music playing |
| `ResumeMusicStream(music: Music)` | `void` | Callable | Resume playing paused music |
| `SeekMusicStream(music: Music, position: number)` | `void` | Callable | Seek music to a position (in seconds) |
| `SetMusicVolume(music: Music, volume: number)` | `void` | Callable | Set volume for music (1.0 is max level) |
| `SetMusicPitch(music: Music, pitch: number)` | `void` | Callable | Set pitch for music (1.0 is base level) |
| `SetMusicPan(music: Music, pan: number)` | `void` | Callable | Set pan for music (-1.0 left, 0.0 center, 1.0 right) |
| `GetMusicTimeLength(music: Music)` | `number` | Callable | Get music time length (in seconds) |
| `GetMusicTimePlayed(music: Music)` | `number` | Callable | Get current music time played (in seconds) |
| `LoadAudioStream(sampleRate: number, sampleSize: number, channels: number)` | `AudioStream` | Callable | Load audio stream (to stream raw audio pcm data) |
| `IsAudioStreamValid(stream: AudioStream)` | `boolean` | Callable | Check if an audio stream is valid (buffers initialized) |
| `UnloadAudioStream(stream: AudioStream)` | `void` | Callable | Unload audio stream and free memory |
| `UpdateAudioStream(stream: AudioStream, data: bigint | null, frameCount: number)` | `void` | Callable | Update audio stream buffers with data |
| `IsAudioStreamProcessed(stream: AudioStream)` | `boolean` | Callable | Check if any audio stream buffers requires refill |
| `PlayAudioStream(stream: AudioStream)` | `void` | Callable | Play audio stream |
| `PauseAudioStream(stream: AudioStream)` | `void` | Callable | Pause audio stream |
| `ResumeAudioStream(stream: AudioStream)` | `void` | Callable | Resume audio stream |
| `IsAudioStreamPlaying(stream: AudioStream)` | `boolean` | Callable | Check if audio stream is playing |
| `StopAudioStream(stream: AudioStream)` | `void` | Callable | Stop audio stream |
| `SetAudioStreamVolume(stream: AudioStream, volume: number)` | `void` | Callable | Set volume for audio stream (1.0 is max level) |
| `SetAudioStreamPitch(stream: AudioStream, pitch: number)` | `void` | Callable | Set pitch for audio stream (1.0 is base level) |
| `SetAudioStreamPan(stream: AudioStream, pan: number)` | `void` | Callable | Set pan for audio stream (-1.0 left, 0.0 center, 1.0 right) |
| `SetAudioStreamBufferSizeDefault(size: number)` | `void` | Callable | Default size for new audio streams |
| `SetAudioStreamCallback(stream: AudioStream, callback: AudioCallback)` | `void` | **Not callable** | Audio thread callback to request new data |
| `AttachAudioStreamProcessor(stream: AudioStream, processor: AudioCallback)` | `void` | **Not callable** | Attach audio stream processor to stream, receives frames x 2 samples as 'float' (stereo) |
| `DetachAudioStreamProcessor(stream: AudioStream, processor: AudioCallback)` | `void` | **Not callable** | Detach audio stream processor from stream |
| `AttachAudioMixedProcessor(processor: AudioCallback)` | `void` | **Not callable** | Attach audio stream processor to the entire audio pipeline, receives frames x 2 samples as 'float' (stereo) |
| `DetachAudioMixedProcessor(processor: AudioCallback)` | `void` | **Not callable** | Detach audio stream processor from the entire audio pipeline |

---

Inventory totals: **35 structures**, **6 callbacks**, **21 enums**, **56 parsed constants**, and **613 functions**.

For editor completion, keep `types/raylib.d.ts` in the generated project and enable JavaScript checking through its `jsconfig.json`.
