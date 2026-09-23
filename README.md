# Ridge Runner

A 3D endless runner set on a mountain ridge at dusk. Dodge boulders, slip past stone pillars, collect lanterns, and survive as the world speeds up and shifts through five colour palettes. The whole game lives in a single HTML file with no build step, no dependencies to install, and no audio assets.

Built with [three.js](https://threejs.org/) r128 and the Web Audio API.

## Table of contents

- [Quick start](#quick-start)
- [How to play](#how-to-play)
  - [Controls](#controls)
  - [Obstacles](#obstacles)
  - [Lives](#lives)
  - [Power-ups](#power-ups)
  - [Scoring](#scoring)
  - [Levels](#levels)
  - [Scoreboard](#scoreboard)
- [Project structure](#project-structure)
- [How it works](#how-it-works)
  - [Rendering and scene](#rendering-and-scene)
  - [The runner character](#the-runner-character)
  - [Spawning and difficulty](#spawning-and-difficulty)
  - [Collision detection](#collision-detection)
  - [Sound](#sound)
  - [Persistence](#persistence)
  - [Mobile and accessibility](#mobile-and-accessibility)
- [Tuning the game](#tuning-the-game)
- [Browser support](#browser-support)
- [Troubleshooting](#troubleshooting)
- [Credits](#credits)

## Quick start

Clone the repo and open the HTML file in a browser. That is the whole setup.

```bash
git clone https://github.com/karthik-zoro-96/ridge_runner.git
cd ridge_runner
open "Ridge Runner.html"        # macOS
# xdg-open "Ridge Runner.html"  # Linux
# start "Ridge Runner.html"     # Windows
```

The page loads the bundled `three.min.js` sitting next to it, so it works fully offline. If that file is missing, the game falls back to loading three.js r128 from cdnjs.

If you prefer to serve it over HTTP, any static server works:

```bash
python3 -m http.server 8000
# then visit http://localhost:8000/Ridge%20Runner.html
```

## How to play

Type a runner name (up to 12 characters), press **Start running** or hit **Enter**, and keep going as long as you can. The run ends when you lose your last life.

### Controls

| Action        | Keyboard                    | Touch                |
| ------------- | --------------------------- | -------------------- |
| Move left     | `←` or `A`                  | Swipe left           |
| Move right    | `→` or `D`                  | Swipe right          |
| Jump          | `Space`, `↑` or `W`         | Tap or swipe up      |
| Start / retry | `Enter`                     | Tap the button       |

There are three lanes. Lane changes are instant on input and the runner glides smoothly into position. You can only jump while on the ground.

### Obstacles

- **Boulders** are low rocks. Jump over them or change lanes.
- **Stone pillars** are too tall to jump. You have to change lanes.

Every row of obstacles blocks one or two lanes and always leaves at least one lane open, so a clean path always exists.

### Lives

You start each run with three lives, shown as hearts in the top right of the HUD.

- Hitting an obstacle costs one life, smashes the obstacle out of your way, shakes the camera, and gives you about a second of invulnerability (the runner blinks while it lasts).
- Losing your last life ends the run. The runner tumbles, the camera shakes, and the game-over card appears.

### Power-ups

Power-ups float in an open lane and appear roughly one in nine rows. Only one is on the field at a time. Active power-ups show as pills in the HUD with a draining timer bar.

| Power-up   | Colour | Duration | Effect                                                                                  |
| ---------- | ------ | -------- | --------------------------------------------------------------------------------------- |
| **Shield** | Cyan   | Until used | Absorbs one crash without costing a life. Shown as a translucent bubble around the runner. Never spawns while you already hold one. |
| **Magnet** | Red    | 8 s      | Pulls nearby lanterns toward you from any lane so you do not have to weave for them.   |
| **Boost**  | Pink   | 4 s      | Runs at 1.5x speed, smashes through every obstacle, widens the camera field of view, and trails pink streaks. A one-second grace period follows so you are not dropped straight into a wall. |

### Scoring

```
score = floor(distance travelled) + 25 × lanterns collected
```

Distance accrues continuously while running, faster as the speed climbs. Lanterns come in lines of three along one open lane. Your best score is persisted and shown under the live score in the HUD.

### Levels

You level up every **1000 points**. Each level raises the difficulty and fades the world into a new palette over about 2.5 seconds. A banner announces the level name and the runner gets an immediate speed kick so the change is felt.

| Level | Name          | Mood                                   |
| ----- | ------------- | -------------------------------------- |
| 1     | Dusk ridge    | Purple sky, pink horizon, warm sun     |
| 2     | Ember pass    | Deep plum sky, orange horizon          |
| 3     | Frost col     | Navy sky, ice-blue horizon, pale sun   |
| 4     | Night summit  | Near-black sky, indigo horizon         |
| 5+    | Storm crest   | Charcoal sky, muted violet horizon     |

Beyond level 5 the palette stays on Storm crest and the banner reads `Storm crest +N`, but difficulty keeps scaling. Per level the game:

- raises top speed by 5 units (capped at 58) and accelerates toward it faster,
- packs obstacle rows about 10% closer together (down to half the base gap),
- blocks two lanes more often (from 15% up to 75%),
- swaps more boulders for unjumpable pillars (from 22% up to 60%),
- makes lantern lines scarcer (from 70% down to 40%).

No matter how fast you go, rows are never spaced closer than half a second of travel time so there is always room to react.

### Scoreboard

The start and game-over screens show a local top-10 scoreboard. Each entry records the runner name, score, level reached and lanterns collected. Ties are broken by whichever run happened first. Your latest run is highlighted on the game-over board, and the note under the score tells you your rank or how many points you needed to make the board.

Use **Change runner** on the game-over screen to go back and enter a different name.

## Project structure

```
ridge_runner/
├── Ridge Runner.html   # The entire game: markup, CSS, and JavaScript
├── three.min.js        # three.js r128, bundled for offline use
└── README.md
```

Inside `Ridge Runner.html` the script is organised into commented sections in this order: renderer and scene, ground, scenery peaks, player model, particles, obstacle factories, power-ups, levels and palettes, game state, lives, sound, scoreboard, spawning, input, game flow (begin / game over), and the main loop.

## How it works

### Rendering and scene

- A `WebGLRenderer` with antialiasing and shadow maps, pixel ratio capped at 2 for performance on high-DPI phones.
- The sky is a large inverted sphere with a custom GLSL gradient shader whose top and bottom colours are uniforms, so level palettes can lerp them per frame.
- Linear fog matches the horizon colour so distant objects dissolve into the sky.
- A flat disc acts as the low sun. Hemisphere light plus one shadow-casting directional light give the dusk look.
- The ground is a long plane with a procedurally drawn canvas texture (two stripes marking the lanes). Scrolling the texture offset each frame creates the illusion of motion; the ground itself never moves.
- Two tilted planes fall away on either side to read as the ridge flanks, and 26 low-poly cones drift past at half speed as a parallax layer of peaks, wrapping around when they pass the camera.

### The runner character

The player is an original low-poly "mountain courier" built entirely from three.js primitives, with no external models:

- Tapered capsules for thighs, shins and forearms (r128 has no `CapsuleGeometry`, so each is a cylinder capped with two spheres).
- Rounded boxes via an extruded, bevelled `Shape` for the shoes, backpack, pocket and reflective strip.
- Stacked squashed spheres for a puffer jacket, a beanie with a folded cuff and pom-pom, hair tufts, goggles with emissive lenses, a rolled sleeping mat, and a blinking teal beacon on the pack.
- A scarf made from a dynamic 14-segment ribbon mesh whose vertices are recomputed every frame. It stretches with speed, ripples with sine waves and lifts when airborne.
- A point light attached to the player adds a soft teal glow.

Animation is procedural. A two-joint run cycle drives hips, knees, shoulders and elbows from a phase that advances with speed. In the air the limbs tuck. On landing the body squashes and a ring of dust puffs out. On a fatal crash the runner pitches forward and is knocked back so it does not overlap the obstacle.

A pool of 48 icosahedron particles handles footstep dust, landing puffs, debris bursts when obstacles smash, and boost streaks. Particles are recycled round-robin, so there are never allocations during play.

### Spawning and difficulty

Spawning is measured in world units travelled rather than time, so the density of obstacles stays consistent as speed rises. When the spawn counter runs out, `spawnRow` places a row at z = −90:

1. Pick one or two lanes to block (probability from the level's `doubleChance`), always leaving one open.
2. For each blocked lane, place a pillar or a boulder (probability from `pillarChance`).
3. With 11% chance, and only if no power-up is already on the field, place a power-up in an open lane and stop.
4. Otherwise, with probability `lanternChance`, place a line of three lanterns in an open lane.

The gap to the next row is a random base of 26 to 34 units scaled by the level's `gapScale`, but never less than half a second of travel at the current speed.

### Collision detection

Collisions are simple axis-aligned checks on the object's position relative to the runner: within 0.9 units on z and 1.0 units on x. Boulders only count if the runner is below a height of 1.0, which is how jumping clears them. Pillars always count. Resolution order on a hit is boost (smash), invulnerability (ignore), shield (consume and smash), spare life (lose one and smash), then game over.

Off-screen objects are removed once they pass z = 12 behind the camera.

### Sound

All audio is synthesised with the Web Audio API, so there are no sound files:

- `tone()` plays an oscillator with an exponential gain envelope and optional pitch slide.
- `thud()` fills a buffer with decaying white noise and runs it through a low-pass filter.

These two primitives compose into the lantern chime, power-up arpeggio, jump whoosh, smash, shield pop, hit, game-over descent and level-up fanfare. The `AudioContext` is created on the first user gesture to satisfy browser autoplay rules.

### Persistence

Everything is stored in `localStorage` under three keys:

| Key           | Contents                                        |
| ------------- | ----------------------------------------------- |
| `ridge-best`  | Best score as a number                          |
| `ridge-board` | JSON array of up to 10 `{name, score, level, lanterns, at}` entries |
| `ridge-name`  | Last runner name entered                        |

All reads and writes are wrapped in try/catch so the game still works in private browsing or when storage is blocked. Corrupt board data is filtered out on load.

### Mobile and accessibility

- The viewport respects safe-area insets on notched phones and the HUD is padded accordingly.
- Portrait mode widens the camera field of view to 75° so the lanes stay visible.
- Touch input uses swipe distance and direction with a tap fallback for jumping. `touch-action: none` prevents the browser from scrolling or zooming.
- Buttons and the name field have visible focus outlines. Keyboard users can start with Enter, and the retry button is focused automatically on game over.
- `prefers-reduced-motion` disables CSS transitions.
- The page adapts its panel translucency to `prefers-color-scheme`.

## Tuning the game

The interesting knobs are all near the top of the script or in clearly named constants:

| What                       | Where                                   |
| -------------------------- | --------------------------------------- |
| Lane x-positions           | `LANES`                                 |
| Points per level           | `LEVEL_POINTS`                          |
| Level names and palettes   | `LEVELS`                                |
| Difficulty curve per level | `levelParams()`                         |
| Starting lives             | `MAX_LIVES`                             |
| Power-up colours/durations | `POWER`                                 |
| Power-up spawn rate        | the `0.11` in `spawnRow()`              |
| Lantern value              | the `25` in `currentScore()`            |
| Jump strength / gravity    | `vy = 11.5` in `jump()`, `32` in the physics step |
| Starting speed             | `speed = 13` in `reset()`               |
| Scoreboard size            | `BOARD_SIZE`                            |

## Browser support

Any modern browser with WebGL and Web Audio: current Chrome, Firefox, Safari and Edge on desktop, plus iOS Safari and Android Chrome. Shadows and the pixel-ratio cap are tuned so it runs smoothly on mid-range phones.

## Troubleshooting

**"Could not load the 3D engine"** on the start screen. The page could not find `three.min.js` next to it and could not reach the CDN. Make sure both files are in the same folder, or reconnect to the internet and reload.

**No sound.** Audio starts only after your first click or key press because browsers block autoplay. If your device is muted or the tab is muted, nothing will play.

**Blank page when opened from a file.** Some browsers restrict `file://` scripts. Serve the folder with a local HTTP server instead (see [Quick start](#quick-start)).

**Scoreboard is empty after reopening.** Private browsing windows discard `localStorage` when closed. Use a normal window to keep scores between sessions.

## Credits

- 3D rendering by [three.js](https://threejs.org/) (MIT).
- Typeface: [Bricolage Grotesque](https://fonts.google.com/specimen/Bricolage+Grotesque) via Google Fonts, with a system font fallback when offline.
- Character model, world, sound design and code are original.
