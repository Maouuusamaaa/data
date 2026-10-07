# Distant Horizons 3.3.4 — Static Analysis

## 1. Sample

| Field | Value |
|---|---|
| File | `DistantHorizons-3.3.4-26.3-fabric-neoforge.jar` |
| SHA-256 | `ba3e5c85a765a109219831df99b99ef5cfcdee05b2dfa6796116993bd48d3947` |
| Size | 28,476,894 bytes |
| ZIP entries | 2,196 |
| Main mod ID | `distanthorizons` |
| Version | 3.3.4 |
| Minecraft | 26.3 |
| Java | >=25 |
| License metadata | LGPL-3 |

This is a static inspection of the supplied JAR. It does not execute the mod.

## 2. High-level architecture

Distant Horizons is structured as a terrain Level-of-Detail (LOD) renderer. Its metadata describes the core purpose as generating and rendering simplified terrain beyond Minecraft's normal view distance.

The package layout separates platform adapters from a shared core:

- `com.seibel.distanthorizons.fabric`
- `com.seibel.distanthorizons.neoforge`
- `com.seibel.distanthorizons.common`
- `com.seibel.distanthorizons.core`
- `com.seibel.distanthorizons.api`

There are approximately 1,019 Distant Horizons class files in the supplied JAR.

## 3. Fabric / NeoForge integration

The same distribution contains both Fabric and NeoForge metadata.

Fabric:

- entrypoint: `com.seibel.distanthorizons.fabric.FabricMain`
- mixin configuration: `DistantHorizons.fabric.mixins.json`
- Minecraft dependency: 26.3
- Java minimum: 25

NeoForge:

- mod ID: `distanthorizons`
- version: 3.3.4
- Minecraft range: `[26.3]`
- mixin configuration: `DistantHorizons.neoforge.mixins.json`

The JAR also embeds a large set of Fabric API JARs under `META-INF/jars/`.

## 4. Mixin / render-pipeline integration

The Fabric mixin configuration contains client hooks including:

- `MixinClientLevel`
- `MixinGameRenderer`
- `MixinFogRenderer`
- `MixinLevelRenderer`
- `MixinChunkSectionsToRender`
- `MixinLightTexture`
- `MixinProjectionMatrixBuffer`
- `MixinIrisFrameBuffer`
- `MixinMinecraft`
- `MixinOptionsScreen`

Server-side hooks include chunk generation/map, entity/player, tick and background-thread integration.

This indicates that DH participates in both world/chunk lifecycle and the client rendering pipeline rather than merely changing the vanilla render-distance number.

## 5. Rendering architecture

The JAR contains a substantial OpenGL renderer layer.

Important classes/packages observed:

- `GlDhMetaRenderer`
- `GlDhTerrainRenderer`
- `GlDhRenderApiDefinition`
- `GLBuffer`
- `GLVertexBuffer`
- `GLIndexBuffer`
- `GlDhFramebuffer`
- `GlShader`
- `GlShaderProgram`
- `GlDhTerrainShaderProgram`

The shader/post-processing layer includes:

- terrain rendering
- temporal anti-aliasing (TAA)
- sharpening
- far-distance fading
- vanilla fade
- fog
- SSAO
- copy/apply passes

This suggests a pipeline closer to an independent secondary renderer/compositor than a simple chunk-distance extension.

## 6. LOD and world-generation pipeline

The class inventory exposes explicit world-generation and LOD-related components:

- `DhWorldGenerator`
- `DhChunkGenerator`
- `DhInternalServerGenerator`
- `DhRoughSurfaceGenerator`
- `EDhApiDistantGeneratorMode`
- `EDhApiGeneratorPlan`
- `IChunkGenerator`
- `IRoughGenerator`

There is also an explicit chunk-update subsystem:

- `ChunkUpdateData`
- `ChunkUpdateQueueManager`
- `WorldChunkUpdateManager`
- `ChunkPosQueue`

This supports the model:

`Minecraft world/chunk data -> generation/translation -> LOD update queue -> LOD representation -> renderer`

The exact proprietary algorithms should not be copied from the binary. The architectural observation itself is useful for independent implementation research.

## 7. Persistent storage

The JAR contains an embedded SQLite implementation under `dh_sqlite/` and native SQLite libraries for multiple platforms.

The inventory includes native SQLite binaries for:

- Linux x86/x86_64
- Linux ARM/ARMv6/ARMv7/ARM64
- Linux RISC-V64
- Linux PPC variants
- Linux musl variants
- Windows x86/x86_64/ARM variants
- macOS x86_64/ARM64
- FreeBSD variants

This strongly indicates persistent local storage is a first-class part of the architecture rather than an incidental cache.

For a future independent renderer, SQLite is a useful candidate for storing spatial LOD tiles, metadata, version information, and regeneration state.

## 8. Compression / native libraries

The JAR bundles native Zstandard JNI libraries and LZ4 libraries.

Observed examples:

- `libzstd-jni_dh-1.5.7-12`
- `liblz4-java`

