# Migrating RayBanManager off the deprecated StreamSession API (SDK 0.6.0 → 0.9.0)

## Why this exists
`project.yml` currently pins `meta-wearables-dat-ios` to `exactVersion: "0.6.0"` because
`RayBanManager.swift` (and possibly `CameraManager.swift`) is written against the SDK's
original streaming API — `StreamSession`, `StreamSessionConfig`, `StreamSessionState`,
`StreamSessionError`. Meta renamed these in **0.7.0** and removed them entirely in
**0.9.0**, replacing them with a `DeviceSession → Camera → Stream` model. Building
against `from: "0.5.0"` (i.e. "any 0.x") silently picked up 0.9.0 in CI and broke the
build with `cannot find type 'StreamSession' in scope` and friends.

0.6.0 works today but is a dead end — do this migration once you have Mac/Xcode access
(your brother's Mac) so you can actually compile-check each step, since I can't build
Swift locally to verify this.

## API mapping (old → new)

| Old (0.6.0) | New (0.9.0) |
|---|---|
| `StreamSession(streamSessionConfig:deviceSelector:)` | `Wearables.shared.createSession(deviceSelector:)` → `DeviceSession`, then `session.addCamera(config:)` → `Camera`, then `camera.stream` → `Stream` |
| `StreamSessionConfig(videoCodec:resolution:frameRate:)` | `StreamConfiguration(...)` — same shape, passed into `addCamera(config:)` rather than the session constructor |
| `StreamSessionState` (`.streaming`, `.stopped`, `.waitingForDevice`, `.starting`, `.stopping`, `.paused`) | `StreamState` on `stream.state` / `stream.statePublisher` — re-verify exact case names against the 0.9 reference before porting the `switch` |
| `StreamSessionError` (`.deviceNotFound`, `.deviceNotConnected`, `.permissionDenied`, `.hingesClosed`, `.thermalCritical`, `.timeout`, `.videoStreamingError`, `.internalError`) | `StreamError` — same categories except `.thermalCritical`/`.videoStreamingError` may not exist 1:1; 0.9's docs list `.deviceNotFound`, `.deviceNotConnected`, `.timeout`, `.permissionDenied`, `.hingesClosed`, `.internalError` — confirm the full case list in Xcode's autocomplete/quick help once you're on a Mac, `formatError()` will need its `switch` adjusted to match |
| `session.statePublisher` / `session.videoFramePublisher` / `session.errorPublisher` | Now live on `Stream`, not the top-level session: `camera.stream.statePublisher`, `camera.stream.videoFramePublisher`, `camera.stream.errorPublisher` |
| `session.start()` | `deviceSession.start()` (connects the device) **then** `camera.stream.start()` (begins streaming) — two-step now instead of one |
| `session.stop()` | `camera.stream.stop()`, then `camera.stop()`, then `deviceSession.stop()` — stopping the parent cascades to children automatically per Meta's docs, so `deviceSession.stop()` alone may be sufficient, but confirm frames actually stop arriving before relying on cascade-only cleanup |
| N/A | New: `Stream.capturePhoto(format:) -> Bool` if still-photo capture is ever wanted alongside streaming (mutually exclusive with active video stream) |

## Concrete steps

1. **Open `ClaudeVision/ios/ClaudeVision/Services/RayBanManager.swift`** on the Mac.
2. In `project.yml`, temporarily bump `exactVersion` to `"0.9.0"` (or `from: "0.9.0"` once
   you trust it) so Xcode/SwiftPM resolves the new API and autocomplete can guide you.
3. Rewrite `start()`:
   - Replace `let session = StreamSession(streamSessionConfig: config, deviceSelector: selector)` with:
     ```swift
     let deviceSession = Wearables.shared.createSession(deviceSelector: selector)
     self.deviceSession = deviceSession   // new stored property, replaces streamSession
     try await deviceSession.start()
     let camera = try await deviceSession.addCamera(config: streamConfig) // streamConfig replaces StreamSessionConfig
     self.camera = camera                 // new stored property
     let stream = camera.stream
     ```
   - Move the `statePublisher`/`videoFramePublisher`/`errorPublisher` `.listen { }` wiring
     (lines ~152–208 today) onto `stream` instead of `session`.
   - Call `stream.start()` after wiring listeners, in place of the current
     `await session.start()` calls (there are three call sites in the permission-handling
     `Task` block — lines ~219, 229, 239 today).
4. Rewrite `stop()`: cascade via `deviceSession?.stop()`, drop the separate `streamSession`
   property in favor of `deviceSession`/`camera`, verify frames actually stop (add a log
   line, confirm it in the Xcode console against real glasses).
5. Update `formatError(_:)`'s `switch` to `StreamError`'s actual case list (check via
   Xcode Quick Help/autocomplete on the Mac — the table above is from Meta's docs page,
   not exhaustively verified against the real enum).
6. Check `CameraManager.swift` too — the original build log flagged a related warning at
   `CameraManager.swift:29` ("property declared here"), so skim it for any of the same
   renamed types even though it wasn't in the hard-error list.
7. Build locally in Xcode (not just CI) so you get real compiler diagnostics and can
   iterate fast, rather than round-tripping through GitHub Actions each time.
8. Once it builds and you've smoke-tested against the actual glasses, set
   `project.yml`'s pin to `from: "0.9.0"` (or newer, if Meta has shipped again) and push
   — CI will then build the up-to-date SDK going forward.

## Reference
- Full API reference: https://wearables.developer.meta.com/docs/reference/ios_swift/dat/0.9/
- `Camera`: https://wearables.developer.meta.com/docs/reference/ios_swift/dat/0.9/mwdatcamera_camera
- `Stream`: https://wearables.developer.meta.com/docs/reference/ios_swift/dat/0.9/mwdatcamera_stream
- `DeviceSession`: https://wearables.developer.meta.com/docs/reference/ios_swift/dat/0.9/mwdatcore_devicesession
