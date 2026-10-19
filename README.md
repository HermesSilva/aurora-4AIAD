# Aurora for Tootega AIAD

This is a fork of **[Autodesk/Aurora](https://github.com/Autodesk/Aurora)**, maintained for
**Tootega AIAD** (Artificial Intelligence-Aided Design), a Windows-only CAD/CAE tool built on
OpenUSD, DirectX 12 and CUDA.

> The original README is kept unchanged in **[README.upstream.md](README.upstream.md)**.
> Everything about Aurora itself (features, requirements, build, usage) is documented there.

## Why this fork exists

AIAD needs a Hydra render delegate that runs on **DirectX 12**. OpenUSD's own renderer (Storm)
only targets OpenGL, Metal and Vulkan, so it does not fit. Aurora already combines Hydra
(`HdAurora`), Direct3D 12 / DXR and MaterialX under the Apache-2.0 license, which makes it the
best starting point.

Aurora is a real-time **path tracer**, though. A CAD viewport also needs things a path tracer
does not provide: sharp edges drawn as lines, a noise-free interactive view, and per-face picking.
This fork adds those without removing what Aurora already does.

## What changes (and why)

### Done (branch `main-4AIAD`)

| Change | Why |
|--------|-----|
| CMake option **`AIAD_INTEGRATION`** (off by default): `Aurora` and `HdAurora` become **static libraries**; `HdAurora` is built **without OpenGL, GLEW and hdx** | AIAD links everything statically into one executable and owns the D3D12 device; OpenUSD is built without a GPU backend (no Storm/hdx) |
| `HdAuroraStaticRegistration.cpp`: registers the renderer plugin through `TF_REGISTRY_FUNCTION` | the shared build registers it from `DllMain`, which a static library does not have |
| `HdAuroraRenderBuffer`: local `HdFormat`→`HgiFormat` mapping, optional `Hgi`, no GL texture path when `HDAURORA_NO_GL` | the host reads the shared D3D12 texture or `Map()`s the pixels itself |
| `HdAuroraPlugin`/`HdAuroraMesh`: OpenUSD **25.11** API (`IsSupported(HdRendererCreateArgs const&)`, `HdMeshComputationResult`) | upstream gates these on 26.08, but they already exist in 25.11 |
| `installExternals.py`: `--usd-root`, `--materialx-static`, `--no-oiio`; Visual Studio 2026 / **MSVC v145**; pin fix for `pystring` in OpenImageIO 3.1.13 | build the externals with AIAD's toolchain and without the Autodesk USD fork |
| `CMAKE_CURRENT_SOURCE_DIR` for the version header | lets Aurora be included with `add_subdirectory()` |
| **Visual Studio projects** for Foundation, Aurora and HdAurora (`Libraries/*/*.vcxproj`, `Aurora4AIAD.props`, `Aurora4AIAD.Runtime.targets`, `VersionFiles/AuroraVersion.h`, `Source/pch.cpp`): the `AIAD_INTEGRATION` build (static, no OpenGL) with the shader packing (Python + slangc) and HLSL compilation as build steps | Tootega AIAD builds with versioned Visual Studio projects (no CMake); the CMake build of upstream stays as it is |
| `HdAuroraMesh`: primvars read only when they hold the expected type | `UncheckedGet` on an empty value (a mesh without normals) crashed the Hydra sync threads |
| `HdAuroraMesh::Sync`: instances are added when there are fewer than needed (was `>`) | a hidden mesh on a moving body took the transform-only path with a null resource and crashed |
| `PTRenderBuffer::data()`: the caller's stride stays the host (tight) pitch; no stale pointer after a resize | the padded GPU row pitch made readers overrun when the width was not 256-byte aligned |
| Motion vectors of moving instances: `PTScene` keeps each instance's transform of the previous frame next to the current one (`gTransformMatrixBuffer`), the closest-hit shader reprojects the hit point with it (`computeObjectMotionVector`) | the motion vectors treated every surface as static, so the denoiser (NRD) and the temporal resolve smeared and scrambled parts moving in an animated scene |

Verified (spike S3b in AIAD): a USD BRep `BrepArray` tessellated by `hdUsdBrep` is path-traced
with DXR through Hydra, in a single statically linked executable (static OpenUSD 25.11), on an
RTX 2060 SUPER.

### Planned

| Change | Why |
|--------|-----|
| **Raster "modeling" mode** next to the path-traced mode, on the same D3D12 device and scene | interactive, deterministic, noise-free viewport for modeling |
| **Edges and curves** (`basisCurves`) drawn as screen-space lines | Aurora only accepts triangles; B-rep edges (already produced by `hdUsdBrep`) are essential in CAD |
| **ID buffer** (prim + element id) and selection highlight | picking faces/edges/bodies, also used to assign loads and supports in simulation studies |
| Face-varying normals from `hdUsdBrep` | today they trigger "normal array length does not match" and faceted shading |
| **Host-provided device** and **CUDA result fields** via D3D12 external memory | simulation results flow from CUDA to the viewport without copies |
| **OpenPBR** materials | AIAD uses OpenPBR as its appearance model |

Upstream already provides the MaterialX → **Slang** generator and the **NRD** denoiser, which were
initially on this list.

B-rep solids (`BrepArray` from the `omniSolid` schema) reach this delegate as meshes through the
`hdUsdBrep` Hydra plugin from [NVIDIA USD BRep](https://github.com/NVIDIA-Omniverse/usd-brep), so no
tessellation code is added here.

Non-Windows backends (Metal, Vulkan / HGI) are left as they are upstream and are not maintained by
this fork.

## Branches

| Branch | Content |
|--------|---------|
| `main` | mirror of `Autodesk/Aurora` `main`, never modified |
| `main-4AIAD` (default) | `main` + the changes listed above, rebased periodically onto upstream |

## License

Same as upstream: **Apache License 2.0** (see [LICENSE.md](LICENSE.md)). Changes made in this fork
are released under the same license.
