# Earth — real-time sun position

A single self-contained page (`index.html`) that renders a 3D Earth globe and marks the
subsolar point (where the Sun is directly overhead) in real time, with the Sun placed in
its true direction so the day/night terminator is visible. It also draws the path the
subsolar point traces over the year at the current time of day: the past in amber and
the future in blue, meeting at the current marker.

The page also shows real satellites in their real orbits, propagated live from TLE data
with the SGP4 model. A curated set of 47 named satellites (ISS, CSS, GPS, Galileo,
GOES, and more) is drawn with orbit lines, and the full Starlink constellation
(~10,700 satellites) is drawn as a live cloud of points. Hover any satellite to see its
name. Two checkboxes in the corner show or hide each layer.

## Live site

<https://gevertex.github.io/earth-sun/>

## Deploy

The site is hosted on GitHub Pages. Pages serves the `main` branch as is.

To publish a change:

```sh
git add -A
git commit -m "describe the change"
git push
```

GitHub Pages rebuilds the site within a minute or two after the push.

Note: Three.js, `satellite.js`, and the Earth textures load from the unpkg.com CDN, and
satellite TLEs load from celestrak.org. Visitors need internet access. Offline, the globe
and Sun still work; the satellites fall back to embedded TLEs (the Starlink layer needs
the live feed and shows `unavailable` without it).

## Run locally

Open it from a local server (Three.js loads from a CDN):

```sh
python3 -m http.server 8080
# then visit http://127.0.0.1:8080
```
