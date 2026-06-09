# HTML Touch Game Kit - template reference

A reusable base for simple HTML games that run fullscreen on Android and iPad/iPhone
touchscreens and install to the home screen as an app icon. Touch, multitouch, swipe,
pinch, accelerometer/tilt, orientation lock, offline play and installability are all
already solved. A new game is built by replacing one fenced section of `index.html`.

This README is written first as a note-to-self for Claude on how to reuse this structure,
and second as deployment instructions for Andrew. The division of labour:

- Claude handles everything inside the files: writing the game, renaming the app,
  regenerating icons, and bumping the cache version.
- Andrew handles hosting and installing (GitHub Pages + Add to Home Screen).

---

## For Claude - building a game from this template

### File structure

```
index.html              Shell + engine + game. Edit only the fenced "YOUR GAME GOES HERE".
manifest.webmanifest    App name, icons, fullscreen display, orientation (Android install).
sw.js                   Service worker. Cache-first. Has a CACHE version string + ASSETS list.
icon-192.png            Android icon (manifest, "any maskable").
icon-512.png            Android icon (manifest, "any maskable").
apple-touch-icon.png    iOS home-screen icon, 180x180, solid background (iOS ignores alpha).
```

Keep all paths relative (`./index.html`, `sw.js`, `icon-192.png`). The published site lives
under `/repo-name/`, so a leading-slash path breaks.

### Where the game goes

Everything above the `ENGINE PLUMBING` banner in `index.html` is the `Game` object. Implement:

```
setup(W, H)                 once at start; W/H = screen size in CSS pixels
update(dt)                  each frame; dt = seconds since last frame
draw(ctx, W, H)             each frame; draw in CSS pixels (canvas is pre-scaled for retina)
onPointerDown(x, y, id)     touch/mouse down, coords already in canvas space
onPointerMove(x, y, id)     optional
onPointerUp(x, y, id)       optional
onTilt(beta, gamma, alpha)  optional; device orientation angles (degrees)
onMotion(ax, ay, az)        optional; accelerationIncludingGravity (m/s^2)
onResize(W, H)              optional; rotate/resize
```

Engine helpers available to the game:

```
store.get(key, default) / store.set(key, value)   localStorage, wrapped in try/catch
Motion.enable()         start sensors; MUST be called from a real DOM click (see below)
Motion.supported/enabled/secure/fileOrigin/permState/motionEvents/hasMotionData/...   diagnostics
Orientation.lockCurrent()   lock rotation to current orientation (Android; needs fullscreen/PWA)
```

The engine owns: hi-dpi canvas + resize, the delta-timed RAF loop, pointer normalisation,
pause-on-background, service-worker registration, the Motion/Orientation helpers, and the
DOM permission button. The game should not need to touch any of it.

### Claude's checklist for every new project or update

Do these automatically, without being asked - they are pure boilerplate:

1. Name. Set the app name in `manifest.webmanifest` (`name`, `short_name`, `description`),
   the `<title>`, and the `apple-mobile-web-app-title` meta tag - keep them consistent.
2. Icons. Regenerate `icon-192.png`, `icon-512.png`, `apple-touch-icon.png` themed to the
   game (keep the same filenames and sizes: 192, 512, and 180 solid-background for Apple).
   Keep the icon shape inside the central ~60% so Android maskable cropping doesn't clip it.
3. Cache version. Bump the `CACHE` string in `sw.js` (`game-v1` -> `game-v2` ...) on every
   change, or phones keep serving the old cached copy. Keep the `ASSETS` list in sync if any
   filenames change.
4. Paths stay relative. Never introduce a leading-slash path.

Andrew should not have to edit any of the above.

---

## Touch protocol (how input works, and the rules that keep it reliable)

- The engine listens for pointer events on the canvas and converts client coords to canvas
  CSS pixels before calling the `onPointer*` callbacks. Work in CSS pixels everywhere.
- Multitouch: each finger is a separate `pointerId` with its own down/move/up. Track active
  touches in a `Map` keyed by `id`. Never assume a single touch.
- Swipe: there is no swipe event. Derive it in `onPointerUp` from the start position/time
  recorded in `onPointerDown` (demo: >40px travel in <700ms, dominant axis = direction).
- Pinch: the changing distance between two simultaneously-active pointers. Capture the start
  distance when the second finger lands; scale = current / start.
- Pointerdown calls `preventDefault()` and the page uses `touch-action: none`,
  `overscroll-behavior: none`, and blocks `gesturestart`/`dblclick`. This is what stops
  scrolling, pull-to-refresh, pinch-zoom and double-tap-zoom from fighting game input.
  Keep these - removing them brings the mobile browser gestures back.

