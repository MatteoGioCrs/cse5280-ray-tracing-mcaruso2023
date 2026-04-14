# CSE5280 — Ray Tracing Engine (Extended)

A from-scratch ray tracing engine implemented in Python as a Jupyter Notebook, built for CSE5280 (Computer Graphics). It extends a basic ray caster into a recursive ray tracer with physically-motivated shading, shadows, reflections, refractions, and support for multiple camera viewpoints.

---

## Features

| Feature | Details |
|---|---|
| **Phong shading** | Ambient + Lambertian diffuse + specular highlight, computed per light per RGB channel |
| **Hard shadows** | Shadow ray cast from each hit point to each light source |
| **Soft shadows** | Area light approximated by sampling N points on a disk; shadow factor averaged |
| **Mirror reflection** | Recursive reflected ray blended by surface `reflectivity` |
| **Glossy reflection** | Gaussian-perturbed reflection directions averaged over `glossy_samples` rays |
| **Refraction** | Snell's Law in vector form; Total Internal Reflection handled automatically |
| **Checkerboard plane** | Procedural tile texture on the floor plane |
| **Multiple viewpoints** | Render any scene from arbitrary camera positions |
| **Feature flags** | All effects toggled via a single `FEATURES` dict — no code changes needed |
| **Gamma correction** | γ = 2.2 applied at render time for correct sRGB display |
| **Debug passes** | Normal-pass and depth-pass helpers for diagnosing artefacts |

---

## Requirements

```
python >= 3.8
numpy
Pillow
```

Install with:

```bash
pip install numpy Pillow
```

Run inside Jupyter (classic or JupyterLab):

```bash
jupyter notebook cse5280-ray-tracing-mcaruso2023.ipynb
```

---

## Notebook Structure

| Section | What it covers |
|---|---|
| 1. Imports & Setup | `numpy`, `Pillow`, `random`, `math` |
| 2. Ray Equation & Class | Parametric ray `p(t) = e + d·t`; `Ray` class with `at(t)` |
| 3. Material Model | `Material` class — color, ambient, diffuse, specular, shininess, reflectivity, transparency, IOR, glossiness |
| 4. Sphere | Quadratic ray–sphere intersection; outward normal |
| 5. Plane | Ray–plane intersection; optional checkerboard via `material_at(p)` |
| 6. Camera | Pinhole camera; pixel→ray mapping; configurable `eye` position |
| 7. Phong Model & Lighting | `PointLight`; `phong_at_hit()` function |
| 8. Shadows | `Scene.shadow_factor()` — hard and soft shadow sampling |
| 9. Reflection & Refraction | `reflect()` and `refract()` helpers; Snell's Law derivation |
| 10. Recursive Tracer | `trace_ray()` — intersection → Phong → reflection → refraction |
| 11. Feature Flags | `FEATURES` dict |
| 12. Scene Assembly | `build_default_scene()` |
| 13. Render | `render()` with gamma correction; saves to PNG |
| 14. Multiple Viewpoints | Front, high-angle, and side renders |
| 15. Feature Comparison | Side-by-side renders with individual features toggled off |
| 16. Debugging Tips | Artefact table + normal-pass / depth-pass code snippets |

---

## Quick Start

Run all cells top to bottom. The main render cell (Section 13) produces `render_full.png` at 256 × 256. Change `nrows`/`ncols` to 512 for a higher-quality output.

```python
nrows, ncols = 512, 512   # higher quality
camera = Camera(focal, nrows, ncols, eye=[0, 30, 100])
out = render(camera, scene, FEATURES)
out.save("render_full.png")
```

---

## Feature Flags

All effects can be toggled without editing any class or function:

```python
FEATURES = {
    'shadows'             : True,   # enable shadow rays
    'soft_shadows'        : True,   # area-light sampling
    'soft_shadow_samples' : 8,      # samples per light (more = smoother penumbra)

    'reflection'          : True,   # recursive mirror/glossy reflections
    'refraction'          : True,   # transparent objects (Snell's Law)
    'glossy_samples'      : 4,      # rays per glossy bounce (1 = perfect mirror)
    'max_depth'           : 4,      # maximum recursion depth

    'ambient_intensity'   : 0.20,   # global ambient light level
}
```

For a fast preview render, disable the expensive effects:

```python
fast = {**FEATURES, 'soft_shadows': False, 'glossy_samples': 1, 'max_depth': 2}
```

---

## Scene Overview

The default scene (built by `build_default_scene()`) contains:

- **Checkerboard floor plane** at y = −80
- **Red diffuse sphere** — back-left, mildly reflective
- **Green glossy sphere** — front-right, 50% reflective with slight blur
- **Glass sphere** — centre, 80% transparent, IOR 1.5, demonstrates refraction and TIR
- **Warm area light** — top-right, intensity 1.0, radius 70 (soft shadows)
- **Cool fill light** — top-left, intensity 0.4, hard shadows

To add your own objects:

```python
scene.add_object(Sphere(center=[0, 0, -500], radius=60, material=my_material))
scene.add_light(PointLight(position=[0, 300, 0], intensity=1.2, area_radius=50))
```

---

## Multiple Viewpoints

```python
viewpoints = {
    'Front'      : [  0,  30,  100],
    'High angle' : [  0, 350, -200],
    'Side'       : [450,  60, -500],
}
for name, eye in viewpoints.items():
    cam = Camera(focal, 128, 128, eye=eye)
    render(cam, scene, FEATURES).save(f'render_{name}.png')
```

---

## Performance Notes

Render time scales with image resolution, `max_depth`, `soft_shadow_samples`, and `glossy_samples`. Rough guidance at 256 × 256:

| Configuration | Approx. time |
|---|---|
| No shadows, no recursion | ~5 s |
| Hard shadows, depth 2 | ~30 s |
| Soft shadows (8 samples), depth 4, glossy (4 samples) | ~5–10 min |

The engine is a pure Python reference implementation — not optimised for speed. For larger renders, consider reducing sample counts or porting the inner loop to NumPy vectorised operations.

---

## Common Artefacts

| Symptom | Cause | Fix |
|---|---|---|
| Black speckles | Shadow ray hits own surface ("shadow acne") | Increase `t_min` epsilon (`1e-4`) |
| Completely black image | Rays miss all objects, or scene coordinates mismatched | Print `t` for a centre ray; check sphere positions |
| No highlights | `shininess` too high, or `v` vector wrong | Ensure `v = -ray.d`; lower `shininess` |
| No refraction | `transparency` or `ior` near default | Set `transparency > 0` and `ior ≠ 1.0` |
| Noisy soft shadows | Too few shadow samples | Raise `soft_shadow_samples` to 16–32 |
| Noisy glossy reflections | Too few glossy samples | Raise `glossy_samples` to 8–16 |
