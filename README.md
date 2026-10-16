Tool Name > # td_perspective_matcher_maya2026-
Tool Creator >>  @Nameisarrtist 
-----------------------------------------------------
For more connection With Me 
linkdin > SameeerKumamekar
Instagram  > _hfxSam23
Email > ArrtixtStudio@Gmail.com

----------------------------------------------------


# TD Perspective Matcher 3.1.0 — Complete Workflow Guide (Maya 2026 / PySide6)

A practical, step-by-step manual for `td_perspective_matcher_maya2026.py`:
installation, the full camera-matching workflow, the three workspace tabs,
calibration files, troubleshooting, and a **verification appendix** with two
numeric checks you can run yourself (plus two real bugs found while verifying,
with exact one-line fixes).

---

## 0. What the tool can and cannot do

**It solves (from your drawn guides):**
- camera **orientation** (a full orthonormal rotation),
- **focal length** (from orthogonal vanishing points, or from a value you enter),
- **principal point / lens shift** (entered manually, or estimated as the
  orthocenter of three finite orthogonal VPs).

**It does NOT solve (physics, not a limitation of the code):**
- camera **translation** — the applied camera keeps whatever position it already had,
- **scene scale** — blockout dimensions are in your current scene units,
- **lens distortion** — feed it undistorted, rectilinear photos only.

Assumptions baked into the math: square pixels, zero skew, three mutually
orthogonal scene axes, pinhole projection. Everything the tool writes to the
scene is undoable (named undo chunks, see §10).

---

## 1. Installation and launch

1. Copy `td_perspective_matcher_maya2026.py` into a folder on Maya's
   `PYTHONPATH` (e.g. `<prefs>/scripts`).
2. In Maya's Script Editor (Python):
   ```python
   import td_perspective_matcher_maya2026 as tpm
   tpm.show()              # dockable window (default)
   tpm.show(dockable=False)  # floating window
   ```
3. Optional shelf button (select a shelf tab first):
   ```python
   import td_perspective_matcher_maya2026 as tpm
   tpm.install_shelf_button()
   ```

Requirements / behaviour at launch:
- Maya **2026+** with **PySide6** (guarded: older Maya raises, batch mode warns).
- **NumPy optional.** Without it the colour preview degrades to opacity-only
  (the sliders disable themselves and a label tells you).
- Dockable via `MayaQWidgetDockableMixin`; if the mixin is missing the tool
  falls back to a plain window.
- Re-running `show()` closes any previous instance (no duplicate docks).
- Preferences (last directory, custom sensor width) persist in the optionVar
  `tdPerspectiveMatcher2026_settings` on close.

---

## 2. UI map (30-second tour)

```
┌────────────────────────────────────────────────────────────────────┐
│ PERSPECTIVE / MATCH                          [Workflow / Help]     │
│ TARGET CAMERA  [ combo ▾ ] [Refresh cameras] [Use selected camera] │
│ ┌───────────────────────────────┬────────────────────────────────┐ │
│ │  Reference canvas             │ Tabs: Match │ Camera │ Images  │ │
│ │  (draw guides here)           │       │ Blockout               │ │
│ │                               │  01 Camera & calibration       │ │
│ │                               │  02 Reference preview          │ │
│ │                               │  03 Solve & apply              │ │
│ │                               │  [log]                         │ │
│ └───────────────────────────────┴────────────────────────────────┘ │
│ X 0/2  Y 0/2  Z 0/2   [Grid][100 px]                               │
│ status line                                                        │
└────────────────────────────────────────────────────────────────────┘
```

- **Target camera combo** (top): the camera every Apply/edit acts on.
  `Use selected camera` adopts the current Maya selection (exactly one
  perspective camera transform or shape).
- **Canvas** (left): guides live in *native image pixels*; large references are
  only downscaled for the preview pixmap (longest edge > 1600 px), the solve
  always uses the original resolution.
- **Tabs** (right): *Match* (the solver), *Camera* / *Images* / *Blockout*
  (manual scene workspaces, §7–§9).

---

## 3. Core workflow — Match tab

