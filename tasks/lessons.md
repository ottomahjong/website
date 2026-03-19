# Otto Tile Studio — Geometry Lessons

## NEVER apply RoundedBoxGeometry per-layer.
Bevel only the top edge of the cap and bottom edge of the base.
All layers share one silhouette.

---

## Lesson 1 — RoundedBoxGeometry is wrong for layered tiles
`RoundedBoxGeometry` rounds ALL 12 edges of a box equally. It cannot produce a
tile with flat vertical sides. Never use it for layered tiles. It was the root
cause of the "muffin" effect where the cap bulged wider than every other layer.

## Lesson 2 — ExtrudeGeometry bevel is the correct tool
`ExtrudeGeometry` bevel only affects the extruded (top) face, not the sides.
This is exactly what a physically-correct laminate tile requires:
- Cap layer: bevel on top face → glassy lozenge / pillow look
- Base layer: bevel on bottom face → symmetric floor-contact curve
- Body / Inlay: bevelEnabled: false → perfectly straight sides, flush silhouette

## Lesson 3 — bevelSize must always be less than the smallest corner radius
Add a safeBevel guard to EVERY bevel calculation:
```js
const safeBevel = Math.min(
  bevel,
  cornerTL, cornerTR, cornerBL, cornerBR,
  th * 0.45   // never deeper than ~half the layer thickness
);
```
If bevel > corner radius, Three.js produces degenerate geometry (triangles fold
inside out at corners). This guard prevents that at all times.

## Lesson 4 — Flipping the base geometry puts the bevel on the bottom face
Three.js ExtrudeGeometry always places the bevel at the extruded (top) face.
To move it to the bottom face without changing mesh.position.y:

```js
// After geo.applyMatrix4(rotMatrix):
//   bevel is at local Y = +th (top)  →  we want it at Y = 0 (bottom)
//
// makeRotationX(π): Y → -Y, Z → -Z   (proper rotation, preserves winding + normals)
//   bevel moves to Y = -th
// makeTranslation(0, +th, 0): shift the whole geo up by th
//   bevel is now at Y = 0  ✓
//   flat top is at Y = +th ✓
//   mesh.position.y stays unchanged ✓

const flipAndShift = new THREE.Matrix4()
  .makeTranslation(0, th, 0)
  .multiply(new THREE.Matrix4().makeRotationX(Math.PI));
geo.applyMatrix4(flipAndShift);
```
Because `makeRotationX(π)` has det = +1, face winding and normals are preserved
correctly. No need to call `computeVertexNormals()` or set DoubleSide.

## Lesson 5 — "Muffin effect" = bevel expanding outward at layer boundaries
Root cause is ALWAYS per-layer rounding on geometry that should share a single
silhouette. The fix: one `buildSilhouette(w, h, tl, tr, bl, br)` shape, N
extrusions using that same shape, no per-layer width or height adjustments.

## Lesson 6 — All layers of a physical laminate share one XY footprint
Corner radii control the 2D XY silhouette shape (viewed from above).
Edge bevel controls the 3D top/bottom face curve (viewed from the side).
These are completely independent controls and must never be confused.
