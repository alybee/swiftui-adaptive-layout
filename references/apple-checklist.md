# API checklist (when Apple's `app-resizability` skill is not installed)

Apple ships an authoritative skill for the API side of resizing, in Xcode 27.1 and later:

```bash
DEVELOPER_DIR=/Applications/Xcode-beta.app/Contents/Developer \
  xcrun agent skills export --output-dir ~/.claude/skills
```

Use it when you can: it is maintained with each Xcode release. What follows is a short summary in our
own words, so an audit is not blind without it. Sources: Apple's *Prepare your app for iPhone Duo*
(<https://developer.apple.com/iphone-duo/prepare/>) and the `app-resizability` skill.

## Project settings (read only, report, change nothing)

- [ ] A launch screen is declared (`UILaunchScreen`, a launch storyboard, or
      `INFOPLIST_KEY_UILaunchScreen_Generation`). iOS 27 rejects uploads without one.
- [ ] The iPad declares all four orientations (or none). A portrait-only iPhone declaration is normal.
- [ ] `UIRequiresFullScreen` is absent. iOS 27 ignores it and resizes the scene anyway; removing it, or
      setting `UIRequiresFullScreenIgnoredStartingWithVersion`, is the developer's call, after the
      layouts are ready.

Read every source of build settings (project file and `.xcconfig`s) before reporting a key as missing.

## Code

- [ ] **No `UIScreen.main`.** Size from the view or window (`view.bounds`, `GeometryReader`,
      `onGeometryChange`), scale from `traitCollection.displayScale` / `@Environment(\.displayScale)`,
      the screen from `view.window?.windowScene?.screen`, windows from `UIWindow(windowScene:)`.
- [ ] **No layout from orientation** (`UIDevice.current.orientation`, `statusBarOrientation`,
      `interfaceOrientation`, `effectiveGeometry.interfaceOrientation`). Compare the width and height of
      the space available instead.
- [ ] **No layout from idiom** (`userInterfaceIdiom`, `UI_USER_INTERFACE_IDIOM()`). Any size-class
      combination can occur on any device; `.regular` width is not "iPad". No hard-coded device sizes.
- [ ] **Scene lifecycle.** Window and UI setup lives in a scene delegate (or a SwiftUI `App`), not in
      `UIApplicationDelegate`.
- [ ] **Size re-read on every change**, not once: `layoutSubviews` / `viewDidLayoutSubviews` /
      `viewWillTransition(to:with:)` in UIKit; SwiftUI re-evaluates on its own.
- [ ] **Safe areas are asymmetric and change at runtime.** No `topLayoutGuide`/`bottomLayoutGuide`; no
      hard-coded bar heights (20, 44, 64, 34…); never apply one edge's inset to both sides; never store
      an inset; never read it from another view or window; never test it against a threshold to infer
      the device.
- [ ] **SwiftUI insets:** scope `ignoresSafeArea(edges:)` to backgrounds; place content with
      `safeAreaInset`, `safeAreaBar` (iOS 26) or `safeAreaPadding` rather than reading insets; a
      `GeometryProxy.size` is already inset.
- [ ] **Fill images:** `.scaleAspectFill` / `.aspectRatio(contentMode: .fill)` on hero images loses
      content on wide displays — set a focal point, or choose fill or fit by aspect ratio.
- [ ] **iOS 27.1 SDK:** standard toolbars and tab bars become vertical, below the side status bar on
      iPhone Duo. Check content against them before shipping a 27.1 build.

## How Apple's skill applies changes

Apply the safe one-to-one replacement where it exists; ask before a change that alters behaviour or a
public signature; leave an actionable TODO only when neither is possible; touch only the lines that
hold the target API; keep every existing guard and `#available` check.
