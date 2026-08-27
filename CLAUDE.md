# CLAUDE.md — earth-sun

Project context for any agent working in this repo. Run `git status` and
`git log` first. Trust the live git state over the notes below.

## Project

A single-page web app that renders a 3D Earth globe in real time. The app
marks the subsolar point (where the Sun is directly overhead). The Sun sits
in its true direction at true scale (1 AU away, 0.53° across), so the
day/night terminator is visible. The app also
draws the path the subsolar point traces over one year at the current time
of day: the past in amber, the future in blue, meeting at the current
marker. The app also shows real satellites in their real orbits, propagated
live from TLE data with the SGP4 model; tapping a satellite shows its name.
The app also paints a pin where the user is, resolved from the browser's
public IP via IP2GeoAPI (no location permission prompt). The app also draws
country borders and US state borders on the globe; tapping a region shows
its name (state in the US, country elsewhere). The app also paints live
radar precipitation over the globe from the RainViewer public API, behind a
"show weather" toggle. The app also shows the Moon at true size and
distance in its true sky position, with the correct phase, plus a marker
at the sublunar point (where the Moon is overhead). The app also shows the
planets, a time-speed control (1x to 1 day/s), a GEO ring, aurora ovals, a
lat/lon grid, and the eclipse umbra when the Moon's shadow hits Earth.

- Live site: https://earth.4runner.online/ (the GitHub Pages URL
  https://gevertex.github.io/earth-sun/ 301-redirects there via a CNAME
  file in the repo)
- Repo: https://github.com/gevertex/earth-sun.git
- Hosting: GitHub Pages serves the `main` branch as is. Push to publish.
- Deployed state: pushed 2026-08-24, the live site serves the true-scale
  Sun and the country/state borders (commit `91070f0`). The custom domain
  is fronted by Cloudflare and resolves; the live page was verified to
  serve the borders code.

## Files

- `index.html` — the entire app: markup, CSS, and JS in one file (~3000 lines)
- `favicon.svg` — the app logo, used as the browser favicon
- `apple-touch-icon.png` — 180 px PNG render of the logo, for iOS home screens
- `README.md` — live site link, deploy steps, local run steps
- `CLAUDE.md` — this file

## Run locally

```sh
python3 -m http.server 8080
# then visit http://127.0.0.1:8080
```

Three.js, `satellite.js`, and the Earth textures load from the unpkg.com CDN.
The Moon texture loads from the three.js GitHub repo (raw.githubusercontent
is CORS-open; the three npm package excludes texture files). Fresh satellite
TLEs load from the satvisor GitHub mirror of Celestrak data (Celestrak is
the fallback). The page needs internet access. Offline, the globe and sun
still work; named satellites use embedded fallback TLEs; the location pin
shows `unavailable`; the borders do not load; the moon is a plain gray
sphere.

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
- Camera: `PerspectiveCamera(45)`, starts at `(0, 8, 30)`, far plane
  1,000,000 (covers the starfield shell and the Sun at 1 AU).
- `OrbitControls`: damping 0.05, distance clamped to 12–120.
- Starfield: 2500 random points on a shell at radius 400,000–800,000,
  beyond the Sun so the stars sit behind every object in the scene.

### Scale

- `R = 10` scene units is one Earth radius (`EARTH_RADIUS_KM = 6371`,
  `KM_TO_SCENE = R / EARTH_RADIUS_KM`).
- The Sun is at true scale: `SUN_RADIUS_KM = 696340` → `SUN_RADIUS`
  (≈ 1093), `AU_KM = 149597870.7` → `SUN_DISTANCE` (≈ 234,811). The disk
  is 0.53° across, the true apparent size of the Sun from Earth.
- The Moon is at true scale: radius `1737.4 km` → ≈ 2.73 scene units,
  distance ≈ 394,000 km → ≈ 618 scene units. The disk is 0.52° across,
  the true apparent size of the Moon from Earth.

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