### Step 1 · Load a reference photograph
`Open image` → PNG/JPG/TIFF/BMP (EXR/HDR will be refused by the canvas with a
hint to convert). EXIF orientation is deliberately **not** applied, so canvas
coordinates stay identical to the coordinates Maya's image plane uses.
The image auto-fits; `Fit [F]` re-fits after zooming.

### Step 2 · Draw guides (the only artistic judgement in the whole tool)
Pick an axis button **X / Y / Z**, then **left-drag along a real scene edge**
that is parallel to that world axis. Repeat until the counter reads `2/2` for
that axis; do the same for at least one more axis.

Rules enforced or implied by the code:
- exactly **2 guides per axis** (drawing a 3rd silently retires the oldest);
- each guide ≥ **4 px** long at release, else discarded;
- the two guides of one axis must be **different, non-parallel-on-screen**
  edges — they must *converge* toward their vanishing point. Two parallel
  drawn lines produce a VP at infinity (allowed, but then that axis gives no
  focal-length information);
- drag the white end handles to refine a guide afterwards;
- `Clear axis` / `Clear all` to start over; **Esc** or right-click cancels the
  rubber-band line.

Good guides: long (> ~300 px), well separated in the frame, traced on crisp
man-made edges (wall/floor junctions, window frames, ceiling grids, crate
edges). Bad guides: texture streaks, shadows, hand-drawn art, curved edges,
two nearly-overlapping lines ("Two guides overlap" error).

Mouse: **LMB** draw / move handle · **MMB** pan · **wheel** zoom (0.05–60×) ·
**F** fit · **Esc** cancel.

### Step 3 · Calibration inputs (group *01 Camera & calibration*)
| Control | Meaning | Notes |
|---|---|---|
| `Create` combo + `New` | builds the target camera | *Free*, *Aim* (managed aim-constraint rig), *Blender-style film gate* (a free camera with overscan 1.0 + gate-mask preset). New cameras start at (0,0,10) |
| `Sensor width` | physical sensor width behind the pixels | presets: Full frame 36, Super 35 24.89, Academy 22, APS-C 22.3, Micro 4/3 17.3, Alexa 65 54.12, Custom (enables the mm spin, 1–100). **This is what converts px-focal to mm** |
| `Known focal length` + mm | fixes f instead of solving it | use when you trust the EXIF/lens, or when a VP is at infinity / only one finite VP exists |
| `Estimate lens shift from 3 finite VPs` | principal point = orthocenter of the VP triangle | needs **three finite** VPs; disables the manual shift fields; refuses flat VP triangles |
| `Shift X / Y` | manual principal-point offset, normalized (±2) | X+ = right, Y+ = up; multiplied by image width/height internally |
| `Flip X/Y/Z` | sign choice of each solved world direction | VPs are sign-ambiguous; flips pick the orientation. The triple must stay **right-handed** (left-handed combos are rejected with an explicit error) |
| `Clip near / far` | clipping planes written on Apply | 0.001–1e7 / up to 1e8, near < far |

### Step 4 · `SOLVE CAMERA`
The result label shows focal (mm), HFOV, orthogonality error (°), principal
point (px, centre-relative), and a message. Read the **orthogonality error**
as your quality meter:
- `< 0.5°` — excellent guides;
- `0.5–2°` — acceptable, refine the worst guide;
- `> 8°` — the solve aborts ("Orthogonality error is …").

Sanity-check the focal: it should look like the lens you shot with. A wildly
wrong mm with tiny residual usually means a mislabelled axis or a curved edge.

**Choosing flips (the one trial-and-error step).** Solve, then Apply to a
*scratch* camera and look through it with the image plane on (§Step 6). If the
view is mirrored/rolled, toggle flips and re-apply. The tool rejects
left-handed triples; of the four surviving combinations exactly one (and its
180°-rolled complement) fits — the viewport check resolves it in seconds.
For the shipped practice plate the correct set is **all three flips ON**
(see §11).

