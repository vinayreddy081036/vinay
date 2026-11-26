# ViniPlay for Apple Vision Pro

ViniPlay is a personal, development-signed visionOS client for PS5 Remote Play.

## Download

Download the `.ipa` asset attached to this release. It is a development build for Vision Pro, not an App Store app.

## Important: who can install it

This IPA is signed with the developer profile used to build it. It will install only when either:

1. your Vision Pro is registered in that profile, or
2. you re-sign the IPA with your own Apple Developer certificate and provisioning profile.

The provided IPA will not automatically install on every Vision Pro.

## Install on your own Vision Pro

You need a Mac with Xcode and your Apple Developer account.

1. Open **Xcode** and sign in with your Apple ID under **Xcode > Settings > Accounts**.
2. Connect and unlock your Vision Pro. Keep it awake and approve any trust prompt.
3. Re-sign the IPA for your own Team ID and provisioning profile, then install it with Xcode or Apple Configurator.
4. On the headset, open **ViniPlay**, add your PS5, then connect using your Remote Play credentials.

## Re-signing

The `.ipa` contains an app under `Payload/ViniPlay.app`. A developer needs to replace its signature and embedded provisioning profile with ones that include the target Vision Pro.

Typical requirements:

- Apple Developer membership
- Xcode with visionOS support
- A development certificate
- An App ID matching `com.visionplay.VisionPlay` (or a changed bundle ID)
- A visionOS development provisioning profile that includes the headset

## Performance

For the best Remote Play result:

- Use Ethernet for the PS5 whenever possible.
- Use a strong 5 GHz or 6 GHz Wi-Fi connection for Vision Pro.
- Start at 1080p / 60 FPS / HEVC.
- Use **Low-Latency Mode** for fast games; it uses the fast single-pass GPU path and a smaller audio buffer.
- Avoid other high-bandwidth downloads on the same network while playing.

## Troubleshooting

| Problem | What to do |
| --- | --- |
| IPA does not install | Re-sign it with a profile that contains your Vision Pro. |
| Vision Pro is not detected | Unlock it, keep it awake, reconnect it to the Mac, then retry. |
| Stream stutters or feels delayed | Enable Low-Latency Mode and check the PS5 network connection. |
| Cannot reconnect to PS5 | End Remote Play on the console, wait a few seconds, then reconnect. |
| Controller is controlling the UI | Close or hide Settings before playing; controller input should remain with Remote Play. |

## Legal and privacy

This is an unofficial personal project and is not affiliated with Sony or PlayStation. Use it only with consoles and accounts you own or are authorized to access. Never share Remote Play credentials, registration keys, or provisioning profiles.

## Build details

- IPA filename: `ViniPlay-v0.9.7-development.ipa`
- Bundle identifier: `com.visionplay.VisionPlay`
- Package integrity: ZIP and code-signature verification passed before release packaging.