These are consistent with a storage/transport path where large amounts of terrain data need compact representation.

Potential independent design lesson:

- keep hot render data in memory/GPU buffers;
- persist colder LOD levels on disk;
- compress disk representations;
- load/decompress asynchronously;
- avoid blocking the render thread.

## 9. Multithreading

The code inventory contains explicit threading infrastructure:

- `RenderThreadTaskHandler`
- `ServerThreadTaskHandler`
- `RateLimitedThreadPoolExecutor`
- `PriorityTaskPicker`
- `ThreadPoolUtil`
- `DhThreadFactory`
- `PositionalLockProvider`

There is also a public/config-facing multithreading API.

This is important for LOD systems because terrain generation, conversion, compression, disk I/O, mesh generation, and GPU upload have different latency characteristics.

A useful independent scheduling model is:

`world changes -> prioritized jobs -> worker threads -> bounded queues -> render-thread/GPU upload`

with distance/visibility and player movement influencing job priority.

## 10. GPU upload

The API/config inventory exposes:

`EDhApiGpuUploadMethod`

This confirms GPU upload is treated as a configurable subsystem. For an independent renderer, separating CPU-side LOD generation from GPU upload is an important architectural boundary.

## 11. Network / updater observations

The class inventory contains:

- `SelfUpdater`
- `GitlabGetter`
- updater GUI classes
- update-branch configuration

The bundled metadata identifies the project source/issue locations on GitLab. The static inspection found updater-related code that can query GitLab project information.

Important distinction: presence of HTTP/update functionality does not by itself indicate malicious behavior. This report does not claim a security verdict for runtime network behavior because the JAR was not executed and network traffic was not dynamically observed.

## 12. Security / trust observations

Static observations:

- The JAR contains native libraries.
- The JAR contains an auto-update subsystem.
- The JAR contains embedded third-party libraries.
- The JAR modifies Minecraft through Mixins.
- The JAR has local database/native-storage functionality.

These are architectural capabilities, not proof of malicious activity.

For stronger security verification, a separate dynamic test should monitor:

1. child process creation;
2. filesystem writes;
3. network destinations;
4. downloaded update artifacts;
5. native library loading;
6. checksum/signature verification;
7. updater execution path.

## 13. Relevance to PVR / independent renderer research

The most useful concepts to carry into independent research are architectural, not copied implementation details:

### A. Multi-resolution terrain

Use several spatial resolutions rather than rendering every distant block.

Example conceptual hierarchy:

`block -> chunk -> LOD-1 -> LOD-2 -> LOD-3 -> ...`

### B. Persistent LOD cache

Store generated distant representations so they do not have to be regenerated every launch.

Candidate schema:

`world_id, dimension, lod_level, tile_x, tile_z, version, payload, compression, last_generated`

### C. Asynchronous generation

Keep expensive generation and compression away from the render thread.

### D. Prioritized streaming

Prioritize work based on:

- camera distance;
- camera direction;
- movement velocity;
- screen-space importance;
- missing/dirty state.

### E. Independent render pipeline

Treat distant terrain as a separate render stage with its own:

- vertex/index buffers;
- shaders;
- depth/fog handling;
- post-processing/compositing;
- GPU upload queue.

### F. Compatibility boundary

Minecraft integration should be isolated behind adapters/interfaces. The DH package structure provides a useful architectural reference for this pattern.

## 14. Proposed independent PVR mapping

For the PVR-Minecraft-Research project, a conceptual mapping is:

`Bedrock/RenderDragon world data`
→ `world adapter`
→ `chunk/voxel sampler`
→ `LOD builder`
→ `compressed tile store`
→ `streaming scheduler`
→ `GPU buffer manager`
→ `PVR renderer`

The PVR implementation should use independently designed code and algorithms. Do not copy DH bytecode, decompile it into source, or redistribute its bundled proprietary assets.

## 15. Confidence levels

| Finding | Confidence |
|---|---|
| Dual Fabric/NeoForge packaging | High |
| Mixin-based Minecraft integration | High |
| Independent OpenGL terrain renderer | High |
| LOD/world-generation subsystem | High |
| SQLite persistence | High |
| Zstandard/LZ4 native support | High |
| Dedicated multithreading infrastructure | High |
| GPU upload abstraction | High |
| Auto-update subsystem | High |
| Malicious behavior | Not established |

## 16. Source sample

The analysis target is the user-supplied JAR. Project metadata inside the JAR identifies the Distant Horizons project and its source location.

This repository stores analysis notes and derived architectural observations, not a redistributed copy of the mod.

## 17. Next research targets

1. Map the complete LOD data flow from chunk ingestion to GPU submission.
2. Identify the persistent database schema and cache invalidation model from static metadata/class signatures.
3. Map renderer passes and framebuffer dependencies.
4. Characterize thread pools and priority scheduling.
5. Extract configuration defaults without reproducing implementation code.
6. Compare the architecture against the PVR research design.
7. Build an independent minimal LOD prototype for Bedrock/PVR experiments.