### Step 5 · Apply options (group *03 Solve & apply*)
- `Create / update camera image plane` — attaches (or re-points) an image plane,
  `fit = 1` ("match best"), `displayOnlyIfCurrent = True`.
- `Look through camera after apply` — switches the focused (or first) model panel.
- `Set render resolution to reference size` — opt-in, **scene-global**:
  `defaultResolution.width/height`, `pixelAspect = 1`, `deviceAspectRatio = w/h`.
  The tool refuses if any of those four plugs is locked or driven.

`APPLY TO TARGET CAMERA` always **re-solves first** (a stale solve can never be
applied), then in one undo chunk (`TPM_Apply` → `TPM_ApplyMatch`):
writes filmback (`filmFit = horizontal`, vertical aperture = sensor·h/w),
focal length, film offsets = −principal point (sign-flipped, mm), clip planes,
resets every projection-killer (`lensSqueezeRatio`, `cameraScale`, `preScale`,
`postScale`, `filmTranslateH/V`, `filmRollValue`, `panZoomEnabled`,
`shakeEnabled`, `filmFitOffset`), rotates the camera **in place** (translation
is preserved), toggles resolution gate + gate mask on, and (optionally)
creates/updates the image plane and pushes exposure/brightness/opacity to it.

Guards you will meet: orthographic targets rejected; locked or
expression/constraint-driven attributes rejected plug-by-plug with the driver
named ("Driven attribute: … ← … Use a new match camera."); missing image file
rejected; a plane selected in *Images* that is not attached to the target
camera rejected.

For an **Aim-type** camera the rotation channels belong to the aim constraint;
Apply instead re-aims: `worldUpVector` becomes the solved camera-up axis and
the managed locator is moved along the solved view axis at the previous
target distance. Exactly one aim target is required.

### Step 6 · Verify in the viewport (do this every time)
With the plane visible and the camera looked through:
1. image edges sit on scene edges at the frame corners **and** the centre;
2. verticals in the photo stay vertical in the view;
3. orbit slightly — the pin should hold (a perspective match is exact, not
   "close enough at one angle").
If it mirrors/rolls: flip combo wrong (§Step 4). If it shears: guides or lens
shift wrong. If it scales oddly: sensor width wrong.

### Step 7 · Finish the match
- Move the camera (or the scene) to fix **translation** — never recoverable
  from guides; the Camera tab has explicit position/rotation controls.
- Scale/translate your blockout to real-world size using one known dimension.
- If render resolution was not set, match it manually before rendering
  (the log reminds you).
- **Save the Maya scene** (.ma/.mb). The calibration JSON does *not* store
  cameras, planes, shaders or meshes (§6).

---

## 4. Reference preview (group *02*) — display-space only

`Exposure / Brightness / Contrast / Gamma / Saturation / Opacity` re-render the
canvas through a NumPy worker thread (single-thread pool, generation-counter
debounce, ~90 ms timer) so the UI never blocks. `Original` bypasses the chain.
`Apply exposure / brightness / opacity to plane` mirrors **only those three**
onto the Maya image plane (`colorGain = 2^exp`, `colorOffset = brightness`,
`alphaGain = opacity`) — gamma/contrast/saturation have no image-plane
equivalent and stay preview-only. This is a viewing aid, not colour management
(no OCIO involvement).

---

## 5. Calibration JSON (Save / Load calibration)

Schema v5 (accepts v4): `image`, `image_size`, `sensor_mm`, `camera_type`,
`lines` (≤2 segments/axis, endpoints in native px), `flips`, `color`,
`camera_settings` (clips, lens shift, known_focal, focal_mm, estimate_pp).
- Save validates its own payload and writes **atomically** (temp file +
  `os.replace`), `allow_nan=False`.
- Load validates *before* touching the UI; a relative `image` path resolves
  next to the JSON; if the file is missing you get a relocate dialog; if the
  located image's pixel size ≠ `image_size` the load is refused (guides are
  never silently rescaled).
- Loaded calibrations always end with "Solve to validate before applying".

---

## 6. Camera tab — manual camera surgery (no solve involved)

