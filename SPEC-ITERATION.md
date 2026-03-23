# ona-walk — Iteration Log

Changes made during the debugging and MVP stabilization session, starting from the initial implementation commit (`3ae8033`).

---

## 1. DevContainer & Tooling

- Added Node.js 20 feature to `.devcontainer/devcontainer.json` (Node.js was not available in the base image).
- All package management uses **pnpm** (npm is blocked by security policy).
- Added `allowedHosts: true` to `vite.config.ts` to allow access from Gitpod preview URLs.

## 2. Globe Coordinate Fix

**Problem:** Zooming into the globe always landed in the ocean. The `triggerTransition` function in `globe.ts` computed lat/lon from the camera's look direction using an incorrect formula — it negated and re-derived coordinates that didn't match the globe's actual geographic mapping.

**Fix:** Derive lat/lon from the camera's normalized position vector instead. The camera orbits the globe, so its position on the unit sphere directly corresponds to the geographic point it's above:

```ts
const camPos = camera.position.clone().normalize();
const lat = Math.asin(camPos.y) * (180 / Math.PI);
const lon = Math.atan2(camPos.z, -camPos.x) * (180 / Math.PI) - 180;
```

When within 15° of a known city, coordinates snap to the city's exact lat/lon.

**File:** `src/systems/globe.ts`

## 3. ReorientationPlugin & Coordinate System

**Problem:** The initial implementation used raw ECEF coordinates from Google 3D Tiles, requiring complex East-North-Up frame calculations for camera movement. The controls system computed local ENU axes from the ellipsoid, which was fragile and hard to debug.

**Fix:** Use `ReorientationPlugin` from `3d-tiles-renderer/three/plugins` to transform the entire tileset so the target lat/lon is at world origin `(0,0,0)` with Y-up. This means:
- Camera at `(0, height, 0)` is directly above the target city
- Standard Y-up movement works without ENU calculations
- Ground surface is near `y=0` (plus local elevation)

Key detail: `tiles.group.position` will show values like `(0, -6371248, -19980)` — this is the inverse ECEF transform applied to the group, not a bug. The children's world-space positions end up near the origin.

**Files:** `src/systems/tiles.ts`, `src/systems/controls.ts`

## 4. Controls Simplification

**Problem:** Controls used ECEF-space ENU frame calculations (`ellipsoid.getPositionToNormal`, cross products for local axes). This was complex, hard to debug, and broke when tile data wasn't loaded yet.

**Fix:** With ReorientationPlugin handling the coordinate transform, controls now work in simple Y-up local space:
- Yaw/pitch Euler rotation (`YXZ` order)
- Movement on the XZ horizontal plane
- No ENU frame computation needed

**File:** `src/systems/controls.ts`

## 5. Tile Loading Limits

**Problem:** The tile renderer made unlimited concurrent requests with per-tile `console.log` calls, causing 170,000+ log entries and browser tab crashes.

**Fix:**
- Throttled logging to once per 2 seconds instead of per-tile
- Set `downloadQueue.maxJobs = 12`, `parseQueue.maxJobs = 12`
- Set `lruCache.maxSize = 1200`
- Set `maxDepth = 50`, `loadSiblings = false`
- Removed deprecated `errorThreshold` option

**Files:** `src/systems/tiles.ts`, `src/test-tiles.ts`

## 6. Dynamic LOD Based on Camera Altitude

**Problem:** A single `errorTarget` value either loaded too few tiles (coarse geometry at street level) or too many (runaway requests when zoomed out).

**Fix:** `errorTarget` adjusts dynamically based on camera height:

| Camera height | errorTarget | Detail level |
|---|---|---|
| < 100m | 0.5 | Maximum (street level) |
| 100–500m | 1 | High |
| 500–2000m | 2 | Medium |
| > 2000m | 4 | Coarse (performance) |

**Files:** `src/systems/tiles.ts` (`updateTiles`), `src/test-tiles.ts`

## 7. Cinematic Descent Transition

**Problem:** Setting the camera directly to street level (`y = EYE_HEIGHT`) before tiles loaded meant the tile renderer never saw the camera in its frustum at the right altitude to trigger loading. Ground raycasts failed because no geometry was loaded yet.

**Fix:** Animated descent from aerial view to street level:
1. Camera starts at `(200, 500, 200)` — angled aerial view
2. Over 6 seconds, smoothly interpolates to `(0, groundY + 1.7, 0)` with eased motion
3. Tiles load progressively as the camera descends (each altitude triggers appropriate LOD)
4. Raycasts for ground throughout the descent
5. Falls back to `y = EYE_HEIGHT` if no ground hit

This doubles as a cinematic city entry effect.

**File:** `src/main.ts` (`descendToStreetLevel`)

