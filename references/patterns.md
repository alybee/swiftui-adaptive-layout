# Layout defects on iPhone Duo and resizable windows

Each pattern was met in a shipping SwiftUI app and passed Apple's `app-resizability` skill. Numbers
match the detection list in `SKILL.md`.

---

## 1. A decoration that grows with the width

**Symptom** — unfolded or on iPad, a background shape or illustration behind a mascot or a title is
huge, and the page scrolls although its content is short.

**Cause** — `.resizable().scaledToFill()` (or `scaledToFit()`) with no height cap, inside a vertical
`ScrollView` or a `ZStack`. The scroll view proposes the full width and an unbounded height, so the
image takes the width and derives its height from its aspect ratio — fine at 354 pt, ~770 pt tall at
700 pt. Being the largest child, it sizes the stack.

**Fix** — bound it by what it decorates, not by the screen.

```swift
Image(decorative: "hero-backdrop")
    .resizable()
    .scaledToFit()
    .frame(maxHeight: 390)   // its height on the reference iPhone: unchanged there
```

Moving it to a `.background` also stops it sizing the stack, but only if the design does not rely on
it reserving space; check the iPhone layout before choosing.

---

## 2. Fixed vertical space tuned for a tall phone

**Symptom** — unfolded (wide but short), a page scrolls although "everything looks like it fits"; an
empty state needs scrolling to reach its text.

**Cause** — literals that only add up on a ~870 pt portrait phone: a 150 pt top inset, a 200 pt header
minimum, a 180 pt mascot plus 2 × 40 pt padding, a clearance for a floating button.

**Fix A, measured** — the space becomes a maximum that yields before the page scrolls. Measure the
viewport and the content *without* the space; there is no feedback loop because neither depends on it.

```swift
ScrollView {
    content
        .onGeometryChange(for: CGFloat.self) { $0.size.height.rounded() } action: { contentHeight = $0 }
        .padding(.top, topSpace)
}
.onGeometryChange(for: CGFloat.self) { $0.size.height.rounded() } action: { viewportHeight = $0 }

var topSpace: CGFloat {
    guard let viewportHeight, let contentHeight else { return 150 }   // phone first
    return min(150, max(0, viewportHeight - contentHeight))
}
```

**Fix B, no measurement** — `ViewThatFits` plus a decoration whose *ideal* size is its smallest.
`ViewThatFits` judges each child by its ideal size; once chosen, the real proposal lets the flexible
frame grow back.

```swift
ViewThatFits(in: .vertical) {
    page.frame(maxHeight: .infinity)
    ScrollView { page }
}

// inside page:
mascot.frame(minHeight: 60, idealHeight: 60, maxHeight: 260)   // 180 + 2 × 40 when room allows
```

A `Spacer` inside a `ScrollView` gets its minimum (the height proposal is unbounded), so
`.frame(minHeight:)` + `Spacer` tricks do not work there. In the scroll fallback the decoration shows
at its ideal size.

---

## 3. A full-bleed band whose image follows the width

**Symptom** — unfolded or on iPad, a header illustration overflows below the header, or is cropped to
its bottom strip.

**Cause** — `scaledToFill()` sized by the width, clipped only under `isRegularWidth` (which the Duo may
not report). Capping the height fixes the overflow but exposes the real problem: a 1.6:1 image cannot
fill a 3–5.5:1 band without losing its subject.

**Fix** — the frame decides the size, the image only fills it; choose the artwork by the band's
*measured* ratio. A `GeometryReader` in a `.background` does not affect layout and picks the variant in
the same pass (no `@State`, no first-frame flash).

```swift
.background {
    GeometryReader { proxy in
        let isWide = proxy.size.width > 1.8 * proxy.size.height
        Image(isWide ? "header-wide" : "header")
            .resizable()
            .scaledToFill()
            .frame(width: proxy.size.width, height: proxy.size.height,
                   alignment: isWide ? .bottomTrailing : .bottom)
    }
    .clipped()
    .padding(.bottom, -50)                        // tucks under the card below
    .ignoresSafeArea(edges: [.top, .horizontal])  // runs under the status bar, top or side
}
```

Modifier order matters: `clipped()` clips to the reader; the negative padding and `ignoresSafeArea`
outside it enlarge what the reader is offered, so it measures the full area.

---

## 4. An inset read from the window

**Symptom** — a control placed from the top inset sits in the middle of the header, or against the
edge, on the Duo.

**Cause** — `UIApplication.shared.connectedScenes.first?.keyWindow?.safeAreaInsets` (not a SwiftUI
dependency, and `.first` may not be this scene on a two-display device), usually forced by a
screen-wide `.ignoresSafeArea(edges: .top)` that zeroes every proxy inside it.

**Fix** — stop ignoring the safe area for the screen; let only backgrounds bleed (pattern 3, 11), and
let content take its place in the safe area. No inset is read at all.

---

## 5. Two elements positioned independently

**Symptom** — a settings button overlaps the greeting on one device and hugs the edge on another.

**Cause** — one element is placed from the top inset (an `overlay` with a computed padding), the other
is bottom-aligned in a minimum-height header. When the inset changes they meet.

**Fix** — one flow. Switch its axis rather than its structure, so identity survives a rotation.

```swift
var layout: AnyLayout {
    isShortScreen
        ? AnyLayout(HStackLayout(alignment: .top, spacing: 0))     // beside, in landscape
        : AnyLayout(VStackLayout(alignment: .leading, spacing: 0)) // above
}

layout {
    settingsButton.clearOfWindowControls().padding(.leading, 24)
    greeting
}
```