Header: `Refresh` (snapshot pose + bind the attribute form), `Look through`,
`Select`, `Open complete Maya Attribute Editor`, rename field + `Rename`,
`Clone as free camera (no image plane)`.

- **Transform group**: world position / rotation fields with `Set position` /
  `Set rotation`, plus channel `Lock/Unlock T/R`. Pose edits are guarded by a
  snapshot: if Maya moved the camera since Refresh you are told to refresh
  instead of clobbering. Aim cameras refuse manual rotation (edit the rig).
- **Managed aim rig group**: target position + up vector + `Apply aim target /
  up` (rejects up ∥ aim direction; protects driven plugs).
- **Attribute form** (30 fields: focal, film apertures/offsets, filmFit,
  squeeze, clips, overscan, scale, COI, DOF/focus/f-stop, renderable, gates,
  safe areas, field chart, pan/zoom, film roll): capability-aware (missing
  attributes disable themselves), snapshot-diffed — **only changed fields are
  written**, and a field whose scene value drifted since Refresh raises
  instead of overwriting. Locked plugs are reported in tooltips and rejected.
- `Use current lens / gate as Match inputs`: copies focal, sensor (as Custom),
  film offsets → normalized shifts, clips into the Match tab and enables
  *Known focal length*. Explicit one-way sync, logged.

---

## 7. Images tab — image planes without solving