- Sun: sphere at true scale in the subsolar direction: radius
  `SUN_RADIUS` (≈ 1093, 109× the Earth radius), distance `SUN_DISTANCE`
  (≈ 234,811, 1 AU), color `0xffe28a`. The disk is 0.53° across, the true
  apparent size of the Sun from Earth.
- `DirectionalLight` (`0xfff2d8`, intensity 4.5) at the Sun's position
  creates the terminator (a directional light uses only the direction).
- `AmbientLight` (`0x8899bb`, intensity 0.05): kept very low so the
  terminator stays a crisp edge.
- Subsolar marker: sphere (radius 0.35, `0xffd24a`) at surface radius + 0.12,
  plus a line from the globe center.
- Sun path: 365 days, one sample per day at the current time of day.
  `past` (amber `0xffb347`) covers -182..0 days, `future` (blue `0x6fd3ff`)
  covers 0..+182 days. `Line2` with linewidth 2.5. Material resolution is
  set on resize. The path rebuilds once per second (`lastPathSecond` check).

### Moon (true size, distance, and sky position)

- `moonPos(date)` returns `{ lambda, beta, dist }` (ecliptic longitude deg,
  ecliptic latitude deg, Earth-Moon distance km) from Meeus A&A 2nd ed.
  §47.2 medium-precision lunar coordinates (truncated ELP2000/82): 60-term
  series `MOON_LR` (longitude, 1e-6 deg; distance, 1e-3 km) and `MOON_B`
  (latitude, 1e-6 deg), with the argument variables `Lp, D, M, Mp, F` and
  the Evection/Perigee precession corrections `A1, A2, A3`. Referred to the
  mean equinox of date, the same frame as the satellite ECI positions.
  Accuracy ≈ 0.2° in position, ≈ 4 km in distance (spot-checked against
  the JPL DE421 ephemeris; the gap is the medium-precision theory error).
- `gmstIAU(date)` is the IAU 1982 GMST in degrees; it matches
  `satellite.js` `sat.gstime` exactly.
- `updateMoon(now)` (called each frame from `tick()`, skipped while the
  layer is hidden): converts ecliptic (λ, β) to equatorial of date
  (RA, Dec) with obliquity `eps = 23.439 − 0.0000004 T`, then
  `lonGeo = RA − GMST` (normalized to `[-180, 180)`). The mesh sits at
  `latLonToVec3(dec, lonGeo, dist * KM_TO_SCENE)`; `moonMesh.lookAt(0,0,0)`
  gives tidal lock (the SphereGeometry texture center u=0.5 faces +Z, so
  the near side faces Earth). The phase comes from the existing
  `DirectionalLight` at the Sun; no extra lighting.
- `moonMesh`: `SphereGeometry(1737.4 * KM_TO_SCENE, 48, 32)` (≈ 2.73
  units), `MeshPhongMaterial`. Texture `moon_1024.jpg` loads from
  `raw.githubusercontent.com/mrdoob/three.js/r160/examples/textures/planets/`
  (the three npm package excludes texture files, so the unpkg URL 404s;
  GitHub raw is CORS-open like the TLE and border fetches). Empty error
  callback: offline the moon is a plain gray sphere.
- Sublunar marker: sphere (radius 0.28, `0xd8d4cc`) at
  `surfaceRadiusAt(u, v) + 0.12` under the Moon, plus a line from the
  globe center (the point where the Moon is directly overhead).
- `moonWorld` group holds the mesh and marker; the "show moon" toggle
  (`#moon-toggle`, checked by default) sets `moonWorld.visible`.
- The moon is not in the tap raycast targets, so it never blocks
  satellite or region taps.
- `tick()` feeds `simNow()` (the time-speed control), so the Moon follows
  the simulated clock when the time scale is above 1x.
- The eclipse umbra layer (a dark disk on the globe) is driven by
  `moonMesh.position` and only appears when the Moon's shadow cone hits
  Earth.

