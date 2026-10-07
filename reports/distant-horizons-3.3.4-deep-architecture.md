# Distant Horizons 3.3.4 — Deep Static Architecture Analysis

## Scope

Target:

- `DistantHorizons-3.3.4-26.3-fabric-neoforge.jar`
- SHA-256: `ba3e5c85a765a109219831df99b99ef5cfcdee05b2dfa6796116993bd48d3947`
- 2,196 ZIP entries
- 1,019 classes under `com.seibel.distanthorizons`

This phase goes beyond package inventory. It reconstructs the responsibilities and data flow of the major classes from bytecode signatures, fields, interfaces, resource metadata, and the public Distant Horizons source/project structure.

The official project describes DH as an LOD system that renders simplified terrain outside normal Minecraft render distance. citeturn0search2turn1search9

This report does not reproduce source code. It records behavior, interfaces, relationships, and independent research implications.

---

## 1. Architectural decomposition

The implementation can be understood as these layers:

```
Minecraft / Fabric / NeoForge
          |
          v
Platform wrappers + Mixins
          |
          v
Level lifecycle
          |
          +----------------------+
          |                      |
          v                      v
Chunk/world acquisition      Multiplayer/network
          |
          v
Full terrain data
          |
          v
LOD generation / lighting
          |
          v
Persistent FullData storage
          |
          v
FullData -> RenderData
          |
          v
ColumnRenderSource / LOD quads
          |
          v
QuadTree + visibility/culling
          |
          v
GPU buffers + shaders
          |
          v
DH render passes / compositor
```

The critical architectural boundary is the separation between **world data** and **render data**. DH does not directly turn every Minecraft block into a GPU mesh at extreme distance. It first constructs compact terrain representations and then derives renderable column/quad data.

---

## 2. Bootstrap and lifecycle

### `core.Initializer`

Responsibilities:

- `preConfigInit()`: initializes dependencies that must exist before configuration is loaded.
- `postConfigInit()`: finishes initialization after configuration is available.

The initializer uses shared Minecraft wrappers rather than directly hard-coding every platform implementation.

### Platform entrypoints

Fabric and NeoForge each provide their own entrypoint/wrapper layer. The common/core code then consumes wrapper interfaces.

This is deliberate architecture:

```
Fabric implementation ----+
                          +--> common/core implementation
NeoForge implementation -+
```

The benefit is that the LOD engine does not need separate copies of its world, storage, and renderer logic for each mod loader.

---

## 3. Wrapper abstraction

The `wrapperInterfaces` package is one of the most important architectural boundaries.

It abstracts:

- blocks and block states
- chunks
- client levels
- server levels
- world generation
- Minecraft client/server
- rendering
- textures
- entities and related game services

The concrete Fabric and NeoForge packages implement these abstractions.

Research implication: an independent renderer can use exactly the same *architectural idea* without copying implementation:

```
Game-specific adapter
       |
       v
Stable internal world interface
       |
       v
LOD engine
```

This is particularly relevant to PVR because RenderDragon/Bedrock should be isolated behind a narrow adapter.

---

## 4. World/chunk acquisition

### `DhWorldGenerator`

Key methods:

- `getPriority()`
- `getSmallestDataDetailLevel()`
- `getLargestDataDetailLevel()`
- `getReturnType()`
- `runApiValidation()`
- `preGeneratorTaskStart()`
- `generateLod(...)`
- `close()`

Functionally, this class is the bridge between DH's generation queue and Minecraft's world/chunk generation facilities.

It can generate LOD data using chunk generation and supports an asynchronous executor.

The 26.3 implementation is particularly important because Minecraft changed world-generation stages. Official DH development notes show that 26.3's terrain step consolidated previous noise/surface/carver behavior, requiring DH's generator to stop at the correct generation boundary. citeturn1search4turn1search5

### `WorldGenerationQueue`

This is the scheduler for distant terrain generation.

Important state:

- waiting tasks by LOD position
- in-progress generation tasks
- regeneration permission
- generation target position
- estimated remaining work
- rolling average generation time
- dedicated queueing thread

