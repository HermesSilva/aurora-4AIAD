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

| Change | Why |
|--------|-----|
| **Raster "modeling" mode** next to the path-traced "render" mode, on the same D3D12 device and scene | interactive, deterministic, noise-free viewport for modeling |
| **Edges and curves** (`basisCurves`) drawn as screen-space lines | Aurora only accepts triangles; B-rep edges and sketches are essential in CAD |
| **ID buffer** (prim + element id) and selection highlight | picking faces/edges/bodies, also used to assign loads and supports in simulation studies |
| **Host-provided device**: the D3D12 device and queues come from the application (`aiad_gpu`), on the same adapter (LUID) as CUDA | lets simulation results flow from CUDA to the viewport without copies |
| **Result fields from CUDA** via D3D12 external memory and fences | color maps of stress, temperature, fields, pressure |
| **Denoiser** for the path-traced mode | upstream lists it as planned (NVIDIA Real-Time Denoisers) |
| **OpenPBR** materials through MaterialX, in addition to Standard Surface | AIAD uses OpenPBR as its appearance model |
| Build with **MSVC v145** against **OpenUSD 25.11** (the build shipped with NVIDIA USD BRep) | one toolchain and one OpenUSD for the whole application |

B-rep solids (`BrepArray` from the `omniSolid` schema) reach this delegate as meshes through the
`hdUsdBrep` Hydra plugin from [NVIDIA USD BRep](https://github.com/NVIDIA-Omniverse/usd-brep), so no
tessellation code is added here.

Non-Windows backends (Metal, Vulkan / HGI) are left as they are upstream and are not maintained by
this fork.

## Status

**No source changes yet.** At this point the only differences from upstream are this README, the
original README renamed to `README.upstream.md`, and the link to it in `Doc/Plasma.md`. The work
above is planned in the AIAD roadmap (milestone F1: raster mode; later: denoiser, OpenPBR).

## Branches

| Branch | Content |
|--------|---------|
| `main` | mirror of `Autodesk/Aurora` `main`, never modified |
| `main-4AIAD` (default) | `main` + the changes listed above, rebased periodically onto upstream |

## License

Same as upstream: **Apache License 2.0** (see [LICENSE.md](LICENSE.md)). Changes made in this fork
are released under the same license.