### Your location pin (IP2GeoAPI)

- `fetchMyLocation()` runs once on load. It fetches
  `https://api.ip2geoapi.com/ip/check?key=...` (`IP2GEO_API_KEY`). The
  service resolves the browser's public IP to a position. No location
  permission prompt: the lookup uses the IP, not the GPS. The fetch aborts
  after 12 s (`LOC_FETCH_TIMEOUT_MS`).
- On success, `myLoc` holds `{ latDeg, lonDeg, city, region, country }` and
  `myLocState` is `'ok'`. `tick()` calls `placePin()` every frame so the pin
  base follows the terrain height once the topology texture loads.
- The pin is a `THREE.Group` (red `0xff5450`): a thin cylinder (the needle,
  0.7 long, base at the surface) and a sphere head (radius 0.22) at the tip.
  `placePin` puts the base at `surfaceRadiusAt(u, v)` and orients the group
  with `quaternion.setFromUnitVectors((0,1,0), dir)`.
- Readout line "Your location" (`#myloc`): `locating…` while the fetch runs,
  then `City, Region, Country · lat, lon`, or `unavailable` on failure (bad
  key, network error, timeout, missing coordinates). On failure the pin stays
  hidden.
- The API key is embedded in the page: the static site has no backend. The
  free plan allows 100,000 lookups per month.

### Country and state borders (tap to name a region)

- `loadBorders()` fetches two GeoJSON files on load (GitHub raw, CORS-open,
  like the TLE fetches): world countries
  (`raw.githubusercontent.com/johan/world.geo.json/master/countries.geo.json`,
  180 features) and US states
  (`raw.githubusercontent.com/PublicaMundi/MappingAPI/master/data/geojson/us-states.json`,
  52 features). A failed fetch leaves the set empty; offline the globe works
  without borders.
- Rendering: `makeBorderLines()` builds one `THREE.LineSegments` per set
  (one draw call each). Country lines are white (`0xffffff`, opacity 0.4);
  state lines are light blue (`0x9fd8ff`, opacity 0.6). Each vertex sits at
  `surfaceRadiusAt(u, v) + 0.03`, so the lines follow the displaced terrain.
  `rebuildBorders()` runs both when the GeoJSON arrives and when the
  topology texture loads, so the async load order does not matter.
- Picking: `buildPickCanvas()` fills a 2048×1024 equirectangular canvas, one
  color per region, where the RGB encodes the region index (index 0 is open
  water). Countries fill first, US states on top, so a tap in the US resolves
  to the state. Rings that cross the antimeridian are unwrapped to
  continuous longitudes and drawn at three shifts (-360, 0, +360) with
  `fill('nonzero')`, so such polygons fill on both canvas edges.
  `pickRegionIndex(lat, lon)` reads the pixel.
- Tap: `pointerdown`/`pointerup` on the canvas. A tap is a pointer that moved
  less than 6 px in under 500 ms (a drag rotates the globe and must not pick).
  `onTap` first raycasts the satellite tap targets (named sats, then
  Starlink, each only when its toggle is on) and shows `#sat-tip` on a hit.
  Otherwise it raycasts the globe, converts the hit to lat/lon, and looks up
  the region. A hit sets `tapped = { point, name, kind }` and shows
  `#region-tip`; a tap on open water or empty space hides both labels.