Important operations:

- submit retrieval request
- remove matching requests
- test whether a position is already requested
- start generation around a target
- select the next task
- start generation task groups
- generate using vanilla chunks
- generate using API chunks
- generate using API data sources
- close/cancel generation

The presence of `TaskDistancePair` and the target position means task selection is not simply FIFO. Spatial distance is part of prioritization.

Conceptually:

```
candidate LOD requests
       |
       +--> duplicate check
       |
       +--> detail-level eligibility
       |
       +--> distance/priority ordering
       |
       v
generation task
```

---

## 5. Retrieval abstraction

### `LodRequestModule`

This module sits between the current level state and the mechanism that retrieves/generates missing LOD data.

It lets DH treat these sources uniformly:

1. local persistent data;
2. remote/server-provided data;
3. freshly generated data.

That abstraction is what allows multiplayer worlds to avoid regenerating terrain already supplied by a server.

### `DataSourceRetrievalTask`

Represents one retrieval/generation request.

The associated result state distinguishes successful retrieval from failure/cancellation conditions.

---

## 6. Full terrain data representation

### `LodDataBuilder`

This is a critical conversion boundary.

Important operations:

- `createFromChunk()`
- `createFromApiChunkData()`
- `convertApiDataPointListToPackedLongArray()`
- `putListInTopDownOrder()`
- `validateOrThrowApiDataColumn()`

Its job is to convert Minecraft/API chunk information into DH's compact FullData representation.

The use of packed `long` arrays is important: terrain column records are compressed into compact primitive representations instead of large object graphs.

Conceptual pipeline:

```
Minecraft chunk
  -> terrain columns
  -> ordered datapoints
  -> validated datapoints
  -> packed primitive data
  -> FullDataSourceV2
```

This is a major performance design choice because primitive arrays reduce object allocation and improve cache locality.

---

## 7. FullData model

The important classes include:

- `FullDataSourceV1`
- `FullDataSourceV2`
- `FullDataPointIdMap`

V2 is the active persistent representation in this artifact.

A FullData source represents terrain information at a spatial section/LOD level before it becomes GPU-oriented render data.

The distinction is:

```
FullData = authoritative/processable terrain representation
RenderData = renderer-oriented representation
```

That separation allows storage and rendering to evolve independently.

---

## 8. Lighting subsystem

### `DhLightingEngine`

Important operations:

- `lightChunk()`
- `bakeChunkBlockLighting()`
- `propagateChunkLightPosList()`
- `bakeDataSourceSkyLight()`
- `recursivelyLightAdjacentDataPoints()`

The engine operates on chunk data and FullData.

The presence of:

- adjacent chunk holder
- stable light-position stack
- get/set light callbacks
- sky-light baking

shows that lighting is processed as terrain data rather than requiring the complete vanilla renderer to be present at distant LOD.

This is essential for far terrain because full per-block lighting at enormous render distances would be too expensive.

---

## 9. Occlusion reduction

### `FullDataOcclusionCuller`

Key methods:

- `cullHiddenDatapointsInColumn()`
- `checkOcclusion()`
- `isTranslucent()`

The purpose is to remove terrain datapoints that cannot contribute to the final visible representation.

It explicitly distinguishes translucent terrain, meaning the culler cannot simply delete everything hidden behind another opaque sample.

Conceptually:

```
FullData column
      |
      +--> neighboring/vertical visibility checks
      |
      +--> translucency exception
      |
      v
reduced render candidate set
```

This reduces CPU work and eventual vertex-buffer size.

---

## 10. FullData -> RenderData

### `FullDataToRenderDataTransformer`

This is one of the most important classes in the entire architecture.

Primary method:

`transformFullDataToRenderSource()`

Supporting operations:

- transform complete FullData into column data
- replace/update individual render columns
- apply texture-set IDs
- create render column views
- represent void/air regions

There is also a pooled-array mechanism and a broken-position set.

The existence of a `PhantomArrayListPool` indicates deliberate reuse of temporary collections to reduce garbage-collector pressure.

The transformation can therefore be represented as:

```
FullDataSourceV2
      |
      +--> column extraction
      +--> datapoint reduction
      +--> occlusion-aware processing
      +--> texture ID assignment
      +--> render column creation
      |
      v
ColumnRenderSource
```

---

## 11. Render column representation

### `ColumnRenderSource`

This represents the render-oriented terrain for a spatial LOD section.

It is downstream of FullData and upstream of buffer building.

The column-oriented representation is important because distant terrain can often be represented as vertical terrain surfaces rather than millions of individual block cubes.

---

## 12. Quad generation

### `LodQuadBuilder`

This class constructs renderable quads.

Important operations:

- add adjacent-face quad
- add upward quad
- add downward quad
- merge quads
- build opaque vertex buffers
- build transparent vertex buffers
- write vertices into byte buffers
- cache/reuse quad objects

The builder maintains separate opaque and transparent quad collections.

This matters because transparency normally requires a different rendering order/path than opaque geometry.

### Quad merging

`mergeQuads()` combines compatible adjacent quads.

The architectural objective is straightforward:

```
many small coplanar faces
        |
        v
fewer larger faces
        |
        v
fewer vertices / draw work
```

This is a major LOD performance optimization.

---

## 13. GPU vertex layout

`LodQuadBuilder` exposes:

- `BYTES_PER_VERTEX`
- `BYTES_PER_QUAD`
- direction-specific index patterns
- ByteBuffer generation

This means the CPU-side builder produces tightly packed GPU-ready buffers rather than handing individual Java objects to the renderer.

The intended path is:

```
terrain datapoints
 -> BufferQuad
 -> packed ByteBuffer
 -> GL vertex/index buffer
 -> shader
```

---

## 14. Render-buffer hierarchy

### `LodBufferContainer`

Represents the GPU/render resources associated with a LOD section.

### `RenderBufferHandler`

Maintains:

- a LOD QuadTree
- loaded buffers ordered near-to-far
- temporary traversal lists
- visible/culled counts
- shadow visible/culled counts

Its primary job is to construct the current render list from spatial LOD data.

This is a spatial rendering system, not a flat list of meshes.

---

## 15. QuadTree

### `LodQuadTree`

The QuadTree is the spatial index used to select appropriate LOD sections.

The renderer can therefore traverse:

```
root
 |
 +-- near/high detail
 |
 +-- medium detail
 |
 +-- far/low detail
```

rather than iterating every distant tile every frame.

This is one of the strongest architectural patterns to reuse in an independent PVR renderer.

---

## 16. Camera/frustum selection

`RenderBufferHandler.buildRenderList()` receives `RenderParams`.

The surrounding render classes contain:

- world-view matrices
- world-view-projection matrices
- frustum representation
- camera position
- shadow-pass state
- visibility counters

Therefore render selection happens before issuing GPU draws.

Conceptually:

```
LOD QuadTree
    |
    +--> frustum test
    +--> distance/LOD selection
    +--> shadow visibility
    +--> near-to-far ordering
    |
    v
visible LodBufferContainer set
```

---

## 17. Client lifecycle

### `DhClientLevel`

This is the client-side world-level controller.

Important state:

- client-level wrapper
- save structure
- remote data provider
- network state
- loaded chunk set
- LOD request module
- sync-on-load request queue

Important methods:

- `clientTick()`
- `shouldDoWorldGen()`
- `getTargetPosForGeneration()`
- `onWorldGenTaskComplete()`
- `clearRenderCache()`
- `updateDataSourcesAsync()`
- `shouldProcessChunkUpdate()`
- `close()`

This class ties together storage, generation, networking and rendering.

---

## 18. Client render API

### `ClientApi`

Important operations:

- `renderLods()`
- `renderDeferredLodsForShaders()`
- `renderFadeOpaque()`
- `renderFadeTransparent()`
- key events
- queued messages
- camera-speed measurement
- boot-time warnings

The API maintains a `DhRenderState` and `RenderParams`.

This demonstrates that DH's renderer has an explicit state machine rather than scattering rendering decisions across Mixins.

