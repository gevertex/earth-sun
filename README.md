# Earth — real-time sun position

A single self-contained page (`index.html`) that renders a 3D Earth globe and marks the
subsolar point (where the Sun is directly overhead) in real time, with the Sun placed in
its true direction so the day/night terminator is visible.

Open it from a local server (Three.js loads from a CDN):

```sh
python3 -m http.server 8080
# then visit http://127.0.0.1:8080
```
