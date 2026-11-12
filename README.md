# Near Miss

**Every close pass between active satellites, screened live in your browser.**

Near Miss loads the public catalogue of active satellites, propagates all of them with SGP4, and runs an all-on-all conjunction screen for the next three hours. Nothing runs on a server: the orbital mechanics, the screening and the rendering all happen in the page.

![Overview](docs/01-overview.png)

On the 12 November 2026 catalogue (15,880 active satellites) it finds:

| | |
|---|---|
| Passes closer than 5 km in 3 hours | **5,062** (one every 2.1 s) |
| Passes closer than 1 km | **157** |
| Closest predicted pass | **44 m**, YAOGAN-43 01D × STARLINK-37776 |
| Work done | 17.7 M SGP4 propagations, 49 M pair checks, about a minute on a laptop |

These are geometric estimates from public element sets, not operational conjunction assessments. See [Accuracy](#accuracy).

## What you can do

- Drag to rotate, scroll or pinch to zoom, hover any dot to identify it.
- Filter passes by miss distance (1, 2 or 5 km) and sort by soonest or closest.
- Click a pass to see both orbits, the miss distance, closing speed, altitude and the point on Earth it happens over.
- **Watch this pass** jumps the clock to 80 seconds before closest approach and plays it at 10×, with a live separation readout.
- The altitude chart shows where satellites are concentrated and where the close passes happen.

![A close pass](docs/02-close-pass.png)

## How the screening works

Comparing every pair directly is 126 million distance checks per time step. The screener in [`src/screening.js`](src/screening.js) cuts that to about 45,000:

1. **Propagate.** Every object is advanced to the step time with SGP4 (position and velocity, TEME frame). Steps are 10 seconds apart.
2. **Broad phase.** Positions go into a uniform 3-D grid with a cell edge of `threshold + vMax × step` (161 km). Any two objects that could come within the threshold during the step must be in the same or neighbouring cells. The grid is an open-addressing hash table in typed arrays, so a step allocates nothing.
3. **Narrow phase.** For each candidate pair, solve for the time of closest approach assuming straight-line relative motion across the step. If that is under the threshold, re-propagate both objects at the estimated time and solve again, which removes the curvature error.
4. **Filter.** Pairs closing slower than 0.1 km/s are docked or flying in formation. They are counted but not reported.

The screen is incremental: each animation frame gives it about 9 ms, so the globe stays interactive while it runs.

`npm test` checks the screener against a brute-force all-pairs search on a 1,500-object subset. Both must find the same passes with miss distances within 20 m of each other.

## Rendering

Running SGP4 for 16,000 objects on every frame is too slow, so positions are propagated at two times a few seconds apart and each frame interpolates between them (direction linearly, radius separately, so orbits stay round). A third snapshot is prepared a slice at a time in the background. The two satellites in a selected pass are propagated exactly on every frame.

The Earth is drawn with d3-geo's orthographic projection. Satellites are projected by hand from TEME to Earth-fixed to screen coordinates, which keeps a frame at a few milliseconds.

## Run it

No build step.

```sh
npm start          # serves the folder at http://localhost:3000
```

Opening `index.html` directly also works, using the bundled snapshot.

When the page is served from somewhere that can reach CelesTrak it fetches the current catalogue on load. Otherwise it uses `data/tle-snapshot.js`. Refresh the snapshot with:

```sh
npm run update-tle
```

Element sets go stale. After about a week the page warns that positions have drifted.

## Layout

```
index.html
src/
  catalog.js      3LE parsing, orbit regime and operator family
  screening.js    all-on-all conjunction screener
  globe.js        canvas renderer and projection
  app.js          clock, position snapshots, selection, panels
  styles.css
vendor/           satellite.js 5.0.0, d3-geo, d3-array
data/             catalogue snapshot, Natural Earth land outline
tools/            fetch-tle.sh
tests/            screener vs brute force
docs/             screenshots and demo video
```

## Accuracy

- Public two-line element sets are good to roughly a kilometre at epoch and get worse with age. A "44 m" pass means the predicted geometry is very close, not that the true miss distance is known to the metre.
- Element sets do not include planned manoeuvres. Starlink and other operators move satellites to avoid passes like these, often after the element set was published.
- Only active satellites are screened. Debris and rocket bodies, which are most of the tracked population, are not included.
- Operators assess risk with covariance data and collision probability, not miss distance alone.

Do not use this for operational decisions.

## Credits

- Orbital elements: [CelesTrak](https://celestrak.org) (Dr. T.S. Kelso)
- SGP4: [satellite.js](https://github.com/shashwatak/satellite-js)
- Projection: [d3-geo](https://github.com/d3/d3-geo)
- Land outline: Natural Earth via [world-atlas](https://github.com/topojson/world-atlas)
- Type: Archivo, JetBrains Mono (SIL Open Font License)

MIT licence.