## 8. Camera Configuration

**Problem:** Camera far plane was 5000, clipping tiles at distance. `ACESFilmicToneMapping` darkened the already-baked tile textures.

**Fix:**
- Far plane: 5000 → 100000
- Tone mapping: `ACESFilmicToneMapping` → `NoToneMapping`

**File:** `src/main.ts`

## 9. PSX Shader Tuning for Photogrammetry

**Problem:** The PSX shader's vertex snapping (`uSnapGrid = 160`) collapsed photogrammetry mesh vertices into huge polygons, making the scene nearly black with occasional bright patches. The effect was designed for low-poly game models, not dense photogrammetry meshes.

**Fix:**
- Disabled vertex snapping (`uSnapGrid = 0`) — photogrammetry tiles are already detailed enough
- Reduced jitter from 0.6 → 0.3
- Kept texture pixelation (`uTexelSize = 128`) for the retro look
- Kept 5-bit color dithering (Bayer 8×8 matrix)
- Kept fog (near=150, far=600)
- Fixed material extraction to handle `MeshBasicMaterial` (Google tiles) not just `MeshStandardMaterial`

The PSX aesthetic now comes from pixelated textures + dithered colors + fog, not geometry destruction.

**File:** `src/shaders/psx.ts`

## 10. Ground Snapping

**Problem:** Continuous ground snapping during walking used a small search range (50m above, 200m far) that missed the ground when elevation varied.

**Fix:** Increased to 100m above camera, 500m range. Smooth interpolation (`lerp 0.3`) prevents jarring vertical jumps.

**File:** `src/main.ts` (`updateGroundSnap`)

## 11. Test Page

Created `test-tiles.html` + `src/test-tiles.ts` as a standalone diagnostic page that loads Google 3D Tiles at Buenos Aires with orbit controls and debug overlays. No PSX shader, no globe, no weather — just tiles. Used to isolate and diagnose rendering issues independently of the main app.

Key diagnostic value: confirmed that `ReorientationPlugin` works correctly and tiles render at the origin, proving the main app's issues were in the shader and coordinate systems, not in tile loading.

**Files:** `test-tiles.html`, `src/test-tiles.ts`

## 12. Import Path Corrections

- `GoogleCloudAuthPlugin`: must import from `3d-tiles-renderer/core/plugins` (not `three/plugins`)
- `ReorientationPlugin`: imports from `3d-tiles-renderer/three/plugins`
- `TilesRenderer.raycast()`: called directly on the renderer instance, not on `tiles.group`

## 13. Ground-Level Camera Placement

**Problem:** Different cities sit at different heights above the WGS84 ellipsoid in the reoriented tileset space (e.g. SF at Y≈-192, Chicago at Y≈-5694). Previous attempts tried to normalize the ground to Y=0 by modifying `group.position.y` or re-calling `transformLatLonHeightToOrigin` with derived heights. Both corrupted the tileset transform — `group.position` is in a rotated frame, so shifting Y moves the tileset in the wrong direction.

**Fix:** Stop fighting the tileset transform. The ground sits wherever the `ReorientationPlugin` puts it. During the cinematic descent, raycast downward to find the actual ground mesh Y, then land the camera at `groundY + EYE_HEIGHT`. Ground snapping during walking also raycasts with a large vertical buffer (500m above camera) to handle any ground elevation. The tileset group transform is never modified after initialization.

**Files:** `src/main.ts` (`descendToStreetLevel`, `updateGroundSnap`), `src/systems/tiles.ts`

## 14. City Marker Click-to-Enter + Hover Tooltips

**Problem:** The only way to enter a city was zooming in with the scroll wheel. No way to select a specific city directly.

**Fix:** Each city marker on the globe is now a group of three objects:
- Visible dot (0.6 radius) — the green sphere
- Invisible hit area (3.0 radius) — makes hovering/clicking much easier
- Hover ring (2.0–2.6 radius) — semi-transparent green ring appears on hover

Clicking a marker triggers the street mode transition to that city's exact coordinates. Drag vs click is distinguished by a 5px movement threshold. Tooltip is PSX-styled (monospace, green-on-dark, uppercase).

**Files:** `src/systems/globe.ts`

## 15. Mobile Touch Support

**Globe mode:**
- One-finger drag → rotate globe
- Pinch → zoom in/out (triggers street transition at threshold)
- Tap city marker → enter that city

**Street mode:**
- Left half of screen → virtual joystick (movement)
- Right half of screen → drag to look around
- Both work simultaneously via multi-touch tracking

The virtual joystick is a DOM overlay (green ring + knob, PSX-styled) that appears only on touch devices in street mode.

**Files:** `src/systems/globe.ts`, `src/systems/controls.ts`

