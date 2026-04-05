# Line Art & Contour Rendering — Three.js Reference

## Overview

Line art rendering extracts visible edges from 3D geometry and renders them as clean strokes
— silhouettes, creases, boundaries — producing a drawing-like aesthetic from 3D scenes.
This reference covers techniques for both Three.js Scene mode (geometry-based edge extraction)
and Three.js Shader mode (screen-space edge detection via normals/depth).

**When to use:**
- Architectural / blueprint visualizations
- Pen-and-ink or woodcut aesthetics
- Toon / cel shading with outlines
- Print-ready SVG line art from 3D scenes
- Technical illustration style
- Combining line art with flat or minimal color fills

## Edge Classification

Three.js line art works with four types of feature edges, analogous to traditional technical
drawing conventions:

| Edge Type | Definition | Visual Role |
|-----------|-----------|-------------|
| **Silhouette** | Edge where one adjacent face points toward the camera, the other away | Object outline — the most important line |
| **Crease** | Edge where the dihedral angle between adjacent faces exceeds a threshold | Surface detail — shows hard folds and bevels |
| **Boundary** | Edge with only one adjacent face (open mesh border) | Mesh edges — holes, open surfaces |
| **Ridge / Valley** | Extrema of principal curvature | Fine surface articulation (advanced) |

## Approach 1: Geometry-Based Edge Extraction (Scene Mode)

Extract feature edges directly from mesh geometry by analyzing face normals and adjacency.
This produces clean vector-quality lines and works well for low-to-mid poly meshes.

### Core Algorithm

```javascript
function extractFeatureEdges(THREE, mesh, camera, creaseAngle = 60) {
  const geo = mesh.geometry;
  if (!geo.index) geo.computeBoundsTree?.(); // optional acceleration

  // Need face normals for silhouette/crease detection
  geo.computeVertexNormals();

  const position = geo.attributes.position;
  const index = geo.index;
  const matrixWorld = mesh.matrixWorld;

  // Build edge → face adjacency map
  const edgeMap = new Map(); // "v0_v1" → [faceIdx, ...]

  const faceCount = index ? index.count / 3 : position.count / 3;
  const faceNormals = [];

  const _v0 = new THREE.Vector3();
  const _v1 = new THREE.Vector3();
  const _v2 = new THREE.Vector3();
  const _edge = new THREE.Vector3();
  const _normal = new THREE.Vector3();

  for (let f = 0; f < faceCount; f++) {
    const i0 = index ? index.getX(f * 3) : f * 3;
    const i1 = index ? index.getX(f * 3 + 1) : f * 3 + 1;
    const i2 = index ? index.getX(f * 3 + 2) : f * 3 + 2;

    _v0.fromBufferAttribute(position, i0).applyMatrix4(matrixWorld);
    _v1.fromBufferAttribute(position, i1).applyMatrix4(matrixWorld);
    _v2.fromBufferAttribute(position, i2).applyMatrix4(matrixWorld);

    // Face normal (world space)
    _edge.subVectors(_v1, _v0);
    _normal.subVectors(_v2, _v0);
    _normal.cross(_edge).normalize();
    faceNormals.push(_normal.clone());

    // Register edges (sorted indices for canonical key)
    const verts = [i0, i1, i2];
    for (let e = 0; e < 3; e++) {
      const a = Math.min(verts[e], verts[(e + 1) % 3]);
      const b = Math.max(verts[e], verts[(e + 1) % 3]);
      const key = `${a}_${b}`;
      if (!edgeMap.has(key)) edgeMap.set(key, []);
      edgeMap.get(key).push(f);
    }
  }

  // Classify edges
  const camPos = new THREE.Vector3();
  camera.getWorldPosition(camPos);
  const creaseThreshold = Math.cos(THREE.MathUtils.degToRad(creaseAngle));
  const featureEdges = [];
  const _center = new THREE.Vector3();

  for (const [key, faces] of edgeMap) {
    const [a, b] = key.split("_").map(Number);
    let isFeature = false;
    let edgeType = "";

    if (faces.length === 1) {
      // Boundary edge
      isFeature = true;
      edgeType = "boundary";
    } else if (faces.length === 2) {
      const n0 = faceNormals[faces[0]];
      const n1 = faceNormals[faces[1]];

      // Face centers for camera direction test
      // (simplified: use edge midpoint)
      _v0.fromBufferAttribute(position, a).applyMatrix4(matrixWorld);
      _v1.fromBufferAttribute(position, b).applyMatrix4(matrixWorld);
      _center.addVectors(_v0, _v1).multiplyScalar(0.5);

      const viewDir = _center.clone().sub(camPos);

      // Silhouette: one face toward camera, one away
      const d0 = n0.dot(viewDir);
      const d1 = n1.dot(viewDir);
      if ((d0 > 0) !== (d1 > 0)) {
        isFeature = true;
        edgeType = "silhouette";
      }

      // Crease: sharp dihedral angle
      if (!isFeature && n0.dot(n1) < creaseThreshold) {
        isFeature = true;
        edgeType = "crease";
      }
    }

    if (isFeature) {
      _v0.fromBufferAttribute(position, a).applyMatrix4(matrixWorld);
      _v1.fromBufferAttribute(position, b).applyMatrix4(matrixWorld);
      featureEdges.push({
        start: _v0.clone(),
        end: _v1.clone(),
        type: edgeType,
      });
    }
  }

  return featureEdges;
}
```

