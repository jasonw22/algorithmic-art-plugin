# Analytic Ray Tracing Reference

## Overview

Analytic ray tracing solves ray-primitive intersections exactly using closed-form math
(quadratic/quartic equations), rather than iteratively marching along the ray. This is
faster and more precise for scenes composed of simple geometric primitives (spheres,
planes, boxes, cylinders, discs, tori).

Use analytic ray tracing when:
- Your scene has a small number of distinct geometric primitives
- You need exact intersections (no stepping artifacts)
- Performance matters and the scene doesn't need complex SDF boolean operations

Use raymarching (see `shaders-glsl.md`) when:
- You need smooth blending, domain repetition, or complex SDF compositions
- The scene has organic or fractal geometry

## Ray Definition

```glsl
// Ray: origin + direction × t
// P(t) = ro + rd * t
// Find the smallest positive t where the ray hits a surface
```

## Primitive Intersections

### Sphere

Solve `|P(t) - center|² = radius²` → quadratic in t:

```glsl
// Returns t (distance along ray), or -1.0 if miss
float raySphere(vec3 ro, vec3 rd, vec3 center, float radius) {
  vec3 oc = ro - center;
  float b = dot(oc, rd);
  float c = dot(oc, oc) - radius * radius;
  float disc = b * b - c;

  if (disc < 0.0) return -1.0;

  float sqrtDisc = sqrt(disc);
  float t0 = -b - sqrtDisc;
  float t1 = -b + sqrtDisc;

  // Return nearest positive intersection
  if (t0 > 0.0) return t0;
  if (t1 > 0.0) return t1;  // inside sphere
  return -1.0;
}

// Sphere normal at hit point
vec3 sphereNormal(vec3 hitPoint, vec3 center) {
  return normalize(hitPoint - center);
}
```

### Plane

Solve `dot(P(t), normal) = offset`:

```glsl
float rayPlane(vec3 ro, vec3 rd, vec3 normal, float offset) {
  float denom = dot(rd, normal);
  if (abs(denom) < 1e-6) return -1.0;  // parallel

  float t = -(dot(ro, normal) + offset) / denom;
  return t > 0.0 ? t : -1.0;
}

// Ground plane (y = 0)
float rayGroundPlane(vec3 ro, vec3 rd) {
  return rayPlane(ro, rd, vec3(0, 1, 0), 0.0);
}
```

### Axis-Aligned Box (Slab Method)

Intersect with 3 pairs of parallel planes:

```glsl
// Returns vec2(tEntry, tExit), or vec2(-1) if miss
vec2 rayBox(vec3 ro, vec3 rd, vec3 boxMin, vec3 boxMax) {
  vec3 invRd = 1.0 / rd;
  vec3 t0 = (boxMin - ro) * invRd;
  vec3 t1 = (boxMax - ro) * invRd;
  vec3 tmin = min(t0, t1);
  vec3 tmax = max(t0, t1);

  float tEntry = max(max(tmin.x, tmin.y), tmin.z);
  float tExit = min(min(tmax.x, tmax.y), tmax.z);

  if (tEntry > tExit || tExit < 0.0) return vec2(-1.0);
  return vec2(tEntry, tExit);
}

// Box normal from hit point
vec3 boxNormal(vec3 hitPoint, vec3 boxCenter, vec3 boxSize) {
  vec3 d = (hitPoint - boxCenter) / boxSize;
  vec3 a = abs(d);
  float maxComp = max(a.x, max(a.y, a.z));

  if (a.x >= maxComp - 0.001) return vec3(sign(d.x), 0, 0);
  if (a.y >= maxComp - 0.001) return vec3(0, sign(d.y), 0);
  return vec3(0, 0, sign(d.z));
}
```

### Disc (Bounded Plane)

```glsl
float rayDisc(vec3 ro, vec3 rd, vec3 center, vec3 normal, float radius) {
  float t = rayPlane(ro, rd, normal, -dot(center, normal));
  if (t < 0.0) return -1.0;

  vec3 hitPoint = ro + rd * t;
  if (length(hitPoint - center) > radius) return -1.0;
  return t;
}
```

### Cylinder (Infinite, then Capped)

```glsl
// Infinite cylinder along Y axis, centered at origin
float rayInfCylinder(vec3 ro, vec3 rd, float radius) {
  // Project to xz plane
  float a = rd.x * rd.x + rd.z * rd.z;
  float b = 2.0 * (ro.x * rd.x + ro.z * rd.z);
  float c = ro.x * ro.x + ro.z * ro.z - radius * radius;
  float disc = b * b - 4.0 * a * c;

  if (disc < 0.0) return -1.0;

  float sqrtDisc = sqrt(disc);
  float t0 = (-b - sqrtDisc) / (2.0 * a);
  float t1 = (-b + sqrtDisc) / (2.0 * a);

  if (t0 > 0.0) return t0;
  if (t1 > 0.0) return t1;
  return -1.0;
}

// Capped cylinder: check height bounds and caps
float rayCappedCylinder(vec3 ro, vec3 rd, vec3 center, float radius, float halfHeight) {
  vec3 oc = ro - center;
  float bestT = 1e10;
  bool hit = false;

  // Side
  float t = rayInfCylinder(oc, rd, radius);
  if (t > 0.0) {
    float y = oc.y + rd.y * t;
    if (abs(y) <= halfHeight) { bestT = t; hit = true; }
  }

  // Top and bottom caps
  for (int i = 0; i < 2; i++) {
    float capY = (i == 0) ? halfHeight : -halfHeight;
    float capT = (capY - oc.y) / rd.y;
    if (capT > 0.0 && capT < bestT) {
      vec2 hitXZ = oc.xz + rd.xz * capT;
      if (dot(hitXZ, hitXZ) <= radius * radius) {
        bestT = capT;
        hit = true;
      }
    }
  }

  return hit ? bestT : -1.0;
}
```

