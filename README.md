# VisionPlay

A native Apple Vision Pro client for **PS5 Remote Play** (PS4 also supported by the core),
built on the open-source [chiaki-ng](https://github.com/streetpea/chiaki-ng) Remote Play
protocol implementation.

```
PS5 ──(Takion/UDP, local network)──► chiaki-ng core (C) ──► VideoToolbox HW decode (H.264/HEVC)
     ──► Metal: Y'CbCr→RGB ──► MetalFX Spatial (or Lanczos-3) ──► CAS sharpening ──► window or curved RealityKit screen
     ──► Opus → AVAudioConverter → AVAudioEngine (bounded ~30–100 ms)
DualSense ──► GameController ──► Remote Play controller state (buttons, sticks, triggers, touchpad, optional motion)
```

> **Status: compiles in Xcode (visionOS simulator SDK), not yet run.** Nothing has been
> launched, unit-tested in the simulator, or tested against a PS5 yet. See *Verification
> status* at the end before relying on any feature.

---

## 1. Requirements

* Apple Silicon Mac with **Xcode 16 or later** and the visionOS SDK/simulator installed
  (the project uses Xcode 16 "synchronized folders", object version 77).
* Apple Vision Pro on visionOS 2.0 or later (deployment target 2.0, required for
  RealityKit `LowLevelTexture`).
* A free or paid Apple ID added in Xcode › Settings › Accounts (a personal team is enough
  for side-loading; no App Store distribution or paid services are needed).
* PS5 with Remote Play enabled, on the same local network. Wired Ethernet on the PS5 and
  5 GHz/6 GHz Wi-Fi for the Vision Pro are strongly recommended.

## 2. Project layout

```
VisionPlay.xcodeproj         3 targets: VisionPlay (app), ChiakiCore (static lib), VisionPlayTests
Config/                      Info.plist (merged with generated keys), Swift bridging header
ChiakiCore/
  chiaki/                    chiaki-ng lib sources (unmodified except: aia.h trimmed, base64.c/ecdh.c renamed
                             to chiaki_base64.c/chiaki_ecdh.c to avoid clashing with mbedtls object names)
  chiaki/proto/              takion.pb.c/.h generated with nanopb from lib/protobuf/takion.proto
  bridge/vp_bridge.{h,c}     C API used by Swift (discovery, registration, LAN + PSN sessions, input, mic, haptics)
  nanopb/  fec/  mbedtls/    vendored dependencies (nanopb, Jerasure + gf-complete, Mbed TLS 2.28.10)
Opus/                        libopus 1.5.2 (float build) — microphone encoder (Opus target)
Curl/                        libcurl 8.11.1 (HTTP(S)+WebSockets only, Mbed TLS) — PSN APIs (Curl target)
JsonC/  MiniUPnPc/           json-c 0.18 and miniupnpc 2.2.8 — used by chiaki-ng's PSN hole punching
VisionPlay/
  App/ UI/ Models/ Services/ RemotePlayCore/ ChiakiBridge/ Streaming/ VideoDecoder/
  MetalRenderer/ Upscaling/ Shaders/ Audio/ Controllers/ Security/ Immersive/ Settings/ Diagnostics/
VisionPlayTests/             XCTest suites (logic + GPU pipeline)
Licenses/                    licence texts and the exact chiaki-ng commit used
```

All C code is compiled by Xcode directly (no CMake, OpenSSL or libevent). Crypto and TLS use
Mbed TLS via chiaki's `CHIAKI_LIB_ENABLE_MBEDTLS` path. Game audio is decoded by Apple's
AudioToolbox; libopus is only used to encode the microphone, because the PS5 requires fixed
40-byte Opus packets. Patches to upstream code, all marked "VisionPlay" in the source:
`holepunch.c` uses chiaki-ng's existing select() path (as on Android) instead of libevent,
`Curl/lib/url.c` takes its default CA bundle from the app (`Resources/cacert.pem`), and
`base64.c`/`ecdh.c` are renamed to avoid clashing with Mbed TLS object names.

## 3. Build

1. Open `VisionPlay.xcodeproj`.
2. Select the **VisionPlay** target › Signing & Capabilities › choose your Team. If Xcode
   says the bundle id is taken, change `com.visionplay.VisionPlay` to something unique.
3. Scheme **VisionPlay**, destination **Apple Vision Pro** simulator › Product › Build (⌘B).
4. Tests: Product › Test (⌘U) on the simulator.

Build output goes to Xcode's default DerivedData location. Do not move it into an
iCloud-synced folder (Desktop/Documents with iCloud): iCloud adds extended attributes that
make `codesign` fail.

## 4. Install on Apple Vision Pro

1. On the Vision Pro: Settings › General › Remote Devices; on the Mac, Xcode › Window ›
   Devices and Simulators, and pair the headset (same Wi-Fi).
2. Enable Developer Mode on the headset when prompted (Settings › Privacy & Security).
3. Choose the Vision Pro as the run destination and press Run.
4. With a free Apple ID the app expires after 7 days; just run it again from Xcode.
5. On first launch allow **Local Network** access — discovery and streaming need it.

## 5. Pair a PS5

1. PS5: Settings › System › Remote Play › **Enable Remote Play**.
2. VisionPlay: **Find PS5** (or **Manual IP Address**). The console appears under *On This Network*.
3. Tap **Register** and get the **PSN account ID** with **Sign in to PlayStation Network**,
   using any of three options:
   * **On This Headset**: Sony's own sign-in page in an embedded private browser.
   * **QR Code**: scan with your iPhone, sign in there, copy the final page's address
     (`https://remoteplay.dl.playstation.net/remoteplay/redirect?code=…`) and paste it on the
     Vision Pro (Universal Clipboard carries it across with the same Apple Account).
   * **Enter Code**: open/share the sign-in link on any browser, then type or paste the code
     (the part after `code=`) or the whole address.
   * Or type a Base64 account ID you already have (chiaki-ng, PXPlay) into the field.
   Sign-in codes are single-use and expire after a few minutes.

   **How sign-in works** (on the sign-in screen, Register, Internet Play and Diagnostics)
   explains each step and shows a live log of what the app did, with codes and tokens
   shortened so they can't be reused. Stored on the device, in the Keychain only: the account
   ID and, after an Internet Play sign-in, the refresh token. Your password is never seen.
4. PS5: Settings › System › Remote Play › **Pair Device** shows an 8-digit code. Enter it
   and tap **Register** within about five minutes.
5. The console moves to *Registered Consoles*. Tap **Connect**.

If a Remote Play login PIN is set on the PS5, VisionPlay asks for it when connecting.

### Why discovery sweeps the subnet

visionOS only allows UDP broadcast/multicast with Apple's restricted
`com.apple.developer.networking.multicast` entitlement, which personal teams cannot get.
VisionPlay therefore sends **unicast** Sony discovery probes to every address on the local
/24 (larger subnets are clamped to the /24 around the headset). A broadcast probe is also
tried in case the entitlement is ever present. If the console is on another subnet, use
**Manual IP Address**. Registration and streaming are unicast and are unaffected.

## 6. Rendering and upscaling

| Mode | Pipeline | Sharpening |
|---|---|---|
| Performance | One fragment pass: Y'CbCr→RGB + bilinear scale straight to the output | none |
| Balanced | Y'CbCr→RGB compute → separable Catmull-Rom (bicubic) compute → compose | user % × 0.5 |
| **Quality (default)** | Y'CbCr→RGB → **MetalFX Spatial Scaler** (`.perceptual`) → compose with CAS | **35 %** |

* Default output target is 3840×2160 from a 1920×1080 stream. **This is GPU upscaling, not
  native 4K** — the PS5 sends at most 1080p over Remote Play.
* If `MTLFXSpatialScalerDescriptor.supportsDevice` is false (e.g. in the simulator), Quality
  falls back to a separable **Lanczos-3** compute scaler. Diagnostics shows the technique
  actually used and the real drawable size.
* **Sharpening**: contrast-adaptive sharpening (after AMD FidelityFX CAS) in the final pass.
  The user amount maps linearly to the CAS peak denominator: 0 % → 8.0, 100 % → 4.5,
  **35 % → 6.775** (`UpscalerConfiguration.casPeakDenominator`). The negative lobe is
  scaled by local contrast, which limits halos and stops macro-blocks being amplified.
* Frames are decoded into IOSurface-backed `CVPixelBuffer`s and wrapped as Metal
  textures via `CVMetalTextureCache` — no CPU copies or GPU readbacks.
* Queueing: at most 2 frames on the GPU plus 1 waiting; a newer frame replaces the
  waiting one, so latency cannot build up.
* No temporal upscaling and no frame generation, as specified.

## 7. Display modes

* **Window (flat)**: a resizable visionOS window with a `CAMetalLayer`; the drawable is
  sized to the output target and the image is letterboxed so the aspect ratio is kept.
* **Theater** (ImmersiveSpace, mixed): a flat or curved screen (cylinder section, 0–100°
  arc) built from a generated mesh. **Screen sizes:** 55″, 65″, 75″, 85″, 105″, 125″,
  175″, 205″ and **IMAX** (22 m wide), measured diagonally like a TV, plus a custom slider up
  to 30 m. **Viewing distance** is yours to set (0.75–40 m); "Use recommended" places the
  screen so it fills about 40° of your view. Big screens are raised so their bottom edge stays
  above the floor, and the control panel stays within reach. In the windowed mode the size is
  whatever you resize the visionOS window to. Metal writes straight into a RealityKit
  `LowLevelTexture`. Size, distance, height and curvature are adjustable; drag the screen
  to move it vertically. The *Dark* option surrounds you with a dim environment sphere.
  The environment is static geometry, separate from the video path.

## 8. Controller

DualSense via GameController: sticks (with radial dead zone), D-pad, face buttons, L1/R1,
analog L2/R2, L3/R3, Options, Create, PS, touchpad click and two-finger touch positions.
Home/Options/Create are set to `preferredSystemGestureState = .disabled` so the app gets them.
Console feedback: rumble (CoreHaptics on the controller), light bar colour, adaptive
triggers (feedback / weapon / vibration decoded to `GCDualSenseAdaptiveTrigger` modes).
Motion forwarding is behind an *experimental* setting (axis mapping unverified).
**Controller Test** shows live input and the exact values sent to the console.

## 9. Licensing

* **chiaki-ng is licensed AGPL-3.0-only with an OpenSSL linking exception.** VisionPlay links
  it statically, so **VisionPlay as a whole is a derivative work and must be distributed under
  AGPL-3.0** if you ever share it (binaries or a hosted service). For personal use on your
  own device there is no obligation to publish, but if you give the app to anyone you must
  provide the complete corresponding source (including your changes) under AGPL-3.0.
* AGPL/GPL terms are generally considered incompatible with App Store distribution terms;
  this project is intended for personal side-loading only.
* Bundled: Mbed TLS (Apache-2.0), nanopb (zlib), Jerasure and gf-complete (BSD-3-Clause),
  libopus (BSD-3-Clause), libcurl (curl licence, MIT-style), json-c (MIT), miniupnpc
  (BSD-3-Clause), and the Mozilla CA certificate list as packaged by certifi
  (`Resources/cacert.pem`, MPL-2.0).
  Licence texts are in `Licenses/`; the upstream chiaki-ng commit is recorded in
  `Licenses/CHIAKI_NG_UPSTREAM_COMMIT.txt`.
* VisionPlay's PSN sign-in uses the same public Remote Play OAuth client id/secret that
  chiaki-ng and other community clients embed. "PlayStation", "PS5" and "DualSense" are
  Sony trademarks; this project is unaffiliated with Sony.
* The CAS shader is an independent implementation of the published algorithm (AMD
  FidelityFX CAS is MIT-licensed).

## 10. Version 0.2–0.3 features

* **HDR (HEVC HDR codec):** the frame's colour tags (BT.2020 / PQ) are read from VideoToolbox.
  Upscaling and sharpening run on the PQ-encoded image; the final pass converts PQ to
  luminance (203 nits = SDR white). In the window the Metal layer switches to EDR
  (`rgba16Float`, extended-linear BT.2020) and highlights roll off smoothly at the "HDR Peak
  Brightness" setting (default 2× SDR white). Theater mode (RealityKit) cannot show EDR, so
  HDR is tone-mapped to SDR and converted to BT.709 there.
* **DualSense haptics:** the PS5's per-grip haptics stream (3 kHz stereo PCM) drives
  CoreHaptics players on the left and right grips. visionOS cannot play raw waveforms on the
  DualSense, so each 10 ms block becomes an intensity (RMS) and sharpness (frequency) value:
  timing and strength match; the exact texture is simplified. Controllers without per-grip
  actuators fall back to rumble.
* **Microphone / party chat:** the Mic button in the gameplay window. Capture uses Apple's
  voice processing (echo cancellation, so game audio isn't sent back), 48 kHz stereo,
  encoded by libopus at the exact constant bitrate the protocol needs (40-byte packets,
  verified on 300 test frames).
* **Internet Play (PSN):** console menu › Internet Play… → sign in to PSN → link the console;
  then console menu › Connect over the Internet. Uses chiaki-ng's hole punching (PSN session
  + STUN; UPnP is attempted but usually unavailable because visionOS blocks the multicast it
  needs without Apple's entitlement). The PSN refresh token is stored in the Keychain.
* **Polish:** layered visionOS app icon; wake-from-rest waits for the console with a progress
  bar (re-sends wake packets every 10 s, up to 45 s); Theater screen position (sideways and
  height) is saved, with a Recenter button.

### Connectivity (v0.3)

* **Smart Connect:** Connect uses the home network when the PS5 answers there (known
  address probed first, usually under a second), otherwise Internet Play if linked.
* **Instant recovery:** the first reconnect is immediate, later ones back off (1, 2, 4… s,
  6 tries). The Vision Pro's own network is watched, so a Wi-Fi drop, roam or network
  change reconnects straight away instead of waiting for a timeout.
* **Adaptive Quality:** six seconds of ≥5 % loss or ≥80 ms round trip lowers quality for
  the next session (reduced bitrate → 720p → 540p); six seconds of ≥12 % loss reconnects
  immediately one step lower; two clean minutes step back up. Remembered per network.
* **Connection Test** (console menu): 30 timed discovery probes → round trip, jitter,
  loss, a grade, specific advice and one-tap suggested settings.
* **Headset off/on:** taking the headset off ends the stream cleanly; putting it back on
  resumes it (waking the PS5 if it went to rest).
* **Router port forwarding (Internet Play):** optional fixed UDP ports (default 9320/9321)
  to forward on a router, since visionOS blocks UPnP.

### Sound, motion, capture, cinema and hand controls (v0.4)

* **Sound from the screen** (on by default): the game's left/right channels are placed at
  the edges of the virtual screen with visionOS spatial audio (HRTF, head tracking). In the
  window they sit at a typical window size; in Theater they follow the screen size and distance.
* **Room acoustics** (off by default, subtle): reverb sized to the screen — living room under
  90″, large room under 130″, cinema hall up to 300″, auditorium for IMAX.
* **Virtual surround** (off by default): soft rear ambience made from the stereo difference
  signal (−11 dB, 12 ms delay). It is an effect, not the game's real surround channels —
  Remote Play only sends stereo.
* **PS5 3D Audio guide** in Settings › Audio: how to turn on Tempest 3D Audio for headphones,
  and why to turn off "Sound From the Screen" when you use it.
* **Smooth Motion** (on by default): frames are presented on the display's refresh (requests
  120 Hz) for an even 60 fps cadence; adds ≤ 1 refresh of delay. The HUD shows the measured Hz.
* **Screenshots and Instant Replay**: Screenshot button saves the current frame (JPEG) to
  Photos. With Instant Replay on, the last ~30 s are kept (hardware H.264 re-encode, 1 s
  keyframes, plus game audio as AAC) and Save 30 s — or hold **Create** and press **Options**
  — writes an MP4 to Photos. Add-only Photos permission.
* **Cinema Room** (Theater, off by default): a procedural auditorium (walls, ceiling, stage,
  rows of red seats) sized to the screen, tinted by the picture's average colour ~8×/s.
  Three meshes, unlit — negligible GPU cost.
* **Hand Controls**: an ornament beside the gameplay window with a pinch-drag touchpad
  (quick pinch = click), PS, Create, Options, D-pad and face buttons. Merged with a physical
  controller, so both work together.

### Picture, latency, profiles, Siri and reports (v0.5)

* **Compression clean-up** (Settings › Motion; Off / Light (default) / Strong): a pass at
  source resolution that smooths blocks and banding in flat areas before upscaling; edges and
  texture exceed its threshold and are left alone. Skipped in Performance and Low-Latency.
* **HDR in Theater (experimental, off)**: passes HDR highlights into the Theater screen's
  float texture instead of tone-mapping. Whether RealityKit shows them above SDR white in an
  immersive space is unverified — compare on the headset and keep whichever looks better.
* **Low-Latency Mode**: 60 FPS, no display pacing, single-pass scaling (no clean-up or
  sharpening), 40 ms audio buffer.
* **Lag estimate** (HUD + Diagnostics): pad (4 ms, assumed) + network RTT (measured) + PS5
  game/encode (one frame, assumed) + decode (measured) + display (measured + half a refresh).
  Remote Play has no input echo, so the console's part can't be timed; use it to compare settings.
* **Game profiles**: save screen/sound/picture settings under a game name; apply from Home ›
  Game Profiles or the stream window's Profile menu. With PSN sign-in, VisionPlay asks PSN's
  presence endpoint once a minute which game is running and applies the matching profile —
  best effort and untested (the Remote Play token may not be allowed to read presence).
* **Several PSN accounts on one PS5**: pair again while signed in as the other account and
  give it a player name; each registration is its own Home entry ("PS5 · Guest").
* **Siri and Shortcuts**: "Play PS5 in VisionPlay", "Play PS5 in Theater with VisionPlay",
  "Stop VisionPlay"; the Play shortcut takes an optional console and Theater switch.
* **Pin Theater screen to the room** (on): dragging the screen saves an ARKit world anchor, so
  Theater reopens in the same spot in your room, also after a restart. Recenter un-pins.
* **Share Problem Report** (Diagnostics): settings, last stream statistics, lag breakdown,
  log and any crash/hang reports MetricKit delivered after a crash, as one text file.

### Picture (v0.6)

* **Picture modes**: Standard (accurate, default), Vivid, Cinema, Game, Dark Room, plus Custom
  sliders for brightness, contrast, colour, warmth and shadow lift. Applied in the final
  compose pass (no extra GPU pass), in every mode including Low-Latency, and saved in game profiles.
* **Shadow lift**: a luma gamma that fades out by the mid-tones — dark areas open up, blacks
  stay black, highlights are untouched.
* **HDR tone-mapping**: ITU-R BT.2390 EETF (Hermite knee in PQ space) from the game's peak to
  the display's peak, so highlight detail rolls off instead of clipping. The game's peak is read
  from HDR10 metadata (MaxCLL, else mastering display); 1000 nits when the PS5 doesn't send it.
* **Smart sharpening** (on): full strength on edges, ~35 % on flat areas, so skin and skies
  don't turn gritty. **Text clarity** (30 %): extra sharpening on hard, colourless edges —
  menus, maps, subtitles. Not in Performance / Low-Latency.
* **Black bars**: letterboxed cutscenes are detected (symmetric bars of 4–30 %, stable for
  ~1.2 s); Pure Black (default) or Zoom to Fill (sides crop). Leave turns it off.
* **Compare**: stream window button; left half is the raw stream (bilinear, no clean-up or
  sharpening), right half is processed, with a thin divider.
* **Picture quality meter** (HUD + Diagnostics): measured video bitrate and bits per pixel,
  graded Excellent / Good / Fair / Heavily compressed, with advice on whether softness is from
  the network (PS5 sending < 70 % of the requested bitrate) or from scaling.

### Takion receive buffer (v0.6.3)

The video stream's receive window and socket buffer are 4 MB (the value the AGPL-3.0 Pylux
iOS client uses) instead of chiaki-ng's 100 KB, so bursts of a large video frame on Wi-Fi aren't
dropped. If the system refuses 4 MB, VisionPlay halves it until it is accepted (down to the
original 100 KB) and tells the PS5 the size it actually got. See `ChiakiCore/chiaki/src/takion.c`.