### Rendering Extracted Edges as Line Segments

```javascript
function buildEdgeLines(THREE, featureEdges, color = 0x000000) {
  const positions = [];
  for (const edge of featureEdges) {
    positions.push(edge.start.x, edge.start.y, edge.start.z);
    positions.push(edge.end.x, edge.end.y, edge.end.z);
  }

  const geo = new THREE.BufferGeometry();
  geo.setAttribute("position", new THREE.Float32BufferAttribute(positions, 3));

  const mat = new THREE.LineBasicMaterial({ color });
  return new THREE.LineSegments(geo, mat);
}

// Usage in sceneSetup:
function sceneSetup(THREE, scene, camera, renderer, params, seed) {
  const mesh = new THREE.Mesh(
    new THREE.IcosahedronGeometry(2, 2),
    new THREE.MeshStandardMaterial({ color: 0xcccccc })
  );
  scene.add(mesh);

  const edges = extractFeatureEdges(THREE, mesh, camera, params.creaseAngle);
  const lines = buildEdgeLines(THREE, edges, 0x000000);
  scene.add(lines);

  return { mesh, lines };
}
```

### Updating Edges on Camera Move

Silhouette edges depend on the view direction, so they must be recalculated when the camera
orbits. Throttle the update to avoid per-frame geometry rebuilds:

```javascript
function sceneAnimate(THREE, scene, camera, state, params, time, delta) {
  // Rebuild every N frames or on significant camera change
  state._frameCount = (state._frameCount || 0) + 1;
  if (state._frameCount % 10 === 0) {
    scene.remove(state.lines);
    const edges = extractFeatureEdges(THREE, state.mesh, camera, params.creaseAngle);
    state.lines = buildEdgeLines(THREE, edges);
    scene.add(state.lines);
  }
}
```

### Styling by Edge Type

Different visual weight per edge type creates hierarchy, like technical illustration:

```javascript
function buildStyledEdgeLines(THREE, featureEdges) {
  const groups = { silhouette: [], crease: [], boundary: [] };
  for (const edge of featureEdges) {
    groups[edge.type]?.push(edge);
  }

  const parent = new THREE.Group();

  // Silhouettes: bold
  if (groups.silhouette.length > 0) {
    parent.add(buildEdgeLines(THREE, groups.silhouette, 0x000000));
  }
  // Creases: medium weight, slightly lighter
  if (groups.crease.length > 0) {
    parent.add(buildEdgeLines(THREE, groups.crease, 0x444444));
  }
  // Boundaries: thin, dashed
  if (groups.boundary.length > 0) {
    const lines = buildEdgeLines(THREE, groups.boundary, 0x888888);
    lines.material.dashSize = 0.05;
    lines.material.gapSize = 0.03;
    lines.computeLineDistances();
    parent.add(lines);
  }

  return parent;
}
```

