# Style: Low-Poly Flat Toon, Garden variant ("Pitch Study")

A reusable art direction for a calm, graphic, toy-diorama world: few colors, no textures,
strong silhouettes, one loud accent color, and a Swiss typographic layer on top.
It started from the Absolute Drift look and has since moved away from it: jade water
instead of cyan, an organic island with a paper fern garden instead of box buildings,
and a mouth instead of a car.

> Scope: this describes a look, not any proprietary asset. Rebuild everything from
> primitives and procedural geometry; do not reuse any game's models, logos, or code.

Reference implementation: the Pitch Study UI/UX sample page. When this spec and the
sample disagree, the sample wins and this file should be corrected.

---

## 1. Principles

1. **Flat before fancy.** Every surface is one color plus a light band. No textures, no normal maps, no PBR.
2. **Palette discipline.** Six colors cover everything. Saturation is reserved for one accent.
3. **Shape carries the image.** Slabs, blocks, folded leaflets, thin bars. Read the scene by silhouette and shadow, not by detail.
4. **Negative space is a feature.** Open cream ground between fern groves and jade water around the island make the few objects feel precious.
5. **Motion is the decoration.** Voice trails, fern sway and camera drift add the richness that textures would.
6. **The scene is the interface.** No panels or widgets over the diorama; type sits directly on the scene, set on a Swiss grid.

---

## 2. Palette

| Token | Hex | Role | Typical coverage |
|---|---|---|---|
| `--ground` | `#efead7` | Island surface, fern base color, research page background | ~45% |
| `--water` | `#93c4ac` | Background / clear color (jade) | ~30% |
| `--wall-shade` | `#d2d0c4` | Shaded faces, leaflet shade band | ~12% |
| `--roof` | `#fcfcf2` | Upward faces, leaflet tops, stepping stones, text on red | ~8% |
| `--accent` | `#ef2e2d` | Gates (the objective), brand dot, primary button outline, current-step dot | 1–2% |
| `--ink` | `#302d26` | Text, the mouth, voice trails, fern stems | <1% in the scene |

Derived accent shades: lit `#ef2e2d`, base `#d62625`, shaded `#9b1a1a`.
Derived ink shades (mouth): top `#45413a`, shaded `#1f1d18`.
Projected shadows: `#6a6658` at 22% opacity.

Rules:

