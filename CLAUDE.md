# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Yourscraft is a Minecraft-style voxel game built in **Unity 6 (6000.5.9f1)** with **URP**, targeting **PC and mobile** from one codebase.

The repo is at the very start of implementation: [Assets/Scripts/PlayerController.cs](Assets/Scripts/PlayerController.cs) is still an empty `MonoBehaviour` stub and [Assets/Scenes/SampleScene.unity](Assets/Scenes/SampleScene.unity) contains only a camera, a directional light, and a global volume. Nearly all gameplay code is yet to be written.

History note: commits before `34d4ed2` ("Unity project setup") are from an **Unreal Engine 5** incarnation of this project (ZixuanCraft → OurCraft → Yourscraft). All UE content was deleted in that commit. Do not treat pre-`34d4ed2` files, `.uasset`, `Config/*.ini`, or `Source/` as reference material.

### Target features

- Player character: move, look, attack, shoot
- Endless terrain generation streamed around the player
- Placing/removing cubes at runtime
- Inventory and crafting
- Enemies: zombie, skeleton, creeper
- Livestock and pets

## Commands

Unity editor: `C:/Program Files/Unity/Hub/Editor/6000.5.9f1/Editor/Unity.exe`. CLI invocations need `-projectPath "c:/Users/szx07/repos/Yourscraft"`, and **fail while the project is open in the Editor** (Unity holds an exclusive lock on `Library/`).

```powershell
$Unity = "C:/Program Files/Unity/Hub/Editor/6000.5.9f1/Editor/Unity.exe"

# Compile-check headlessly (exit code != 0 on compile errors)
& $Unity -batchmode -quit -nographics -projectPath . -logFile -

# Run tests (no -quit with -runTests; Unity exits itself)
& $Unity -batchmode -nographics -projectPath . -logFile - `
  -runTests -testPlatform EditMode -testResults ./Temp/editmode-results.xml
& $Unity -batchmode -nographics -projectPath . -logFile - `
  -runTests -testPlatform PlayMode -testResults ./Temp/playmode-results.xml

# Single test / subset
& $Unity -batchmode -nographics -projectPath . -logFile - -runTests -testPlatform EditMode `
  -testFilter "Yourscraft.Tests.ChunkMeshTests.EmptyChunkProducesNoTriangles"
```

`com.unity.test-framework` is installed but there are **no test assemblies yet** — tests require creating `Assets/Tests/{EditMode,PlayMode}/` with an `.asmdef` referencing `UnityEngine.TestRunner` + `UnityEditor.TestRunner` before any of the above returns results.

Builds currently have no `-executeMethod` entry point; they are made from the Editor (File ▸ Build Profiles). Adding a static build method under an `Editor/` folder is the way to make builds scriptable/CI-able. `.github/workflows/` exists but is empty.

`Assembly-CSharp.csproj`, `Assembly-CSharp-firstpass.csproj`, and `Yourscraft.slnx` are Unity-generated and gitignored — never hand-edit them.

## Architecture

### Assemblies

No `.asmdef` files exist, so every script under `Assets/Scripts/` lands in the monolithic `Assembly-CSharp`, and `Assets/Plugins/` in `Assembly-CSharp-firstpass`. Introduce asmdefs when splitting subsystems (terrain / gameplay / UI / tests) — the test runner requires them regardless.

### Input — one action set, five control schemes

Input System 1.20.0 is the *only* handler (`activeInputHandler: 1`; legacy `Input` API is disabled and will not compile). [Assets/InputSystem_Actions.inputactions](Assets/InputSystem_Actions.inputactions) is registered as the **project-wide actions asset** (wired through `ProjectSettings/EditorBuildSettings.asset` → `m_configObjects`), so actions resolve globally:

```csharp
var move = InputSystem.actions.FindAction("Player/Move");
```

Wrapper-code generation is **off** in the `.meta`, so there is no generated `InputSystem_Actions` C# class — don't reference one.

- Maps: `Player` (Move, Look, Attack, Interact, Crouch, Jump, Previous, Next, Sprint) and `UI`.
- Control schemes: `Keyboard&Mouse`, `Gamepad`, `Touch`, `Joystick`, `XR`.

Since PC and mobile ship from the same build path, every new player-facing action needs bindings for keyboard/mouse, gamepad, **and** touch. Touch coverage in the template is minimal — mobile needs on-screen sticks/buttons (`OnScreenStick`/`OnScreenButton` bound to the same actions) plus schemes-aware UI that shows/hides them based on the active control scheme, rather than a parallel mobile input path.

### Rendering — per-platform URP assets

[Assets/Settings/](Assets/Settings/) holds two parallel URP stacks: `PC_RPAsset` + `PC_Renderer` and `Mobile_RPAsset` + `Mobile_Renderer`. They are selected by quality level, not by code: `QualitySettings` defines **Mobile at index 0** (Android's per-platform default) and **PC at index 1**. Rendering changes must be applied to the right tier — editing only `PC_Renderer` silently leaves mobile untouched.

### Third-party

DOTween (`Assets/Plugins/Demigiant/`) with the `DOTWEEN` scripting define set for all platforms. Cinemachine 3.1.7, AI Navigation 2.0.14 (for mob pathing), Visual Scripting, and Timeline are also in [Packages/manifest.json](Packages/manifest.json).

## Working in this repo

- **Commit `.meta` files with their assets.** Adding an asset without its `.meta` (or deleting one but not the other) breaks GUID references in every scene/prefab that points at it.
- Unity YAML assets (`.unity`, `.prefab`, `.asset`) are text but merge badly. Prefer expressing behavior in C# and wiring it at runtime over deep scene/prefab hierarchies, and avoid parallel edits to the same scene.
- **Voxel-specific:** do not represent blocks as GameObjects. Blocks are entries in a chunk's flat array; chunks build a single combined mesh (greedy or per-face culled) and are streamed in/out by player position. A per-cube-GameObject design will not survive the mobile target.
- Mobile is a first-class target: allocation in per-frame or per-chunk code paths, uncapped draw calls, and main-thread mesh generation are the constraints to design against from the start (prefer Jobs/Burst-friendly data layouts for terrain even before those packages are added).
