# OpenGhost Site

Marketing site for the [OpenGhost](https://github.com/EduardoAllen02/openghost) agentic AI framework.  
Built with **Vite + Three.js**. No UI framework. Pure HTML/CSS/JS.

**Live:** https://eduardoallen02.github.io/openghost-site/

---

## Features

### Ghost 3D — Three.js + GLB + Particle Body

The mascot is rendered entirely in Three.js on a full-viewport `<canvas>`.

**Architecture:**
- **GLB mesh** (`openghost-mascot.glb`) — exports the tech ring and crystal eyes from Blender. Exported with `blender --background --python _export_glb_original.py`.
- **Particle body** — since Blender metaballs don't export to GLB, the ghost body is a `THREE.Points` cloud of ~2800 particles shaped by a GLSL parametric function.
- **ghostGroup** — a single `THREE.Group` contains both the particle layers and the loaded GLB, so all movement happens at one level.

**Ghost shape GLSL** — `ghostR(lat, lon)` defines the surface radius at each latitude/longitude:
```glsl
float ghostR(float lat, float lon) {
  float y = cos(lat);
  if (y > -0.20) {
    return 0.82 + 0.04 * y;           // oval upper body
  } else {
    float t    = (-0.20 - y) / 0.80;
    float taper = 1.0 - t * 0.55;
    float wave  = 1.0 + 0.18 * sin(3.0 * lon + 0.5);  // 3-lobe wavy tail
    return 0.86 * taper * wave;
  }
}
```

**Crystal sparkle** — particles have a per-vertex `aSparkle` attribute (Fibonacci sphere distribution):
- 68% base glass particles
- 22% medium sparkle
- 10% bright highlight

**Eye material** — `MeshPhysicalMaterial` with transmission + clearcoat for a crystal glass look:
```js
const eyeMat = new THREE.MeshPhysicalMaterial({
  transmission: 0.55, thickness: 0.3,
  ior: 1.52, clearcoat: 1.0, clearcoatRoughness: 0.02,
  emissive: new THREE.Color(0.0, 0.898, 0.765), emissiveIntensity: 3.0,
})
```
Requires `renderer.toneMapping = THREE.ACESFilmicToneMapping` to render correctly.

---

### Scroll-Based Animation

The ghost follows scroll-driven **waypoints** defined as normalized screen positions:

```js
const waypoints = [
  { sx: 0.70, sy: 0.38, scale: 1.50 },  // hero — large, right side
  { sx: 0.74, sy: 0.25, scale: 0.80 },  // philosophy
  { sx: 0.86, sy: 0.40, scale: 0.80 },  // architecture
  // ...
]
```

Each waypoint maps `(sx, sy)` to world coordinates via inverse camera projection. The ghost lerps smoothly between positions using `lerp(current, target, 0.04)` every frame. Scale and floating animation amplitude also interpolate per section.

---

### Water Ripple Effect

Circular water ripple distortion on the hero title text (`h1`) on `mousemove`.

**How it works:**

1. A `128×80` canvas acts as a **displacement map** (R = horizontal displacement, G = vertical).
2. The map is driven by a **2D wave equation** simulation running in two Float32 buffers:
   ```js
   // Discrete wave propagation (per frame):
   bufPre[i] = (bufCur[i-1] + bufCur[i+1] + bufCur[i-RW] + bufCur[i+RW]) * 0.5 - bufPre[i]
   bufPre[i] *= DAMP   // energy decay
   // swap buffers
   ```
3. Mouse movement **drops a disturbance** into the buffer (strength ∝ cursor velocity).
4. The canvas gradient is encoded as a data URL and fed into an SVG `feDisplacementMap` filter applied to the `h1`.

The result: physically accurate concentric rings that spread outward from wherever the mouse moves — like water.

**Key insight:** the gradient of the height field `∇h = (∂h/∂x, ∂h/∂y)` gives the displacement direction. Encoding it as `R = gx * GRAD + 128, G = gy * GRAD + 128` maps to the `[0,255]` range where `128` = no displacement.

**Performance tip:** pre-initialize `feImage.href` with the neutral map at load time to warm the browser's filter pipeline and eliminate first-hover latency.

---

## Stack

| Layer | Tech |
|-------|------|
| Bundler | Vite 8 |
| 3D | Three.js r184 |
| 3D models | Blender 4.4 → GLB |
| Shaders | GLSL (inline in JS) |
| Water effect | SVG filters + Canvas 2D |
| CSS | Custom properties, no framework |
| Deploy | GitHub Actions → GitHub Pages |

---

## Development

```bash
npm install
npm run dev      # http://localhost:5173
npm run build    # outputs to dist/
```

---

## Blender export

To regenerate the GLB mascot:

```bash
"C:\Program Files\Blender Foundation\Blender 4.4\blender.exe" \
  --background --python blender/_export_glb_original.py
```

The script exports: tech ring + crystal eyes + 4 hidden orbs (hidden in JS via `obj.visible = false`).
