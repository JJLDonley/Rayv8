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
editor completion for the JavaScript raylib API.

## Licenses

RayV8 executables include third-party open-source software. The applicable
license texts are distributed in the `licenses/` directory. They must remain
with redistributed copies of the toolkit where their terms require it.

These permissive licenses allow commercial and closed-source games. When
distributing a game executable built with RayV8, include the `licenses/`
directory with the game, or reproduce the applicable notices in its installer,
documentation, or other accompanying materials.