---

## 6. A control pushed into a side margin

**Symptom** — in one landscape direction a control is visible, in the other it is off screen.

**Cause** — `.padding(.leading, -24)` moving it into the left margin. iPhone landscape margins are
symmetric (notch one side, corner the other); the Duo's camera is on one side only, so one margin is
~0.

**Fix** — keep it in the layout (pattern 5). Never assume the opposite edge carries an inset.

---

## 7. A top control against the edge

**Symptom** — on the Duo and on iPad, a close or back button sits at the very top, hard to reach; fine
on iPhone.

**Cause** — the safe area is respected, but the top inset is ~0 on the Duo (status bar on the side),
~24 pt on iPad, 0 in iPhone landscape and in a sheet. A margin applied only `if isShortScreen` misses
the Duo.

**Fix** — a minimum distance from the top edge, topping up whatever the safe area gives.

```swift
extension View {
    func minimumTopMargin(_ length: CGFloat) -> some View { modifier(MinimumTopMargin(length: length)) }
}

private struct MinimumTopMargin: ViewModifier {
    let length: CGFloat
    @State private var topInset: CGFloat = 0

    func body(content: Content) -> some View {
        content
            .padding(.top, max(0, length - topInset))
            .onGeometryChange(for: CGFloat.self) { $0.safeAreaInsets.top } action: { topInset = $0 }
    }
}

// usage: .minimumTopMargin(isShortScreen ? 16 : 40)
```

With `safeAreaPadding(.top)` in the chain, apply it **before** `minimumTopMargin`: the other way round
the inner view measures 0 and gets the full length added on iPhone too. Apple's guidance prefers not
to read insets at all; this is the exception for a *minimum*, which no modifier expresses.

---

## 8. A top-leading button under iPadOS window controls

**Symptom** — on iPad in a window, the close/minimise/full-screen controls cover a custom button.

**Cause** — for apps built with the iPadOS 26 SDK, window controls are **not** part of the safe area.
Standard toolbars avoid them; custom buttons do not.

**Fix**

```swift
extension View {
    @ViewBuilder
    func clearOfWindowControls() -> some View {
        if #available(iOS 26, *) {
            containerCornerOffset(.leading, sizeToFit: true)
        } else {
            self
        }
    }
}
```

The offset reacts to the button's whole tap frame (44 pt), so the button may shift right even when its
glyph alone would clear the controls. Keep it: it is what protects large text sizes, where there is no
vertical slack.

It applies inside presentations too: a `fullScreenCover` fills the window, so its top-leading close
button meets the controls exactly like a pushed screen's. A `.sheet` on iPad is a form sheet inset from
the window corner and is normally clear; check it rather than assume. Not every flagged button is a
defect — a screen whose button already sits below the controls needs nothing; confirm on screen.

---

## 9. One column on the folded Duo

**Symptom** — folded, a two-column grid shows one column of oversized cards.

**Cause** — `GridItem(.adaptive(minimum: 160))`: the side status column leaves ~280 pt, under the
336 pt two columns need; an iPhone has ~354.

**Fix** — compute the count from the measured width with a floor, keep `.adaptive` until measured.

```swift
guard let width, width.isFinite, width > 0 else {
    return [GridItem(.adaptive(minimum: 160), spacing: 16)]
}
let fitting = Int((width - 2 * 24 + 16) / (160 + 16))
let count = min(max(2, fitting), max(itemCount, 1))
return Array(repeating: GridItem(.flexible(), spacing: 16), count: count)
```

Guard non-finite widths: `Int(.nan)` traps during zero-size layout passes.

---

## 10. A card across the fold

**Symptom** — unfolded and not flat, the fold runs through a card.

**Cause** — an odd column count, or an even one centred on the **safe area** while the fold sits at
the **physical** centre (the side status column shifts the safe-area centre by ~25 pt). A single
centred card lands on it too.

**Fix** — with the iOS 27.1 SDK: read the `division` region; when a vertical one crosses the grid,
round the count up to even, centre the grid on the fold's `midX`, widen the central gutter by its
thickness, round odd item counts up as well. Without a fold, change nothing. See `apis.md` for the SDK
gating problem.

When the shipping SDK cannot read the fold yet, **report it, do not drop it**: list the grid as a
deferred finding with what blocks it and the date the SDK ships, so it is picked up then.

---

## 11. Artwork that stops at the side status column

**Symptom** — a header colour reaches the right edge but its illustration stops a column short.

**Cause** — `.background(Color)` bleeds into safe areas automatically; an `Image` or a shape does not.

**Fix** — `.ignoresSafeArea(edges: .horizontal)` (with `.top` if it also runs under the status bar) on
the artwork, *and* on any shape the artwork tucks under (a card with rounded top corners), or a strip
of artwork shows beside it. Controls stay in the safe area.

---

## 12. Content stretched edge to edge

**Symptom** — on iPad full screen and the unfolded Duo, form fields, settings cards and text rows run
the full width: long lines, labels far from their values.

**Cause** — nothing caps the width; on a phone the screen does.

**Fix** — a design decision more than a defect, so propose it rather than apply it. Cap the readable
column and centre it, leaving full-bleed backgrounds alone:

```swift
VStack { cards }
    .frame(maxWidth: 640)          // a readable column
    .frame(maxWidth: .infinity)    // centred in the space
```

Leave grids alone: their column count (patterns 9, 10) already uses the width.