- Accent never fills large areas. If red exceeds ~3% of the frame, the image loses its punch.
- Red marks only what the singer must aim for. In the scene that means the gates and nothing else: no red props, rings, cranes or crates.
- Plants stay inside the palette. Ferns are cream and white, drawn like paper cut-outs; their ink stems and their shadows do the work green would.
- No gradients in the environment. Gradients exist only in lighting (soft band edges, vignette).
- Pure black and pure white are not used. Darkest is `--ink` (`#1f1d18` only as the mouth's shaded band), brightest is `--roof`.
- Text on the water must keep at least 4.5:1 contrast. Ink on jade is about 7:1. A dark sea is not a color swap: it would require flipping all text over water to light.
- Single theme by design: the palette does not change in dark mode.

```css
:root{
  --ground:#efead7; --water:#93c4ac; --wall-shade:#d2d0c4;
  --roof:#fcfcf2;   --accent:#ef2e2d; --ink:#302d26;
}
```

---

## 3. Geometry

- **Primitives and procedural shapes only:** boxes, slabs extruded from outlines, low-segment cylinders, flattened icosahedra, folded triangles. Hard edges, flat-shaded faces, no bevels.
- **Island:** a thick slab with a visible faceted rim floating over the water, never an infinite plane. Its outline is an ellipse perturbed by a few low-frequency harmonics (orders 3, 5 and 11) plus slight jitter, extruded about 2.4 units deep. A smaller, deeper tier underneath (about 84% of the outline) reads as a chunk of earth lifted out of the water.
- **Pond:** an irregular hole cut through the island, showing the water below, ringed with small rocks.
- **Islets:** two or three small floating slabs near the island, each with one fern and a rock.
- **Scale rhythm:** many low ferns, the exercise staircase as the main mass, a few tree ferns on trunks as the rare tall verticals.
- **Generative ferns:** each frond is a quadratic curve from a crown, with leaflet pairs that taper toward the tip and lean forward. Each leaflet is a folded diamond (two triangles with a raised midpoint), so it catches two shading bands. Six to nine fronds per fern, about 60 ferns clustered into groves by a smooth field, kept out of the main interaction corridor. Merge all leaflets into one mesh.
- **Tree ferns:** a slender seven-sided trunk 4 to 7.5 units tall with a larger, droopier crown.
- **Rocks:** flattened icosahedra, half sunk, around the pond and along the shore.
- **Stepping stones:** low hexagonal slabs in `--roof`, in a loose line along the front of the staircase.
- **Staircase:** one block per note, 4.6 units wide, height proportional to pitch (a scale reads as stairs, an up-and-down arpeggio as a pyramid). Rebuilt when the exercise changes, at most 8 blocks.
- **Gates:** the objective. Two thin posts and two crossbars in the accent color on each block; the gap between the bars is the current tolerance. Passed gates sink into their block.
- **The mouth:** the hero object. Low-poly lips extruded from a cupid's-bow outline, in mouth ink, with a `--roof` teeth bar and a dark cavity that opens when singing. Tilted toward the camera.
- **Placement is seeded** (`mulberry32`), so the generated garden is identical on every load.

---

## 4. Camera directive

| Setting | Value | Why |
|---|---|---|
| Projection | Perspective | Verticals lean outward; orthographic looks sterile |
| FOV | 32° | Narrow enough to avoid fisheye, wide enough for lean |
| Pitch | 66° above horizon | Mostly top-down but still shows block sides and some height |
| Distance | 46 units in use, 78 on establishing shots (×1.55 in portrait) | Water visible at the frame edges in wide shots |
| Follow | Frame-rate-independent lerp, factor 0.06 at 60 fps | Lazy, floaty feel |
| Yaw sway | ±0.18 rad, very slow | Calmer than ±0.25 during a concentration task |
| Framing | `setViewOffset`: diorama pushed right by 15% of width on wide screens, down by 12% on narrow ones | Leaves the left column for type |
| Tone mapping | None | Keeps flat colors on palette |
| Antialiasing | On (MSAA) | Hard edges need clean lines |
| Reduced motion | Sway and drift off | Respect the user's setting |

Height is foreshortened at this pitch, so anything whose height carries meaning needs extra cues: a plumb line to the surface below and its long projected shadow.

---

## 5. Lighting and shading

- **One directional light**, upper left, low: `L = normalize(-0.55, 0.62, 0.35)`.
- **Two light bands** per surface (lit / shaded) with a narrow soft transition (~0.15 in N·L). No continuous ramp.
- **Roofs** (normal facing up) use `--roof`, regardless of light direction.
- **Double-sided leaves:** fern leaflets render both sides; the shading normal is flipped on back faces so it always faces the viewer.
- **Wall foot darkening:** subtle vertical gradient, up to 25% darker at ground level, clean at the roofline.
- **Distance fade** toward `--ground`, at most 35% mix, only for far geometry.
- **Vignette** on top: transparent until ~60% radius, then up to ~16% warm dark `rgba(40,35,20,.16)`.

### Shadows

- Flat, long, cast on the ground plane only (`y ≈ 0.02`).
- Color `#6a6658` at 22% opacity, which lands near `--wall-shade` on cream.
- Implementation: project the casting geometry onto the ground along the light direction. The island and islets write stencil 1 and render first; shadows test equal to 1 and increment, so they are clipped to land and never darken where they overlap. No shadow ever falls on water or into the pond.
- Shadow meshes use `matrixAutoUpdate = false` and set their local `matrix`; setting `matrixWorld` is overwritten every frame.
- Fern shadows sway with the ferns: the same vertex displacement runs before the projection.
- Avoid shadow maps. Their soft noisy edges clash with the graphic look.

---

## 6. Reference shader (GLSL, Three.js `ShaderMaterial`)

```glsl
// Fragment: two-band toon with roof color and wall AO
uniform vec3  uBase, uTop, uShade, uLight, uFog;
uniform float uFogNear, uFogFar, uAO;
varying vec3 vN, vW;

void main(){
  vec3 n    = normalize(vN);
  // double-sided leaves only: if (!gl_FrontFacing) n = -n;
  float up  = smoothstep(.55, .9, n.y);              // roof mask
  float band= smoothstep(-.05, .12, dot(n, uLight)); // lit vs shaded
  vec3 col  = mix(mix(uShade, uBase, band), uTop, up);

  float ao  = mix(1., smoothstep(0., 7., vW.y)*.25 + .75, uAO*(1.-up));
  col *= ao;

  float d = length(cameraPosition - vW);
  col = mix(col, uFog, smoothstep(uFogNear, uFogFar, d)*.35);
  gl_FragColor = vec4(col, 1.);
}
```

Color management: the sample uses Three.js r128 with `outputEncoding = LinearEncoding` and no colorspace include, so hex values reach the screen unchanged. On r152 or later, add `#include <colorspace_fragment>` and set the output color space so the palette still lands exactly; sample a screenshot to verify.

Wind sway (vertex shader, ferns, their stems and their shadows):

```glsl
uniform float uTime;
attribute float aW;            // 0 at the crown, 1 at the frond tip
vec3 sway(vec3 p){
  float w  = aW * aW;
  float ph = p.x * .23 + p.z * .31;
  return p + vec3(sin(uTime * 1.15 + ph), 0., cos(uTime * .9 + ph * 1.3)) * .16 * w;
}
```

Presets:

| Material | `uBase` | `uTop` | `uShade` | `uAO` |
|---|---|---|---|---|
| Block, trunk, rock | `#efead7` | `#fcfcf2` | `#d2d0c4` | 1 |
| Island and islets | `#efead7` | `#efead7` | `#d2d0c4` | 0 |
| Fern leaflets | `#efead7` | `#fcfcf2` | `#d2d0c4` | 0 |
| Stepping stone | `#fcfcf2` | `#fcfcf2` | `#d2d0c4` | 0 |
| Gate (accent) | `#d62625` | `#ef2e2d` | `#9b1a1a` | 0 |
| Mouth lips | `#302d26` | `#45413a` | `#1f1d18` | 0 |
| Mouth teeth | `#fcfcf2` | `#fcfcf2` | `#d2d0c4` | 0 |

---

## 7. Motion and effects

- **Voice trails:** three thin parallel lines per note, `--ink` at ~85% opacity, recording the mouth's path while the voice is detected. Persistent; the previous run stays at ~22% opacity for comparison, and trails are cleared when the staircase is rebuilt. Lines are 1 px, not ribbons.
- **Mouth read:** the mouth rolls ahead of its vertical speed, leaning into a pitch change the way a drifting car leads its slide. Its jaw opens while singing.
- **Fern stems:** thin `--ink` lines along each frond at ~60% opacity, swaying with the leaves.
- **Gate tolerance** is the one deliberately animated UI value: the bars ease toward the new tolerance after each note.
- **Easing:** everything eases; nothing snaps. The camera is always a little late.
- **Frame loop safety:** clamp the frame delta to [0, 0.05] s (the first animation frame can arrive with a negative delta) and keep the loop body free of code that can throw, or the canvas goes black.
- **No bloom, no depth of field, no film grain, no particles, no color grading beyond the vignette.**

---

## 8. UI and typography (Swiss layer)

- **Grid:** 12 columns, 24 px gutter, 40 px side margins; 4 columns below 860 px. Everything flush left, ragged right. A G key toggles a faint column overlay.
- **Typeface:** one family, Archivo (variable width and weight). Condensed (62% width, weight 800 to 850) for note names, titles and the brand; normal width for body text. Fallback: Helvetica Neue, Helvetica, Nimbus Sans, sans-serif.
- **Hierarchy by scale, not decoration:** a giant condensed note name is the one bold element; everything else is small and quiet. Labels in sentence case, never tracked-out capitals.
- **Rules:** a 2 px ink rule above the top bar and above each research finding, a 1 px rule above the bottom strip.
- **Controls:** text buttons with a 2 px underline when pressed; selects drawn as an underlined word with a small chevron. At least 44 px tall. No cards, pills or panels over the scene.
- **Primary button:** rectangular, 3 px outline in `--accent`, transparent fill, bold condensed label. Hover fills with accent and the label turns `--roof`.
- **Accent dot:** a single red circle as brand punctuation after the wordmark and titles.
- **Research page:** a full-screen `--ground` page on the same grid. Each finding has a text column on the left and an animated SVG figure on the right, in palette colors only. Figures play once when scrolled into view and show their final state with reduced motion. Schematic figures are labelled as schematic.

---

## 9. Do / Don't

| Do | Don't |
|---|---|
| Keep open ground between fern groves | Fill every corner with plants |
| Use red only for what the singer must aim for | Use red for props or decoration |
| Let shadows and ink stems define the ferns | Add green or any new hue for the garden |
| Show water at frame edges | Use an infinite ground plane |
| Give the island an organic outline | Use a rectangular slab |
| Add slow camera drift and fern sway | Use hard camera cuts, shake or a moving background color |
| Set type directly on the scene, on the grid | Put panels or cards over the diorama |
| Stick to the six palette colors | Introduce new hues for variety |

---

## 10. Quick-start prompt

> Create a scene in the "Low-Poly Flat Toon, Garden variant" style: tilted top-down perspective camera (FOV 32°, pitch 66°), an organic faceted island slab with a pond hole floating on flat jade water, small floating islets, a seeded garden of generative low-poly ferns in cream and white with thin dark stems and a gentle wind sway, a few tree ferns as tall verticals, flattened rocks and white hexagonal stepping stones, a single low light from the upper left, two-band toon shading with near-white tops, long flat projected shadows clipped to the land, one red accent used only for objectives, thin dark trail lines, a light vignette, no textures, no tone mapping. Overlay a Swiss 12-column typographic layer in Archivo, flush left. Palette: ground `#efead7`, water `#93c4ac`, shade `#d2d0c4`, roof `#fcfcf2`, accent `#ef2e2d`, ink `#302d26`.

---

## 11. Checklist before shipping a frame

- [ ] Colors stay inside the six-color palette (sample the screenshot to verify).
- [ ] Red covers under ~3% of the frame and appears only on objectives.
- [ ] Water visible on at least two frame edges in wide shots.
- [ ] Every shadow falls the same direction, never darkens when overlapped, and never lands on water.
- [ ] No texture, no tone mapping, no bloom.
- [ ] Tops read lighter than sides; walls darken toward the ground.
- [ ] Text over water and ground keeps at least 4.5:1 contrast.
- [ ] The main interaction corridor is clear of plants.
- [ ] With reduced motion on, sway and camera drift are off and the scene still reads.