Combo of planes attached to the target camera + `Refresh`; read-only file path.
- `Attach matcher reference as NEW plane` / `Browse image + attach NEW plane`
  (dialog includes EXR/HDR for planes even though the canvas can't show them),
- `Replace selected plane's image file`,
- `Load selected plane image into Match` (asks before clearing drawn guides),
- attribute form (display mode, only-if-current, locked-to-camera, fit,
  maintain ratio, depth, size, offsets, alpha gain, color gain/offset,
  sequence frame/offset) with the same changed-only/locked-aware semantics,
- `Open complete image-plane Attribute Editor`, `Delete selected image plane…`
  (deletes the shape only — a shared parent transform is never touched).

---

## 8. Blockout tab — manual massing in scene units

Nothing here is inferred from the photo; you are modelling by eye against the
matched camera.

- **Primitive settings**: name prefix, Size (local extents before rotation),
  Position, Rotation, placement mode (*World coordinates / Selected bounds
  center / Along target camera view* at a given camera distance), ground snap
  + ground elevation (respects Y-up and Z-up Maya), blockout colour.
- **Create**: Cube / Plane / Cylinder / Sphere / Locator → created, scaled +
  frozen, assigned a tool-owned lambert `TPM_Blockout_<hex>_MAT` (+ its SG),
  tagged `tpmBlockout`, parented under `TPM_BLOCKOUT_GRP` (tagged
  `tpmBlockoutRoot`), optionally floor-snapped, selected.
- **Selection tools**: read bounds+pose into the fields; apply pose; resize a
  single mesh to world extents (vertex-level, keeps flat axes flat, rejects
  instanced/referenced meshes); snap bases to ground; assign blockout colour;
  duplicate; move into the blockout group. Multi-select supported where noted.
- **Viewport helpers**: look through target, select group, frame selection,
  wireframe-on-shaded toggle, AE of selection.

Typical loop: Apply match → look through camera → drop a cube where the photo
shows a mass → `Read single selection` → tweak → `Apply position / rotation` →
repeat. Use one real-world measurement to set scene scale.

---

## 9. Undo, logging, and safety conventions

- Every scene mutation runs inside a named undo chunk: `TPM_CreateCamera`,
  `TPM_Apply` (wraps `TPM_ApplyMatch`), `TPM_RenameCamera`, `TPM_CloneCamera`,
  `TPM_CameraPosition/Rotation/ChannelLocks`, `TPM_EditAim`,
  `TPM_EditCameraAttributes`, `TPM_AttachImagePlane`, `TPM_ReplacePlaneImage`,
  `TPM_EditImagePlane`, `TPM_DeleteImagePlane`, `TPM_PlaneColor`,
  `TPM_CreateBlockout`, `TPM_PlaceBlockout`, `TPM_ResizeBlockoutVertices`,
  `TPM_GroundBlockout`, `TPM_ColorBlockout`, `TPM_DuplicateBlockout`,
  `TPM_OrganizeBlockout`. One Ctrl+Z reverts a whole action.
- Inspectors never write on open/refresh — only explicit buttons write.
- The log pane + status line echo every action and error; errors also pop a
  modal with the same text.
- Maya commands and QPixmap work stay on the main thread; only QImage+NumPy
  pixel math runs in the worker.

---

## 10. Troubleshooting (exact messages → fixes)

| Message | Cause → fix |
|---|---|
| `Guide segments must be at least 4 image pixels long.` | drag longer guides |
| `Two guides overlap. Use separate parallel scene edges.` | pick genuinely distinct edges |
| `Draw two guides on each of at least two world axes.` | finish the 2/2 counters |
| `Focal length is underdetermined. Enable Known focal length or provide two finite vanishing points.` | a VP is at infinity (parallel drawn guides) → enter focal, or re-draw converging guides on another axis |
| `Guides do not describe orthogonal directions for this principal point…` | wrong axis labels, curved edges, or real lens shift → fix guides / enter shift |
| `Implausible focal length (…) mm.` | guides/sensor inconsistent |
| `Orthogonality error is X°.` | refine guides (>8° aborts) |
| `Axis signs form a left-handed frame. Toggle one axis flip.` | flip combo not a rotation → toggle exactly one flip |
| `Vanishing-point triangle is too flat to estimate lens shift.` | uncheck *Estimate*, enter shift manually |
| `Select an existing camera.` / `Not a camera: …` / orthographic error | pick/refresh a perspective camera |
| `Locked attribute: …` / `Driven attribute: … ← …` | unlock, remove the driver, or use a fresh match camera |
| `Clipping planes must satisfy 0 < near < far.` | fix clip fields |
| `Reference image no longer exists: …` | relink/reattach the file |
| `Selected image plane is not attached to the target camera. Refresh Images.` | refresh the Images combo |
| `Cannot decode image with this Qt installation…` | convert EXR/HDR to PNG/TIFF for the canvas |
| `Reference dimensions differ from the saved calibration…` | locate the original-resolution image |
| `{attr} changed in Maya since Refresh.` / `Camera pose changed in Maya.` | Refresh, then edit |
| `Open a Maya model viewport first.` | focus a model panel (blockout/look-through tools) |

---

## 11. Verification appendix — practice plate, numbers, and two real fixes

To make this guide testable, a synthetic rectilinear "photograph" was rendered
from a known camera and the shipped solver was run on it **outside Maya**
(stubbed `maya.*`, real PySide6/NumPy), then re-checked with simulated Maya
`xform` matrix semantics.

**Files (folder `tpm_workspace_example/`):**
- `tpm_reference_room.png` — 3000×2000 practice plate (wireframe room: floor &
  ceiling grids, two walls, doorway, crates). Load it and trace guides.
- `tpm_reference_room_annotated.png` — same plate with the six verified guides
  drawn in the tool's axis colours (X red, Y green, Z blue).
- `tpm_reference_room_calibration.json` — loadable calibration containing those
  exact guides and the verified flip set.
- `_make_reference.py` — the generator + ground-truth printer.
- `tpm_verify_in_maya.py` — run **inside Maya** (`tv.run()`): builds the same
  room as curves, a camera with a known pose, solves from the six guides, and
  prints the reprojection error of the applied orientation.

**Ground truth of the plate:** 24 mm on 36 mm → f = 2000 px, HFOV 73.74°,
camera at (6, 2.4, 7) looking at (0.6, 1.0, 0.6), world +Y up.
Verified guide endpoints (native px) and VPs:

```
X: (1224.3,1619.9)->(2949.3,1851.4)   (1244.4,699.3)->(2768.0,248.8)     VPx (-903.4, 1334.4)
Y: (1706.7,  95.8)->(1741.1,1936.3)   (1068.4,398.7)->(1016.2,1771.3)    VPy (1500.2,-10954.5)
Z: (1811.7,1612.9)->( 309.2,1912.1)   (1789.4,713.6)->( 473.2,138.8)     VPz (3210.7, 1334.3)
```

**What the shipped v3.1.0 code gets right:** focal length
`23.9934 mm` (0.03 % off truth; exactly 24.0000 with unrounded guide coords),
HFOV `73.755°`, orthogonality residual `≈0.013°`, principal-point estimate
`(0.0, 0.0)`, handedness rejection of the four left-handed flip combos, and
the 1–10000 mm / clip / shift range guards.

**What it gets wrong (verified numerically, two independent one-line bugs):**

1. **`solve_camera` — inverted image-v convention.**
   The camera-plane y coordinate is built as `(height*.5-pp[1])*v[2]-v[1]`
   (y-down) while the back-projection `(u, v, -f)` expects y-up. Effect: the
   solved frame is vertically mirrored; with the practice plate the accepted
   flip set produces a view whose vertical is flipped (reprojection error
   ≈ 2 300 px in a Maya-semantics simulation).
   **Fix** (one line in `solve_camera`):
   ```python
   # was:  (height*.5-pp[1])*v[2]-v[1], v[2]) for ax, v in vps.items()}
                (v[1]-(height*.5-pp[1])*v[2], v[2]) for ax, v in vps.items()}
   ```
2. **`apply_camera` — rotation matrix handed to `xform` is transposed.**
   `solution.camera_axes` rows are the camera axes **in world space**; Maya's
   `xform -matrix` (row-vector convention) needs them as **columns**, i.e. the
   transpose. As shipped, the camera's right/up/back end up as the world-axis
   directions instead.
   **Fix** (replace the single `matrix = …` line in `apply_camera`):
   ```python
   m = (xa, ya, za)   # rows = camera axes in world
   matrix = [m[0][0], m[1][0], m[2][0], 0.,
             m[0][1], m[1][1], m[2][1], 0.,
             m[0][2], m[1][2], m[2][2], 0.,
             *position, 1.]
   ```
   (The aim-camera branch already uses `ya`/`za` correctly and is untouched.)

**With both fixes applied**, for flip set `{X: on, Y: on, Z: on}` the matrix
that reaches `xform` reproduces the photograph with a maximum reprojection
error of **0.15–1.5 px** over 8–12 scene points (the residual is the 0.1 px
rounding of the published guide coordinates), and equals the ground-truth
look-at frame to **4.2e-5**. `tpm_verify_in_maya.py` prints these numbers
inside a live Maya session; expect `< 2 px` PASS with the fixes and
thousands-of-pixels FAIL without them.

**Practice session with the plate (post-fix expectations):**
Load `tpm_reference_room.png` → draw/trust the six guides above (or
`Load calibration`) → sensor *Full frame* → all three flips ON →
`SOLVE CAMERA` ⇒ ≈ **23.99 mm / HFOV 73.75° / residual ≈ 0.01°** → create a
Free camera → `APPLY` → look through: the wireframe room's edges sit exactly on
the photo's edges.

---

## 12. Quick reference cards

**Mouse / keys (canvas):** LMB draw · drag handle adjust · MMB pan · wheel zoom
· Esc cancel rubber · F fit · right-click cancel.

**Sensor presets:** Full frame 36 · Super 35 24.89 · Academy 22 · APS-C 22.3 ·
Micro 4/3 17.3 · Alexa 65 54.12 · Custom 1–100 mm.

**Input ranges:** focal 1–10000 mm · shift ±2 · near 0.001–1e7 · far ≤1e8 ·
colour: exposure ±4, brightness ±1, contrast 0–3, gamma 0.2–4, saturation 0–3,
opacity 0–1 · grid step 10–1000 px.

**Recommended order for a new shot:** open image → guides (X, Z first, Y if
verticals converge) → sensor → solve → sanity-check mm/residual → flips via
viewport test → create Free camera → apply (plane + look-through on) → verify
pin at corners/centre → translate camera / scale scene → blockout → save
scene; save JSON only to reuse the match inputs later.
