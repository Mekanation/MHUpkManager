# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build

The repo SDK is pinned to `.NET SDK 8.0.419` via `global.json`. The stable local build command is:

```powershell
dotnet build src/MHUpkManager.csproj -c Debug -m:1
```

The `-m:1` flag disables parallel build; omitting it is flaky during project-graph evaluation on this machine. Release builds use `-c Release`. Known post-build warnings are nullable-annotation warnings in `src/EnemyConverter/EnemyAvatarPrototypeFieldIdBridgeAnalyzer.cs` — these are not regressions.

There are no test projects and no lint/format tooling configured.

## What This Is

MHUpkManager is a Windows Forms/.NET 8 desktop tool for modifying Marvel Heroes Omega game assets stored in Unreal Engine 3 `.upk` package files. Core capabilities: skeletal mesh FBX import/export, texture injection, 3D preview, and skeletal retargeting.

## Project Layout

```
src/                    WinForms GUI application (main entry: Program.cs → MainForm.cs)
UpkManager/             Core UPK format parser library (headers, export/import tables, compression)
DDSLib/                 DDS texture format read/write
SharpGL/                OpenGL wrapper (SharpGL + SharpGL.WinForms)
UpkIndexGenerator/      Console tool to build SQLite/MessagePack UPK indices
tools/                  Diagnostic console probes (MeshParseProbe, SkeletalExportProbe)
archive/docs/           Development worklogs and implementation notes
```

## Architecture

### MainForm.cs

Central coordinator (~4600 lines). Manages UPK file load/navigation, hosts all sub-panels, and owns the export-tree context menus. Most UI entry points begin here.

### UpkManager library

Low-level UPK binary parser. Key types:
- `UnrealHeader` — parses package headers, export/import tables; has staged read methods (`ReadTablesAsync`, `ReadExportObjectAsync`) used by Mesh Preview to avoid loading full objects just to enumerate exports
- `UpkFileRepository` — centralized UPK file access
- `Models/UpkObjects/` — UE3 object types (USkeletalMesh, UTexture2D, etc.)
- Compression via `lzo2_64.dll` + `msvcr100.dll` native binaries

### Mesh Import Pipeline (`src/MeshImporter/`)

**Two parallel importers exist intentionally — do not merge them:**
- `src/Model/Import/` — original importer, proven in-game valid, kept as reference
- `src/MeshImporter/` — new standalone importer being built to the same spec without depending on the original

The new importer flow: `FbxMeshImporter` (Assimp) → `NeutralMesh` → `BoneRemapper` + `WeightNormalizer` → `UE3VertexBuilder` / `UE3LodBuilder` / `UE3LodSerializer` → `UpkSkeletalMeshInjector`. The injector patches only the target LOD byte range inside the original export buffer, then repacks export-table offsets and writes `<file>.upk.bak` before overwrite.

**Critical serializer constraints:**
- Skeleton hierarchy is never replaced — only geometry/weights
- `RequiredBones` order is always preserved from the original
- Material indices and section ordering are preserved from the original LOD
- Weights are forced to exactly 4 influences summing to 255
- `RawPointIndices` bulk-data uses a sentinel for zero-length; absolute offsets are patched post-injection

### Mesh Export (`src/Model/SkeletalFbxExporter.cs`)

Exports binary FBX (not ASCII — Blender 5.0+ rejects ASCII FBX). Uses Assimp. The `System.Numerics.Matrix4x4 → Assimp.Matrix4x4` conversion requires a transpose before building the Assimp matrix — without this, bone heads collapse to the origin in Blender.

### Retargeting (`src/Retargeting/`)

One-click bind to original MHO skeleton. Pipeline order:
1. Scale to reference mesh
2. `ReferenceAlignmentProcessor` — translate into original MHO body frame
3. Pose conform
4. Weight transfer
5. Bind to original MHO skeleton

### Mesh Preview (`src/MeshPreview/`)

Real-time 3D preview with two renderer backends:
- `OpenTkMeshPreviewViewport` — OpenGL via OpenTK
- `VorticeMeshPreviewViewport` — Direct3D 11 via Vortice; compiles HLSL shaders to temp files at runtime

Both share `IMeshPreviewViewportBackend`. The dominant load-time cost is `UE3ToPreviewMeshConverter.Convert(...)` (not UPK I/O). `BuildUvSeams` is a suspected hotspot — seams are detected from UV-differing edges across adjacent triangles.

### Texture Injection (`src/TexturePreview/Services/TexturePreviewInjector.cs`)

Requires a loaded `TextureFileCacheManifest.bin`. Converts source image to target UE3 format (DXT1/DXT5/A8R8G8B8) and generates matching mip levels. If replacement payload exceeds original cache allocation, new mip chain is appended to end of `.tfc` and manifest offsets are updated. Injection is blocked unless source dimensions exactly match target `SizeX`/`SizeY` (UPK Texture2D size metadata is not rewritten).

## Key Dependencies

| Package | Purpose |
|---|---|
| AssimpNet 5.0.0-beta1 | FBX import/export |
| OpenTK 4.9.3 + OpenTK.GLControl | OpenGL preview |
| Vortice.Direct3D11/DXGI/D3DCompiler 3.6.2 | D3D11 preview |
| MessagePack 3.1.4 | Fast binary cache serialization |
| Microsoft.EF.Sqlite 9.0.8 | UPK index database |
| Be.Windows.Forms.HexBox | Hex viewer control |

## Diagnostic Logs

Failed imports write timestamped logs to:
- `Desktop\MHUpkManager_ImportLogs\fbx-import-YYYYMMDD-HHMMSS.log`
- `Desktop\MHUpkManager_ImportLogs\` (retarget one-click diagnostics)
- `Desktop\MHUpkManager_TextureLogs\` (texture injection diagnostics)
- `Desktop\MHUpkManager_StartupLogs\` (startup crash logs)
