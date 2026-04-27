# 3D Gaussian Splats

## Philosophy

Gaussian splats are a relatively recent addition to the rendering vocabulary
(Kerbl et al., 2023). Each splat is a small oriented 3D Gaussian — a position,
an anisotropic covariance (rotation + 3 scales), a color, and an opacity —
rasterized as a soft anisotropic blob. Originally trained from photographs to
reconstruct real scenes, the same representation is **directly constructable
from algorithms**.

The interesting move for algorithmic art: skip training, write each splat
analytically. Strange attractor trajectory points become anisotropic ribbons
along the tangent. Flow-field particles become oriented streaks. Implicit
surface samples become a fuzzy volume. SDF gradient fields become directional
glow. Anywhere you'd otherwise use particles or volumetric data, splats give a
real-time renderable representation with proper depth sorting, transparency,
and a standard file format.

**Conceptually**: meshes describe surfaces with infinite resolution, splats
describe **light at locations** with finite resolution. They render
photoreally because they encode what the camera sees, not what's there.

## The 3DGS PLY Format

Standard 3DGS `.ply` (binary little-endian) — degree-0 spherical harmonics is
the minimum and works in every viewer. 17 floats per vertex:

```
ply
format binary_little_endian 1.0
element vertex N
property float x          ─┐
property float y           │ position (world space)
property float z          ─┘
property float nx         ─┐
property float ny          │ unused; write zeros
property float nz         ─┘
property float f_dc_0     ─┐
property float f_dc_1      │ DC SH coefficient per RGB channel
property float f_dc_2     ─┘ encoded as (rgb − 0.5) / SH_C0
property float opacity     │ pre-sigmoid (logit)
property float scale_0    ─┐
property float scale_1     │ pre-exp (log of std-dev along local axes)
property float scale_2    ─┘
property float rot_0      ─┐
property float rot_1       │ unit quaternion (w, x, y, z)
property float rot_2       │ NOTE: w-first, not the xyzw convention
property float rot_3      ─┘
end_header
<binary float32 records, packed in property order>
```

Encoding constants:

```
SH_C0 = 0.28209479177387814   (sqrt(1/(4π)) — the degree-0 SH basis function)
f_dc  = (rgb − 0.5) / SH_C0   so rgb in [0,1] decodes back via SH_C0·f_dc + 0.5
scale = exp(scale_log)         (always positive after exp)
opacity = sigmoid(opacity_logit)
```

Higher-degree SH (`f_rest_0` … `f_rest_44`, 15 coefficients × 3 channels) is
optional. Most viewers tolerate its absence and treat splats as
view-independent. Include only if you have a reason — for purely procedural
art, degree 0 is plenty.

## Direct Construction (Python)

Pattern: emit one record per algorithmic point. The orientation comes from
whatever local frame your generator naturally produces.

```python
import math, struct
import numpy as np

SH_C0 = 0.28209479177387814

def quat_from_basis(x_axis, y_axis, z_axis):
    """Unit quaternion (w,x,y,z) from an orthonormal basis (Shoemake)."""
    m00, m10, m20 = x_axis
    m01, m11, m21 = y_axis
    m02, m12, m22 = z_axis
    tr = m00 + m11 + m22
    if tr > 0:
        s = math.sqrt(tr + 1.0) * 2.0
        w, x = 0.25*s, (m21 - m12)/s
        y, z = (m02 - m20)/s, (m10 - m01)/s
    elif m00 > m11 and m00 > m22:
        s = math.sqrt(1.0 + m00 - m11 - m22) * 2.0
        w, x = (m21 - m12)/s, 0.25*s
        y, z = (m01 + m10)/s, (m02 + m20)/s
    elif m11 > m22:
        s = math.sqrt(1.0 + m11 - m00 - m22) * 2.0
        w, x = (m02 - m20)/s, (m01 + m10)/s
        y, z = 0.25*s, (m12 + m21)/s
    else:
        s = math.sqrt(1.0 + m22 - m00 - m11) * 2.0
        w, x = (m10 - m01)/s, (m02 + m20)/s
        y, z = (m12 + m21)/s, 0.25*s
    q = np.array([w, x, y, z])
    return q / np.linalg.norm(q)

def write_splat_ply(path, positions, rgb01, scales_xyz, opacities, quats_wxyz):
    """positions: (N,3); rgb01: (N,3) in [0,1]; scales_xyz: (N,3) world units;
       opacities: (N,); quats_wxyz: (N,4) unit."""
    n = len(positions)
    f_dc          = (rgb01 - 0.5) / SH_C0
    scale_log     = np.log(np.clip(scales_xyz, 1e-6, None))
    opacity_logit = np.log(opacities / (1.0 - opacities))

    properties = ["x","y","z","nx","ny","nz",
                  "f_dc_0","f_dc_1","f_dc_2","opacity",
                  "scale_0","scale_1","scale_2",
                  "rot_0","rot_1","rot_2","rot_3"]
    header  = "ply\nformat binary_little_endian 1.0\n"
    header += f"element vertex {n}\n"
    for p in properties:
        header += f"property float {p}\n"
    header += "end_header\n"

    rows = np.empty((n, 17), dtype=np.float32)
    rows[:, 0:3]   = positions
    rows[:, 3:6]   = 0.0
    rows[:, 6:9]   = f_dc
    rows[:, 9]     = opacity_logit
    rows[:, 10:13] = scale_log
    rows[:, 13:17] = quats_wxyz

    with open(path, "wb") as f:
        f.write(header.encode("utf-8"))
        f.write(rows.tobytes())
```

## Anisotropic Frames From a Generator

