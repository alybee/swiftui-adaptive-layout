# Testing

## Device matrix

| Device / state | Why |
|---|---|
| iPhone portrait | the reference: every fix must leave it unchanged, or say by how much it moves |
| iPhone landscape | short height; symmetric side insets; no status bar |
| Duo folded, portrait | side status column, ~0 top inset, ~280 pt usable width |
| Duo folded, both landscapes | asymmetric insets: camera on one side only |
| Duo unfolded | wide but short; regular width; the fold |
| iPad full screen, portrait and landscape | short status bar (~24 pt), wide bands |
| iPad in a window | window controls outside the safe area; arbitrary sizes |
| One accessibility text size (`.accessibility3`) | removes the slack that hides collisions |

Rotation to portrait-upside-down does nothing on iPhones and the Duo, in every app: not a defect.

## Tools

- **iPhone Duo simulator** — Xcode 27.1 or later. Two simulators may share the name "iPhone Duo":
  target by id. Also keep an iPad Air 11" simulator: in portrait it is the
  narrowest regular-width case (pattern 16).
- **Device Hub** (Xcode > Open Developer Tool; `Contents/Developer/Applications/DeviceHub.app`) — the only way to
  fold, unfold or rotate the Duo: no `simctl` command does it. Plan the pass so the developer sets
  each pose and tells you; you capture and measure. `simctl io <udid> screenConfig power` does not
  switch displays, it leaves both black until the simulator is shut down and booted again.
- **Previews with Duo variants** — with the preview canvas device set to iPhone Duo, previews accept
  variant overrides `Display: Outer | Inner`, `Orientation`, `Dynamic Type`. Through Xcode's MCP
  bridge (`RenderPreview`, `previewVariantOverrides`) this checks a fix without launching the app.
- **Resizable simulator / iPhone Mirroring on macOS 27** — resize to extremes in both directions.
- Apple's guide: <https://developer.apple.com/iphone-duo/prepare/>.

## Previews that crash

A view that reads a dependency the app injects (a manager with no valid default, which traps when
missing) crashes its preview unless the preview injects it too. Give previews one helper that injects
every such dependency, rather than repeating the lines in each `#Preview`.

## Seed data for runs and screenshots

Grids, cards and odd item counts need data. A DEBUG launch argument that swaps the store for an
in-memory one, seeded from the preview fixtures, keeps fake data out of the user's iCloud:

```swift
private static var isSeeding: Bool {
    #if DEBUG
        UserDefaults.standard.bool(forKey: "seedPreviewData")
    #else
        false
    #endif
}
// ModelConfiguration(isStoredInMemoryOnly: true, cloudKitDatabase: .none) when seeding
```

- Each scheme argument needs its value: `-seedPreviewData YES`. A bare `-seedPreviewData` reads as false.
- Photos for the seed go in a development asset catalog (`DEVELOPMENT_ASSET_PATHS`): included in runs,
  excluded from archives — usable for App Store screenshots.
- Seed an odd number of items: it is what an even-column layout has to absorb.
- Size seed photos to the exact aspect of the frame that shows them (a multiple of the frame ratio,
  under the encoder's maximum dimension), or they are centre-cropped.

## Driving the Duo simulator from a terminal

- **Capture per display**: `xcrun simctl io <udid> screenshot --display=3 out.png` is the inner
  display (2007 × 2853 px, 669 × 951 pt), `--display=1` the outer one (1398 × 2034 px, 466 × 678 pt).
  The inactive display renders black. The command can hang ("Timeout waiting for screen surfaces"):
  run it in the background and kill it after ~8 s; the file is written before the hang.
- **Positions**: `axe describe-ui --udid <udid>` (Accessibility) gives frames in points, in the
  landscape coordinate space when the app is in landscape (951 wide). It is the reliable way to read
  where a control ended up; a screenshot alone needs a pixel-to-point conversion (3×).
- **Taps and swipes** (`idb`, `axe`) drop intermittently on the Duo: check with `axe describe-ui`
  after each action, or a lost swipe passes as "the panel stayed" (pattern 15).
- **Reaching a screen** (paywall, onboarding) without tapping through: write the app's defaults with
  `xcrun simctl spawn <udid> defaults write <plist path> key value` and relaunch. An unsigned build
  (`CODE_SIGNING_ALLOWED=NO`) has **no App Group**: anything the app stores there is silently lost.
- **Third-party overlays**: full-screen SDK presentations (ads, consent forms) are screens too. On the
  unfolded Duo an ad's close button sat under the status column (pattern 17, note); the consent form
  rendered broken in landscape. Note them; they are not fixable from the app's layout code.

## Prove "unchanged on iPhone"

The promise every fix makes is checkable. Capture the reference iPhone before the change (keep the
built `.app` aside), capture after, and diff:

```python
from PIL import Image, ImageChops
a = Image.open("before.png").convert("RGB"); b = Image.open("after.png").convert("RGB")
diff = ImageChops.difference(a, b).convert("L").point(lambda v: 255 if v > 8 else 0)
print(diff.getbbox())   # None = pixel-identical; else the changed area in px (÷ scale for pt)
```

Reach the same state both times (same defaults, same prompts dismissed: rating, ATT, an ad). A
non-empty box with a bbox the size of the screen usually means a dialog or an ad, not the fix.

Without PIL, `cmp before.png after.png` proves a byte-identical capture. A capture may draw the
Dynamic Island in one shot and not the other: when `cmp` differs, compare below the status bar.
