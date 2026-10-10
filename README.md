# swiftui-adaptive-layout

An agent skill that audits and fixes the **visual layout** of SwiftUI apps on the foldable
**iPhone Duo** — outer display, inner display, every pose — and on resizable **iPad windows**.

Unlike skills built from Apple's documentation, this one comes out of real adaptation passes of shipping
SwiftUI apps: every pattern is a defect that actually showed up on the simulator, with the fix that worked.

Apple ships its own skill, `app-resizability`, which replaces the APIs that break resizing
(`UIScreen.main`, orientation and idiom checks, app lifecycle, inset reads). It is the right first step,
and it does not look at what a screen *looks like*. Every defect in this skill passed Apple's cleanly
and still broke on the iPhone Duo simulator:

- a decorative image that grows with the width until the page scrolls;
- fixed spacing tuned for a tall portrait iPhone, on a display that is wide but short;
- a header illustration cropped to a strip, because a phone-ratio image cannot fill a 4:1 band;
- buttons against the top edge, because the status bar moved to a side column;
- a control pushed into a side margin that only exists in one landscape direction;
- custom buttons under iPadOS window controls, which are not part of the safe area;
- a two-column grid collapsing to one on the folded display;
- a category grid with a card across the fold;
- a footer with a hard-coded 34 pt home-indicator clearance, and a composed background that stops at
  the side status column;
- a custom keypad that should sit beside the list, not under it, on the unfolded display;
- a settings list that pushes its detail where two columns fit;
- a toast window attached to the wrong scene.

The skill turns each of these into a pattern — symptom, cause, fix, code — and a detection table to
find them before a tester does.

## What's inside

```
swiftui-adaptive-layout/
├── SKILL.md                       # when to use it, workflow, detection table, rules
└── references/
    ├── patterns.md                # 17 layout defects with symptom, cause, fix and SwiftUI code
    ├── apis.md                    # the APIs and their availability, incl. the iOS 27.1 fold APIs
    ├── testing.md                 # device matrix, driving the Duo simulator, before/after diffs, seed data
    ├── assets.md                  # briefing a designer for a wide illustration
    └── apple-checklist.md         # the API side, summarised, when Apple's skill is not installed
```

## Install

Copy the folder into your skills directory:

```bash
git clone https://github.com/alybee/swiftui-adaptive-layout ~/.claude/skills/swiftui-adaptive-layout
```

Use `~/.claude/skills/` for every project, or `.claude/skills/` at a project's root for that project
only. The agent picks the skill up from its description; you can also ask for it by name.

Recommended alongside it, Apple's skills (Xcode 27.1 or later):

```bash
DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer \
  xcrun agent skills export --output-dir ~/.claude/skills
```

## Use

> Adapt the screens for the iPhone Duo.

> On the unfolded iPhone Duo, the onboarding page scrolls for no reason.

> On iPad, when the app runs in a window, the close/minimise buttons cover my back button.

> On the unfolded Duo the fold runs through a card of my category grid.

> Can the settings list show its detail beside it on iPad and the unfolded Duo?

That last one is easy to miss: it only shows once the app is in a window, never in full screen. For apps
built with the iPadOS 26 SDK, the window controls sit in the top-leading corner **outside the safe area**.
Standard toolbars move out of their way; a custom button does not. The skill offsets it only when the
controls are there:

```swift
backButton
    .clearOfWindowControls()   // containerCornerOffset(.leading, sizeToFit: true) on iOS 26+
```

Check every screen with a top-leading custom button (back, close, settings) with the app in a window,
not only in full screen.

The skill works one defect at a time: it names the cause, proposes a fix with its effect on iPhone stated
(unchanged unless said otherwise), applies it, and hands over for verification on the simulator. It
does not sweep the codebase with speculative fixes.

## Status and limits

- Built from real adaptation passes of two shipping SwiftUI apps, in the **Xcode 27.1 beta and RC**
  (build 27A9275) simulators, October 2026. No physical iPhone Duo was used.
- Measurements such as "~280 pt usable when folded" or "~780 × 540 pt unfolded" come from simulator
  screenshots and are approximate; the second pass measured the inner display at 951 × 669 pt in
  landscape (669 × 951 portrait) and the outer one at 466 × 678 pt, with `axe describe-ui`.
- **Pattern 10 (a card across the fold) ran on the 27.1 RC simulator**: 7 adaptive columns became 6
  centred on the fold; the code in `patterns.md` is what shipped. It ran in both apps.
- Patterns 13–17 (hard-coded bottom clearance, composed backgrounds, a side pane, master/detail
  settings, overlay windows) come from the second pass; iPhone was checked pixel-identical for each.
- SwiftUI only. UIKit layout is covered by Apple's skill.

Corrections from anyone with a device are welcome — open an issue or a pull request.

## Sources

- Apple, [Prepare your app for iPhone Duo](https://developer.apple.com/iphone-duo/prepare/)
- Apple, [Designing for iPhone Duo](https://developer.apple.com/design/human-interface-guidelines/designing-for-iphone-duo)
- Wesley Matlock, [Through the Never: SwiftUI Layout for the iPhone Duo](https://www.wesleymatlock.com/iphone-duo-swiftui-width-driven-layout/)
- Apple, [NavigationSplitView](https://developer.apple.com/documentation/swiftui/navigationsplitview) (collapsing in narrow sizes, "typically the root view in a Scene")

## License

This project is licensed under the MIT License - see [LICENSE](LICENSE) for details.