- Label: `#region-tip` is a fixed tooltip (name over kind, e.g. "Texas /
  state"). `tick()` calls `updateRegionTip()` each frame, which projects the
  point to the screen and hides the label when the point faces away from the
  camera (`dir.dot(toCam) <= 0.02`).
- Toggle: `#borders-toggle` (a "show borders" checkbox, checked by default)
  sets `.visible` on both line sets.

### Satellites (real orbits via SGP4)

- `SATELLITES` holds 47 real satellites (name, NORAD catalog number, and a
  fallback TLE with a 2026-08 epoch). The set spans LEO (26), MEO (10), and
  GEO (11) with varied inclinations:
  - LEO: ISS (Zarya, Nauka), CSS (Tianhe, Wentian, Mengtian), Hubble,
    NOAA 19/20/21, MetOp-B/C, Sentinel-2A/3A/3B, Jason-3, Terra, Aqua,
    Landsat 8/9, ICESat-2, OCO-2, GCOM-W1, TerraSAR-X, Starlink-1008,
    OneWeb-0050/0250.
  - MEO: GPS BIIR-5, BIIF-1/4/7, Galileo GSAT0101/0201/0202, O3B FM9,
    Beidou-3 M1/M4.
  - GEO: GOES-16/17/18/19, Himawari-8/9, Meteosat-10/11/12,
    GEO-KOMPSAT-2A, Beidou-2 IGSO-7.
- `refreshTles()` fetches group TLE files from the satvisor GitHub mirror
  (`raw.githubusercontent.com/satvisorcom/satvisor-data/.../tle/{group}.tle`)
  on load and every 5 minutes (`TLE_REFRESH_MS`). Groups: stations, weather,
  resource, science, gnss, oneweb, other-comm, satnogs. It matches records
  to `SATELLITES` by catalog number and rebuilds if any line changed. Each
  URL aborts after 12 s (`TLE_FETCH_TIMEOUT_MS`); Celestrak
  (`gp.php?GROUP=<group>&FORMAT=tle`) is the fallback. A `visibilitychange`
  handler re-fetches when the tab returns to view with stale data. The
  embedded TLEs stay if every source fails. `tleRefreshing` is cleared in
  `finally` so the readout leaves `updating…` when the fetch ends.
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
- Each satellite has a visible dot (`SphereGeometry(0.1)`) and a larger
  invisible tap target (`SphereGeometry(0.4)`, `MeshBasicMaterial` with
  `visible: false`; the raycaster still hits it).
- Tap: `onTap` (the same pointer tap as the region lookup) raycasts the hit
  targets before the globe. On a hit it sets `tappedSat` and shows
  `#sat-tip` (a fixed tooltip anchored above the dot, `translate(-50%,
  -135%)`) with the name and orbit type. `tick()` calls `updateSatTip()`
  each frame, which projects the dot's world position to the screen, so the
  tag tracks the satellite as it moves; the tag hides while the dot faces
  away from the camera (`dir.dot(toCam) <= 0.02`). Tapping the globe or
  empty space clears the tag; the tag and the region label are mutually
  exclusive (one tap shows one tag).
- Toggle: `#sat-toggle` (a "show satellites" checkbox, top-right; bottom-
  right on screens under 640px wide) sets `satWorld.visible`. Unchecking
  hides every dot and orbit line and hides the tag if it belongs to this
  layer.

### Starlink constellation (full set, live)

- `refreshStarlink()` fetches the whole constellation on load and every
  30 minutes (`STARLINK_REFRESH_MS`). `fetchStarlinkTle()` tries the
  satvisor GitHub mirror first (`STARLINK_TLE_FALLBACK_URL`, updated every
  few hours), then Celestrak (`gp.php?GROUP=starlink&FORMAT=tle`). Each
  fetch aborts after 12 s (`TLE_FETCH_TIMEOUT_MS`). A failed refresh
  schedules a retry after 60 s (`STARLINK_RETRY_MS`) until one succeeds;
  success cancels the retry and the 30 min cycle resumes. `refreshing`
  stays true until `initStarlinkPoints` so the readout keeps `fetching…`
  through the chunked build. There is no embedded fallback: on total
  failure it keeps the last good points and the readout shows the last
  good state, or `unavailable` if nothing loaded yet.
- `parseTleText(text)` splits the TLE text into `{ name, line1, line2 }`
  records (shared with the named-sat refresh). `buildStarlink(records)`
  builds the `satrec`s in chunks: 8 ms of `twoline2satrec` per
  `requestAnimationFrame`, guarded by a generation counter (`slBuildGen`)
  so a refresh mid-build discards the stale build.
- All sats render as one `THREE.Points` (one draw call) with a
  `PointsMaterial` (`vertexColors`, `size 0.07`, `opacity 0.9`,
  `depthWrite: false`). A manual `boundingSphere` (radius 15) keeps the
  tap raycast cheap; `frustumCulled` is off.
- The points live in their own `slWorld` group (a sibling of `satWorld`),
  so the "show satellites" toggle does not hide them. `tick()` sets
  `slWorld.rotation.y = -gmst` each frame (the same GMST as `satWorld`),
  which rotates the inertial positions into the Earth-fixed view.
- `tickStarlink(now)` propagates the sats in time slices: it advances a
  cursor through the array, calling `sat.propagate` until it has used 8 ms
  (`SL_PROPAGATE_BUDGET_MS`), then resumes next frame. Unpropagated sats
  park far away (1e6) until their turn.
- Orbit lines: one merged `THREE.LineSegments` in `slWorld` (one draw
  call, `LineBasicMaterial`, `SL_COLOR`, opacity 0.05, `depthWrite:
  false`, `frustumCulled` off) holds every satellite's orbit, 64 samples
  each (`SL_ORBIT_SAMPLES`). The buffer starts zeroed (an unfilled orbit
  is a degenerate segment at the globe center). `tickStarlink` builds each
  orbit on the first successful propagation of its satellite: `orbitBasis`
  derives the osculating ellipse (semi-major axis, eccentricity, in-plane
  basis) from the ECI position and velocity, and `fillOrbitLine` writes
  the ellipse in group-local coords. The per-frame upload covers only the
  newly filled range (`addUpdateRange`), so the full set builds within the
  first propagation cycle (~1 s) without blocking a frame.
- Color `SL_COLOR` (`0x9fd8ff`). Tap raycasts the `Points` with
  `raycaster.params.Points.threshold = 0.1` (tight: the shell is dense, and
  a wide threshold steals taps meant for the globe), rejects dots behind
  the globe, and picks the dot closest to the tap ray (the frontmost dot
  may sit off to the side). `onTap` then shows the same `#sat-tip` tag as
  the named satellites, with the name and `Starlink (LEO)`.
- Toggle: `#starlink-toggle` (a "show Starlink" checkbox) sets
  `starlink.points.visible` and `starlink.orbits.visible` (the orbits
  belong to the layer) and hides the tag if it belongs to this layer.
  `#starlink-orbits-toggle` (a "show Starlink orbits" checkbox) sets
  `starlink.orbits.visible` on its own.

### Weather radar (live, RainViewer)

- `refreshWeather()` fetches `https://api.rainviewer.com/public/weather-maps.json`
  on load and every 10 minutes (`WEATHER_REFRESH_MS`). The service is public,
  keyless, and CORS-open. The JSON gives a `host` and a `radar.past` list of
  frames; the last frame is the current one (~10-min cadence). The fetch
  aborts after 12 s (`TLE_FETCH_TIMEOUT_MS`). A failed refresh keeps the last
  good frame.
- `applyWeatherFrame(host, path, frameTime)` fetches the current frame as a
  4×4 grid of 256 px Web Mercator tiles (zoom `WEATHER_ZOOM` = 2, color
  scheme 2 "Universal Blue"):
  `{host}{path}/256/2/{x}/{y}/2/1_0.png`. It composites the grid onto a
  1024×1024 Mercator canvas, then remaps it to a 2048×1024 equirectangular
  canvas (`weatherCanvas`). The remap draws one output row per latitude
  band, stretching the matching Mercator band; the polar caps (|lat| ≥
  85.05°) stay clear. A post-pass drops pixels with alpha < 32 (the row
  stretch leaves low-alpha residue that would haze the globe).
- `weatherTex` is a `THREE.CanvasTexture` on `weatherCanvas`, sRGB, with
  `minFilter = LinearFilter` (no mipmaps: they haze the sparse alpha). The
  overlay is a translucent sphere at `R + DISPLACEMENT + 0.08`
  (`MeshBasicMaterial`, `opacity 0.9`, `depthWrite: false`), so it sits above
  the displaced terrain and matches the globe UV mapping.
- The overlay is a separate mesh and is not in the tap raycast targets, so it
  does not block satellite or region picks.
- Toggle: `#weather-toggle` (a "show weather" checkbox, checked by default)
  sets `weatherOverlay.visible` (only when a frame has loaded).
- Readout line "Weather radar" (`#weather`): `updated HH:MM:SS UTC` (the
  frame time), or `fetching…`, or `unavailable` when no frame has loaded.
- The `visibilitychange` handler re-fetches when the tab returns with stale
  data. The hint line credits RainViewer (required by their API terms).

### Readout (top-left panel)

The panel starts collapsed to a one-line summary (`#readout-summary`,
`Subsolar <lat> · <lon> · HH:MM UTC`). Tapping the line (`#readout-head`,
`role="button"`; Enter and Space also toggle) shows the full body
(`#readout-body`). The caret shows `▸` when collapsed, `▾` when expanded.
The head has `pointer-events: auto` (the rest of the panel stays
`pointer-events: none` so drags pass through to the globe). On screens
under 640px wide the panel font drops to 12px so the summary stays on
one line. The summary updates every frame in `tick()`.

The body shows UTC time, subsolar latitude, subsolar longitude, solar
declination, the equation of time, a legend for the sun path and the
satellite orbit types (LEO/MEO/GEO), the TLE data status (last update
time, "updating…", or "embedded fallback"), the Starlink status
(`#starlink`: `N sats · updated HH:MM:SS UTC`, or `fetching…`, or
`unavailable`, or `satellite.js failed to load`), and the weather radar
status (`#weather`: `updated HH:MM:SS UTC`, or `fetching…`, or
`unavailable`).

### Logo and favicon

- `favicon.svg` is the app logo: a dark rounded square holding a blue
  globe with a night-side crescent and city lights, a glowing sun, the
  amber/blue sun path meeting at the subsolar marker, and a satellite
  orbit with a dot.
- `apple-touch-icon.png` is a 180 px PNG render of the same mark, for
  iOS home screens.
- `index.html` links both in the head: `rel="icon"` (SVG) and
  `rel="apple-touch-icon"` (PNG).

## Recent changes (as of 2026-08-24)

- `16446e5` Add live weather radar from RainViewer with a show/hide toggle
- `7fec186` Add Starlink orbit lines with a show/hide toggle
- `dd971fb` Show satellite names on tap; the tag tracks the dot and hides
  with its layer toggle
- `91070f0` Add country and US state borders, tap the globe to name a region
- `6a3c17c` Put the Sun at true scale and distance (1 AU away, 0.53° across)
- `5d2a1aa` Add a logo and set it as the favicon
- `cde6ee9` Collapse the readout to a one-line summary, tap to expand
- `eb8e9d2` Add a location pin from the browser's public IP via IP2GeoAPI
- `d477164` Fetch TLE data from the satvisor GitHub mirror first
- `2516030` Add a timeout and mirror fallback to the Starlink TLE fetch
- `08f9290` Add the full Starlink constellation as a live SGP4 point cloud
- `3573ebc` Add 35 satellites, shrink the marker dot, add a show/hide toggle
- `f20f0a4` Re-fetch satellite TLEs every 5 minutes while the app runs
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
- Batch responses so a single reply stays under the maximum response
  length. Group independent tool calls into one message. Keep replies
  short; give the full detail in a follow-up message only if the user
  asks.
