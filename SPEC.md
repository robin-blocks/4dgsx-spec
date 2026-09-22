# 4DGSX — the 4D Gaussian Splatting eXchange bundle

_Open specification, licensed [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
Canonical copy: [4dgsx.com/spec](https://4dgsx.com/spec). Source, the reference
player and a sample bundle: [github.com/robin-blocks/4dgsx-spec](https://github.com/robin-blocks/4dgsx-spec)._

A match (or any simulated 4D scene) exported as one portable, self-contained
directory that any platform — web, Quest/WebXR, native — can play back with
a free camera. 4DGSX covers the **3D content** (geometry + gaussian splat
sets + a rigid transform track), the **UI** (semantic data + a layered,
user-toggleable presentation system with publisher-designed panels and
clock-mapped media, e.g. the broadcast feed beside the volumetric render),
an optional **programme** (video-independent rolls, inserted replays and scheduling),
and the **audio** (one or more user-toggleable sources — broadcast mix,
crowd, commentary, spatial mics — each with an explicit clock map). RFL is the reference producer
(`gauntlet/volumetric.py`) and `index.html` in each bundle is the reference
player; the layout is deliberately sport-agnostic.

Format ids carried in the files: `4dgsx` (scene manifest), `4dgsx-hud`
(data track), `4dgsx-ui` (presentation track). Version stamps are
semver-ish: additive fields bump minor, layout changes bump major.
**Must-ignore rule everywhere:** consumers skip unknown fields, unknown
event types, unknown UI component/anchor/content types. That rule — not
any particular field — is what future-proofs the format. **Exception: an explicit
`scene.program.version` is a playback contract. Programme-aware readers MUST
refuse unsupported or malformed explicit programme versions, not silently use
legacy timing.** This cannot change already-deployed legacy readers; see the
reader-first release requirement below.

## Design rules (the part other projects should copy)

1. **Geometry once, transforms per frame.** Rigid bodies get one static
   geometry/splat set each, in the body's local frame; playback is one 4×4
   per body per frame. Never re-ship or re-train per-frame surfaces for
   rigid content.
2. **Text and UI are data, not pixels.** Names, scores, clocks, chatter
   ship as a semantic track (`hud.json`); how they LOOK ships separately
   (`ui.json` + optional HTML panels) and the user can toggle every layer.
   Burned-in overlays are wrong under a free camera; 3D text meshes freeze
   typography; text in splats is both plus blurry.
3. **Audio is a sidecar of independent sources, each with an explicit
   clock map.** Splat formats have no audio container, and a broadcast
   timeline legitimately diverges from sim time (inserted replays). Ship
   each track (mix, crowd, commentary, mics) as its own user-toggleable
   source with a piecewise-linear `[match_t, audio_t]` map; a source may
   anchor to a body for spatial playback (v0.2 — see the audio section).
4. **One authoritative transport, explicit clock domains.** Track frames, HUD
   events and UI bindings use recorded scene time. A versioned `scene.program`
   maps extended scene time to programme seconds, including rolls and inserted
   replays. Neither a video element nor a UI panel owns the transport. HUD
   presentation clocks (count-down, halves) are derived from scene time.
5. **The platform ships surfaces; publishers ship panels.** Dock slots,
   layer toggles, the clock/data feed and media playback are format
   concerns; what a stats or line-up panel SHOWS and how it LOOKS belongs
   to the bundle's producer, as sandboxed HTML. The spec standardises a
   panel data shape only after several sports converge on one.

## Versioning

Each track carries its own `format` + `version` and they move
**independently**: `scene.json` (`4dgsx`), `hud.json` (`4dgsx-hud`),
`ui.json` (`4dgsx-ui`). A ui 0.2 bundle whose scene manifest gained nothing
still stamps `"version": "0.2"` in `scene.json`.

- **Minor bump = additive.** New optional fields; older players can parse
  bundles by ignoring what they do not know (the must-ignore rule). Parsing
  compatibility does not imply programme timing compatibility: programme/1
  exports additionally require the reader-first release check below.
- **Major bump = a break.** An existing field changes meaning or disappears.
- **A player never refuses a bundle over a minor it does not recognise.** The
  version states what MAY be present; it is not a gate. Nothing in the
  reference player gates on it — it is for humans, validators and logs. This
  is distinct from the explicit `program.version` sub-contract below.
- Bundle ids are immutable, so a published bundle keeps the version it was
  stamped with. A bump applies to new bundles, never to a re-cut.

The scene manifest is at **0.5** (optional versioned `program`, explicit
`ui: null`). `hud.json` is at 0.2, `ui.json` at 0.2. Existing 0.1–0.4 bundles
retain legacy playback unless they explicitly opt into `program.version: "1"`.

## Bundle layout

```
web/
  scene.json      manifest: version, buffers, prims, draws, bodies, camera,
                  pointers to hud/ui, audio + time map
  geometry.bin    mesh vertex + index data
  track.bin       per-frame rigid transforms (the "4D" in 4DGSX)
  hud.json        DATA: teams, players, anchors, clock, score, events, sfx
  ui.json         PRESENTATION: layers + components (see UI system)
  ui/*.html       designable sandboxed panels referenced by ui.json
  media/*         clock-mapped media for `media` components (broadcast.mp4)
  badges/*        club badge assets referenced by hud.json teams (hud 0.2)
  textures/*      surface tiles referenced by draws[].tex (turf 0.3)
  audio.m4a       v0.1 premix (optional); v0.2 stems live under audio/*
  splats/*.ply    per-body 3DGS gaussian sets (standard 3DGS PLY fields)
  points.bin      lightweight splat preview (optional viewer convenience)
  index.html      reference player (no dependencies, WebGL2)
```

## scene.json

```jsonc
{
  "format": "4dgsx", "version": "0.5",   // 0.1–0.4 bundles stay valid
  "meta": { "hz": 25.0, "nframes": 2251, "camera": {...},
            "grass": {...},                      // the turf, see below
            "platform": "microduck", "scale": 4.0 },  // provenance (0.4), below
  "hud": "hud.json",
  "ui": "ui.json",  // optional; null explicitly means no bundle presentation
  "program": {...},  // optional programme/1 contract, below
  "audio": {"file": "audio.m4a", "map": [[0,0], [25.8,25.8], [25.8,30.1], ...],
            "sources": [...]},
           // or null. map: [match_t, audio_t] breakpoints, slope 1 between;
           // a duplicated match_t is inserted media (a goal replay).
           // file+map = v0.1 legacy premix; sources[] = v0.2, audio section.
  "bodies": ["corner_0", ..., "ball", "r0_pelvis", ...],   // track order
  "prims":  [{"vo","vc","io","ic"}, ...],   // ranges into geometry.bin
  "draws":  [{"p": primIdx, "b": bodyIdx, "rgba": [..], "checker": 0|1,
              "tex": {...}}, ...],           // tex: turf 0.3, see below
            // checker: 1 = this draw IS the turf (see meta.grass), not
            // "paint a checker on it". The name is 0.1 legacy.
  "points": [{"b": bodyIdx, "ofs": pointOfs, "cnt": n}, ...],
  "buffers": {"vertexCount","vertexBytes","indexCount","indexBytes"},
  "times": [t_first, t_last]
}
```

- `draws[].b` indexes `bodies` **plus one** (0 = world/static, baked in
  world frame). A draw's vertices are already in that body's local frame.
- Many draws may reference one prim — instancing falls out for free (the
  RFL robots share one set of link meshes).

### `program` — video-independent programme (scene 0.5)

`program` standardises the existing RFL shape (`t`, `duration_s`, `map`,
`segments`) by adding **`"version": "1"`**. An absent, null, or **unversioned**
`program` is LEGACY, including old producer-private RFL programmes: consumers
MUST NOT retroactively reinterpret it as this contract. Once `version` is
present, anything other than the string `"1"` (including `1`, null and unknown
versions), or malformed version-1 data, MUST fail closed in programme-aware
readers and publishers. Unknown non-contract fields are ignored.

```json
{
  "version": "1",
  "t": [-2, 12],
  "duration_s": 17,
  "map": [[-2, 0], [0, 2], [4, 6], [4, 9], [10, 15], [12, 17]],
  "segments": [
    {"id": "pre", "kind": "pre-roll", "t": [-2, 0], "bodies": "hold", "clock": "kickoff"},
    {"id": "match", "kind": "live", "t": [0, 10], "bodies": "live", "clock": "match"},
    {"id": "post", "kind": "post-roll", "t": [10, 12], "bodies": "hidden", "clock": "fulltime"}
  ]
}
```

For this example `scene.times` is `[0,10]`; no MP4, audio, UI, or preview points
file is needed to express the full 17-second programme.

**Four distinct notions of time:**

- **Recorded scene time** (`scene.times`, seconds) addresses the transform track
  and semantic events. It need not begin at zero. Rendering clamps to those
  bounds; the manifest's recorded range is not the programme's duration.
- **Extended scene time** (`program.t`, seconds) includes pre/post-roll outside
  that recording. It is the first coordinate of `program.map`. It is never an
  excuse to index outside the track.
- **Programme seconds** start at zero and end at `duration_s`; they are the
  second map coordinate and the authoritative playback transport. Duplicate
  scene-time knots insert positive-duration windows while the event clock holds.
- **Wall start** is supplied by hosting/scheduling, not baked into the bundle:
  `startEpochMs` / entry `startsAt` means programme start, **not kickoff**.
  Scheduled position is `(nowMs - startEpochMs) / 1000` clamped to the programme.

**Validation (normative):** all numeric values are finite JSON numbers (not
strings or booleans). `t` and `scene.times` are two-number ranges; `t` has
positive span and covers the ordered recorded range. `duration_s > 0`.
`map` contains at least two two-number knots, starts at `[t[0],0]` and ends at
`[t[1],duration_s]`. Programme coordinates strictly increase, scene coordinates
never decrease. Between distinct scene coordinates the differences in the two
coordinates MUST be equal (unit slope); equal scene coordinates are inserts.
Endpoint/alignment/unit-slope comparisons permit absolute error **1e-6 seconds**;
monotonicity and positive-duration checks are strict. No extrapolated duration,
implicit cuts, negative-rate intervals or video-duration inference is allowed.

`segments` is a nonempty, ordered partition of `t`, without gaps or overlaps;
each segment has positive scene-time span. Each carries `t`, `bodies`, and a
string `clock`. Optional `id` and `kind` are descriptive, not control flow.
`bodies` is exactly `live`, `hold` or `hidden`:

- `live`: render the mapped scene sample, or the inserted replay sample.
- `hold`: render the segment's opening scene frame, clamped to the recording.
- `hidden`: hide tracked bodies, not static geometry or the transport itself.

`clock` is a presentation hint (`kickoff`, `match`, `fulltime` are conventional);
unknown strings are ignored, never a new transport or a validation error.

An optional segment `program: [startSeconds,endSeconds]` states its programme
range. When omitted it is derived from its `t` endpoints. Internal boundaries
use the **first/before-insert** programme coordinate at that scene time (unit
slope between knots); the first start is zero and the final end is `duration_s`.
Thus an insert on an omitted boundary belongs to the segment **starting** there.
An explicit boundary MAY instead choose the **last/after-insert** coordinate at
that scene time, assigning the whole insert to the preceding segment (as RFL's
final replay does). An interior point of an insert is not a segment boundary.
Both adjacent segments MUST agree on the chosen boundary, and their ranges
MUST partition `[0,duration_s]`. Intervals are half-open; the exact final endpoint uses the last
segment. Arbitrary replay sample ranges are not part of version 1.

**Sampling and transport:** inside an insert `[b0,b1)`, event/HUD time `t` holds
at the duplicate scene coordinate. Only `renderT` sweeps the previous
`b1-b0` seconds, clamped to `scene.times`: `t - (b1-b0)*(1-progress)`. The exact
insert start has progress 0; the exact end exits replay. This matches the
reference `invMap`. Scores/events follow held `t`, never replay `renderT`.
`hold` overrides the rendered pose; `hidden` controls visibility independently.

VOD starts at kickoff (the forward map of recorded start), not programme zero;
it advances programme time, traverses inserts, and stops at the final sample
without looping. A scene-time seek clamps to the recording and resolves
**after all inserts** at its destination. Programme-time seeks can enter rolls
or inserts. Scheduled playback ignores local pause, rate and delta-time while
locked: it catches up from the wall clock, including late joins and background
tabs. Before start it is `upcoming`/paused; from zero through strictly less than
`duration_s` it is `live`/playing. At **elapsed >= duration_s**, it holds the
final sample, pauses, becomes `replay`, and unlocks transport permanently for
VOD seeking; a backwards wall-clock change cannot relock a completed schedule.
An inserted replay does not change scheduled state to `replay`: that state
means on-demand/finished availability, distinct from the replay-sample flag.

The renderer-independent reference API is `readProgram(scene): Program | null`,
`ProgrammeController(program,t0,t1,scheduled?)`, `sample()`,
`update(dtSeconds,nowMs,playing,speed=1)`, `seek(sceneTime)` and
`seekProgramme(seconds)`. Invalid VOD deltas (negative/nonfinite), rates
(nonpositive/nonfinite) and seeks (nonfinite) are rejected. Locked seeks return
the unchanged sample. Snapshots expose `programmeTime`, held/event `t`,
`renderT`, insert `replay`, `state`, `playing`, `ended`, `bodies`, and `preKick`
(seconds until kickoff while in pre-roll, otherwise null). The controller
initialises scheduled position from the wall clock immediately.

**Audio/video target rule (normative):** a source with a map **equal to the
programme map** consumes programme seconds directly, including inserts and
rolls. Equality means the same number/order of knots and each coordinate within
1e-6 seconds. A source with a different or absent map uses the ordinary forward
map of held event `t`; an absent map means identity, **never** an inferred
programme edit. Sources cut from the programme edit MUST carry that equal map,
even when video is absent or presentation is disabled. The shared reference
`sourceTime(program,snapshot,map?)` implements this rule for BOTH audio and
video. No UI/media component may secretly become the master clock.

**Reader-first release requirement:** a scene minor bump or adding
`program.version` cannot fix old external readers which ignore the field.
Hosting MUST upgrade/verify its intended readers, including pinned/cached SDKs,
embeds and any exported standalone player, before enabling versioned exports.
The SDK advertises programme capability; this is a release check, not an
assumption that every reader of scene 0.5 supports it. `bin/4dgsx-publish`
requires explicit **`--programme-reader-ready`** acknowledgement for a versioned
programme; `--dry-run` permits local validation without it and uploads nothing.
For validated version 1, entry duration comes ONLY from `program.duration_s`,
not HUD, track, UI, media metadata, or a hidden video. Malformed/unknown explicit
versions refuse publication. The legacy unversioned longer-programme guard
remains intact; adding an unversioned duration does not bypass it. This change
neither mutates old published bundles nor changes spoiler-safe URL/score gates.

### Presentation is independent of programme

`scene.ui: null` explicitly means **no bundle presentation**. Do not fetch
`ui.json`, bundle panels, or invent fallback publisher overlays in that case.
An absent `ui` property retains the legacy optional `ui.json` fetch/fallback;
a string selects that presentation file as before. `hud.json` remains semantic
data, usable for events and clocks even with no panels. A versioned programme
continues unchanged with `ui: null` and without video.

The player/SDK option `presentation: 'none' | 'all' | string[]` independently
selects no presentation, all available components, or the specified component IDs.
It does not disable audio, remove the programme, change schedule/transport,
or override `scene.ui: null` by inventing presentation. Explicit empty UI files
also mean no fallback overlays. Readers SHOULD bound optional UI discovery so
a stalled UI sidecar cannot prevent the scene mounting (the supplied readers
allow two seconds, then continue without it). Layer defaults and
user choices apply to the layers used by the permitted components. Attribution supplied by the host
outside bundle UI is unchanged and is not a publisher layer that this option
can suppress.

### `draws[].tex` — surface tiles (turf 0.3)

Any draw may carry an image tile, sampled in **world space**:

```jsonc
"tex": {
  "src": "textures/turf.png",   // bundle-relative; any format a browser decodes
  "scale_m":  [4.0, 4.0],       // world metres spanned by one tile, per axis
  "offset_m": [-2.0, -2.0]      // world coords of the tile's -x/-y corner
}
```

`uv = (world.xy - offset_m) / scale_m`, wrapped. Three consequences worth
stating, because they are the whole point:

- **No vertex UVs, and none needed.** Ground quads carry position and normal
  only. World-planar mapping means the pattern is anchored to the world, so
  **widening the ground quad does not move the pattern** — the failure that
  made the parametric fields necessary cannot occur.
- **Row 0 of the image file is the `+y` edge of the tile.** That is the GL
  convention (v = 0 at the image's *bottom* row) and it is stated here so
  publishers do not have to discover it by rendering.
- **The tile supplies the draw's rgb**; `rgba[3]` still applies as alpha. A
  tile outranks `meta.grass` on the same draw.

Tiles do not need power-of-two dimensions. Players are expected to mipmap and,
where available, sample anisotropically — a mown surface viewed down its own
length is the aliasing worst case, and it is the reason this is an image and
not a formula.

**This is the mechanism for ground appearance.** A publisher ships what their
surface looks like — turf, boards, hardwood, ice, clay, sponsor paint — and no
player change is required for a surface nobody has thought of yet.

### `meta.grass` — the mown turf

The playing surface is one flat draw with `checker: 1`; its pattern is
synthesised by the player from `meta.grass`, in **world metres**, measured
off the model at export time:

```jsonc
"grass": {
  "pattern": "stripes",        // "stripes" | "checker"
  "axis": "x",                 // alternation axis (stripes only)
  "period_m": 4.0,             // one light band + one dark band
  "phase_m": 0.0,              // a band boundary sits at this coordinate;
                               // the band running +axis from it is rgb2
  "blend_m": 0.11,             // linear seam across a boundary, total width
  "rgb1": [0.283, 0.545, 0.290],   // light band
  "rgb2": [0.156, 0.345, 0.176],   // dark band
  "repeat": 6.0, "mark": [..]  // LEGACY — frozen, DO NOT READ
}
```

`repeat` and `mark` are the 0.1 form and describe the exporter's *old* turf:
`repeat` was a MuJoCo `texrepeat` under `texuniform`, so it never meant
"across the pitch" — one tile spans `2 / repeat` metres — and `mark` was that
builtin checker's per-CELL border, nothing to do with pitch markings. RFL's
literal went stale on 2026-08-20 when the pitch changed under it, and
published bundle ids are immutable, so **every bundle still carries both,
still stale**: the manifest cannot tell an old pitch from a new one. A player
that reads them draws the wrong turf. They stay in the schema only because
removing them would make shipped bundles disagree with the spec.

With the world-unit fields absent, assume **4 m stripes along x, boundary on
0** — right for RFL from s2-m6 on, and merely coarse before it.

**Pitch markings are geometry**, never synthesised: the halfway line, circle
and boxes ship as flat draws like anything else.

### `meta.platform` / `meta.scale` — capture provenance (scene 0.4)

Both optional. `platform` names the rig the moving bodies came from, as a
free token the producer chooses (`"unitree_g1_12dof"`, `"microduck"`).
`scale` is the factor the exporter multiplied the native scene by to land
it in metres — `4.0` for a robot simulated at quarter scale and exported
up to the real pitch, `1.0` (or absent) when the simulation already was in
metres. Time is never scaled: one bundle second is one match second. They
exist so a catalogue, a dataset index or a log can say what a bundle holds
without opening `geometry.bin`, and so a season that fields two robots on
one pitch can be told apart by machines.

**Nothing in playback depends on them.** The track is in metres whatever
`scale` says — a player MUST NOT rescale by it — and body names, label
offsets and ball size all come from the bundle itself (`bodies`,
`hud.players[].anchor`, the geometry). A bundle with 1.0 m bodies and a
0.28 m ball plays in the same player as one with 1.3 m bodies and a 0.70 m
ball, on the same 14 × 9 m pitch, with no switch anywhere. That is the
rule these fields must not erode: a consumer that finds itself keying
behaviour on `platform` wants something that belongs in the bundle instead.

## geometry.bin

Little-endian, two consecutive sections:

1. `vertexCount × 24 B` — interleaved `float32 x,y,z, nx,ny,nz`
2. `indexCount × 4 B` — `uint32` triangle indices into section 1

## track.bin

`float32 [nframes][nbodies][7]` — `x y z` position then `w x y z` unit
quaternion (MuJoCo convention), world frame, `bodies` order, at `meta.hz`.
Interpolate: lerp position, nlerp shortest-path quaternion. 90 s of RFL
≈ 3.6 MB (≈ 40 kB/s); quantization is planned for a later version.

## hud.json — the data track

```jsonc
{
  "format": "4dgsx-hud", "version": "0.1",
  "clock": {"mode": "down", "duration_s": 600, "halves": 2,
            "half_breaks": [305.2]},
  "teams":   [{"id": "A", "name", "code", "color": [r,g,b,a],
               "badge": "badges/a.svg"}, ...],   // badge: hud 0.2, optional
  "players": [{"id": "r0", "team": "A", "number": 1, "name": "CR-7000",
               "anchor": {"body": "r0_pelvis", "offset": [0,0,0.62]}}, ...],
  "score":  [{"t": 0, "a": 0, "b": 0}, {"t": 25.8, "a": 1, "b": 0}, ...],
  "events": [{"t": 25.8, "type": "goal", "team": "A", "player": "r2",
              "replay_s": 4.3},
             {"t": 31.0, "type": "radio", "player": "r1",
              "text": "man on!", "dur_s": 3.5},
             {"t": 40.0, "type": "coach", "team": "B",
              "text": "Drop deep!", "dur_s": 5.0},
             {"t": 305.2, "type": "half", "n": 1},
             {"t": 88.0, "type": "drop"}],
  "sfx":    [{"t": 12.4, "kind": "kick", "mag": 0.8}, ...]
}
```

- `anchor` = a body from `scene.json bodies` + an offset in that body's
  frame. It is the contract for where a player IS: the root body is
  whatever the producer names there, and the offset already carries that
  rig's head height. Consumers MUST NOT key on body-name patterns
  (`r0_pelvis` is one rig's root, not a convention) or assume a body
  count, a player count per team or a ball size — all of it is the
  bundle's own, and a rig one metre tall plays exactly like one that is
  taller. `score` is a step track (value at t = last step ≤ t).
- `teams[].badge` (hud 0.2, optional) — bundle-relative path to a square
  club badge: SVG preferred, or a raster ≥256 px, transparent background,
  legible at 16 px. Players SHOULD use it wherever they mark a team
  (scorebug, score chips); catalogues/registries SHOULD copy it to their
  own public listing surfaces (an unaired bundle's path may be withheld,
  the badge is not a spoiler). Absent → kit-colour marks, as before.
- **Events are an open envelope** `{t, type, ...}` — a sport adds types
  freely (`card`, `pit_stop`, `checkpoint`); the five above stay stable
  where they apply. `sfx` is the impulse tape, usable for platform-side
  spatialized audio.
- hud.json is the TRUTH. A player with no ui.json support must be able to
  build complete native UI from this file alone.

## ui.json — the layered presentation system

The user-facing requirements this answers: labels tracked to bodies in 3D;
floating panels; every piece of UI toggleable per user; designable by
non-engine people; extensible to rich animated content and media without
replacing anything.

```jsonc
{
  "format": "4dgsx-ui", "version": "0.2",
  "layers": [
    {"id": "players.names",  "title": "Player names",        "default": true},
    {"id": "players.radio",  "title": "Radio chatter",       "default": true},
    {"id": "match.scorebug", "title": "Score bug",           "default": true},
    {"id": "match.panel3d",  "title": "Stadium score panel", "default": false},
    {"id": "panels.video",   "title": "Broadcast feed", "icon": "video",
     "default": true},
    {"id": "panels.stats",   "title": "Game stats",     "icon": "stats",
     "default": true}
  ],
  "components": [
    {"id": "np_r0", "layer": "players.names",
     "anchor": {"type": "body", "body": "r0_pelvis",
                "offset": [0,0,0.62], "billboard": true},
     "content": {"type": "nameplate", "player": "r0"}},
    {"id": "scorebug", "layer": "match.scorebug",
     "anchor": {"type": "screen", "pos": [0.5, 0.03], "align": "top-center"},
     "content": {"type": "scoreboard", "style": "bug"}},
    {"id": "panel3d", "layer": "match.panel3d",
     "anchor": {"type": "world", "pos": [0, 0, 3.4], "billboard": "yaw",
                "size_m": [4.6, 1.55]},
     "content": {"type": "html", "src": "ui/score_panel.html",
                 "bind": ["teams", "score", "clock", "events"]}},
    {"id": "screen", "layer": "panels.video",                      // ui 0.2
     "anchor": {"type": "dock", "slot": "main"},
     "content": {"type": "media", "kind": "video",
                 "src": "media/broadcast.mp4", "mute": true,
                 "aspect": 1.778, "map": [[0,0], [21,21], [21,26]]}},
    {"id": "gamestats", "layer": "panels.stats",                   // ui 0.2
     "anchor": {"type": "dock", "slot": "left"},
     "content": {"type": "html", "src": "ui/stats.html", "aspect": 0.68}}
  ]
}
```

**Layers** are the unit of user control. Players render a toggle menu from
this list, apply `default`, and persist the user's choices. User prefs
always beat bundle defaults. `default` describes the publisher's intended
FULL presentation; how much of it a given surface can carry is the
player's call (the same split as dock geometry): a player MAY start
scene-space layers — those with body, world, or dock components — hidden
on flat screens, where stacked overlays can bury the content itself. Every
layer still appears in the toggle menu, and immersive sessions, which
place panels beside the content rather than over it, honour `default` as
written. Layers that own dock components SHOULD also
surface as a quick-access nav (the player's "pill bar"), ordered
left → main → right to mirror the docks; `icon` (ui 0.2, optional) names a
glyph token for it — `stats`, `lineup`, `video` are defined, unknown
tokens get a neutral glyph.

**Components** are `{id, layer, anchor, content}`:

- `anchor.type`:
  - `screen` — classic HUD; `pos` in normalized [0..1] viewport coords.
  - `world` — fixed world transform (`pos`, optional `billboard: "yaw" |
    true`, `size_m` physical size — in VR this is REAL size at REAL depth).
  - `body` — tracked to a `scene.json` body + offset in its frame;
    follows the transform track automatically (nameplates, bubbles).
  - `dock` (ui 0.2) — a SEMANTIC slot: `"slot": "main" | "left" |
    "right"` (unknown slots: skip the component). The player owns dock
    geometry per form factor — overlay panels on a flat page, stage quads
    around the arena in XR — which is what keeps one bundle portable
    across phone, desktop and headset. Bundles never encode XR metres.
- `content.type`, v0.1 built-ins: `nameplate`, `bubble`, `scoreboard`,
  `coach-strip`, `event-banner` — all data-bound to hud.json, styled
  natively by the player.
- `content.type: "html"` — **the extension escape hatch.** A sandboxed
  HTML/CSS/JS document from the bundle, rendered as a panel at the anchor
  (web: iframe/texture; native/VR: web-view texture on a quad). Anything
  the built-ins can't express becomes an html panel: animated scoreboards,
  sponsor boards, stats tickers, line-ups — without any format change.
  Designers edit one file. At a dock, `aspect` (w/h) hints panel
  proportions and pointer events are enabled (see the action channel).
- `content.type: "media"` (ui 0.2, was reserved) — clock-mapped media the
  player renders natively, because sandboxed HTML cannot drive an XR video
  texture: `{kind: "video" (others reserved), src, map?, mute?, aspect?}`.
  `src` is bundle-relative; `map` has exactly the audio-source semantics
  (match-time → media-time breakpoints, identity when absent, inserted
  replay windows are skipped for legacy scene-time-driven playback) and the
  same drift-corrected seek behaviour. With versioned `scene.program`, the
  programme source-target rule above takes precedence, including inserted dwell.
  `mute: true` is the norm when the bundle ships audio sources — the
  stems are the sound, the media is picture. The broadcast feed beside
  the free-camera render is the canonical use; a later version points
  `src` at a live stream URL and nothing else changes.

  **The broadcast window (legacy media-master playback).** Without a versioned
  `scene.program`, the following existing media behaviour remains unchanged.
  The media file SHOULD be the broadcast
  programme end to end — the stream's pre-roll, the match (baked replays
  included), the post-roll — so the bundle and the simulcast are the same
  timeline. The map already expresses this: media time before the first
  breakpoint is the intro (players present the scene held at the match
  start — a capture whose opening frames are the line-up standing still
  reads best), media past the last breakpoint is the outro (scene held at
  the final frame — extend the capture past full-time if the performers
  should exit before the hold). On-demand playback starts at the first
  breakpoint (kick-off) and the seek range stays the match window — the
  roll regions belong to the scheduled broadcast. A wall-clock scheduled
  match plays the WHOLE file in lockstep with the simulcast: its start is
  the STREAM's start instant, not kick-off, and pre-roll, dwelling replays
  and post-roll air exactly as broadcast. When stems and media coexist
  they MUST be cut from the same edit (equal maps, rolls included).
- Reserved for later versions (players must skip them today): `text`
  (bare template string) and namespaced custom types (`"x-rfl:heatmap"`).

**The html panel contract** (host ⇄ panel, JSON postMessage):

- host → panel: `{type:"4dgsx:init", hud}` once on load;
  `{type:"4dgsx:tick", t, clock, score, playing}` at ≥4 Hz;
  `{type:"4dgsx:event", event}` when a hud event fires.
- panel → host (ui 0.2): `{type:"4dgsx:frame", svg, w, h, buttons}` — a
  SELF-CONTAINED SVG snapshot of the panel plus its tappable regions
  (`buttons: [{action, ..., rect: [x,y,w,h]}]`, normalized, origin
  top-left). Immersive sessions have no DOM, so this is how an html panel
  exists in a headset: the iframe keeps running invisibly, posts a frame
  whenever its content changes, and the host rasterises it onto a quad
  and ray-tests the declared rects. Rules learned the hard way: system
  fonts and no external refs (the host must be able to rasterise it);
  pin the snapshot root's size in px (percentage heights do not resolve
  in rasterised foreignObject content); post from the tick/event handlers,
  not timers or rAF (hidden pages throttle both). Panels that never post
  frames are simply 2D-only — nothing breaks.
- panel → host actions: `{type:"4dgsx:action", action, ...}`.
  `toggle-layer` (`{action:"toggle-layer", layer}`) is ACTIVE as of
  ui 0.2 — a panel's ✕ hides its own layer; hosts validate the layer id
  against the bundle's declared layers. `seek` and `set-camera` remain
  reserved. Unknown actions: ignore.
- Sandbox rules: `allow-scripts` only — no network, no host DOM, no
  storage. A panel is a pure function of the messages it receives, which
  is what makes third-party designs safe to ship in bundles.

Screen/world panels remain display-only; dock panels get pointer events
in 2D and their `frame`-declared buttons in XR, both limited to the
action vocabulary above.

## audio — sources + the time map

`scene.audio` is `null` (silent bundle) or an object:

```jsonc
"audio": {
  // v0.1 legacy — the premixed broadcast track. Producers SHOULD keep
  // shipping it when a single premix exists; v0.1 players read only this.
  "file": "audio.m4a",
  "map":  [[0,0], [25.8,25.8], [25.8,30.1], ...],

  // v0.2 — independent, user-toggleable sources on the shared match clock
  "sources": [
    {"id": "crowd",      "title": "Crowd",        "file": "audio/crowd.m4a",
     "default": true, "gain": 1.0, "map": [...]},        // non-positional
    {"id": "pitch",      "title": "Pitch sounds", "file": "audio/pitch.m4a",
     "default": true, "gain": 1.0, "map": [...]},
    {"id": "commentary", "title": "Commentary",   "file": "audio/commentary.m4a",
     "default": true, "gain": 1.0, "map": [...]},
    {"id": "mic_r0", "title": "BLU 1 mic", "file": "audio/r0.m4a",
     "default": false,
     "anchor":  {"type": "body", "body": "r0_pelvis", "offset": [0, 0, 0.5]},
     "rolloff": {"ref": 2.0, "max": 45.0}}          // spatial, follows the body
  ]
}
```

- Each **source** is an independent audio file keyed to match time.
  Per-source `map` (optional; identity when absent) has exactly the v0.1
  semantics. `gain` (default 1) is the source's mix level. Unknown fields:
  must-ignore, as everywhere.
- **Toggling mirrors the UI layer rules:** players list sources
  (`title`, `default`) in the same menu as UI layers, persist the user's
  choices, and user prefs beat bundle defaults.
- **`anchor` makes a source spatial**, reusing the ui.json anchor
  vocabulary: `body` (offset in that body's frame — the source moves with
  the transform track automatically, so a mic follows its player for free)
  or `world` (`pos`, fixed). No anchor = non-positional. Spatialisation is
  a SHOULD: a player without positional audio plays anchored sources as
  plain stereo (their `gain` still applies).
- **Moving sources are bodies.** To animate a sound independently of any
  visible character, export a *virtual body* — an entry in `bodies` +
  `track.bin` that no draw or splat set references — and anchor to it.
  One clock, one track, one interpolator; audio never grows a second
  keyframe system.
- `rolloff` (optional): `{ref, max}` meters — reference distance and
  attenuation cutoff hints for the player's distance model (defaults
  1 / 60). Hints, not physics; players pick the curve.
- **Compatibility:** a v0.2 player reading a v0.1 bundle synthesizes one
  source from `file` + `map`. A bundle with only `sources` may set
  `file: null` — consumers must guard. RFL v0.2 bundles carry `crowd`,
  `pitch` and `commentary`, all non-positional, plus the premix as the
  legacy `file`.

**The time map** (per source): the broadcast timeline inserts `replay_s`
seconds at each goal; `map` encodes exactly that. Legacy scene-time-driven
reference behaviour: skip the inserted windows. With `scene.program.version: "1"`,
use the programme source-target rule: equal maps play inserts and rolls directly
from programme seconds, independently of whether a media component exists. A richer player may dwell at a jump and let the
replay audio play over its own replay presentation. For fully synthesized
spatial audio, use `hud.sfx` instead of (or on top of) the sources.

## splats/*.ply

One file per dynamic body, standard 3DGS PLY fields (`x y z nx ny nz
f_dc_0..2 opacity scale_0..2 rot_0..3`, binary little-endian), positions
in the body's local frame, driven by the same track. v0.1 sets are
surface-sampled isotropic gaussians; a trained-appearance upgrade replaces
these files and nothing else. Budget note: the whole RFL scene is ~120k
gaussians — inside the ~400k standalone-headset ceiling.

## points.bin (optional preview)

`pointCount × 20 B`: `float32 x,y,z`, `uint8 rgba`, `float32 radius`,
grouped per body by `scene.json points[]`. A cheap splat stand-in for
renderers without a gaussian rasterizer.

## Producing a bundle (RFL)

- League fixtures record + export automatically (`league.play_next` →
  `runs/league/s<N>/m<K>_<home>_<away>/web/`).
- Ad-hoc matches: `RFL_EXPORT_STATES=1 python -m gauntlet football ...`,
  then `python scripts/export_match_web.py <match_dir>`.
- Inputs consumed: `states.npz`, `scene_build.json`, `match.json`,
  `comms.jsonl`/`tactics.jsonl`, `*_tv.mp4` (audio source; from ui 0.2
  also stream-copied video-only into `media/broadcast.mp4`, sharing the
  audio map).
- Serve anywhere static: `python3 -m http.server -d <dir>/web`.
## Changelog

- **programme 0.5** (2026-09-22; scene manifest 0.4 → 0.5; HUD/UI tracks
  unchanged): optional explicit `program.version: "1"`, validated programme map,
  segments, duration, replay/roll sampling and scheduled/VOD transport independent
  of media/UI; explicit `ui: null` and independent host presentation selection.
  Unversioned programmes stay legacy. Reader-aware validation fails closed on
  malformed/unknown explicit programme versions; reader-first publishing is
  mandatory because the minor stamp cannot upgrade existing consumers.

- **provenance 0.4** (2026-09-08, additive; **scene manifest 0.3 → 0.4**,
  `hud`/`ui` untouched): optional `meta.platform` (the rig the bodies came
  from, a free token) and `meta.scale` (the factor the exporter applied to
  land the native scene in metres). Provenance only — playback reads
  neither, and body names, label offsets and ball size stay the bundle's
  own, restated under `hud.json`. Written for RFL season 4, which fields
  a second robot (a 1.0 m biped with a 0.28 m ball) on the same pitch from
  2026-10-02; the first bundles carrying it are that division's pre-season
  friendlies in late September 2026.
- **vocabulary** (2026-09-01, wording only — no version change): the entry
  mode `premiere` is renamed `scheduled`. "Premiere" is film language and
  reads as nonsense for live sport, which is what this format carries. The
  behaviour is unchanged — a wall-clock-locked start that becomes a replay —
  and `premiere` is still accepted on read, so bundles and registry entries
  already published stay valid. Producers should emit `scheduled`.

- **ui clarification** (2026-08-26, wording only — no version change):
  layer `default` is the publisher's intent for the full presentation;
  form-factor presentation is the player's call. A player MAY start
  scene-space layers (body/world/dock components) hidden on flat screens
  — at embed sizes the overlays of a real bundle covered the entire
  pitch — while immersive sessions honour `default` as written. Toggles
  and user-pref persistence are unchanged and still beat everything.
- **turf 0.3** (2026-08-21, additive; **scene manifest 0.2 → 0.3**, the
  first bump of that track since 0.2 — `hud`/`ui` are untouched):
  `draws[].tex` — an image tile on any
  draw, mapped in world space (`uv = (world.xy - offset_m) / scale_m`), no
  vertex UVs required. Supersedes the parametric `meta.grass` fields, which
  stay for bundles that predate it. The format ships assets everywhere else
  and shipped a *description* only here; that description drifted from the
  model it described, twice, and a `pattern` enum would have made every new
  kind of ground wait on a player release. An image needs neither.
- **turf 0.2** (2026-08-21, additive): `meta.grass` gains world-unit fields
  — `pattern`, `axis`, `period_m`, `phase_m`, `blend_m` — measured off the
  model at export time, and `rgb1`/`rgb2` become the light/dark band. `repeat`
  and `mark` are frozen legacy and must not be read; `draws[].checker` is
  documented as the turf FLAG it always was. Written after a player drew the
  pitch flat green next to a broadcast feed mown in 2 m stripes.
- **hud 0.2 + broadcast window** (2026-08-21, additive): optional
  `teams[].badge` (club badge asset, used by players and catalogues); the
  `media` broadcast-window semantics made explicit — pre/post-roll live in
  the media file outside the map's breakpoints, VOD starts at kick-off,
  a scheduled match plays the whole file 1:1 with the simulcast from the
  STREAM's start instant, stems must share the broadcast edit.
- **ui 0.2** (2026-08-20, additive — the `4dgsx-ui` track only; the scene
  manifest stays 0.2 and v0.1 players skip everything here): `dock`
  anchors (`main`/`left`/`right` semantic slots, geometry player-owned
  per form factor — the tabletop-broadcast layout); `media` content type
  (clock-mapped broadcast video beside the volumetric render, audio-map
  semantics; the one native dock type); layer `icon` tokens + the pill-bar
  convention; html panels at docks; panel→host `4dgsx:frame` snapshots
  (XR presence for publisher HTML) and the first activated action,
  `toggle-layer`. Principle codified as design rule 5: the platform ships
  surfaces, publishers ship panel content. Reference dock panels:
  `ui/stats.html` (publisher-embedded data) + `ui/lineup.html` (renders
  purely from the `init` hud) in the current example bundle.
- **0.2** (2026-08-19, additive — v0.1 bundles stay valid):
  `audio.sources[]` — multiple independent, user-toggleable audio tracks;
  optional spatial `anchor` (`body` | `world`, the ui.json vocabulary);
  the virtual-body pattern for independently moving sources; per-source
  `map` / `gain` / `rolloff`. Legacy `audio.file` + `audio.map` unchanged,
  still recommended as a premixed fallback. RFL emits `crowd` + `pitch` +
  `commentary` stems.
- **0.1** — initial: geometry/track/splats, hud/ui/panels, single
  broadcast mix + clock map.
