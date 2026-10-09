# swiftui-adaptive-layout

An agent skill that audits and fixes the **visual layout** of SwiftUI apps on the foldable
**iPhone Duo** (outer display, inner display, every pose) and on resizable **iPad windows**.

Unlike skills built from Apple's documentation, this one comes out of a real adaptation pass of a shipping
SwiftUI app: every pattern is a defect that actually showed up on the simulator, with the fix that worked.

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
- a two-column grid collapsing to one on the folded display.

The skill turns each of these into a pattern — symptom, cause, fix, code — and a detection table to
find them before a tester does.

## What's inside

```
swiftui-adaptive-layout/
├── SKILL.md                       # when to use it, workflow, detection table, rules
└── references/
    ├── patterns.md                # 12 layout defects with symptom, cause, fix and SwiftUI code
    ├── apis.md                    # the APIs and their availability, incl. the iOS 27.1 fold APIs
    ├── testing.md                 # device matrix, previews with Duo variants, seed data
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
DEVELOPER_DIR=/Applications/Xcode-beta.app/Contents/Developer \
  xcrun agent skills export --output-dir ~/.claude/skills
```

## Use

> Adapt the screens for the iPhone Duo.

> On the unfolded iPhone Duo, the onboarding page scrolls for no reason.

> On iPad, when the app runs in a window, the close/minimise buttons cover my back button.

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

- Built from a real adaptation pass of a shipping SwiftUI app, in the **Xcode 27.1 beta** simulator,
  October 2026. No physical iPhone Duo was used.
- Measurements such as "~280 pt usable when folded" or "~780 × 540 pt unfolded" come from simulator
  screenshots and are approximate.
- **Pattern 10 (a card across the fold) is untested.** The fold APIs exist only in the iOS 27.1 SDK;
  the section describes the intended approach, not code that has run.
- SwiftUI only. UIKit layout is covered by Apple's skill.

Corrections from anyone with a device are welcome — open an issue or a pull request.

## Sources

- Apple, [Prepare your app for iPhone Duo](https://developer.apple.com/iphone-duo/prepare/)
- Apple, [Designing for iPhone Duo](https://developer.apple.com/design/human-interface-guidelines/designing-for-iphone-duo)
- Wesley Matlock, [Through the Never: SwiftUI Layout for the iPhone Duo](https://www.wesleymatlock.com/iphone-duo-swiftui-width-driven-layout/)

## License

This project is licensed under the MIT License - see [LICENSE](LICENSE) for details.