---

## 19. Shader integration

The artifact contains render integration for:

- terrain rendering
- fog
- fade
- TAA
- sharpening
- SSAO
- Iris framebuffer interaction

The `MixinIrisFrameBuffer` integration is especially important: DH must coordinate its rendering with shader-mod framebuffer state.

Official DH documentation also warns that unsupported shader configurations can prevent DH rendering, confirming that the renderer is integrated into the shader pipeline rather than being a completely isolated overlay. citeturn1search11

---

## 20. Persistent storage

### `FullDataSourceProviderV2`

Responsibilities:

- open/load FullData from the database
- asynchronously retrieve data
- retrieve neighboring sections
- determine missing data positions
- queue missing positions
- track generation plan
- update stored data
- report timestamps
- expose debug information
- close storage

### `FullDataUpdaterV2`

Responsibilities:

- serialize FullData to DTO
- update database asynchronously
- maintain positional update locks
- track queued updates per position
- notify listeners

The positional locking is important.

Without it, two worker threads could write different versions of the same LOD section concurrently.

Conceptually:

```
update(position)
   |
   v
PositionalLockProvider
   |
   +--> serialize
   +--> database transaction/update
   +--> listener notification
   v
unlock
```

---

## 21. Database synchronization

The API exposes configuration related to database synchronization and multiplayer.

This allows DH to distinguish:

- locally generated LOD
- server-provided LOD
- data that should be synchronized
- data that should remain client-local

The server-owner documentation confirms that DH can generate LOD server-side and transmit it to clients; default server configuration also has measurable bandwidth requirements. citeturn0search9

---

## 22. Multiplayer path

The client level contains:

- `ClientNetworkState`
- `ScopedNetworkEventSource`
- `RemoteFullDataSourceProvider`
- `SyncOnLoadRequestQueue`

This gives a second terrain-data path:

```
Server DH
    |
    v
FullData network messages
    |
    v
RemoteFullDataSourceProvider
    |
    v
local FullData
    |
    v
normal render pipeline
```

Thus network terrain data joins the rendering pipeline at the FullData boundary instead of bypassing it.

---

## 23. Configuration architecture

The configuration tree is extensive and separated into:

- client
- server
- common
- graphics
- fog
- texture
- multithreading
- multiplayer
- world generation
- LOD building
- debugging

Event handlers include:

- quick render toggle
- reload LODs
- world curvature
- world generation plan
- render quality presets
- thread presets

This means configuration changes are not merely passive values; selected settings trigger runtime behavior.

---

## 24. Threading architecture

Important components include:

- `RateLimitedThreadPoolExecutor`
- `PriorityTaskPicker`
- `RenderThreadTaskHandler`
- `ServerThreadTaskHandler`
- `ThreadPoolUtil`
- `DhThreadFactory`
- `PositionalLockProvider`

The architecture separates:

1. world-generation workers;
2. storage/update workers;
3. render-thread operations;
4. server-thread integration.

This separation is crucial because GPU API calls and certain Minecraft operations must remain on their required thread.

Recent official 26.3 development work also shows why this matters: world-generation threads could otherwise block waiting on the Minecraft server thread. A JFR measurement in the project showed substantial waiting in `ServerChunkCache.getChunk()`. citeturn1search5

---

## 25. Native libraries

The artifact contains native libraries for multiple CPU/OS targets.

Observed subsystems include:

- SQLite
- Zstandard
- LZ4
- LWJGL-related native components

The native layer exists primarily to accelerate or enable storage/compression/graphics functionality.

It should not be confused with Minecraft gameplay logic.

---

## 26. Updater

Relevant classes:

- `SelfUpdater`
- `GitlabGetter`
- update UI classes
- changelog UI

Static evidence shows the mod has a self-update subsystem.

Security conclusion:

**No malicious verdict is established by this static inspection.**

A proper dynamic security test would separately observe:

- DNS/network destinations;
- downloaded artifacts;
- checksum/signature validation;
- process creation;
- filesystem writes;
- update execution.

---

## 27. Mixin integration model

