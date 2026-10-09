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

- **iPhone Duo simulator** — Xcode 27.1. Xcode can host two installs (stable + beta); point
  `DEVELOPER_DIR` at the beta for Duo runs. Two simulators may share the name "iPhone Duo": target by id.
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