### Torus

Quartic equation (4th degree polynomial). Requires numerical root finding or
specialized solver:

```glsl
// Simplified: use raymarching for torus (quartic is expensive and complex)
// Or use Newton's method after initial bracketing with bounding sphere
float rayTorus(vec3 ro, vec3 rd, float R, float r) {
  // R = major radius, r = minor radius
  // Bounding sphere check first
  float boundT = raySphere(ro, rd, vec3(0), R + r);
  if (boundT < 0.0) return -1.0;

  // Refine with raymarching inside bounding sphere
  float t = max(boundT, 0.0);
  for (int i = 0; i < 64; i++) {
    vec3 p = ro + rd * t;
    float q = length(p.xz) - R;
    float d = length(vec2(q, p.y)) - r;
    if (d < 0.001) return t;
    t += d;
    if (t > 100.0) break;
  }
  return -1.0;
}
```

## Scene Composition

Combine primitives by testing each and taking the nearest hit:

```glsl
struct Hit {
  float t;
  vec3 normal;
  vec3 albedo;
};

Hit sceneIntersect(vec3 ro, vec3 rd) {
  Hit best;
  best.t = 1e10;

  // Ground plane
  float tPlane = rayGroundPlane(ro, rd);
  if (tPlane > 0.0 && tPlane < best.t) {
    best.t = tPlane;
    best.normal = vec3(0, 1, 0);
    best.albedo = vec3(0.4);
  }

  // Red sphere
  float tSphere = raySphere(ro, rd, vec3(0, 1, 0), 1.0);
  if (tSphere > 0.0 && tSphere < best.t) {
    vec3 p = ro + rd * tSphere;
    best.t = tSphere;
    best.normal = sphereNormal(p, vec3(0, 1, 0));
    best.albedo = vec3(0.8, 0.2, 0.1);
  }

  // Blue box
  vec2 tBox = rayBox(ro, rd, vec3(2, 0, -1), vec3(3, 1.5, 0.5));
  if (tBox.x > 0.0 && tBox.x < best.t) {
    vec3 p = ro + rd * tBox.x;
    best.t = tBox.x;
    best.normal = boxNormal(p, vec3(2.5, 0.75, -0.25), vec3(0.5, 0.75, 0.75));
    best.albedo = vec3(0.1, 0.3, 0.8);
  }

  if (best.t > 1e9) best.t = -1.0;
  return best;
}
```

## When to Combine with Raymarching

You can mix analytic and raymarched objects in one scene:

1. Test analytic primitives first (fast, exact)
2. If the analytic hit is far enough, also run a raymarch for SDF objects
3. Take the nearest hit from either method

```glsl
Hit analyticHit = sceneIntersect(ro, rd);
float sdfHit = raymarch(ro, rd);

if (sdfHit > 0.0 && (analyticHit.t < 0.0 || sdfHit < analyticHit.t)) {
  // SDF object is closer
  // ... shade SDF object
} else if (analyticHit.t > 0.0) {
  // Analytic object is closer
  // ... shade analytic object
}
```

## Reflection and Refraction

Analytic tracing makes recursive reflection/refraction natural:

```glsl
vec3 traceReflection(vec3 ro, vec3 rd, int maxBounces) {
  vec3 color = vec3(0.0);
  vec3 throughput = vec3(1.0);

  for (int i = 0; i < maxBounces; i++) {
    Hit hit = sceneIntersect(ro, rd);
    if (hit.t < 0.0) {
      color += throughput * sky(rd);
      break;
    }

    vec3 p = ro + rd * hit.t;

    // Direct lighting
    color += throughput * hit.albedo * directLight(p, hit.normal);

    // Reflect
    rd = reflect(rd, hit.normal);
    ro = p + hit.normal * 0.001;
    throughput *= 0.5;  // reflection attenuation
  }
  return color;
}

// Snell's law refraction
vec3 refractRay(vec3 I, vec3 N, float eta) {
  float cosi = -dot(I, N);
  float k = 1.0 - eta * eta * (1.0 - cosi * cosi);
  if (k < 0.0) return reflect(I, N);  // total internal reflection
  return eta * I + (eta * cosi - sqrt(k)) * N;
}
```

## Key References

- **Peter Shirley** — "Ray Tracing in One Weekend" (sphere, plane, box intersections)
- **Inigo Quilez** — iquilezles.org/articles/intersectors/ — definitive ray-primitive catalog
- **Real-Time Rendering** — Akenine-Möller et al., ray intersection chapter
- **Shadertoy** — search "analytic" for exact-intersection examples