## Approach 2: Screen-Space Edge Detection (Shader Mode)

Detect edges in screen space using depth and normal buffers. This works on any geometry
complexity and doesn't require adjacency analysis, but produces raster (not vector) outlines.

### Depth + Normal Edge Detection (Sobel / Roberts Cross)

```glsl
// Requires: depth texture and normal texture as uniforms
// In Three.js Scene mode, use MRT or post-processing passes

uniform sampler2D u_depth;
uniform sampler2D u_normals;
uniform vec2 u_resolution;

float sampleDepth(vec2 uv) {
  return texture2D(u_depth, uv).r;
}

vec3 sampleNormal(vec2 uv) {
  return texture2D(u_normals, uv).rgb * 2.0 - 1.0;
}

// Roberts Cross edge detection (2x2 kernel, faster than Sobel)
float robertsCrossDepth(vec2 uv, vec2 texel) {
  float d00 = sampleDepth(uv);
  float d11 = sampleDepth(uv + texel);
  float d10 = sampleDepth(uv + vec2(texel.x, 0.0));
  float d01 = sampleDepth(uv + vec2(0.0, texel.y));
  return sqrt(pow(d00 - d11, 2.0) + pow(d10 - d01, 2.0));
}

// Sobel edge detection on normals (catches surface discontinuities)
float sobelNormal(vec2 uv, vec2 texel) {
  vec3 n00 = sampleNormal(uv + vec2(-texel.x, -texel.y));
  vec3 n10 = sampleNormal(uv + vec2( 0.0,     -texel.y));
  vec3 n20 = sampleNormal(uv + vec2( texel.x, -texel.y));
  vec3 n01 = sampleNormal(uv + vec2(-texel.x,  0.0));
  vec3 n21 = sampleNormal(uv + vec2( texel.x,  0.0));
  vec3 n02 = sampleNormal(uv + vec2(-texel.x,  texel.y));
  vec3 n12 = sampleNormal(uv + vec2( 0.0,      texel.y));
  vec3 n22 = sampleNormal(uv + vec2( texel.x,  texel.y));

  vec3 gx = -n00 - 2.0*n01 - n02 + n20 + 2.0*n21 + n22;
  vec3 gy = -n00 - 2.0*n10 - n20 + n02 + 2.0*n12 + n22;

  return length(gx) + length(gy);
}

float detectEdge(vec2 uv) {
  vec2 texel = 1.0 / u_resolution;
  float depthEdge = robertsCrossDepth(uv, texel);
  float normalEdge = sobelNormal(uv, texel);

  // Combine: depth catches silhouettes, normals catch creases
  return clamp(depthEdge * 50.0 + normalEdge * 2.0, 0.0, 1.0);
}
```

### Three.js Scene Mode: Normal + Depth Post-Processing Pass

Use a two-pass approach — render normals/depth first, then detect edges:

