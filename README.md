# Touch Game Kit

A copy-and-reuse starter for simple HTML games that run **fullscreen** on Android
and iPhone/iPad touchscreens and install to the **home screen as an app icon**.

All the fiddly parts are already solved: hi-dpi canvas, the frame loop, touch input,
multitouch, accelerometer/tilt, no-zoom / no-scroll behaviour, pause-on-background,
offline play, and home-screen install. You write game logic in one clearly marked
section of `index.html`.

The demo that ships in the kit is a **tablet capability test** - press it with several
fingers, tilt the device, and swipe/pinch to confirm multitouch, the accelerometer, and
gestures all work before you build a real game on top. Replace that section with your own.

## What's in here

```
index.html              The shell + engine + your game (edit the fenced section)
manifest.webmanifest    Makes it installable on Android (name, icons, fullscreen)
sw.js                   Service worker - installability + full offline play
icon-192.png            App icon (Android)
icon-512.png            App icon (Android, also splash)
apple-touch-icon.png    App icon (iOS home screen, 180x180)
```

## The workflow

1. Copy this folder. Rename it per game.
2. Open `index.html`, edit the section marked **YOUR GAME GOES HERE**.
3. Push the folder to GitHub Pages (or any HTTPS host).
4. Open the URL on your phone and install it to the home screen.

That's it. Home-screen-installed = launches chromeless and fullscreen automatically,
on both platforms.

## Why it must be hosted over HTTPS

Service workers and Android's install prompt only work over HTTPS (or `localhost`).
Opening the file directly with `file://`, or over plain `http`, will run the game but
**won't** be installable. GitHub Pages gives you free HTTPS, which is why it's the
recommended host below.

## Hosting on GitHub Pages

GitHub Pages serves a repo's files as a static HTTPS site for free. The result is a URL
like `https://your-username.github.io/repo-name/` that you open on your phone and install.

### One-time setup (per game) - via the website, no command line

1. **Create the repo.** On github.com, click **New** (top-left **+** -> *New repository*).
   Give it a name (e.g. `touch-game`). Set it **Public** - Pages is free for public repos.
   Leave everything else default and click **Create repository**.
2. **Upload the kit files.** On the empty repo page, click **uploading an existing file**
   (or **Add file -> Upload files**). Drag in the **contents** of the `game-kit` folder -
   that is `index.html`, `manifest.webmanifest`, `sw.js`, and the three PNG icons.

   Important: upload the files themselves, **not** the `game-kit` folder. `index.html` must
   sit at the repo root, or the paths and service worker won't resolve.
3. Click **Commit changes**.
4. **Turn on Pages.** Go to the repo's **Settings** tab -> **Pages** (left sidebar).
   Under *Build and deployment*, set **Source = Deploy from a branch**, **Branch = main**,
   folder **/(root)**, then **Save**.
5. Wait ~1-2 minutes. Refresh the Pages settings page; it shows the live URL:
   `https://<your-username>.github.io/<repo-name>/`. Open that on your phone and install
   (see the next section).

### Updating after you edit a game

- **Website way:** open the changed file in the repo, click the pencil (Edit), paste your
  changes, **Commit**. Or **Add file -> Upload files** and drop the new versions in to
  overwrite. Pages redeploys automatically in a minute or two.
- **Git way (if you clone the repo locally):**

```
git add .
git commit -m "update game"
git push
```

- Either way, **bump the `CACHE` version in `sw.js`** (e.g. `touch-game-v1` -> `v2`) when
  you change files, or installed phones keep serving the old cached copy. See the
  *After you change anything* section below.

### GitHub Pages gotchas

- **Use relative paths.** This kit already does (`./index.html`, `sw.js`, `icon-192.png`).
  Avoid leading-slash paths like `/icon-192.png` - on Pages the site lives under
  `/repo-name/`, so a leading slash points at the wrong place.
- **First deploy can lag.** If you get a 404 right after enabling Pages, give it a couple of
  minutes and hard-refresh.
- **One game per repo is simplest.** You can host several under one repo in subfolders, but
  then each game's URL is `.../repo-name/game-folder/` and everything (manifest `start_url`,
  `scope`, the `sw.js` register path) must stay relative - which this kit already is.

### Testing locally first (optional)

To try a game on your phone over your own network before publishing, run a quick server on
your PC and open your PC's LAN IP from the phone:

```
python -m http.server 8000
```

LAN `http` still won't pass the install/offline checks - that needs HTTPS - but it's fine
for trying the gameplay itself.