### Connection help (v0.7)

* **Live connection steps** on Home while connecting: Contacting the PS5 → PS5 accepted the
  session → Control link open → Network test → Video arriving, each ticked or crossed, with the
  error code at the step that stopped. Followed from the chiaki core's own log lines.
* **Automatic safe retry**: a stream that fails to start is retried once at 720p, H.264,
  5 Mbps (after looking for the PS5 again in case its address changed). If that works, Home says
  so and your normal settings are left unchanged.
* **Pair Again**: when the PS5 rejects the saved keys (VP-109), a button opens pairing for
  that console.
* **Test Connection**: now also checks TCP 9295, and **Full Stream Test** connects for real,
  shows the steps, waits for video, then disconnects with a pass/fail and the code.
* **Copy Log** (Home, Diagnostics): copies the last connection attempt's log to paste in a message.

### Controller and vibration on visionOS 26 (fix)

visionOS 26 keeps a paired DualSense for the system (X = tap, sticks = scroll, PS = Home)
unless the app claims it. The playback window and Custom Screen now use
`handlesGameControllerEvents(matching: [.gamepad], withOptions: .receivesEventsInView(false))`
(`Controllers/GameControllerOwnership.swift`), so the controller plays the game and the PS5's
vibration can reach it. The haptic engines also no longer auto-shut down, and restart when the
system stops or resets them (at most once every 2 s). Check: Settings › Controller › Test
Vibration, and Diagnostics › DualSense haptics blocks / vibration events while playing.

