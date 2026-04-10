# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**YooAsset** (`com.tuyoogame.yooasset`, v2.3.18) is a Unity 3D resource management system (Unity Package) that handles asset bundle building, hot updates, and runtime asset loading for commercial games. Minimum Unity version: 2019.4.

Official docs: https://www.yooasset.com/

## Project Structure

This is a Unity Package embedded under `Assets/YooAsset/`. All source lives in two top-level folders:

- `Runtime/` — runtime C# code (included in game builds)
- `Editor/` — editor-only tools and build pipeline (excluded from builds)
- `Samples~/` — optional sample projects (Space Shooter, Mini Game, Extension Sample, UniTask Sample, Test Sample)

## Runtime Architecture

The runtime is organized around these key systems:

### Entry Point
- `Runtime/YooAssets.cs` — static main API; manages `ResourcePackage` instances and the `YooAssetsDriver` MonoBehaviour
- `Runtime/YooAssetsExtension.cs` — extension methods on the static API

### ResourcePackage
`Runtime/ResourcePackage/ResourcePackage.cs` — the central abstraction. Each package encapsulates a `ResourceManager`, an `IPlayMode` implementation, and an `IBundleQuery`. Packages are created via `YooAssets.CreatePackage()` and initialized with one of four play modes.

**Play modes** (in `Runtime/ResourcePackage/PlayMode/`):
- `EditorSimulateMode` — no bundle build required; simulates using editor asset database
- `OfflinePlayMode` — uses only built-in (streaming assets) resources
- `HostPlayMode` — downloads from a CDN/server, caches locally (the primary production mode)
- `WebPlayMode` — WebGL + mini-game platforms

### FileSystem
`Runtime/FileSystem/` provides 6 built-in implementations of `IFileSystem`:
- `DefaultBuildinFileSystem` — reads from StreamingAssets
- `DefaultCacheFileSystem` — persistent local cache for downloaded bundles
- `DefaultEditorFileSystem` — editor simulation
- `DefaultUnpackFileSystem` — decompression for built-in packages
- `DefaultWebRemoteFileSystem` / `DefaultWebServerFileSystem` — WebGL variants

Custom file systems implement `IFileSystem` in `Runtime/FileSystem/Interface/IFileSystem.cs`.

### Resource Loading
- `Runtime/ResourceManager/` — loads assets via `Provider` objects; returns typed `Handle` objects to callers
- `Runtime/OperationSystem/` — coroutine-based async operation runner; all async work in YooAsset uses `GameAsyncOperation` (subclasses of `AsyncOperationBase`)

### Services (Extension Points)
`Runtime/Services/` contains interfaces for customization:
- `IRemoteServices` — CDN URL resolution
- `IDecryptionServices` / `IEncryptionServices` — bundle encryption
- `IManifestProcessServices` / `IManifestRestoreServices` — manifest transformation
- `ICopyLocalFileServices` — copying builtin files

### DiagnosticSystem
`Runtime/DiagnosticSystem/` — remote debugger that connects the editor's `AssetBundleDebuggerWindow` to a running player over `EditorConnection`.

## Editor Architecture

### AssetBundleCollector
`Editor/AssetBundleCollector/` — configures which assets get collected into which bundles. Settings saved as `AssetBundleCollectorSetting` ScriptableObject. Rules implement interfaces in `CollectRules/` and `DefaultRules/`.

### AssetBundleBuilder
`Editor/AssetBundleBuilder/` — build window and pipeline execution.

**Build pipelines** (in `BuildPipeline/`):
- `BuiltinBuildPipeline` — Unity's classic `BuildPipeline.BuildAssetBundles`
- `ScriptableBuildPipeline` — Unity SBP (`com.unity.scriptablebuildpipeline`)
- `RawFileBuildPipeline` — copies raw files as-is into the output
- `EditorSimulateBuildPipeline` — generates the simulate manifest without building bundles

The build system in `BuildSystem/` uses a task-runner pattern: `BuildRunner` executes a list of `IBuildTask` steps, sharing data through a `BuildContext` dictionary.

### Other Editor Windows
- `AssetBundleDebugger/` — runtime asset loading debugger
- `AssetBundleReporter/` — post-build bundle report
- `AssetArtScanner/` / `AssetArtReporter/` — art asset quality analysis

## Key Conventions

- All async operations return a subclass of `GameAsyncOperation` (or `AsyncOperationBase`). They are driven by `OperationSystem` each frame; you `yield return` them in coroutines or `await` them with UniTask.
- `EFileClearMode` enum governs cache cleanup strategies; new modes are added here.
- Build parameters live in `BuildParameters.cs` (base) and pipeline-specific subclasses (`BuiltinBuildParameters`, etc.).
- Editor UI uses **UIElements** (UXML + USS), not IMGUI, for all major windows.
