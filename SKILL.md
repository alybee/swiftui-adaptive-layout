---
name: swiftui-adaptive-layout
description: "Audit and fix the visual layout of a SwiftUI app on the foldable iPhone Duo (outer and inner displays, all poses) and on resizable iPad windows. Use when a screen looks wrong, scrolls for no reason, clips, or puts a control under the status bar, the camera or iPadOS window controls on iPhone Duo or a resized window; when preparing screenshots or a test pass for iPhone Duo; or when asked to 'adapt the screens for the iPhone Duo'. Complements Apple's app-resizability skill, which replaces deprecated APIs but does not catch layout defects such as fill images that grow with width, fixed heights tuned for a tall phone, or asymmetric insets."
---

# SwiftUI adaptive layout (iPhone Duo, resizable iPad windows)

Apple's `app-resizability` skill removes the *APIs* that break resizing (`UIScreen.main`, orientation,
idiom, app lifecycle, inset reads). It does not see what a screen *looks like*. Every defect in
[patterns.md](references/patterns.md) passed that skill cleanly and still broke on the iPhone Duo. Run
`app-resizability` first if it is installed, then this skill. If it is not, walk
[apple-checklist.md](references/apple-checklist.md) before the layout pass.

## The device, in one paragraph

Folded, the outer display is compact width, a little wider than an iPhone, with the **status bar in a
column on the right edge**: the top safe area is close to zero and the right one carries the whole
inset. Unfolded, the inner display is **regular width and wide but short** — roughly 780 × 540 pt in
the screenshots that drove this skill — with the same side column. A fold (`division` reserved region)
crosses the inner display when the device is not flat. The camera sits on one side only, so landscape
insets are **asymmetric**. With the iOS 27.1 SDK the system moves standard toolbars and tab bars to a
vertical bar below that status column. iPhones and the Duo never rotate to portrait-upside-down: that
is system behaviour, not an app setting.

Size classes observed in the Xcode 27.1 simulator — the code's branches depend on them, so state them in
every finding rather than reporting "if it reports regular height":

| State | Horizontal | Vertical | Top inset | Notes |
|---|---|---|---|---|
| Duo folded, portrait | compact | regular | ~0 | status column on the right; ~280 pt usable after margins |
| Duo folded, landscape | compact | **compact** | 0 | camera on one side only: one side inset ~0 |
| Duo unfolded | **regular** | **regular** | ~0 | ~780 × 540 pt: wide but short, yet *not* compact height |
| iPad full screen | regular | regular | ~24 | |
| iPad in a window | any | any | ~24 | window controls outside the safe area |

The unfolded row is the trap: a layout that drops its large spacing only when `verticalSizeClass ==
.compact` keeps it on a 540 pt-tall display.

## Workflow

Work one defect at a time. The developer reviews and verifies each fix before the next one starts.

1. **Set up the matrix** — see [testing.md](references/testing.md). At minimum: iPhone portrait and
   landscape (the reference that must not regress), Duo folded both landscapes, Duo unfolded, iPad full
   screen, iPad in a window, one accessibility text size.
2. **Scan** with the detection list below. Report what you find; do not fix speculatively. A scan finds
   candidates; only a screenshot proves a defect.
3. **Per reported screen**: name the cause from [patterns.md](references/patterns.md) — they recur —
   then propose the fix with its effect on iPhone stated explicitly. "Unchanged on iPhone" is the
   default promise; say so when it does not hold, and by how many points.
4. **Fix**, build, check in a preview with the Duo display variants when the tooling allows it
   ([testing.md](references/testing.md)), then hand over for verification on the simulator.
5. **Factor on the second occurrence**, not the first: a modifier used by two screens earns a file.

## Detection list

