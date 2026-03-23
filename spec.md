# ona-walk — Specification

A browser-based walkable world that renders real Google 3D city data through a PSX-era visual filter. First-person exploration of any city on earth, with real-time weather and time of day, populated by low-poly NPCs.

---

## Problem Statement

There is no web experience that lets you walk through real-world cities rendered in a retro PSX aesthetic. Google's Photorealistic 3D Tiles provide accurate geometry for most major cities worldwide, but they are presented photorealistically. This project re-renders that data through a 1997 PlayStation-style shader pipeline, turning the real world into a lo-fi walking simulator.

---

## Core Experience

1. **Globe View** — The app opens on a PSX-styled low-poly earth. The user can rotate and zoom into any location. As they zoom past a threshold, the view transitions into first-person street level.

2. **First-Person Walking** — WASD movement + mouse look. The user walks at human pace through the city. Collision is approximated against the 3D tile geometry. The camera is at eye height (~1.7m).

3. **PSX Rendering** — All geometry is rendered through a shader pipeline that produces authentic 1997 PlayStation visuals: vertex jitter/snapping, affine texture mapping, reduced texture resolution, color dithering, and distance fog.

4. **Live Weather & Time** — The sky, lighting, fog, and particle effects reflect the actual current weather and local time of the city being viewed. Rain and snow are rendered as PSX-style low-poly particles.

5. **NPC Life** — Major cities have pedestrians walking sidewalks and cars driving roads. NPC models are city-appropriate (e.g., yellow cabs in NYC, tuk-tuks in Bangkok). Rural/suburban areas have no NPCs.

6. **City Ambience** — Spatial audio of city sounds: traffic hum, footsteps, wind, distant horns. Intensity scales with NPC density. Rural areas are quiet (wind, birds).

---

## Technical Architecture

### Stack

| Layer | Technology |
|---|---|
| Rendering | Three.js |
| 3D Tile Loading | `3d-tiles-renderer` (NASA-AMMOS) |
| 3D Data Source | Google Maps Photorealistic 3D Tiles API |
| Weather Data | OpenWeatherMap API (free tier: current weather) |
| Timezone/Time | Calculated from longitude or via timezone API |
| Build Tool | Vite |
| Language | TypeScript |

### System Diagram

```
┌─────────────────────────────────────────────────┐
│                   Browser                        │
│                                                  │
│  ┌──────────┐   ┌──────────────┐   ┌──────────┐ │
│  │  Globe    │──▶│  Street View │──▶│  HUD /   │ │
│  │  Scene    │   │  Scene       │   │  UI      │ │
│  └──────────┘   └──────┬───────┘   └──────────┘ │
│                        │                         │
│         ┌──────────────┼──────────────┐          │
│         ▼              ▼              ▼          │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐   │
│  │ PSX Shader │ │ NPC System │ │ Weather    │   │
│  │ Pipeline   │ │            │ │ System     │   │
│  └────────────┘ └────────────┘ └────────────┘   │
│         ▲              ▲              ▲          │
└─────────┼──────────────┼──────────────┼──────────┘
          │              │              │
   Google 3D Tiles   Built-in      OpenWeatherMap
       API           Assets           API
```

---

## Detailed Requirements

### R1: Globe View

| ID | Requirement |
|---|---|
| R1.1 | On load, display a low-poly PSX-styled earth globe centered in the viewport. |
| R1.2 | Globe rotates with click-drag. Scroll wheel zooms. |
| R1.3 | Globe surface uses a simplified flat-shaded mesh (icosphere or similar, ~2000 faces) with continent colors. |
| R1.4 | Major city markers are visible as small glowing dots on the globe. |
| R1.5 | When zoom passes a threshold (approximate altitude < 500m), transition to street-level first-person view. The transition is a camera dive with a brief fade or PSX-style wipe. |
| R1.6 | The globe itself is rendered with the PSX shader (vertex jitter, dithering). |

### R2: First-Person Navigation

| ID | Requirement |
|---|---|
| R2.1 | WASD keys for forward/back/strafe. Mouse controls look direction. |
| R2.2 | Pointer lock on click. ESC releases pointer lock and shows a minimal menu. |
| R2.3 | Walking speed ~5 km/h (human pace). Optional shift-to-run at ~12 km/h. |
| R2.4 | Camera height fixed at ~1.7m above ground. Gravity snaps camera to tile surface. |
| R2.5 | Basic collision: raycast downward against loaded tile geometry for ground detection. Horizontal collision against nearby geometry to prevent walking through walls. |
| R2.6 | A minimap or compass in the corner showing cardinal directions and approximate location. |

### R3: PSX Shader Pipeline