### Error codes

Every connection error shows a code (also listed in Diagnostics › Error Codes). VP-102 is refined to VP-114–117 when the PS5's reply says more.

| Code | Meaning | Fix |
|---|---|---|
| VP-100 | Network problem | Check the headset's Wi-Fi and that the PS5 is on and connected. |
| VP-102 | The PS5 turned the session down | Restart the PS5 (hold the power button until it beeps twice) and try again. |
| VP-103 | The PS5 refused the connection | On the PS5 turn on Settings › System › Remote Play › Enable Remote Play, and make sure no one else is using Remote Play. |
| VP-104 | Remote Play is already in use | Close the PlayStation Portal or the other Remote Play app, then try again. |
| VP-105 | Remote Play crashed on the PS5 | Restart the PS5 and try again. |
| VP-106 | The PS5 didn't accept this Remote Play version | Update the PS5's system software, then remove the console in VisionPlay and pair it again. |
| VP-107 | The control connection failed | Try again; if it repeats, restart the PS5 and the router. |
| VP-108 | Couldn't reach the PS5 on TCP port 9295 | Put the headset and PS5 on the same network, turn off any VPN or iCloud Private Relay, and check Local Network access for VisionPlay is on. |
| VP-109 | The PS5 rejected the saved pairing | Remove the console in VisionPlay and pair it again (PS5: Settings › System › Remote Play › Pair Device). |
| VP-110 | The video stream never arrived | The router is probably blocking traffic between devices (UDP 9296–9297): turn off AP/client isolation and guest network, or connect the PS5 by Ethernet. |
| VP-111 | The PS5 closed the stream | Reconnect. If it keeps happening, check the PS5 isn't running out of bandwidth (wire it, or lower the resolution). |
| VP-112 | The PS5 shut down or went into rest mode | Wake the PS5 and connect again. |
| VP-113 | PSN registration for Internet play failed | Sign in to PSN again in Internet Play. |
| VP-114 | The PS5 says this pairing isn't valid any more | Remove the console in VisionPlay, then on the PS5 open Settings › System › Remote Play › Pair Device and pair again. Also check the PS5 user you play as is the one you paired with. |
| VP-115 | The PS5 doesn't recognise the PSN account | Pair again using the PSN account ID of the user signed in on the PS5 (sign in to PSN in VisionPlay to fill it in automatically). |
| VP-116 | The PS5 didn't reply to the session request | The PS5 was found but closed the connection. Turn on Settings › System › Power Saving › Features Available in Rest Mode › Stay Connected to the Internet and Enable Turning On PS5 from Network, restart the PS5, and try again. |
| VP-117 | The PS5's reply couldn't be read | Update the PS5's system software and pair again. If it repeats, send the Copy Log text. |
| VP-199 | The session ended unexpectedly | Try again. If it repeats, share a problem report from Diagnostics. |
| VP-201 | Couldn't search the local network | Allow Local Network access for VisionPlay in Settings › Privacy & Security › Local Network. |
| VP-202 | Pairing failed | Keep the code on the PS5 screen, and use the PSN account that is signed in on the console. |
| VP-203 | The PS5 rejected the pairing code | Get a new code on the PS5 and enter it straight away. |
| VP-204 | The PS5 is in rest mode | Waking it can take up to 30 seconds; VisionPlay keeps trying. |
| VP-205 | No saved pairing for this console | Pair the console again. |
| VP-206 | Unsupported stream settings | Set Codec to H.264 and Resolution to 1080p or lower in Settings. |
| VP-207 | The PS5 wasn't found on this network | Make sure the PS5 is on (or in rest mode with 'Stay connected to the internet' on) and on the same Wi-Fi, or use Internet Play. |
| VP-208 | The PS5 didn't answer in time | Try again; if it repeats, restart the PS5 and router. |
| VP-301 | The video couldn't be decoded | Switch Codec to H.264 in Settings and reconnect. |