Touch and gestures work on any origin, including `file://` and inside in-app browsers.
Sensors do not (see below) - that asymmetry caused most of the confusion we hit.

## Sensor protocol (accelerometer / tilt) and the traps to avoid

How it works in this kit:

- `Motion.enable()` requests permission on iOS (`DeviceMotionEvent.requestPermission` /
  `DeviceOrientationEvent.requestPermission`) and auto-starts on Android. It then listens to
  `devicemotion` -> `onMotion(ax, ay, az)` and `deviceorientation` -> `onTilt(...)`, also
  starts the Generic Sensor API `Accelerometer` (better error reporting), and reads the
  Permissions API state. It only reports real data, never null-coerced zeros.
- Drive tilt gameplay from `onMotion` (`accelerationIncludingGravity`), not `onTilt`.

The traps we hit, and how this template avoids each one:

- file:// blocks sensors. A downloaded `index.html` opened directly is a `file://` origin.
  It reports `isSecureContext === true` (misleading) but is an opaque origin, so motion
  sensors are denied (`permission: denied`, `NotAllowedError`). Always serve over HTTPS.
  The diagnostic card now prints `origin: file:// needs https` so this is obvious.
- HTTPS is mandatory for both the service worker (installability/offline) and the sensors.
  GitHub Pages provides it for free.
- In-app WebView / iframe blocks sensors via permissions policy (shows `context: in-app
  WebView` or `embedded iframe`). Open in real Chrome, or launch the installed home-screen
  app. If embedding deliberately, the iframe needs `allow="accelerometer; gyroscope;
  magnetometer"`.
- iOS permission must come from a genuine DOM click. iOS Safari will not raise the motion
  prompt from a canvas pointer event. That is why the enable control is a real
  `<button id="enableBtn">` overlaid on the canvas, with its `onclick` calling
  `Motion.enable()`. Do not move that action back onto a canvas tap.
- Android `deviceorientation` is often empty even when motion works. `devicemotion` is the
  reliable source on Android, which is why the ball is driven by it.
- Device/browser settings that block sensors (not code): Android's "Sensors off" quick tile,
  Chrome's per-site Motion sensors permission, and iOS Settings -> Safari -> Motion &
  Orientation Access. The card's `permission:` / `sensor:` lines reveal these.

Orientation lock: `Orientation.lockCurrent()` enters fullscreen first (the Screen
Orientation API only locks while fullscreen or installed as a PWA), then locks to the
current orientation. Android/Chrome only; iOS ignores programmatic locks. For a permanent
lock on the installed app, set `"orientation": "portrait"` (or `"landscape"`) in the
manifest.

---

## For Andrew - hosting and installing

### Publish to GitHub Pages (free HTTPS)

1. Create a public repo on github.com.
2. Add file -> Upload files, and drop in the kit's contents - `index.html`,
   `manifest.webmanifest`, `sw.js`, and the three PNG icons. Upload the files themselves, not
   the folder, so `index.html` sits at the repo root. Commit.
3. Settings -> Pages -> Source: Deploy from a branch, Branch: main, folder: /(root). Save.
4. Wait ~1-2 minutes; the live URL appears: `https://<username>.github.io/<repo>/`.

Open that URL on the phone (not a `file://` copy, not an in-app browser link).

### Install to the home screen

- Android (Chrome): open the URL, then the install prompt or menu -> Install app / Add to
  Home screen. Launches fullscreen.
- iPad / iPhone (Safari): open the URL in Safari, Share -> Add to Home Screen. Launches in a
  chromeless standalone view. On iPad this is also the most reliable way to get the motion
  sensor working (it avoids desktop-mode quirks).

Installing to the home screen gives a top-level secure context, so sensors, fullscreen and
orientation all behave - which is the environment the finished game actually ships in.

### Updating after Claude changes the files

Re-upload the changed files (Add file -> Upload files, overwrite) or push with git. Claude
already bumps the `sw.js` cache version, so phones pick up the new build on next launch. If a
device looks stale, hard-refresh once.

### Quick troubleshooting

The accelerometer card prints a live diagnosis. Read it top to bottom:

- `origin: file:// needs https` -> you opened a local file. Use the GitHub Pages URL.
- `NOT https` -> not a secure origin. Use HTTPS.
- `permission: denied` / `sensor: NotAllowedError` -> blocked by context or settings; check
  the `context:` line (WebView/iframe vs top-level) and the device sensor settings above.
- `context: in-app WebView` -> open in real Chrome or the installed app.
- moving `accel x/y/z` numbers -> working.
