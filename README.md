# FaceLint-Real-Time-Face-Detection-and-Monitoring-System

# facelint

A macOS menu bar app that watches your webcam and nudges you when you touch your face, helping you break the habit for healthier skin.

Everything runs locally. No video ever leaves your Mac.

![platform](https://img.shields.io/badge/platform-macOS-black)
![python](https://img.shields.io/badge/python-3.9%20to%203.12-blue)
![license](https://img.shields.io/badge/license-MIT-lightgrey)

## Install

```sh
brew tap chandansgowda/tap
brew install facelint
facelint
```

On first launch, macOS asks for Camera permission. Approve it under System Settings > Privacy & Security > Camera, then relaunch.

## Features

- Real time face and hand tracking with MediaPipe.
- Configurable sensitivity, hold time, and minimum time between nudges.
- Ignores the chin and beard area, so resting your chin or stroking your beard while thinking does not raise a false alert. Detection is shape based, not colour based, so it works for any skin or beard colour.
- Sound and optional spoken cues, plus a desktop notification and a daily counter.
- Live preview window with on screen detection overlays for the face box, hand skeletons, and the active touch zone.
- Turns the camera off, including the indicator light, when you pause or step away from the keyboard.
- Light on resources: roughly 9% of one CPU core while actively monitoring, and close to zero when idle.

## Usage

Click the menu bar icon to open the menu.

| Section | Options |
| --- | --- |
| Top | Show camera preview, Pause or resume monitoring |
| Detection | Sensitivity, Hold time, Ignore chin and beard area |
| Alerts | Time between nudges, Alert cue (sound, voice, or both) |
| Privacy and power | Turn the camera off after a chosen period of inactivity |

Settings and your daily count are stored in `~/Library/Application Support/facelint/config.json`.

## How it works

MediaPipe locates your face and hands in each frame. A touch is flagged when a hand stays inside the face region for longer than the hold time. Sensitivity adjusts that region: Low requires a clear touch, High also reacts to hands near the edge. The chin and beard zone below the mouth can be excluded to avoid common thinking poses.

To stay light, the capture loop runs at a few frames per second while idle, speeds up only when a hand is in frame, skips the hand model when no face is present, and releases the camera when you pause or step away.

## Privacy

facelint never records, stores, or transmits anything. Frames are processed in memory and discarded immediately. The only network access is a one time download of the MediaPipe model files, about 8 MB, into `~/Library/Application Support/facelint/models` on first run.

## Development

```sh
git clone https://github.com/Darshansr/facelint
cd facelint
uv venv --python 3.12
uv pip install -e .
python -m tests.test_geometry   # pure logic tests, no camera needed
facelint
```

MediaPipe publishes wheels for Python 3.9 to 3.12, so facelint targets that range.

## License

MIT. See [LICENSE](LICENSE).





## * CLAUDE 
# facelint

macOS menu bar app that watches the webcam and nudges the user when they touch
their face, to build the habit of keeping hands off for healthier skin.
Everything runs locally; no video leaves the machine.

## Commands

```sh
# dev setup (Python 3.12 is required, see Gotchas)
uv venv --python 3.12
uv pip install -e .

python -m tests.test_geometry   # pure-logic tests, no camera needed
facelint                        # run the app (or: python -m facelint)

# regenerate icons after editing the generator
python -m facelint.resources.make_icons

# build a release tarball
uv build --sdist                # -> dist/facelint-<v>.tar.gz
```

There is no separate lint/format config; match the surrounding style (type
hints, module docstrings, `from __future__ import annotations`).

## Tech stack

- `rumps` for the menu bar (pulls in pyobjc / AppKit).
- MediaPipe **Tasks API** (`FaceDetector` + `HandLandmarker`) for detection.
- OpenCV (`opencv-contrib-python`, comes via MediaPipe) for camera capture.
- Pillow + AppKit `NSWindow` for the live preview window.
- macOS `afplay` (sound) and `say` (voice) for the cue. No extra audio deps.
- Idle detection via `ioreg` (no Quartz dependency).

## Module map

- `facelint/app.py` - `rumps.App` subclass. Owns the menu, state icons,
  two `rumps.Timer`s (UI refresh + preview refresh), and the preview window.
  All UI/AppKit work happens here on the main thread.
- `facelint/detector.py` - `FaceTouchDetector`. Runs the camera + MediaPipe
  loop on a background thread. Also holds the pure geometry helpers
  (`BBox`, `touch_region`, `hands_touch_face`, `draw_overlays`) which have no
  camera/MediaPipe dependency and are unit-tested.
- `facelint/config.py` - `Config`, a thread-safe JSON store at
  `~/Library/Application Support/facelint/config.json`, plus daily stats.
- `facelint/models.py` - downloads + caches the two MediaPipe `.task`/`.tflite`
  model files on first run (the arm64 wheel ships no models).
- `facelint/preview.py` - `render_dashboard()` (Pillow composite) and
  `PreviewWindow` (AppKit window).
- `facelint/resources/` - generated icons + `make_icons.py` generator.

## Detection design

Per frame: run `FaceDetector` (cheap) to get the face box + the mouth keypoint
(BlazeFace keypoint index 3). Run `HandLandmarker` only when a face is present.
A "touch" is any selected hand landmark falling inside the active region.

- **Sensitivity is a signed margin** around the face box (`SENSITIVITY_PROFILES`):
  - low `-0.10` (inset, fingertips only): hand must be clearly inside.
  - medium `0.0` (exact box, fingertips + joints).
  - high `+0.12` (grown, all 21 points): reacts to near/edge touches.
  Negative margin shrinks the box so edge grazes do not count. Do not flip the
  sign; growing the box on "low" was the original accuracy bug.
- **Chin/beard exclusion** (`ignore_chin`, default on): drop landmarks below the
  mouth line so resting the chin or stroking a beard does not alert. Falls back
  to the bottom `_CHIN_FALLBACK_FRACTION` of the box if no mouth keypoint.
  This is shape-based, not colour-based, so it is beard/skin-colour agnostic.
- **Hold + interval**: a touch must persist `hold_seconds` before it counts, and
  two alerts are always at least `nudge_interval_seconds` apart (no nagging).
- **Face memory**: a vanished face stays valid for `FACE_MEMORY_SECONDS` so a
  hand covering the face still counts even when detection briefly loses it.
- The frame is mirrored (`cv2.flip`) for a natural selfie view; detection is
  symmetric so this does not affect results.

## Performance and power

Designed to be light enough to run all day:

- **Adaptive frame rate**: `IDLE_FPS` (4) while nothing happens, `ACTIVE_FPS`
  (13) for `ACTIVE_LINGER_SECONDS` after a hand appears, `PREVIEW_FPS` (15)
  while the preview is open.
- **Skip the hand model when no face is present** (nothing to touch).
- **Downscale** frames to `PROCESS_WIDTH` (480) before inference.
- **Release the camera** (and its LED) entirely when paused or when the user is
  idle. Steady state is roughly 9% of one core active, near zero idle.

Measured CPU regressed badly at 30 fps (about 41%); keep the adaptive caps.

## Threading model

- The `rumps` run loop owns the main thread. Never touch AppKit (icon, menu,
  notifications, the preview window) off the main thread.
- The detector runs one background thread. It publishes plain state through a
  lock (`_State`) and exposes thread-safe properties.
- `app.update_ui` (every 0.2s) reads detector state and updates icon/menu, and
  pulls one-shot alerts via `detector.consume_alert()` so the notification fires
  on the main thread.
- The detector plays the sound/voice cue itself (subprocess, thread-safe) for
  low latency; only the desktop notification is deferred to the main thread.
- `_stop.wait(...)` is used for pacing so shutdown is immediate.

## Preview window

Optional calibration view (menu: "Show camera preview"). The detector annotates
the frame (`draw_overlays`: face box, translucent touch zone, hand skeletons)
only while `_preview` is set, and stores the latest frame. `app.update_preview`
composites the premium dashboard with Pillow (`render_dashboard`: header, camera
card, status pill, stat chips), converts to `NSImage`, and shows it in
`PreviewWindow`. Opening the preview forces the camera on (overrides
pause/idle); closing it (red button) is detected via `isVisible()` and turns
preview mode back off. The preview is heavier (about 26% CPU); it is a tool, not
a default.

## Config schema (`config.json`)

`monitoring` (bool), `sensitivity` (low|medium|high), `hold_seconds`,
`nudge_interval_seconds`, `ignore_chin` (bool), `pause_when_idle` (bool),
`idle_timeout_seconds` (0 = never), `cue` (sound|voice|both), `sound` (path),
`camera_index`, and `stats` {date, today, total}. Defaults live in
`config.py:DEFAULTS`; unknown keys are ignored on load and the file is written on
first run.

## Gotchas (read before debugging)

- **Python 3.12 only.** The machine default may be 3.13/3.14. MediaPipe ships
  wheels for 3.9 to 3.12; pin 3.12 via `uv` or `python@3.12`.
- **The arm64 MediaPipe wheel is Tasks-only.** `mediapipe.solutions` and
  `mediapipe.python` do not exist. Use `mediapipe.tasks.python.vision`. Models
  are not bundled, hence `models.py` downloads them.
- **Camera from a background thread needs** `OPENCV_AVFOUNDATION_SKIP_AUTH=1`
  (set at import time in `detector.py`). Without it OpenCV tries to request
  camera auth from a non-main thread and fails ("can not spin main run loop").
  macOS TCC still shows the normal permission prompt.
- VIDEO running mode needs **strictly increasing timestamps**; we derive ms from
  `time.monotonic()` and clamp with `max(prev+1, ...)`.
- `FaceDetector` returns the bounding box in **pixels**; normalize by frame size.
  Hand landmarks are already normalized.
- MediaPipe prints chatty native logs and a harmless "clearcut" upload error;
  `GLOG_minloglevel` is set to quiet most of it.
- rumps sizes icons to 20pt, so ship 40px PNGs for crisp Retina. Template icons
  must be black-on-transparent; the alert icon is non-template (colored).

## Packaging and release

Installed as an isolated `python@3.12` virtualenv via Homebrew. The formula does
`pip install` of the package (deps resolve from PyPI) rather than vendoring the
large MediaPipe tree as brew resources.

- Public repo: `Darshansr/facelint`.
- Tap: `Darshansr/homebrew-tap` (install: `brew tap Darshansr/tap &&
  brew install facelint`). The formula lives there and is mirrored in
  `Formula/facelint.rb` for reference; keep both identical.

Release steps when cutting a new version:

1. Bump `version` in `pyproject.toml` and `__version__` in `facelint/__init__.py`.
2. `uv build --sdist` and note `shasum -a 256 dist/facelint-<v>.tar.gz`.
3. `gh release create v<v> dist/facelint-<v>.tar.gz` on `Darshansr/facelint`.
4. Update `url` + `sha256` in both formula copies (repo + tap) and push the tap.
5. Verify: `brew update && brew install Darshansr/tap/facelint`.

## Conventions

- README and user-facing text: professional, concise, no emojis, no em-dashes.
- Keep detection geometry pure and unit-tested; keep AppKit on the main thread.
- Prefer config + adaptive behavior over hard-coded constants when user-facing.
