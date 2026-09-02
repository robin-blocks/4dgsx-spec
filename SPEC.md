# 4DGSX — the 4D Gaussian Splatting eXchange bundle (v0.2)

A match (or any simulated 4D scene) exported as one portable, self-contained
directory that any platform — web, Quest/WebXR, native — can play back with
a free camera. 4DGSX covers the **3D content** (geometry + gaussian splat
sets + a rigid transform track), the **UI** (semantic data + a layered,
user-toggleable presentation system with publisher-designed panels and
clock-mapped media, e.g. the broadcast feed beside the volumetric render),
and the **audio** (one or more user-toggleable sources — broadcast mix,
crowd, commentary, spatial mics — each with an explicit clock map). RFL is the reference producer
(`gauntlet/volumetric.py`) and `index.html` in each bundle is the reference
player; the layout is deliberately sport-agnostic.

Format ids carried in the files: `4dgsx` (scene manifest), `4dgsx-hud`
(data track), `4dgsx-ui` (presentation track). Version stamps are
semver-ish: additive fields bump minor, layout changes bump major.
**Must-ignore rule everywhere:** consumers skip unknown fields, unknown
event types, unknown UI component/anchor/content types. That rule — not
any particular field — is what future-proofs the format.

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
4. **One clock.** Track frames, hud events, UI bindings, the audio map and
   media panels are all keyed to match time in seconds. Presentation clocks
   (count-down, halves) are declared in `hud.json` and derived by the player.
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

- **Minor bump = additive.** New optional fields; older players keep working
  by ignoring what they do not know (the must-ignore rule).
- **Major bump = a break.** An existing field changes meaning or disappears.
- **A player never refuses a bundle over a minor it does not recognise.** The
  version states what MAY be present; it is not a gate. Nothing in the
  reference player reads it — it is for humans, validators and logs.
- Bundle ids are immutable, so a published bundle keeps the version it was
  stamped with. A bump applies to new bundles, never to a re-cut.

The scene manifest is at **0.3** (`draws[].tex`). `hud.json` is at 0.2,
`ui.json` at 0.2.

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
  "format": "4dgsx", "version": "0.3",   // 0.2 bundles stay valid
  "meta": { "hz": 25.0, "nframes": 2251, "camera": {...},
            "grass": {...} },                     // the turf, see below
  "hud": "hud.json",
  "ui": "ui.json",
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
  four RFL robots share one set of link meshes).

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
  frame. `score` is a step track (value at t = last step ≤ t).
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
  replay windows are skipped) and the same drift-corrected seek behaviour.
  `mute: true` is the norm when the bundle ships audio sources — the
  stems are the sound, the media is picture. The broadcast feed beside
  the free-camera render is the canonical use; a later version points
  `src` at a live stream URL and nothing else changes.

  **The broadcast window.** The media file SHOULD be the broadcast
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
seconds at each goal; `map` encodes exactly that. Reference behaviour:
skip the inserted windows. A richer player may dwell at a jump and let the
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