## Installing on the phone

**Android (Chrome):** open the URL. Chrome shows an install prompt, or use
menu -> "Add to Home screen" / "Install app". Launches fullscreen with no browser bar.

**iPhone / iPad (Safari):** open the URL in **Safari** (not Chrome - only Safari can
install on iOS). Tap Share -> "Add to Home Screen". Launches in a chromeless standalone
view. iOS uses `apple-touch-icon.png` for the icon, not the manifest icons.

## Making a new game

The engine calls these methods on the `Game` object for you:

- `setup(W, H)` - run once; `W`/`H` are screen size in CSS pixels
- `update(dt)` - run each frame; `dt` is seconds since the last frame
- `draw(ctx, W, H)` - run each frame; draw with the 2D context in CSS pixels
- `onPointerDown(x, y, id)` / `onPointerMove` / `onPointerUp` - touch + mouse
- `onTilt(beta, gamma, alpha)` - device tilt in degrees (after `Motion.enable()`)
- `onMotion(ax, ay, az)` - acceleration including gravity (after `Motion.enable()`)
- `onResize(W, H)` - optional, fires on rotate/resize

Use `store.get(key, default)` and `store.set(key, value)` for high scores etc.
The canvas already scales for retina displays, so just draw in plain pixels.

### Multitouch

Every finger fires its own `onPointerDown` / `onPointerMove` / `onPointerUp` with a unique
`id`. Track them in a `Map` keyed by `id` (the demo does exactly this) to handle several
touches at once - pinch, two-thumb controls, multi-tap, etc.

### Accelerometer / tilt

Sensors are off until you call `Motion.enable()` **from inside a tap handler** - iOS 13+
only grants motion permission in response to a user gesture, which is why the demo has a
"Tap to enable motion" button. After that, `Game.onTilt` and `Game.onMotion` start firing.
Read `Motion.supported` and `Motion.enabled` to drive your UI.

Two different sensor events feed these:

- `Game.onMotion(ax, ay, az)` comes from `devicemotion` (`accelerationIncludingGravity`).
  This is the **reliable** one on Android/Chrome (Pixel etc.) and is what the demo uses to
  roll the ball - gravity's x/y components tell you which way the device is tilted, and it
  also spikes when you shake. Prefer this for tilt-controlled gameplay.
- `Game.onTilt(beta, gamma, alpha)` comes from `deviceorientation` (tilt angles in degrees).
  Cleaner angles, but Chrome on Android sometimes never delivers it - so treat it as a
  bonus, not your primary input. The demo shows it as a readout and falls back to it only
  if `devicemotion` is silent.

Gotchas if the readout stays blank / the ball won't move:

- **HTTPS required.** Sensors only fire over HTTPS (or `localhost`). A plain `http` LAN
  address discovers the sensor but delivers no data - publish to GitHub Pages and test the
  real URL, or install to the home screen.
- **Chrome site setting.** Chrome Android has Settings -> Site settings -> Motion sensors
  (or a per-site permission). If it's off, no events arrive even over HTTPS.
- The demo's card shows live `accel x/y/z` and `tilt β/γ`. If both stay at "waiting…",
  it's HTTPS/permission. If `accel` shows numbers, the ball will respond.

### Swipe / gesture

There's no dedicated swipe event - derive it in `onPointerUp` from the distance and time
since that pointer's `onPointerDown` (the demo flags a swipe when a finger travels >40px in
under 700ms and picks the dominant axis for direction). Pinch is just the changing distance
between two active pointers.

## Locking orientation

Edit `manifest.webmanifest` -> `"orientation"`: `"portrait"`, `"landscape"`, or `"any"`.
Android respects this for the installed app. iOS standalone does **not** enforce it - if
you need a fixed orientation on iPhone, design your `draw()` to handle both, or show a
"rotate your device" overlay when `W`/`H` are the wrong way round.

## After you change anything

The service worker caches files aggressively so the game loads instantly and works
offline. When you edit a file, bump the version string in `sw.js`:

```
const CACHE = "touch-game-v2";   // was v1
```

Phones pick up the new version on next launch. Without bumping it, they keep serving the
old cached copy.

## Renaming the app

Change the name in two places: `manifest.webmanifest` (`name` / `short_name`) and the
`apple-mobile-web-app-title` / `<title>` tags in `index.html`. Replace the three icon PNGs
with your own (keep the same filenames and sizes: 192, 512, and 180 for Apple).