| ID | Requirement |
|---|---|
| R3.1 | **Vertex snapping**: Snap vertex positions to a coarse grid in the vertex shader to simulate low-precision vertex transforms. Grid resolution ~1/128 of view distance. |
| R3.2 | **Vertex jitter**: Subtle per-frame vertex position noise to simulate floating-point instability. |
| R3.3 | **Affine texture mapping**: Disable perspective-correct texture interpolation (or approximate it in the fragment shader) to produce the characteristic PSX texture warping. |
| R3.4 | **Texture resolution reduction**: Downsample tile textures to ~64x64 or 128x128 equivalent resolution. Nearest-neighbor filtering (no bilinear). |
| R3.5 | **Color dithering**: Apply ordered dithering (Bayer matrix) to reduce effective color depth to ~15-bit (5-5-5 RGB). |
| R3.6 | **Distance fog**: Dense fog that fades geometry to a solid color at medium distance (~200-400m). Fog color matches sky/time of day. |
| R3.7 | **Draw distance**: Aggressive draw distance culling. Geometry beyond fog range is not rendered. |
| R3.8 | The shader must work as a custom material applied to all 3D Tiles geometry via `3d-tiles-renderer`'s `onLoadModel` callback. |

### R4: 3D Tile Loading

| ID | Requirement |
|---|---|
| R4.1 | Use `3d-tiles-renderer` with the `GoogleCloudAuthPlugin` to stream Google Photorealistic 3D Tiles. |
| R4.2 | Configure tile LOD to prefer lower-detail tiles (fits the PSX aesthetic and reduces bandwidth). |
| R4.3 | Apply the PSX custom material to each tile mesh as it loads. |
| R4.4 | Handle tile loading/unloading gracefully — no pop-in beyond what the fog hides. |
| R4.5 | Works for any location where Google has 3D tile coverage. Falls back to flat terrain where coverage is unavailable. |

### R5: Weather System

| ID | Requirement |
|---|---|
| R5.1 | On entering a location, fetch current weather from OpenWeatherMap using the lat/lon coordinates. |
| R5.2 | Re-fetch weather every 10 minutes while the user remains in the same area. |
| R5.3 | **Sky color**: Map weather condition codes to sky gradients (clear blue, overcast grey, stormy dark, sunset oranges based on time). |
| R5.4 | **Rain**: PSX-style elongated quad particles falling. Density proportional to rain intensity from API. Subtle splash particles on ground contact. |
| R5.5 | **Snow**: PSX-style small quad particles drifting down with slight horizontal sway. |
| R5.6 | **Fog**: Increase fog density during foggy/misty conditions. Reduce visibility to ~100m. |
| R5.7 | **Wind**: Particle drift direction matches wind data from API. |

### R6: Time of Day

| ID | Requirement |
|---|---|
| R6.1 | Calculate local time from the city's longitude (or use a timezone lookup). |
| R6.2 | **Sunlight direction**: Approximate sun position based on time of day and latitude. Directional light rotates accordingly. |
| R6.3 | **Sky gradient**: Dawn (pink/orange), day (blue), dusk (orange/purple), night (dark blue/black). Smooth transitions. |
| R6.4 | **Night lighting**: At night, reduce ambient light. City areas get warm point lights simulating street lamps (placed procedurally along roads or at intervals). |
| R6.5 | **Stars**: At night, render a low-poly star field on the skybox. |

### R7: NPC System

| ID | Requirement |
|---|---|
| R7.1 | NPCs are low-poly humanoid models (~50-100 triangles each) with PSX-style textures. |
| R7.2 | Cars are low-poly box-shaped vehicles (~30-80 triangles) with flat-color textures. |
| R7.3 | NPCs spawn within a radius around the player (~100m) and despawn beyond ~150m. |
| R7.4 | Pedestrians walk along sidewalks/paths using simple waypoint or random-walk AI. They avoid walking through buildings (basic navmesh or raycast avoidance). |
| R7.5 | Cars move along roads at city speeds. Simple path-following, no complex traffic simulation. |
| R7.6 | **City-specific models**: A mapping of city/country to NPC variant sets. MVP can start with 3-4 generic sets (Western, Asian, South American, Middle Eastern) and expand. |
| R7.7 | **Density**: Use a built-in dataset of major metro area bounding boxes. Inside a metro area → spawn NPCs. Outside → no NPCs. Density tiers: mega-city (high), large city (medium), small city (low). |
| R7.8 | NPCs and cars are rendered with the same PSX shader as the environment. |

### R8: Audio

| ID | Requirement |
|---|---|
| R8.1 | Spatial audio using Web Audio API. |
| R8.2 | **City ambience**: Looping ambient track of city sounds (traffic hum, distant horns, murmur). Volume scales with NPC density. |
| R8.3 | **Footsteps**: Player footstep sounds synced to walking pace. |
| R8.4 | **Weather sounds**: Rain patter (intensity-scaled), wind howl, thunder cracks during storms. |
| R8.5 | **Rural/quiet areas**: Wind, occasional bird sounds, silence. |
| R8.6 | Audio starts muted. User clicks to unmute (browser autoplay policy). |