```javascript
// In sceneSetup:
function sceneSetup(THREE, scene, camera, renderer, params, seed) {
  // Normal render target
  const normalTarget = new THREE.WebGLRenderTarget(
    window.innerWidth, window.innerHeight,
    { type: THREE.HalfFloatType }
  );

  // Depth render target
  const depthTarget = new THREE.WebGLRenderTarget(
    window.innerWidth, window.innerHeight,
    { depthTexture: new THREE.DepthTexture() }
  );
  depthTarget.depthTexture.type = THREE.UnsignedIntType;

  // Override material for normal pass
  const normalMat = new THREE.MeshNormalMaterial();

  // Edge detection quad
  const edgeQuad = new THREE.Mesh(
    new THREE.PlaneGeometry(2, 2),
    new THREE.ShaderMaterial({
      uniforms: {
        u_normals: { value: normalTarget.texture },
        u_depth: { value: depthTarget.depthTexture },
        u_resolution: { value: new THREE.Vector2(window.innerWidth, window.innerHeight) },
        u_lineColor: { value: new THREE.Color(params.lineColor || "#000000") },
        u_bgColor: { value: new THREE.Color(params.bgColor || "#ffffff") },
        u_depthSensitivity: { value: params.depthSensitivity || 50.0 },
        u_normalSensitivity: { value: params.normalSensitivity || 2.0 },
      },
      vertexShader: `
        varying vec2 vUv;
        void main() {
          vUv = uv;
          gl_Position = vec4(position.xy, 0.0, 1.0);
        }
      `,
      fragmentShader: `
        uniform sampler2D u_normals;
        uniform sampler2D u_depth;
        uniform vec2 u_resolution;
        uniform vec3 u_lineColor;
        uniform vec3 u_bgColor;
        uniform float u_depthSensitivity;
        uniform float u_normalSensitivity;
        varying vec2 vUv;

        float robertsCrossDepth(vec2 uv, vec2 texel) {
          float d00 = texture2D(u_depth, uv).r;
          float d11 = texture2D(u_depth, uv + texel).r;
          float d10 = texture2D(u_depth, uv + vec2(texel.x, 0.0)).r;
          float d01 = texture2D(u_depth, uv + vec2(0.0, texel.y)).r;
          return sqrt(pow(d00 - d11, 2.0) + pow(d10 - d01, 2.0));
        }

        float sobelNormal(vec2 uv, vec2 texel) {
          vec3 n00 = texture2D(u_normals, uv + vec2(-texel.x, -texel.y)).rgb;
          vec3 n10 = texture2D(u_normals, uv + vec2(0.0, -texel.y)).rgb;
          vec3 n20 = texture2D(u_normals, uv + vec2(texel.x, -texel.y)).rgb;
          vec3 n01 = texture2D(u_normals, uv + vec2(-texel.x, 0.0)).rgb;
          vec3 n21 = texture2D(u_normals, uv + vec2(texel.x, 0.0)).rgb;
          vec3 n02 = texture2D(u_normals, uv + vec2(-texel.x, texel.y)).rgb;
          vec3 n12 = texture2D(u_normals, uv + vec2(0.0, texel.y)).rgb;
          vec3 n22 = texture2D(u_normals, uv + vec2(texel.x, texel.y)).rgb;
          vec3 gx = -n00 - 2.0*n01 - n02 + n20 + 2.0*n21 + n22;
          vec3 gy = -n00 - 2.0*n10 - n20 + n02 + 2.0*n12 + n22;
          return length(gx) + length(gy);
        }

        void main() {
          vec2 texel = 1.0 / u_resolution;
          float depthEdge = robertsCrossDepth(vUv, texel);
          float normalEdge = sobelNormal(vUv, texel);
          float edge = clamp(depthEdge * u_depthSensitivity + normalEdge * u_normalSensitivity, 0.0, 1.0);
          vec3 color = mix(u_bgColor, u_lineColor, edge);
          gl_FragColor = vec4(color, 1.0);
        }
      `,
    })
  );

  const edgeScene = new THREE.Scene();
  const edgeCamera = new THREE.OrthographicCamera(-1, 1, 1, -1, 0, 1);
  edgeScene.add(edgeQuad);

  return {
    normalTarget, depthTarget, normalMat,
    edgeScene, edgeCamera, edgeQuad
  };
}

function sceneAnimate(THREE, scene, camera, state, params, time, delta) {
  const renderer = state.renderer; // stash in setup

  // Pass 1: render normals
  scene.overrideMaterial = state.normalMat;
  renderer.setRenderTarget(state.normalTarget);
  renderer.render(scene, camera);

  // Pass 2: render depth
  scene.overrideMaterial = null;
  renderer.setRenderTarget(state.depthTarget);
  renderer.render(scene, camera);

  // Pass 3: edge detection
  renderer.setRenderTarget(null);
  renderer.render(state.edgeScene, state.edgeCamera);
}
```