## 16. PSX Shader Mobile Compatibility

**Problem:** Tiles rendered on desktop but not on iOS Safari. Three shader issues:
1. `precision mediump float` — tile vertex positions go through `modelViewMatrix` involving ECEF coordinates in the millions. `mediump` (16-bit float) can't represent these.
2. `const float[64]` array syntax — requires WebGL2/GLSL ES 3.0. iOS Safari can fall back to WebGL1.
3. Bayer dithering matrix used unsupported array constructor.

**Fix:**
1. Changed to `precision highp float` in both vertex and fragment shaders
2. Replaced `const float[64]` with a 4x4 Bayer matrix using `mat4` (universally supported)
3. Dither quality reduced from 8×8 to 4×4 — visually indistinguishable at PSX resolution

**File:** `src/shaders/psx.ts`

## 17. Distance-Based PSX Effect Scaling

**Problem:** PSX effects (texture pixelation, vertex jitter, vertex snapping, color dithering) distorted close-up geometry badly — streets and buildings within 10-20m were unreadable.

**Fix:** All four effects now scale with view distance via `smoothstep(5.0, 80.0, distance)`:

| Effect | < 5m | > 80m |
|---|---|---|
| Texture pixelation | None (full resolution) | Full (256 texel grid) |
| Vertex snapping | None | Full grid snap |
| Vertex jitter | None | Full wobble |
| Color dithering | 8-bit (256 levels) | 5-bit (32 levels) |

Also: brightness lift (`color * 1.3 + 0.05`) and contrast boost (`mix(0.5, color, 1.15)`) to make streets/sidewalks readable against dark baked shadows. Fog pushed back (near 400m, far 1200m).

**Files:** `src/shaders/psx.ts`, `src/main.ts`

## 18. Tile Response Caching (Service Worker)

**Problem:** Each city visit downloads ~1200 tile requests (~150MB). Revisiting the same city or walking back through previously loaded areas re-downloads everything.

**Architecture:**

```
Browser ──fetch──▶ tile.googleapis.com
                        │
                   Service Worker intercepts
                        │
                   ┌─────▼──────┐
                   │ In cache?   │
                   └─────┬───────┘
                    yes  │  no
                    │    │
                    │    ▼
                    │  fetch from network
                    │    │
                    │    ├──▶ store in Cache API
                    │    │
                    ▼    ▼
               Return response
```

- **Strategy:** Cache-first for all `tile.googleapis.com` requests except `createSession` (auth tokens)
- **Storage:** Browser Cache API — persists across sessions, up to ~500MB
- **Eviction:** When entries exceed ~3000, oldest 20% are deleted
- **Stats:** HUD displays cache hit rate in bottom-left corner (polled every 3s via `postMessage` to the SW)
- **Activation:** SW registers on first load, starts intercepting on second load

**Files:** `public/tile-cache-sw.js`, `src/main.ts` (registration), `src/systems/hud.ts` (stats display)

## 19. Tile Request Throttling

**Problem:** During transition, the main `animate()` loop and the descent animation both called `tiles.update()` every frame — double tile updates caused 1000+ API requests in seconds.

**Fix:**
- `animate()` does nothing during `mode === 'transition'` — the descent loop is the sole owner of tile updates during that phase
- `downloadQueue.maxJobs`: 12 → 6
- `parseQueue.maxJobs`: 12 → 6
- `lruCache.maxSize`: 1200 → 800
- Fixed `errorTarget` to 4 (was dynamic but broke when ground Y varied per city)

**Files:** `src/main.ts` (`animate`), `src/systems/tiles.ts`

---

## Current State (MVP)

Working:
- Globe view with day/night shader, city markers, terrain displacement
- Click/tap city markers to enter directly, with hover tooltips
- Zoom-to-enter transition with cinematic descent
- Google 3D Tiles rendering at correct coordinates for all cities
- Camera lands on actual ground regardless of city elevation
- PSX visual filter with distance-based scaling (clean close, retro far)
- First-person WASD + mouse look controls (desktop)
- Virtual joystick + touch look controls (mobile)
- Ground snapping via raycast (handles any ground elevation)
- Real-time weather display (OpenWeatherMap)
- Time of day with sky colors
- City ambience audio (procedural Web Audio)
- HUD with compass, weather, time, cache stats
- Tile response caching via Service Worker (persists across sessions)
- Mobile-compatible shaders (highp precision, no WebGL2-only features)

Known limitations:
- NPCs not yet visible in street mode
- Weather particles not rendering
- Street lights not visible
- No collision detection (walk through buildings)
- Globe coordinate mapping has minor offset from visual position
- Service Worker activates on second page load (first load registers it)
- Tile cache eviction is entry-count based, not byte-size based
