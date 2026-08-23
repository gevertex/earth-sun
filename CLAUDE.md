# CLAUDE.md — earth-sun

Project context for any agent working in this repo. Run `git status` and
`git log` first. Trust the live git state over the notes below.

## Project

A single-page web app that renders a 3D Earth globe in real time. The app
marks the subsolar point (where the Sun is directly overhead). The Sun sits
in its true direction, so the day/night terminator is visible. The app also
draws the path the subsolar point traces over one year at the current time
of day: the past in amber, the future in blue, meeting at the current
marker. The app also shows real satellites in their real orbits, propagated
live from TLE data with the SGP4 model; hovering a satellite shows its name.

- Live site: https://gevertex.github.io/earth-sun/
- Repo: https://github.com/gevertex/earth-sun.git
- Hosting: GitHub Pages serves the `main` branch as is. Push to publish.
- Deployed state: verified 2026-08-22, the live site serves commit
  `486caf3` (the satellite feature is present).

## Files

- `index.html` — the entire app: markup, CSS, and JS in one file (~510 lines)
- `README.md` — live site link, deploy steps, local run steps
- `CLAUDE.md` — this file

## Run locally

```sh
python3 -m http.server 8080
# then visit http://127.0.0.1:8080
```

Three.js, `satellite.js`, and the Earth textures load from the unpkg.com CDN.
Fresh satellite TLEs load from celestrak.org. The page needs internet access.
Offline, the globe and sun still work; satellites use embedded fallback TLEs.

## Deploy

```sh
git add -A
git commit -m "describe the change"
git push
```

GitHub Pages rebuilds the site within a minute or two after the push.

## Implementation notes (`index.html`)

### Structure

- Three.js 0.160.0 via an import map (unpkg CDN). Uses `OrbitControls` and
  `Line2`/`LineGeometry`/`LineMaterial` from `three/addons/`.
- `satellite.js` 5.0.0 (unpkg CDN) for SGP4 orbit propagation, loaded with a
  dynamic `import()` so a CDN failure only disables satellites, not the globe.
- Globe: `SphereGeometry(10, 256, 256)` (high segment count keeps the
  terrain displacement smooth), `MeshPhongMaterial`, radius `R = 10`.
- Camera: `PerspectiveCamera(45)`, starts at `(0, 8, 30)`.
- `OrbitControls`: damping 0.05, distance clamped to 12–120.
- Starfield: 2500 random points on a shell at radius 500–800.

### Solar math (low-precision, Meeus/NOAA style)

- `subSolarPoint(d)` returns `{ latDeg, lonDeg, eotMin }`.
  - Declination: mean longitude `L0`, mean anomaly `M`, ecliptic longitude
    `lambda = L0 + 1.91460 sin(M) + 0.02 sin(2M)`, obliquity
    `eps = 23.439 - 0.0000004 T` (deg), then
    `decl = asin(sin(eps) sin(lambda))`.
  - Subsolar longitude: the Sun is overhead where apparent local time is
    12:00. `lon = 15 (12 - h) - 15 eot/60`, normalized to `[-180, 180)`.
  - Equation of time: low-precision fit
    `eot = 9.87 sin(2B) - 7.53 cos(B) - 1.5 sin(B)`, minutes.

### Coordinate mapping

- `latLonToVec3(latDeg, lonDeg, radius)` matches the default
  `THREE.SphereGeometry` UV mapping for an equirectangular texture
  (`u = 0` at 180°W).
- Texture coordinates for a lat/lon point:
  `u = (lon + 180) / 360`, `v = (lat + 90) / 180`.

### Textures (all from unpkg `three-globe`)

- `earth-blue-marble.jpg` → `map` (day side)
- `earth-night.jpg` → `emissiveMap` (city lights), `emissiveIntensity 2.5`
- `earth-topology.png` → `displacementMap` (scale 0.6) + `bumpMap`
  (scale 0.05)
- Offline fallbacks: plain blue globe (`0x1d3f8f`), no city lights, smooth
  globe. Every texture load has an empty error callback.

### Night-side city lights (shader patch)

- `globeMat.onBeforeCompile` injects a `uSunDir` uniform (Sun direction in
  view space) into the fragment shader.
- The patch multiplies `totalEmissiveRadiance` by
  `nightFactor = smoothstep(0.12, -0.12, dot(normalize(vNormal), uSunDir))`.
- `tick()` updates `uSunDir.value` every frame from the current subsolar
  direction transformed into view space.

### Terrain sampling

- The topology texture is drawn to a canvas; `hData` holds the pixel data.
- `surfaceRadiusAt(u, v)` returns `R + (height/255) * 0.6`.
- Used to float the subsolar marker above the terrain and to place the sun
  path on the terrain surface.