## Approach 3: Inverted-Hull Outline (Scene Mode)

The simplest outline method — duplicate the mesh, flip normals, scale up slightly, and render
in a solid color behind the original. Fast and GPU-friendly but only produces outer silhouettes,
not internal creases.

```javascript
function createOutlineMesh(THREE, originalMesh, thickness = 0.03, color = 0x000000) {
  const outlineMesh = originalMesh.clone();
  outlineMesh.material = new THREE.MeshBasicMaterial({
    color,
    side: THREE.BackSide, // render only back faces
  });
  outlineMesh.scale.multiplyScalar(1.0 + thickness);
  return outlineMesh;
}

// Usage:
const mesh = new THREE.Mesh(geo, material);
const outline = createOutlineMesh(THREE, mesh, params.outlineThickness);
scene.add(mesh);
scene.add(outline);
```

**Limitations:** Uniform scaling distorts outlines on non-uniform geometry. For better results,
push vertices along their normals in a custom vertex shader:

```javascript
const outlineMat = new THREE.ShaderMaterial({
  uniforms: {
    u_thickness: { value: 0.03 },
    u_color: { value: new THREE.Color(0x000000) },
  },
  vertexShader: `
    uniform float u_thickness;
    void main() {
      vec3 pos = position + normal * u_thickness;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
    }
  `,
  fragmentShader: `
    uniform vec3 u_color;
    void main() {
      gl_FragColor = vec4(u_color, 1.0);
    }
  `,
  side: THREE.BackSide,
});
```

## Approach 4: Toon / Cel Shading with Outlines

Combines flat-shaded color bands with outline rendering for a cartoon or comic book aesthetic.

### Stepped Lighting (Toon Ramp)

```glsl
// In a ShaderMaterial or as part of a shader mode piece
float toonShade(float NdotL, float steps) {
  return floor(NdotL * steps) / steps;
}

// With custom ramp texture for artistic control:
uniform sampler2D u_toonRamp; // 1D gradient texture
float toonShadeRamp(float NdotL) {
  return texture2D(u_toonRamp, vec2(NdotL * 0.5 + 0.5, 0.5)).r;
}
```

### Three.js MeshToonMaterial + Outline

```javascript
function sceneSetup(THREE, scene, camera, renderer, params, seed) {
  // Toon material with 3-step gradient
  const gradientMap = new THREE.DataTexture(
    new Uint8Array([0, 0, 0, 255, 128, 128, 128, 255, 255, 255, 255, 255]),
    3, 1, THREE.RGBAFormat
  );
  gradientMap.minFilter = THREE.NearestFilter;
  gradientMap.magFilter = THREE.NearestFilter;
  gradientMap.needsUpdate = true;

  const toonMat = new THREE.MeshToonMaterial({
    color: new THREE.Color(params.fillColor || "#4488ff"),
    gradientMap,
  });

  const mesh = new THREE.Mesh(new THREE.TorusKnotGeometry(1, 0.4, 128, 32), toonMat);
  scene.add(mesh);

  // Outline via inverted hull
  const outline = mesh.clone();
  outline.material = new THREE.ShaderMaterial({
    uniforms: {
      u_thickness: { value: params.outlineThickness || 0.02 },
      u_color: { value: new THREE.Color(params.outlineColor || "#000000") },
    },
    vertexShader: `
      uniform float u_thickness;
      void main() {
        vec3 pos = position + normal * u_thickness;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
      }
    `,
    fragmentShader: `
      uniform vec3 u_color;
      void main() { gl_FragColor = vec4(u_color, 1.0); }
    `,
    side: THREE.BackSide,
  });
  scene.add(outline);

  scene.add(new THREE.DirectionalLight(0xffffff, 1.5));
  scene.add(new THREE.AmbientLight(0x404040, 0.5));

  return { mesh, outline };
}
```

## Approach 5: EdgesGeometry (Built-in Three.js)