## 11. Unsupported / not yet implemented

* On-screen keyboard events from the console — ignored.
* MetalFX temporal upscaling: needs per-pixel motion vectors and depth, which a video stream
  doesn't carry; without them it ghosts. Not used.
* SharePlay watching and Persona in party chat: not possible — see the v0.5 notes in chat
  SharePlay can't carry a live 60 fps game stream to another headset, and PS5 party chat is
  voice-only, so there is nowhere to show a Persona.
* HDR in Theater mode is tone-mapped (RealityKit textures are SDR).
* Adaptive bitrate: Chiaki's congestion control reports loss to the console, which
  adapts; VisionPlay adds no extra ABR logic.

## 12. Verification status

| Item | Status |
|---|---|
| chiaki-ng core + Mbed TLS + nanopb + Jerasure + bridge | **Compiled, linked and run natively (Linux clang 18)**: lib init OK, discovery response parsing OK, discovery socket and unicast probe OK |
| Swift app, Metal shaders, Xcode project (v0.1) | **Compiled and linked** in Xcode-beta (visionOS 27 simulator SDK) with no code errors; that build stopped only at code signing (iCloud attributes, fixed) |
| v0.2 additions (HDR, haptics, microphone, Internet Play, icon, 4 new library targets, single-copy video, RT-safe audio) | **Not yet compiled in Xcode.** C parts verified natively: libopus (40-byte CBR packets), libcurl + Mbed TLS (real HTTPS request), json-c, miniupnpc, chiaki-ng holepunch and the remote-session bridge all build, link and run (an invalid PSN token fails cleanly) |
| Unit tests (settings, Keychain, discovery parsing, registration validation, state machine, reconnection, controller mapping, trigger effects, upscaling config, colour conversion, shader compilation, texture creation, end-to-end GPU render, frame timing, audio buffer, resource cleanup, curved mesh) | **Written, not yet run** |
| v0.3–v0.5 Swift additions (connectivity, spatial audio, Smooth Motion, capture, cinema room, hand controls, clean-up shader, low latency, profiles, Siri, room pinning, reports) | **Not yet compiled in Xcode**; written against the visionOS 2 SDK. ExperienceTests (13) and ProfilesAndLatencyTests (6) and PictureTests (9) written, not yet run; v0.6 shader changes (picture, BT.2390, smart sharpening, bars, compare) not yet compiled |
| Discovery / registration / streaming with a real PS5 | **Untested** — needs the console |
| DualSense input, rumble, adaptive triggers, motion | **Untested** — needs the controller and headset |
| MetalFX on Vision Pro, 4K output, 60 FPS, GPU timing | **Unmeasured** — needs the headset; the Diagnostics screen and HUD report the actual numbers |