| Look for | Usually means | Pattern |
|---|---|---|
| `scaledToFill()` / `.aspectRatio(contentMode: .fill)` on an image in scroll content or a band, with no height cap | grows with width | 1, 3 |
| `.padding(.top, <large literal>)`, `minHeight: <literal>` on a header, a mascot with a fixed `frame(maxHeight:)` plus fixed padding | tuned for a tall phone | 2 |
| `if isRegularWidth` / `horizontalSizeClass == .regular` guarding a clip, a size or a variant | treats a size class as a device | 1, 3 |
| `safeAreaInsets` read from `UIApplication`, `connectedScenes.first`, `keyWindow` | wrong scene on a two-display device | 4 |
| A screen-wide `.ignoresSafeArea(edges: .top)` followed by hand-placed controls | content placed from an inset it no longer sees | 4 |
| Two header elements placed independently (one from the top, one bottom-aligned, or an `overlay`) | they collide when the inset changes | 5 |
| A negative padding pushing a control into a side margin | assumes symmetric insets | 6 |
| A top-leading custom button (close, back, settings) | iPadOS window controls; ~0 top inset on Duo | 7, 8 |
| `GridItem(.adaptive(minimum:))` | one column on the folded Duo; odd count across the fold | 9, 10 |
| A background image that stops at a safe-area edge while its colour does not | side status column | 11 |
| A rounded or filled shape *beside* or *below* a band that bleeds sideways (a card with rounded top corners) | a strip of the band shows next to it | 11 |
| A width or height picked by a short-screen check (`isShortScreen ? 120 : 240`) | pattern 2 in disguise: the tall value applies unfolded | 2 |
| Forms, settings cards or text rows with no `frame(maxWidth:)` | stretched edge to edge on iPad and unfolded | 12 |

Starting points for the scan (adjust the path; each hit is a candidate, not a defect):

```bash
SRC=App/Sources
grep -rnE 'scaledToFill|contentMode: *\.fill' $SRC                         # 1, 3
grep -rnE '\.padding\(\.top, *[0-9]{3}|minHeight: *[0-9]{3}' $SRC            # 2
grep -rnE '\? *[0-9]{2,} *: *[0-9]{2,}' $SRC                                  # 2, size by branch
grep -rnE 'isRegularWidth|horizontalSizeClass *== *\.regular' $SRC           # 1, 3
grep -rnE 'keyWindow|connectedScenes|UIScreen\.main|safeAreaInsets' $SRC      # 4
grep -rnE 'ignoresSafeArea\((edges: *\[?\.top|\))' $SRC                     # 4, 11
grep -rnE 'overlay\(alignment: *\.top(Leading)?' $SRC                          # 5, 7, 8
grep -rnE '\.padding\(\.(leading|trailing), *-' $SRC                          # 6
grep -rnE 'xmark|chevron\.(left|backward)|gearshape' $SRC                     # 7, 8
grep -rnE 'GridItem\(\.adaptive' $SRC                                         # 9, 10
```

## Rules

- **Decide from measured space, never from the device.** Size classes are fine for *height is short*
  (`verticalSizeClass == .compact`); they are not a proxy for iPad, for the Duo, or for an aspect ratio.
- **Content in the safe area, backgrounds bleed.** Colour backgrounds bleed into safe areas on their
  own; images and shapes need an explicit `.ignoresSafeArea(edges:)` naming only the edges they reach.
- **Prefer layout over measurement.** Put elements in one flow, use `ViewThatFits`, `AnyLayout`,
  flexible frames. Measure (`onGeometryChange`, rounded to whole points) only when no container does
  the job, and measure *outside* any `ignoresSafeArea` that would zero the value.
- **Make decorations yield first.** When space runs out, whitespace and illustrations shrink before
  the page scrolls; text and controls do not.
- **A wrong aspect ratio has no code-only fix.** A phone-ratio illustration cannot fill a 3–5:1 band
  without losing its subject. Ask for a wide asset ([assets.md](references/assets.md)).
- **Unavailable APIs fail into the tested path.** Fold geometry needs the iOS 27.1 SDK; if it is not
  the SDK you ship with, defer rather than gate on SDK internals ([apis.md](references/apis.md)).

## References

- [patterns.md](references/patterns.md) — the defects, each with symptom, cause, fix and code
- [apis.md](references/apis.md) — the APIs, their availability, and the SDK-gating trap
- [testing.md](references/testing.md) — the device matrix, previews with Duo variants, seed data
- [assets.md](references/assets.md) — specifying a wide illustration for a designer
- [apple-checklist.md](references/apple-checklist.md) — the API side, when Apple's skill is not installed