Three.js has a built-in `EdgesGeometry` that extracts edges exceeding a threshold angle.
Simple but static — doesn't detect silhouettes (view-dependent edges).

```javascript
const geo = new THREE.IcosahedronGeometry(2, 1);
const edgesGeo = new THREE.EdgesGeometry(geo, 30); // threshold angle in degrees
const edgeLines = new THREE.LineSegments(
  edgesGeo,
  new THREE.LineBasicMaterial({ color: 0x000000 })
);
scene.add(edgeLines);
```

**Combine with solid fill:**
```javascript
const fillMesh = new THREE.Mesh(geo, new THREE.MeshBasicMaterial({ color: 0xffffff }));
scene.add(fillMesh);
scene.add(edgeLines); // edges render on top
```

This is the quickest path to a wireframe-on-solid look but won't produce true silhouette
outlines that change with camera angle.

## SVG Export from Three.js Line Art

For print-ready vector output, project 3D edge positions to 2D and write SVG:

```javascript
function edgesToSVG(THREE, featureEdges, camera, width, height) {
  const projMatrix = new THREE.Matrix4()
    .multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);

  let svg = `<?xml version="1.0" encoding="UTF-8"?>\n`;
  svg += `<svg xmlns="http://www.w3.org/2000/svg" `;
  svg += `width="${width}" height="${height}" `;
  svg += `viewBox="0 0 ${width} ${height}">\n`;

  const _v = new THREE.Vector3();

  for (const edge of featureEdges) {
    // Project start
    _v.copy(edge.start).applyMatrix4(projMatrix);
    if (_v.z < -1 || _v.z > 1) continue; // behind camera or beyond far plane
    const x1 = ((_v.x + 1) / 2) * width;
    const y1 = ((1 - _v.y) / 2) * height; // flip Y for SVG

    // Project end
    _v.copy(edge.end).applyMatrix4(projMatrix);
    if (_v.z < -1 || _v.z > 1) continue;
    const x2 = ((_v.x + 1) / 2) * width;
    const y2 = ((1 - _v.y) / 2) * height;

    // Stroke weight by type
    const sw = edge.type === "silhouette" ? 2.0 :
               edge.type === "crease" ? 1.0 : 0.5;

    svg += `  <line x1="${x1.toFixed(2)}" y1="${y1.toFixed(2)}" `;
    svg += `x2="${x2.toFixed(2)}" y2="${y2.toFixed(2)}" `;
    svg += `stroke="#000" stroke-width="${sw}" stroke-linecap="round"/>\n`;
  }

  svg += `</svg>\n`;
  return svg;
}

// Download as file:
function downloadSVG(svgString, filename = "line-art.svg") {
  const blob = new Blob([svgString], { type: "image/svg+xml" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = filename;
  a.click();
  URL.revokeObjectURL(url);
}
```

## Line Weight & Style Techniques

### Distance-Based Thickness

Lines farther from camera get thinner — simulates aerial perspective:

```javascript
function buildWeightedEdgeLines(THREE, featureEdges, camera, minWidth = 0.5, maxWidth = 3.0) {
  // Use Line2 from Three.js examples for variable-width lines
  // Requires: LineGeometry, LineMaterial, Line2 from three/examples/jsm/lines/
  const positions = [];
  const widths = [];
  const camPos = new THREE.Vector3();
  camera.getWorldPosition(camPos);

  let minDist = Infinity, maxDist = 0;
  for (const edge of featureEdges) {
    const mid = edge.start.clone().add(edge.end).multiplyScalar(0.5);
    const dist = mid.distanceTo(camPos);
    minDist = Math.min(minDist, dist);
    maxDist = Math.max(maxDist, dist);
  }

  for (const edge of featureEdges) {
    positions.push(edge.start.x, edge.start.y, edge.start.z);
    positions.push(edge.end.x, edge.end.y, edge.end.z);
    const mid = edge.start.clone().add(edge.end).multiplyScalar(0.5);
    const t = (mid.distanceTo(camPos) - minDist) / (maxDist - minDist + 0.001);
    const w = THREE.MathUtils.lerp(maxWidth, minWidth, t);
    widths.push(w, w);
  }

  return { positions, widths };
}
```