### Sun, marker, and sun path

- Sun: sphere (radius 1.6, `0xffe28a`) at distance 70 in the subsolar
  direction.
- `DirectionalLight` (`0xfff2d8`, intensity 4.5) at distance 50 creates the
  terminator.
- `AmbientLight` (`0x8899bb`, intensity 0.05): kept very low so the
  terminator stays a crisp edge.
- Subsolar marker: sphere (radius 0.35, `0xffd24a`) at surface radius + 0.12,
  plus a line from the globe center.
- Sun path: 365 days, one sample per day at the current time of day.
  `past` (amber `0xffb347`) covers -182..0 days, `future` (blue `0x6fd3ff`)
  covers 0..+182 days. `Line2` with linewidth 2.5. Material resolution is
  set on resize. The path rebuilds once per second (`lastPathSecond` check).

### Satellites (real orbits via SGP4)

- `SATELLITES` holds 12 real satellites (name, NORAD catalog number, and a
  fallback TLE with a 2026-08 epoch). The set spans LEO, MEO, and GEO with
  varied inclinations: ISS, CSS (Tianhe), Hubble, NOAA 19, Sentinel-2A,
  Jason-3, Terra, Aqua, Starlink-1008 (LEO); GPS BIIR-5, Galileo GSAT0101
  (MEO); GOES-16 (GEO).
- `refreshTles()` fetches fresh TLEs from Celestrak
  (`gp.php?CATNR=<id>&FORMAT=tle`) on load and every 5 minutes while the
  app runs (`TLE_REFRESH_MS`), and rebuilds if any change. A
  `visibilitychange` handler re-fetches when the tab returns to view with
  stale data. It falls back to the embedded TLEs on failure.
- Each satellite: `sat.twoline2satrec(line1, line2)` builds a `satrec`;
  `sat.propagate(satrec, date)` gives the ECI position (km) each frame.
- Orbit regime by altitude: LEO < 2000 km (`0x6fd3ff`), MEO 2000–35000 km
  (`0x9dff6f`), GEO above (`0xff6fb3`).
- `eciToLocal(p)` maps an ECI vector to group-local scene coords:
  `(x, z, -y) * (R / 6371)`. A `satWorld` group holds every orbit line and
  dot; `tick()` sets `satWorld.rotation.y = -sat.gstime(now)` (GMST) each
  frame, which rotates the inertial points into the Earth-fixed view. The
  dot and its orbit line stay aligned because both live in the group.
- Orbit path: one full period (`2π / satrec.no`, `satrec.no` in rad/min),
  180 samples propagated in the inertial frame, drawn as a closed `Line2`
  ellipse (linewidth 1.4, opacity 0.45).
- Each satellite has a visible dot (`SphereGeometry(0.3)`) and a larger
  invisible hover target (`SphereGeometry(1.1)`, `MeshBasicMaterial` with
  `visible: false`; the raycaster still hits it).
- Hover: a `pointermove` handler raycasts against the hit targets. On a hit
  it shows `#sat-tip` (a fixed tooltip at the cursor) with the name and
  orbit type, scales the dot 2x, and brightens its orbit line.

### Readout (top-left panel)

UTC time, subsolar latitude, subsolar longitude, solar declination, the
equation of time, a legend for the sun path and the satellite orbit
types (LEO/MEO/GEO), and the TLE data status (last update time,
"updating…", or "embedded fallback"). Updated every frame in `tick()`.

## Recent changes (as of 2026-08-22)

- `a838c56` Add real satellites with live SGP4 orbits and hover names
- `1e2735b` Draw the annual subsolar path at the current time of day
- `0996d31` Document the GitHub Pages deployment in the README
- `7561a38` Darken the night side so the terminator is visible

Verify this list with `git log` before relying on it.

## User preferences (follow in every session)

- Verify web app changes in the browser before declaring the work complete.
  Exercise the changed feature end to end: click, type, submit, navigate.
  A single render screenshot is insufficient. Check every page and route
  that shares the state you touched. Hunt for regressions in surrounding
  flows. Check empty, error, and flag-variant states. Check both desktop
  and mobile viewports when layout or styling changed. If browser tools are
  unavailable, use the closest substitute (tests, curl against the dev
  server) and state what you could not verify.
- Write all text to the user in ASD-STE100 Simplified Technical English:
  one word for one meaning, active voice, short sentences (max 20 words in
  an instruction, 25 in a description), present tense, no metaphors or
  idioms, keep articles, noun groups of at most three words, bullet lists
  for steps and conditions.
- Never use the negative-positive antithesis ("not X, but Y"). Write the
  true statement in one sentence. Delete any clause that names a rejected
  idea or a contrast.