The covariance shape is what makes splats expressive. Pick local axes from the
algorithm's natural geometry:

| Source | Local x (long axis) | Aesthetic |
|--------|---------------------|-----------|
| Trajectory (attractor / parametric curve) | tangent | ribbon tracing the path |
| Flow / curl noise particle | velocity | streak |
| Gradient field of an SDF / scalar field | gradient | oriented "fur" along gradient |
| Hopf fibration / linked rings | fiber tangent | smooth tubes |
| Mesh surface sample | surface normal (or tangent) | shell-like layers |

To turn a single direction `t` into a full orthonormal basis, Gram-Schmidt off
a reference axis (handle the parallel case with a fallback):

```python
up = np.array([0.0, 1.0, 0.0])
binormal = np.cross(t, up)
if np.linalg.norm(binormal) < 1e-6:
    binormal = np.cross(t, np.array([1.0, 0.0, 0.0]))
binormal /= np.linalg.norm(binormal)
normal = np.cross(t, binormal)
quat = quat_from_basis(t, binormal, normal)
```

Anisotropic scale tuning (in world units, since these are Gaussian std-devs):
- **Ribbon**: long ≫ medium ≈ short  (e.g. 0.18 / 0.06 / 0.012)
- **Streak**: long ≫ short ≈ short   (e.g. 0.20 / 0.02 / 0.02)
- **Disk**: long ≈ medium ≫ short    (e.g. 0.10 / 0.10 / 0.010)
- **Isotropic blob**: all equal       (e.g. 0.05 / 0.05 / 0.05)

## In-Browser Approximate Rendering (Three.js)

Full anisotropic 3DGS rasterization (covariance projection, depth sorting,
EWA splatting) is a substantial implementation. For a quick visualization use
isotropic billboards — same colors, same positions, same gaussian falloff,
just round instead of oriented. Looks great with additive blending.

```javascript
const SPLAT_VERT = `
attribute vec3 aColor;
varying vec3 vColor;
uniform float uSizeScale;   // world-space size
uniform float uPxPerUnit;   // domHeight / (2 tan(fov/2))
void main() {
  vec4 mv = modelViewMatrix * vec4(position, 1.0);
  gl_Position = projectionMatrix * mv;
  gl_PointSize = uSizeScale * uPxPerUnit / max(-mv.z, 0.001);
  vColor = aColor;
}`;

const SPLAT_FRAG = `
varying vec3 vColor;
uniform float uBrightness;
void main() {
  vec2 d = gl_PointCoord - 0.5;
  float r2 = dot(d, d) * 4.0;
  if (r2 > 1.0) discard;
  float a = exp(-r2 * 4.0);
  gl_FragColor = vec4(vColor * uBrightness * a, a);
}`;

const mat = new THREE.ShaderMaterial({
  uniforms: { uSizeScale: { value: 0.06 }, uPxPerUnit: { value: pxPerUnit }, uBrightness: { value: 1.6 } },
  vertexShader: SPLAT_VERT, fragmentShader: SPLAT_FRAG,
  transparent: true, blending: THREE.AdditiveBlending,
  depthWrite: false, depthTest: true,
});
const points = new THREE.Points(geom, mat);
```

`pxPerUnit = renderer.domElement.height * 0.5 / Math.tan(0.5 * camera.fov * Math.PI/180)`
gives the world-units-to-pixels conversion factor at distance 1.

The `.ply` produced by the Python side is the ground-truth artifact for
proper anisotropic viewers below; the in-browser approximation is for
fast iteration on the algorithm.

## Tools That Consume the .ply

| Tool | Role | Notes |
|------|------|-------|
| [PlayCanvas SuperSplat](https://playcanvas.com/supersplat/editor) | Browser editor & viewer | Drag-drop, full anisotropic, MIT-licensed |
| [@mkkellogg/gaussian-splats-3d](https://github.com/mkkellogg/GaussianSplats3D) | Three.js library | `DropInViewer` integrates with existing Three scenes |
| Babylon.js `GaussianSplattingMesh` | Babylon engine | First-party support |
| [Brush](https://github.com/ArthurBrussee/brush) | Native Rust+wgpu | Cross-platform desktop, also trains from photos |
| [gsplat](https://github.com/nerfstudio-project/gsplat) | Python training/rendering | If you ever want to *train* splats on a procedurally-rendered scene |

## Render-and-Train Path

The dual to direct construction: render any procedural scene from N viewpoints
(GLSL shader, Three.js scene, Blender), then train splats on the renders with
`gsplat` or Brush. You get a splat representation of *any* algorithmic shader
output — useful for capturing painterly or volumetric looks that don't map
cleanly to direct construction. Good viewpoint coverage matters more than
viewpoint count; sample spherically around the scene.

## Composition Patterns

- **Splats + mesh scaffold**: extracted SDF mesh provides solid geometry,
  splats fill the negative space as light/dust/atmosphere. See
  [isosurface-extraction.md](isosurface-extraction.md). Render order matters
  for correct compositing — see [order-independent-transparency.md](order-independent-transparency.md).
- **Splats from attractor + splats from flow field**: superpose two generators
  with different color palettes, additive blending compounds at intersections.
- **Hierarchical splats**: large slow-moving splats for atmosphere, small fast
  ones for detail — same geometry, different scale ranges.

## Notable Practitioners

- **Kerbl, Kopanas, Leimkühler, Drettakis** — original 3DGS paper (SIGGRAPH 2023)
- **Niantic Spatial / Spark** — Three.js-integrated splat rendering
- **Refik Anadol** — point-cloud / splat-adjacent installations from data archives
- **PostShot, Polycam, Luma** — commercial capture-to-splat tooling