### R9: UI / HUD

| ID | Requirement |
|---|---|
| R9.1 | Minimal HUD. PSX-era font (e.g., a bitmap pixel font). |
| R9.2 | **Location label**: City/country name displayed when entering a new area. Fades after a few seconds. |
| R9.3 | **Compass**: Small compass or cardinal direction indicator. |
| R9.4 | **Weather icon**: Small icon showing current weather condition. |
| R9.5 | **Time display**: Local time of the city in a retro digital clock style. |
| R9.6 | **Menu (ESC)**: Return to globe, adjust settings (fog distance, NPC density override, audio volume). |
| R9.7 | All UI elements styled to match the PSX aesthetic (low-res, dithered edges, muted colors). |

---

## Acceptance Criteria

1. **Globe → Street transition works**: User can spin the globe, zoom into any city with Google 3D coverage, and seamlessly enter first-person walking mode.
2. **PSX aesthetic is convincing**: Side-by-side with a 1997 PSX game screenshot, the visual style is recognizably similar — vertex wobble, texture warping, dithering, fog are all present.
3. **Real weather is reflected**: Standing in a city where it's currently raining shows rain particles and overcast sky. A city at night is dark with artificial lighting.
4. **NPCs appear in cities**: Walking through central Tokyo or Manhattan shows pedestrians and cars. Walking through rural Kansas shows none.
5. **Performance**: Maintains 30+ FPS on a mid-range laptop (integrated GPU) at 1080p. The PSX aesthetic helps here — low LOD tiles, aggressive fog culling.
6. **Audio matches environment**: City sounds in cities, quiet in rural areas, rain sounds when raining.
7. **Works globally**: Any location with Google 3D Tiles coverage is walkable. Locations without coverage show flat terrain with fog.

---

## Implementation Plan

### Phase 1: Foundation
1. **Project scaffolding** — Vite + TypeScript + Three.js setup. Dev server, basic scene rendering.
2. **Google 3D Tiles integration** — Load tiles via `3d-tiles-renderer` with Google API key. Render photorealistic tiles in a basic Three.js scene with orbit controls. Verify tiles load and display.
3. **PSX shader pipeline** — Write the custom ShaderMaterial: vertex snapping, affine textures, texture downsampling, dithering, fog. Apply to all tile meshes via `onLoadModel`.
4. **First-person controls** — WASD + mouse look with pointer lock. Ground-snapping via raycasts against tile geometry. Basic horizontal collision.

### Phase 2: World Systems
5. **Globe view** — Low-poly earth mesh with PSX shader. City markers. Camera zoom transition from globe to street level. Coordinate mapping (globe click → lat/lon → tile loading position).
6. **Weather system** — OpenWeatherMap integration. Sky gradient based on weather + time. Rain/snow particle systems with PSX-style quads. Fog density adjustment.
7. **Time-of-day lighting** — Sun position from time + latitude. Directional light rotation. Sky color transitions. Night mode with procedural street lights and star skybox.

### Phase 3: Life & Polish
8. **NPC system** — Low-poly pedestrian and car models. Spawn/despawn radius. Simple movement AI (random walk for pedestrians, path follow for cars). Metro area density lookup.
9. **City-specific NPC variants** — Model/texture sets per region. Mapping of coordinates to NPC variant.
10. **Audio system** — Web Audio API spatial audio. City ambience loops, footsteps, weather sounds. Volume scaling by density and weather.
11. **HUD & UI** — PSX-styled bitmap font. Location label, compass, weather icon, time display. ESC menu with settings and return-to-globe.

### Phase 4: Hardening
12. **Performance optimization** — Tile LOD tuning, shader performance, NPC culling, draw call batching. Target 30+ FPS on integrated GPU.
13. **Edge cases** — No-coverage fallback, ocean handling, tile loading failures, API rate limits.
14. **Deployment** — Build pipeline, environment variable management for API keys, hosting setup.

---

## API Keys Required

| Service | Purpose | Tier |
|---|---|---|
| Google Maps Platform (Map Tiles API) | Photorealistic 3D Tiles | Pay-as-you-go (free $200/month credit) |
| OpenWeatherMap | Current weather by coordinates | Free tier (1000 calls/day) |

---

## Open Questions / Future Ideas

- **Shareable URLs**: Encode lat/lon/heading in URL so users can share locations.
- **Day/night cycle speed**: Option to accelerate time (e.g., 1 min = 1 hour) for watching sunrise/sunset.
- **Interior spaces**: Some Google 3D tiles include building interiors — could be explored.
- **Mobile support**: Touch controls for mobile browsers.
- **Multiplayer**: See other users walking (future phase).