The Mixin layer connects DH to Minecraft lifecycle points.

The important conceptual groups are:

### Client render

- level renderer
- game renderer
- fog renderer
- light texture
- projection matrix
- Iris framebuffer

### World/chunk

- client level
- chunk sections
- chunk generator
- chunk map

### Runtime

- Minecraft lifecycle
- background executor
- tracing executor
- server/player lifecycle

This is how the common engine receives Minecraft events without hard-coding the entire Minecraft implementation.

---

## 28. What actually makes DH efficient

The performance advantage is not one single optimization.

It is the combination:

1. **LOD** — distant terrain has less geometric detail.
2. **Spatial hierarchy** — QuadTree avoids scanning everything.
3. **Packed primitive data** — less allocation and memory overhead.
4. **Occlusion reduction** — hidden terrain is removed before meshing.
5. **Quad merging** — fewer vertices.
6. **Separate opaque/transparent buffers**.
7. **Asynchronous generation**.
8. **Prioritized generation**.
9. **Persistent storage** — previously generated terrain can be reused.
10. **GPU-ready ByteBuffers**.
11. **Buffer reuse/pooling**.
12. **Independent lighting bake**.
13. **Frustum/culling before rendering**.
14. **Dedicated render state**.

The architecture is therefore better described as a **terrain streaming system with a hierarchical renderer** than as simply a "far render-distance mod".

---

## 29. Independent PVR design derived from the analysis

For PVR-Minecraft-Research, the clean independent architecture is:

```
RenderDragon / Bedrock adapter
            |
            v
     Chunk sampler
            |
            v
   Full terrain representation
            |
            +--> lighting/material extraction
            |
            v
     LOD reducer
            |
            +--> occlusion reduction
            +--> surface extraction
            +--> quad merging
            |
            v
     Persistent tile store
            |
            v
     Priority streaming queue
            |
            v
       CPU mesh builder
            |
            v
       GPU upload queue
            |
            v
       PVR QuadTree
            |
            v
       culling / LOD selection
            |
            v
       renderer / compositor
```

The key lesson is to preserve the **data boundaries**, not DH's implementation.

---

## 30. What should NOT be copied

For this repository, do not copy:

- decompiled DH source;
- DH bytecode;
- proprietary Minecraft assets;
- DH shader source unless independently licensed/used according to its license;
- generated binary data from another user's installation.

Store only:

- observations;
- class/function maps;
- interface descriptions;
- measurements;
- independently written pseudocode;
- architecture diagrams;
- experimental results.

The official project identifies DH as an open-source project and publishes its source/build structure; its build uses multiple subprojects and platform-specific packaging. citeturn1search9turn0search1

---

## 31. Confidence model

### High confidence

- FullData -> RenderData separation
- LOD generation queue
- persistent FullData provider/updater
- lighting subsystem
- QuadTree render hierarchy
- packed vertex buffers
- separate opaque/transparent geometry
- wrapper/platform abstraction
- client/server/multiplayer data paths
- native storage/compression components

### Medium confidence

- exact ordering of some internal asynchronous callbacks
- exact database transaction boundaries without tracing runtime SQL
- exact shader pass ordering for every configuration
- exact LOD-selection heuristic for every renderer mode

### Not established

- malicious runtime behavior
- undocumented remote execution
- hidden network destinations
- exact performance impact on the supplied machine without profiling

---

## 32. Next deep-analysis phase

The next phase should be method-level rather than package-level:

1. enumerate every class and public/private method;
2. map caller -> callee relationships for critical paths;
3. reconstruct the FullData binary packing format;
4. reconstruct the SQLite schema and queries;
5. reconstruct QuadTree node selection;
6. reconstruct LOD detail-level transitions;
7. reconstruct quad-merge rules;
8. map exact GPU vertex attributes;
9. map shader inputs/uniforms;
10. map every Mixin injection point;
11. map network message types;
12. map configuration value -> runtime subsystem;
13. produce independent pseudocode for the complete pipeline.

That phase will produce a much more useful research dataset than simply decompiling all 1,019 classes into a giant text dump.