### Hatching / Cross-Hatching

Fill shaded regions with parallel lines for an engraving aesthetic. Best done in shader mode:

```glsl
float hatch(vec2 uv, float angle, float density, float thickness) {
  float c = cos(angle), s = sin(angle);
  vec2 rotated = vec2(uv.x * c - uv.y * s, uv.x * s + uv.y * c);
  return smoothstep(thickness, 0.0, abs(fract(rotated.x * density) - 0.5));
}

// Layer multiple angles for cross-hatching based on shade level
float crossHatch(vec2 uv, float shade, float density) {
  float h = 0.0;
  if (shade < 0.75) h += hatch(uv, 0.0, density, 0.4);           // horizontal
  if (shade < 0.50) h += hatch(uv, 1.5708, density, 0.4);        // vertical
  if (shade < 0.25) h += hatch(uv, 0.7854, density * 0.7, 0.3);  // 45 degrees
  return clamp(h, 0.0, 1.0);
}
```

## Compositing Line Art with Color

### Lines Over Flat Color (Scene Mode)

```javascript
// Render order ensures lines draw on top
fillMesh.renderOrder = 0;
edgeLines.renderOrder = 1;
edgeLines.material.depthTest = false; // always visible
```

### Lines Over Lit Scene (Post-Process)

Render the lit scene normally, then composite edges on top by blending:

```glsl
// In the edge post-process fragment shader:
uniform sampler2D u_sceneColor; // original lit render
// ... edge detection code ...

void main() {
  vec2 texel = 1.0 / u_resolution;
  float edge = detectEdge(vUv, texel);
  vec3 sceneColor = texture2D(u_sceneColor, vUv).rgb;
  vec3 lineColor = vec3(0.0);
  vec3 finalColor = mix(sceneColor, lineColor, edge);
  gl_FragColor = vec4(finalColor, 1.0);
}
```

## Parameter Design for Line Art Pieces

Follow the skill's concept-first naming convention:

```javascript
const PARAMS = {
  // Drawing folder
  inkWeight:      { value: 2.0, min: 0.5, max: 5.0, step: 0.1, label: "Ink Weight", folder: "Drawing" },
  creaseDetail:   { value: 60,  min: 10,  max: 120, step: 5,   label: "Crease Detail", folder: "Drawing" },
  showCreases:    { value: true, label: "Show Creases", folder: "Drawing" },
  showSilhouette: { value: true, label: "Show Silhouette", folder: "Drawing" },

  // Style folder
  lineColor:    { value: "#1a1a1a", type: "color", label: "Ink Color", folder: "Style" },
  paperColor:   { value: "#f5f0e8", type: "color", label: "Paper Color", folder: "Style" },
  fillMode:     { value: "none", options: ["none", "flat", "toon", "hatched"], label: "Fill", folder: "Style" },

  // Geometry folder
  complexity:   { value: 2,   min: 0,   max: 4,   step: 1,   label: "Complexity", folder: "Geometry" },
  instanceCount:{ value: 8,   min: 1,   max: 24,  step: 1,   label: "Forms", folder: "Geometry" },
  spread:       { value: 4.0, min: 1.0, max: 10.0,step: 0.5, label: "Spread", folder: "Geometry" },
};
```

## Performance Notes

- **Geometry-based extraction** scales with edge count — fine for meshes under ~50k faces.
  For dense meshes, use screen-space detection instead.
- **EdgesGeometry** precomputes at init — fast at runtime but static.
- **Screen-space detection** is constant cost regardless of geometry complexity (it's a
  fullscreen shader pass), but produces pixel-width lines that may alias at low resolution.
- **Inverted hull** is the cheapest — just one extra draw call, no CPU analysis.
- Recalculating silhouette edges every frame is expensive. Throttle to every 5-10 frames
  or trigger on camera change events.
