# Session handoff — AR drift test + cube fiducial

Everything from the session that isn't already in `field-build-plan.md`, `calibration.md`,
or `course-code.md`. Read alongside `CLAUDE.md`.

Repo: `github.com/theMentalShark/MotoGymkhanaHelper`

## Repo layout as pushed

```
CLAUDE.md  README.md  .gitignore
ar-cone-test.html          # WebXR ARCore drift test (root = short URL)
web/index.html             # the converter (done)
docs/                      # course-code, field-build-plan, calibration, this file
field/                     # empty — Python work starts here
```

GitHub Pages (Settings → Pages → main, `/root`) serves:
- `https://thementalshark.github.io/MotoGymkhanaHelper/ar-cone-test.html`
- `https://thementalshark.github.io/MotoGymkhanaHelper/web/`

**WebXR requires a secure context.** `file://` and `http://192.168.x.x` will not expose
`navigator.xr`. Only `https://` or `http://localhost` work — hence Pages, or
`python -m http.server 8000` + `adb reverse tcp:8000 tcp:8000`. Also needs Chrome on Android
plus **Google Play Services for AR** installed (separate from Chrome; usual cause of
"AR not supported").

## `ar-cone-test.html` — what it is

A standalone WebXR page (three.js r128 from cdnjs) that measures **pure ARCore accuracy**
with no lighthouse and no laser. The 2022 MGymfun OLC Round 1 course is hardcoded:
13.5 × 17 m, origin = SW corner, +X east, +Y north.

Cones (metres): `SW(0,0) G1(6,0) G2(7.5,0) SE(13.5,0) G3(6,3) G4(7.5,3) S1(6,4) S2(6,8)
A1(10,8) S3(6,12) NW(0,17) NE(13.5,17)` — verified against the layout sheet's 6/1.5/6
bottom split and 4/4/4/5 vertical spacing.

**Flow:** hit-test the ground → tap to anchor the SW corner → rotate the whole course with a
slider to fit the lot → lock → walk to each ghost cone, drop a real cone, tap PLACED →
switch phase → walk back, aim the reticle at each real cone base, tap CHECK.

**Why CHECK uses the reticle hit-point** rather than the phone's standing position: it
measures where the *cone* is, not where your body is.

**Outputs:** per-cone drift (cm), ΔX/ΔY, distance walked at time of check; summary gives mean
drift, max drift, total walked, and **max drift as % of path length** — the metric that
matters, since VIO drift scales with path travelled, not course size. CSV export.
Course is ~65 m to lay, ~130 m round trip.

**Expected:** literature suggests 1–2 % of path → roughly 1.5–2.5 m. If the measured number
is far lower (e.g. 15 cm), that changes ARCore's role. Validate at least one cone with a tape
measure — the app cannot detect a systematic scale error on its own. Run once in flat overcast
and once in hard sun/shadow; lighting is a major variable on open asphalt.

**Bug fixed this session:** the WebGL canvas was `position:fixed; inset:0` appended after the
launch UI, so it sat on top and swallowed all taps (button visible, unclickable). Fixed with
explicit stacking — canvas `z-index:0`, launch `10`, AR overlay `15`, results `20` — plus the
START AR button pinned as a fixed footer and the launch screen made touch-scrollable.

## Why ARCore is not the primary design

- **Drift scales with path length** (~1–2 % handheld VIO), so error *accumulates* as you place
  more cones — the opposite of the lighthouse's single-origin polar property, where cone 30 is
  as accurate as cone 2.
- **Scale error** (1–3 %) is systematic, not walkable-off: 2 % at 20 m = 40 cm.
- **Worst-case environment**: low-texture asphalt, repetitive parking-lot geometry inviting
  mis-relocalisation, moving shadows, thermal throttling, screen glare.
- **The drift is invisible** — nothing in the app tells you the frame has slipped.
- Geospatial API / VPS is ~1 m at best and poorly mapped on open lots.

**Verdict:** not a replacement. Viable as a *coarse guidance layer* (10–50 cm is plenty for
"walk over there") on top of instrument truth, with periodic re-anchoring on a known reference
cone to stop drift compounding. The AR ghost-cone UX prototyped here is reusable for Phase 5
regardless of what supplies the coordinates.

## Cube fiducial variant (promising, not yet built)

Idea: an **AprilTag cube** (not QR — AprilTag/ArUco are built for 6-DoF pose, better corner
localisation and oblique robustness) placed at the **course centre**; the walker's phone
resects off it continuously.

**Why a cube:** a single planar marker suffers pose ambiguity (flip); seeing two or three
faces plus the shared edge removes it, and per-face IDs make orientation unambiguous from any
approach. Gives 360° coverage as you walk around.

**Error model** (tag size *s*, distance *d*, corner σ ≈ 0.3 px, 7a main lens f ≈ 3200 px):
- lateral ≈ `d·σ/f` — ~1 mm at 10 m, ~2 mm at 20 m. Excellent.
- depth ≈ `d²·σ/(f·s)` — **quadratic**. 300 mm cube: ~0.8 cm @5 m, ~3 cm @10 m, ~12 cm @20 m.

Same signature as the tripod-camera design (angle superb, range poor) → same remedy: **Bosch
supplies range, camera supplies angle.**

**Centre placement is the key move.** Max distance drops ~22 m → ~10.9 m, and because depth
error is quadratic, that quarters it: ~15 cm → **~3.7 cm**, inside the 5 cm bar. A 400 mm cube
gives ~2.8 cm.

**If the cube is visible from everywhere, ARCore is unnecessary** — every fix is an independent
resection, so error never accumulates. Simpler than the ARCore-hybrid (no session management,
tracking loss, or fusion).

**Do not use an inflatable.** Three independent failures: pressure/thermal change alters the
edge length (which *is* the scale reference), inflated faces bow so tags stretch (~2 % → 20 cm
depth error at 10 m, systematic), and low mass means wind moves the datum silently.
Build instead: **40–50 cm coroplast/foam-core cube**, flat-packable with cut corner brackets,
weighted or staked, matte finish (gloss + low sun = blown-out detections), edge length measured
with calipers and recorded.

**Practical:** use tag *bundles* per face (more constraints, glare redundancy); stand still for
each fix (motion blur degrades σ); mind the height offset (phone chest-height, cone at feet —
want horizontal not slope distance); back-sight a known reference cone periodically to detect
if the cube moved; place centre cones last since the cube occupies that space.

**Best synthesis so far:** cube at the lighthouse origin → the walker recovers their own
position directly in the polar frame the `LH1` code is already written in. No frame alignment,
no radio corrections, one person instead of two.

## Next steps

1. **Run the AR drift test** on the real lot; record the numbers (CSV + tape-measure check).
   That single measurement decides ARCore's role.
2. **Phase 1 calibration** (`docs/calibration.md`) — required for the cube variant *and* the
   tripod-camera variant, so it is not blocked by the decision above. `field/calibrate.py`.
3. If cube looks good, write `docs/cube-tracking.md` and fold it into the build plan as an
   alternative to the tripod-camera station.
