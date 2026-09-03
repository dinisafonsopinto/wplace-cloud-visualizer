# wplace pixel tracker — web viewer

A static, zero-build page that shows your Cloudflare-backed pixel discovery
database on top of the live wplace.live world map. Deploys to Cloudflare
Pages for free, as one HTML file.

## What changed from the original plan

The plan was to gut Hugi-R's `wplace-daily-archives` repo for its frontend.
After actually cloning it, that turned out to be a poor fit:

- Its frontend is a **Rust/WASM module** (`wasm-pack` build, `wimage` crate)
  that decodes weekly `.zst`-compressed pixel-diff archives through a custom
  `merged://tiles/...` protocol — built for browsing *historical snapshots*,
  not a live "who painted what" lookup.
- It also corrected a wrong assumption from earlier: wplace.live tiles are
  **real Web Mercator / lat-lng tiles** layered over OpenStreetMap (it's
  literally pixel art over the real world map), not a flat Cartesian grid.

So instead of forcing your live per-pixel JSON data through a pipeline built
for a different data shape, this is a small purpose-built viewer that
follows the original two-layer blueprint (base tiles + JSON overlay +
tooltip), just with the corrected projection, using Leaflet instead of
MapLibre/WASM since Leaflet has a built-in `maxNativeZoom` feature for
exactly this situation (data only exists at one zoom level, let users zoom
past it and it scales client-side).


## How the overlay actually renders

- **At or past your native zoom (11+):** each displayed map tile is either
  exactly one wplace sector or a cropped piece of one. The renderer fetches
  that sector's JSON once (cached in memory per session) and draws each
  pixel as a small filled square, with a thin outline so single pixels are
  visible even fully zoomed in.
- **Zoomed out past native zoom:** drawing every individual pixel would be
  both illegible and would fetch hundreds of sectors per pan. Instead each
  covered sector gets a single dot sized by how many pixels were discovered
  there — a density indicator rather than exact pixels. This is also gated
  by `OVERLAY_MIN_ZOOM` so very-zoomed-out views don't fetch anything.
- **Hover tooltip** only activates at native zoom or closer (below that,
  "one screen pixel = one wplace pixel" stops being true, so exact
  coordinate hit-testing isn't meaningful).

## Protecting your free tier

Per the original plan: set a `Cache-Control: public, max-age=120,
s-maxage=300` (or similar) header on your Worker's `GET /tile/:tx/:ty`
response. Cloudflare's edge will then serve repeat requests for the same
sector straight from cache without invoking your Worker or reading D1 again,
regardless of how many people have the page open.

## Deploying to Cloudflare Pages (free)

1. Set `CONFIG.WORKER_BASE_URL` (and check `normalizePixel`) in `index.html`.
2. Go to the Cloudflare dashboard → **Workers & Pages** → **Create** → **Pages**.
3. Choose **Upload assets**, and drag in just this one `index.html` file
   (or connect a GitHub repo containing it, for auto-deploy on push).
4. Cloudflare gives you a free `your-project.pages.dev` URL immediately —
   that's your shareable link. No server, no build step, no bandwidth cost.

## Known limitations / things to sanity-check against your real data

- Sector→tile alignment assumes wplace's own tile indices line up 1:1 with
  standard Web Mercator tile x/y at your `NATIVE_ZOOM`. This mirrors how
  Hugi-R's own frontend treats the `wplace` source (same z/x/y grid as the
  OSM base layer), but it's worth spot-checking one known pixel against
  your Worker to confirm before sharing the link widely.
- The density-dot fallback when zoomed out is a placeholder aesthetic — you
  may want to tune the radius formula or swap it for a heatmap once you see
  real coverage density.
- No pagination/streaming is implemented for `GET /tile/:tx/:ty` — if a
  sector can hold thousands of discovered pixels, consider capping what the
  Worker returns per request so a single busy sector doesn't stall a tile
  render.