# 360° Live Player

A browser-based 360° live-stream player built for [Flussonic](https://flussonic.com) deployments.  
Designed for **Meta Quest 3** browser use, with staff-assisted sphere centering for assisted viewing.

## Features

- **WebRTC WHEP** playback from a Flussonic stream (sub-second latency)
- **Equirectangular 360** video mapped on an inside-out sphere (Three.js)
- **Staff controls**: Center, Left nudge, Right nudge
- **Head-tracking** via Device Orientation API (Quest 3 / mobile)
- **Mouse / touch drag** fallback for desktop
- **Gamepad / controller** support (dual-stick + buttons)
- **LocalStorage** yaw offset persistence across page reloads
- **Fixed-interval auto-reconnect** (no exponential back-off)
- **Stall watchdog**: if video freezes while connected, player flushes buffers and reconnects
- **Short URL aliases** for QR-friendly links (same behavior, fewer characters)
- **Assistive joystick tuning** via URL params (deadzone/speeds/pitch limits)
- **Configurable via URL query parameters**

---

## Quick start

Open the player in any browser that can reach the Flussonic server:

```
https://<your-flussonic-host>/360player.html?stream=<stream_name>
```

Replace `<stream_name>` with your Flussonic stream name. The player will
connect to `https://<your-flussonic-host>/<stream_name>/whep` automatically.

---

## URL parameters

| Parameter | Short alias | Default | Description |
|-----------|-------------|---------|-------------|
| `stream`  | `s`   | `live`  | Flussonic stream name. Derives WHEP URL as `<origin>/<stream>/whep`. |
| `whep`    | `w`   | *(derived)* | Override the full WHEP URL, e.g. `?whep=https://server/mystream/whep` |
| `host`    | `h`   | *(current origin)* | Override just the base host, e.g. `?host=https://192.168.1.50:8080` |
| `yaw`     | `y`   | `0`     | Initial yaw offset in degrees. Loaded from localStorage if previously saved. |
| `nudge`   | `n`   | `15`    | Degrees to rotate per Left / Right button press. |
| `reconnectMs` | `r` | `350` | Fixed reconnect delay in ms between retries. |
| `iceWait` | `i`   | `800` | Max ICE gather wait in ms before WHEP POST (lower can reduce startup delay). |
| `stallMs` | `st`  | `1200` | If video time stops moving this long, force reconnect. |
| `maxBufferSec` | `b` | `1.2` | If buffered lead grows beyond this, reconnect to get back to live edge. |
| `deadzone` | `dz` | `0.20` | Gamepad deadzone for all stick axes. |
| `lookSpeed` | `ls` | `0.035` | Left-stick look speed (radians per frame near full deflection). |
| `yawSpeed` | `ys` | `0.35` | Right-stick X sphere yaw speed (degrees per frame near full deflection). |
| `trimSpeed` | `ts` | `0.021` | Right-stick Y vertical trim speed (radians per frame). |
| `assistPitchMax` | `ap` | `60` | Max up/down assistive pitch in degrees while head-tracking is active. |

### Examples

```
# Stream named "quest360" on the same host
https://flussonic.example.com/360player.html?stream=quest360

# Override WHEP URL completely
https://flussonic.example.com/360player.html?whep=https://10.0.0.5:8080/live/whep

# Start with 45° yaw and 10° nudge steps
https://flussonic.example.com/360player.html?stream=quest360&yaw=45&nudge=10

# Deterministic live-demo recovery profile
https://flussonic.example.com/360player.html?stream=quest360&reconnectMs=250&iceWait=500&stallMs=900&maxBufferSec=0.9

# Same profile using short aliases (better for QR)
https://flussonic.example.com/360player.html?s=quest360&r=250&i=500&st=900&b=0.9

# Accessibility-tuned profile (precise control for assisted demos)
https://flussonic.example.com/360player.html?s=quest360&r=250&i=500&st=900&b=0.9&dz=0.16&ls=0.030&ys=0.28&ts=0.018&ap=45
```

### FQDN shortening for ops/QR

Yes. The best approach is to expose the player on a short hostname (for example `q.example.com`) that points to your Flussonic host. Then use short aliases in the query string.

Example:

```
https://q.example.com/360player.html?s=quest360&r=250&i=500&st=900&b=0.9
```

---

## Deploying to a Flussonic instance

Flussonic serves static files from its web root directory.
Copy the player file there and it will be accessible over HTTP/HTTPS on the same host as your streams.

```bash
# Copy the file to the Flussonic web root
scp 360player.html user@your-flussonic-host:/opt/flussonic/wwwroot/

# Or on the server directly
cp 360player.html /opt/flussonic/wwwroot/
```

Then open:

```
https://<your-flussonic-host>/360player.html?stream=<stream_name>
```

> **Note — HTTPS:** WebRTC and Device Orientation both require a [secure context](https://developer.mozilla.org/en-US/docs/Web/Security/Secure_Contexts).
> Use HTTPS on your Flussonic instance for full functionality (especially on Quest).

### Hosting Three.js locally (optional)

By default the player loads Three.js from jsDelivr CDN. If the Flussonic server
has no internet access, download Three.js first:

```bash
curl -L https://cdn.jsdelivr.net/npm/three@0.163.0/build/three.min.js \
     -o /opt/flussonic/wwwroot/three.min.js
```

Then edit the `<script src="…three.min.js">` line in `360player.html` to use
a relative path:

```html
<script src="/three.min.js"></script>
```

---

## Staff centering workflow

The player separates two independent rotations:

| Rotation | Controlled by | How |
|----------|--------------|-----|
| **Camera yaw / pitch** | The viewer's head movement | Device Orientation API (Quest gyroscope) or mouse/touch drag |
| **Sphere yaw offset** | Staff | Left / Center / Right buttons |

### Button reference

| Button | Action | Keyboard | Gamepad |
|--------|--------|----------|---------|
| **⊕ Center** | Reset sphere yaw to the initial value (`?yaw=` param, default 0°) | `Space` or `C` | Button A (index 0) |
| **◀ Left** | Rotate sphere left by `nudge` degrees | `←` or `A` | Button X (index 2) |
| **Right ▶** | Rotate sphere right by `nudge` degrees | `→` or `D` | Button B (index 1) |
| **Reset Assist** | Zero only assistive view offset (does not change sphere yaw) | `R` | Button Y / Triangle (index 3) |
| **🔊** | Toggle audio mute | — | — |

Joystick mapping (accessibility mode):
- Left stick X/Y: move the viewing direction (camera yaw/pitch) to place the focus spot. On Quest, this is applied as an assistive offset on top of head tracking.
- Right stick X: rotate sphere yaw for staff-assisted alignment.
- Right stick Y: fine vertical focus trim (assist pitch) for comfort.

For precision and comfort, stick input uses a response curve so small movements near center are easier to control.

### Typical demo procedure

1. Start stream on Flussonic (RTMP ingest or any source).
2. Open `360player.html?stream=<name>` on the Quest 3 browser.
3. Press **Start** (required once for orientation permission on iOS; Quest skips this).
4. Put the headset on the visitor.
5. Staff observes the output (e.g. a mirrored TV feed) and uses **Left / Right** to
   rotate the sphere until the interesting content is in front of the viewer.
6. Press **Center** at any time to snap back to the default orientation.

The yaw offset is **saved automatically** to `localStorage` keyed by stream name,
so it persists across page reloads for the same stream.

---

## Flussonic WHEP endpoint

The Flussonic WHEP URL format is:

```
https://<flussonic-host>/<stream_name>/whep
```

You can verify the stream works first with the built-in Flussonic player:

```
https://<flussonic-host>/<stream_name>/embed.html?proto=webrtc
```

If that works, the custom player will work too.

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Black sphere / no video | Check stream is live; verify WHEP URL in browser network tab |
| `HTTP 404` on WHEP URL | Wrong stream name or Flussonic WebRTC not enabled |
| No head-tracking on Quest | Make sure page is loaded over HTTPS; tap **Start** to grant permission |
| Audio not playing | Tap the 🔊 button; browsers require user interaction for audio |
| CORS errors | Host the HTML on the **same** Flussonic server (same origin as WHEP URL) |
| Player reconnects constantly | Check Flussonic license / stream health; check browser console for ICE errors |
| Connected but frozen frame | Player auto-recovers when `stallMs` is reached; lower `stallMs` or `reconnectMs` for faster recovery |
| Delay grows over time | Lower `maxBufferSec` (e.g. `0.8`) so player re-syncs to live edge sooner |
| Stick feels too sensitive | Increase `deadzone`, or reduce `lookSpeed` / `yawSpeed` / `trimSpeed` |
| Can’t keep focus comfortably | Lower `assistPitchMax` (e.g. `45`) and use Reset Assist (`R` / button 3) between visitors |

---

## Architecture overview

```
Camera (RTMP/SRT) → Flussonic server → WHEP endpoint
                                            │
                    Browser (Quest 3) ──────┘
                    RTCPeerConnection → <video> element
                                            │
                    Three.js scene ─────────┘
                    SphereGeometry (inside-out, 500 units radius)
                    VideoTexture (equirectangular)
                    Camera at origin, head-tracked via DeviceOrientation
                    Sphere yaw offset controlled by staff buttons
```

---

## File listing

| File | Description |
|------|-------------|
| `360player.html` | Self-contained player — copy to Flussonic `wwwroot` |
| `README.md`      | This documentation |