### Hardware test checklist

1. Build & ⌘U in the simulator; note any failing tests.
2. On the headset: Find PS5 → console appears with the right state (Ready/Rest mode).
3. Register with a fresh PIN → console is listed; quit and relaunch → still listed.
4. Connect → banner goes Connecting → Streaming only when video arrives; check Diagnostics:
   stream 1920×1080 HEVC, output 3840×2160, technique "MetalFX Spatial Scaler", presented ≈60 FPS,
   GPU time, decode→display latency.
5. Audio plays in sync; volume works; take the headset off/on (interruption) → audio resumes.
6. Controller Test → every button/stick/trigger/touch registers; in a game: rumble,
   adaptive triggers (e.g. Astro's Playroom), light bar.
7. Pull the PS5 network cable for 5 s → Reconnecting → Streaming.
8. Theater: flat/curved, size, distance, dark background; exit → window resumes.
9. Instruments › Metal System Trace while streaming in Quality mode.
10. Sound From the Screen: turn your head — sound stays at the screen; Room Acoustics on/off
    in Theater at 65″ vs IMAX; Virtual Surround on → faint rear ambience.
11. HUD shows "Smooth Motion · display 90/100/120 Hz"; pans look even.
12. Screenshot → appears in Photos; Instant Replay on, play 40 s, Create+Options → 30 s MP4
    with sound in Photos.
13. Theater › Cinema → room, seats, glow follows the picture.
14. Hand Controls → touchpad swipe/click and PS button work with and without the DualSense.
15. Compression Clean-up Off vs Strong on a dark sky: less blocking, no lost detail.
16. Low-Latency Mode on: lag estimate drops; HUD shows "Present on decode".
17. Save a profile, change settings, apply it → settings come back. Signed in to PSN: does
    "Playing now" show the game? (If not, auto-switching isn't available.)
18. "Hey Siri, play PS5 in VisionPlay" → wakes and connects.
19. Move the Theater screen, quit the app, reopen Theater → same spot in the room.
20. HDR game in Theater with "HDR in Theater" on vs off.
21. Picture modes: Standard vs Vivid/Cinema/Game/Dark Room; Shadow Lift in a dark game.
22. HDR game: sun/neon keep detail; Diagnostics shows the peak nits if the PS5 sends metadata.
23. Compare on: left raw, right processed; Smart Sharpening on/off on a sky; Text Clarity on a map.
24. Letterboxed cutscene: bars turn pure black (or Zoom to Fill) after ~1 s; no flicker in dark scenes.
25. HUD "Picture:" grade with Wi-Fi vs wired PS5.



**Hands keep full control with a controller connected.** `GCSupportsControllerUserInteraction`
is now `false` in Info.plist. When it was `true`, visionOS treated the DualSense as a way to
navigate the app's own buttons, so connecting it switched the interface into controller
navigation and made look-and-pinch unreliable. Game input doesn't need it: it comes through
GameController (`GCControllerDidConnect`) and `handlesGameControllerEvents`, as in Apple's
"Explore game input in visionOS". The controller plays the game; your hands run the app.

## One game screen (v0.8)

The game plays only on the **Custom Screen**: one screen, at its real size and distance.
There is no separate game window or Settings window any more.

* **Home** is the start screen. Press Play and the Custom Screen opens and Home closes. When the
  game ends (End Game, the PS5 ends it, or it fails), the screen closes and Home comes back,
  with any error shown there.
* **The panel** (on your left by default, movable with its grab bar) has Size and Distance,
  Screenshot and Save 30 s, **All Settings** (every ViniPlay setting, shown inside the panel),
  the PS5 PIN when the console asks for it, and **End Game**. Done folds it to a small button.
* A "Connecting…" card sits in front of you until the picture arrives.
* Settings › Preview Screen opens the screen from Home without a game, to try sizes.
* Trade-offs: while the screen is open, other apps' windows are hidden (it is a full space),
  and HDR games are tone-mapped (no brighter-than-white highlights).

**Controller vibration removed (v0.8).** ViniPlay no longer vibrates the DualSense: rumble and
the PS5 haptics stream are ignored and no CoreHaptics engines are created. On Vision Pro the
vibration work was the cause of stutter in hand input while a controller was connected.
Adaptive-trigger resistance and the light bar still work.

## v0.9

* **Controller and hands, no limit.** The controller plays the game and can also use ViniPlay's
  screens and panel; hands work everywhere too. End Game asks "End the game?" first, so a stray
  controller press can't end a game.
* **Controller Vibration (off by default).** Settings › Controller & Overlay. Rebuilt to run on
  its own thread (never the main thread), sending only real changes, with at most 3 engine
  rebuilds a minute. The PS5's haptics are played as a simple rumble (visionOS can't play raw
  DualSense waveforms). Strength slider and Test Vibration.
* **120 FPS Frame Generation (off by default).** Settings › Picture. Apple's low-latency frame
  interpolation (VideoToolbox, visionOS 26) makes an in-between picture for every pair of 60 fps
  frames. Adds about 8 ms of delay; off in Low-Latency Mode; switches itself off if the device
  can't keep up. May not support 10-bit HDR streams (it then switches off).
* **Custom Bitrate, 2–100 Mbps.** Settings › Stream. The PS5 may cap what it really sends; if it
  refuses, ViniPlay retries with safe settings.
* **Controller battery** shown in the panel header (refreshed every 30 s).

## v0.9.1 — back to the smooth v0.9, plus today's fixes

Built from v0.9 (the last version with no lag). The separate game window, curved window,
light spill and Live 2D→3D that came after it (v1.0+) are **not** included.

Added on top of v0.9:
* **No vibration.** Removed completely; the PS5's rumble and haptics are ignored. Adaptive
  triggers and the light bar still work.
* **Lag-causing options removed:** Apple AI Upscaling, 120 FPS Frame Generation, Low-Latency
  Mode, Smooth Motion switch (frame pacing is now always on) and Instant Replay. 4K upscaling
  stays on the fast GPU path.
* **Proper distance for each TV size:** 55″ 2.0 m, 65″ 2.2 m, 75″ 2.4 m, 85″ 2.6 m, 105″ 2.9 m,
  125″ 3.2 m, 175″ 3.8 m, 205″ 4.2 m (IMAX unchanged). A **2 km** size button, and big screens can
  come closer (up to 140° of view).
* **Full quality every game:** a lower quality chosen for a bad connection lasts only for that
  game (it used to be remembered per network).
* **Quality changes mid-game:** resolution, frame rate, codec or bitrate restart the stream
  with the new setting in a few seconds; the PS5 is given 2.5 s to release the old stream so
  you don't get VP-104 ("already in use").
* **Simpler PSN sign-in:** one big *Sign in with PlayStation* button; *Use My Phone* and
  *Enter a Code* underneath.

## v0.9.2 — game window back (picture code untouched)

* **Game window:** Play opens the game in a normal visionOS window. Drag it anywhere by its
  window bar, resize it from a corner, and play inside Apple's Environments.
* **Toolbar under the picture** (always visible): Screen (TV sizes 55″–205″ at their proper
  distance, Cinema), Volume, PS button, Microphone, controller battery and More (Settings,
  Screenshot, End Game with a confirmation).
* **Settings window:** its own window, movable anywhere.
* **Cinema** (the Custom Screen) is one tap away from the toolbar; its panel has **Back to
  Window**, and the game carries on in the window.
* The renderer, decoder, upscaling and frame pacing are exactly as in v0.9.1 — the window uses
  the same picture path (MetalStreamView) the app already had. No light spill, curved window
  or 3D.

### v0.9.3
* **Move the Cinema screen anywhere:** pinch the picture and drag. The screen swings around you
  (left/right, up/down), keeps its size and distance and always faces you; the spot is saved.
  **Center Screen** in the panel puts it back straight ahead. (Hand movement is scaled, so a
  giant far-away screen still follows: about 90° for half a metre of hand movement.)
* Apple's Environments: visionOS shows them only around windows. They work with the game window
  (Play opens it); while the Cinema (a full space) is open, visionOS hides them for every app.
* Picture, decoding, upscaling and frame pacing untouched.
* **Game window: any screen size.** Toolbar › Screen size: a slider from 40″ to 300″ (plus quick
  55″–205″ buttons). The window resizes to look like that TV from a proper seat and stays a
  normal window — drag its bar to put it anywhere, and it works in Apple's Environments. The
  size is remembered.
* **Rounded corners** on the game window and its picture (no sharp edges).

### v0.9.4
* **Cinema screen drag bar:** a bar sits under the Cinema screen (like a window's bar). Pinch it
  and drag to move the screen anywhere around you; pinching the picture itself works too. It
  grows with distance so it's easy to hit. **Center Screen** puts the screen back ahead.
* **Game window sizes change reliably:** the size popover closes first (visionOS could ignore a
  resize while it was open), and a resize that didn't take is retried up to three times. Sizes
  now grow in step with the size you pick, so 55″ and 65″ look clearly different.

### v0.9.5
* **Lag from v0.9.3/0.9.4 removed:** the rounded corners were a SwiftUI clip on the video view,
  which added a mask pass to every frame. The corners are now the video layer's own corner
  radius (cheap). The picture itself is no longer a pinch target; move the Cinema screen with
  its bar.
* **"Only the settings were visible" when connecting:** the Cinema screen could open at a spot
  saved from last time that was out of view. It now always starts straight ahead, and dragging
  can't take it more than 100° to either side.

### v0.9.6
* **Sound but no picture (window mode):** if the game window from the last game was still closing,
  a flag stayed on and no new window opened. Home now always asks for the game window when a game
  starts (asking again just brings it to the front), brings it back when Home appears during a
  game, and has a **Show Game** button next to Disconnect.

### v0.9.7
* **Airplane** (Cinema panel): a 10-second fly-by of real-size airliners (about 70 m long)
  taking off from behind the screen, climbing over its top edge and flying over your head, to
  show how big the screen is. Only when you press it; nothing stays in the scene afterwards.
* **Smooth Cinema drag bar:** gentler response (about 45° for half a metre of hand movement)
  with smoothing, so the screen follows your hand instead of flying off.
* **Game window stays 16:9:** resizing is locked to the picture's shape (no black bars), the
  minimum size is smaller so 55″ and 65″ are never both clamped to the same size, and a size
  that didn't take is retried (width and height checked).
* **Digital Crown recenter:** long-press the Digital Crown and the Cinema screen comes back in
  front of you.
