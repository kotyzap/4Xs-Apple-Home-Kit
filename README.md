# 4Xs Apple Home Kit

Expose an **Axis IP camera as a native Apple HomeKit accessory** — directly from the camera, with no hub, bridge box, or cloud service. It runs as a [CamScripter](https://camstreamer.com/camscripter) microapp on the camera itself, using [HAP-NodeJS](https://github.com/homebridge/HAP-NodeJS) to speak the HomeKit Accessory Protocol.

> **Version 2.2.0** · Vendor: Pavel Kotyza · Requires CamScripter ≥ 1.5.3

Add the camera in the Apple **Home** app and you get a live camera tile, snapshots, and two motion sensors — one for AXIS Video Motion Detection (VMD4) and one for Object Analytics (AOA) — including motion notifications and HomeKit automations.

<p align="center">
  <img src="docs/img/ui-light.png" alt="CamScripter settings UI (light)" width="48%" />
  <img src="docs/img/ui-dark.png" alt="CamScripter settings UI (dark)" width="48%" />
</p>
<p align="center">
  <img src="docs/img/IMG_4411.PNG" alt="Live view in the Apple Home app" width="48%" />
  <img src="docs/img/IMG_4416.PNG" alt="Motion detected in the Security view" width="24%" />
</p>

> A full screenshot walkthrough (pairing, live tiles, motion) is on the project page in [`docs/`](docs/index.html).

## Features

- **Native HomeKit IP camera** — pairs straight from the Home app via a QR code or PIN; no Homebridge, no extra hardware.
- **Live video** — pulls the camera's own H.264 RTSP stream and repackages it as SRTP for iOS. Defaults to zero-transcode passthrough (`-c:v copy`) so it's light on the ARTPEC CPU, with an optional `libx264` transcode fallback for compatibility.
- **Snapshots** — VAPIX JPEG snapshots with Basic **and** Digest auth (modern AXIS OS disables Basic by default), short-lived caching so HomeKit polling doesn't hammer the camera.
- **Two motion sensors** — VMD4 and AOA each publish their own HomeKit MotionSensor (“VMD Motion Sensor” / “AOA Motion Sensor”), so you get separate notifications and automations per detector, each with a configurable hold/debounce window so a flapping detector doesn't fire a burst of notifications.
- **Audio** — OPUS audio alongside the video stream (iOS requires an advertised audio config even for video-only cameras).
- **Stable identity** — a persisted HAP "MAC" so the accessory survives restarts and stays paired.

## How motion works (and what Apple does with it)

The app subscribes to the camera's VAPIX RuleEngine event stream and publishes VMD4 and AOA detections as **two separate HomeKit Motion Sensors**, each driving its own `MotionDetected` characteristic. This lets you enable notifications or build automations on one detector independently of the other.

Important: HomeKit is **stateful, not an event log**. It only tracks the current state (`Motion Detected` / `Clear`) and a "last activity" timestamp — it does **not** count events. To get notifications you must enable them per accessory in the Home app: long-press the sensor → settings → **Status and Notifications**. For an event timeline/history you need a HomeKit Secure Video camera (iCloud+) or a third-party app like Eve.

The `motion_hold_s` setting keeps each sensor's `MotionDetected` true for N seconds after its last active event, debouncing detector flapping so you don't get a storm of alerts.

## Installation

1. Build the package zip:

   ```bash
   npm install
   npm run zip:package      # generic CamScripter package
   # or, architecture-specific bundles that include the ffmpeg binary:
   npm run zip:arm64        # ARTPEC-8 / ARTPEC-9 (aarch64)
   npm run zip:armhf        # ARTPEC-6 / ARTPEC-7 (armv7)
   ```

2. In the camera UI, open **CamScripter → Apps → Upload package** and select the generated zip.
3. Open the app's settings UI and fill in the camera credentials and HomeKit options (see below).
4. In the iOS **Home** app: **+ → Add Accessory**, scan the QR code shown in the settings UI (or enter the PIN), then **More options** if it isn't auto-discovered.

## Configuration

Settings are validated with [zod](https://zod.dev) (`src/schema.ts`) and stored in `settings.json`. Defaults:

### Camera

| Setting | Default | Notes |
| --- | --- | --- |
| `protocol` | `http` | `http` or `https` |
| `ip` | `127.0.0.1` | usually localhost — the app runs on the camera |
| `port` | `80` | |
| `user` / `pass` | — | account needs at least viewer rights |

### HomeKit

| Setting | Default | Notes |
| --- | --- | --- |
| `name` | `Axis Camera` | accessory name shown in Home |
| `pincode` | `031-45-154` | pairing PIN, format `XXX-XX-XXX` |
| `port` | `51826` | HAP port |
| `live_stream` | `true` | `true` = SRTP live view, `false` = snapshot-only |
| `snapshot_resolution` | `1920x1080` | |
| `stream_resolution` | `1920x1080` | |
| `stream_fps` | `30` | 1–60 |
| `audio_enabled` | `true` | OPUS audio in the stream |
| `video_mode` | `copy` | `copy` (passthrough) or `transcode` (libx264 fallback) |
| `stream_bitrate_kbps` | `0` | 0 = camera default |
| `motion_vmd` | `true` | enable AXIS Video Motion Detection source |
| `motion_aoa` | `true` | enable AXIS Object Analytics source |
| `motion_hold_s` | `5` | keep "detected" N s after last event (0–300); debounce |
| `debug` | `false` | verbose logging (`DEBUG=HAP-NodeJS:*`, ffmpeg verbose) |

## Project layout

```
src/
  bootstrap.ts   Entry point — sets HAP debug env before hap-nodejs loads, then requires main
  main.ts        Builds & publishes the HomeKit accessory, wires camera + motion services
  schema.ts      zod settings schema and TSettings type
  snapshot.ts    VAPIX JPEG snapshot fetch with Basic/Digest auth + caching
  stream.ts      FfmpegStreamingDelegate — RTSP → SRTP live streaming (copy/transcode)
  motion.ts      MotionMonitor — VAPIX VMD4/AOA events → HomeKit MotionDetected
  peers.ts       PeerMonitor — mDNS discovery of sibling bridges on the LAN
  constants.ts   Snapshot timeouts, fixed HomeKit setup ID
  logger.ts      Timestamped console logger with a debug toggle
html/            Settings UI (index.html, index.js, QR code generation)
bin/             ffmpeg fetch + package zip helper scripts
docs/            GitHub Pages site (index.html + img/)
manifest.json    CamScripter package manifest
```

## Development

```bash
npm install
npm run build                 # tsc → dist/
npm run try                   # build + run locally with PERSISTENT_DATA_PATH=./localdata/
```

Saving settings sends `SIGINT`; CamScripter then restarts the app with the new config — so the process intentionally idles rather than exits when credentials are missing.

## Tech

TypeScript · [camstreamerlib](https://www.npmjs.com/package/camstreamerlib) v3 · [hap-nodejs](https://github.com/homebridge/HAP-NodeJS) · zod · bundled static ffmpeg (ARM).

## License

See repository. © Pavel Kotyza.
