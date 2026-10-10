# Layout defects on iPhone Duo and resizable windows

Each pattern was met in a shipping SwiftUI app and passed Apple's `app-resizability` skill. Numbers
match the detection list in `SKILL.md`.

| # | Pattern | Typical trigger |
|---|---|---|
| 1 | A decoration that grows with the width | `scaledToFill` with no height cap |
| 2 | Fixed vertical space tuned for a tall phone | large literal paddings and minimum heights |
| 3 | A full-bleed band whose image follows the width | phone-ratio header art |
| 4 | An inset read from the window | `keyWindow`, `connectedScenes` |
| 5 | Two elements positioned independently | overlay vs bottom-aligned header |
| 6 | A control pushed into a side margin | negative side padding |
| 7 | A top control against the edge (+ `minimumTopMargin`/`minimumBottomMargin`) | bar-less screens |
| 8 | A top-leading button under iPadOS window controls | custom back/close buttons |
| 9 | One column on the folded Duo | `GridItem(.adaptive)` |
| 10 | A card across the fold (fold-aware grid, verified code) | odd column count unfolded |
| 11 | Artwork that stops at the side status column | images and shapes in bands |
| 12 | Content stretched edge to edge (+ readable `List`/`Form`) | settings, forms, onboarding |
| 13 | A hard-coded home-indicator clearance | `.padding(.bottom, 34)` |
| 14 | A composed background that stops at the status column | `Color.shadow/mask` footers |
| 15 | A bottom panel that becomes a side pane | custom keypad under a list |
| 16 | A list that pushes a detail (`NavigationSplitView` in regular only) | settings, library, favourites lists |
| 17 | An overlay window attached to the wrong scene | toast and modal windows |

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

**Cause** — `scaledToFill()` sized by the width, clipped only behind a size-class check
that the screen in hand does not pass. Capping the height fixes the overflow but exposes the real
problem: a 1.6:1 image cannot fill a 3–5.5:1 band without losing its subject.

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
    verticalSizeClass == .compact
        ? AnyLayout(HStackLayout(alignment: .top, spacing: 0))     // beside, when height is short
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
~24 pt on iPad, 0 in iPhone landscape and in a sheet. A margin applied only on short screens misses
the Duo.

**Fix** — a minimum distance from the edge, topping up whatever the safe area gives. One modifier
serves both edges: the bottom variant is pattern 13's fix.

```swift
extension View {
    /// At least `length` pt between the top edge and this view.
    func minimumTopMargin(_ length: CGFloat) -> some View {
        modifier(MinimumSafeAreaMargin(edge: .top, length: length))
    }
    /// Same from the bottom: pinned buttons stay in the safe area and still clear the edge on an
    /// iPhone without a home indicator (pattern 13).
    func minimumBottomMargin(_ length: CGFloat) -> some View {
        modifier(MinimumSafeAreaMargin(edge: .bottom, length: length))
    }
}

private struct MinimumSafeAreaMargin: ViewModifier {
    let edge: VerticalEdge
    let length: CGFloat
    @State private var inset: CGFloat?        // nil until measured: adds nothing, phone first

    func body(content: Content) -> some View {
        let edge = edge
        content
            .padding(edge == .top ? .top : .bottom, extra)
            // Measured on the padded view: its edge does not move with the padding, so there is
            // no measure → pad → measure loop.
            .onGeometryChange(for: CGFloat.self) { proxy in
                (edge == .top ? proxy.safeAreaInsets.top : proxy.safeAreaInsets.bottom).rounded()
            } action: { inset = $0 }
    }

    private var extra: CGFloat {
        guard let inset else { return 0 }
        return max(0, length - inset)
    }
}

// usage, at the root of a bar-less screen: .minimumTopMargin(16)
```

Why the optional: a `@State` that starts at 0 pads by the full `length` on the first frame, so the
reference iPhone drops by 16 pt and snaps back once the measurement lands. `nil` adds nothing until
the inset is known, which is the phone's behaviour anyway (its ~60 pt inset exceeds the length).

Apply it at the **root** of the screen: a view lower down measures an inset of 0 and would always get
the full length. For content placed in an `overlay` (a close button), apply it *after* the overlay,
or the overlay keeps its old origin. With `safeAreaPadding(.top)` in the chain, apply it **before**
`minimumTopMargin` for the same reason. Apple's guidance prefers not to read insets at all; this is
the exception for a *minimum*, which no modifier expresses.

Unfolded, a top control moves from ~0–8 pt to the minimum; iPhone portrait stays pixel-identical.

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

**Fix** (iOS 27.1 SDK, verified on the 27.1 RC simulator) — read the `division` region on the grid's
own proxy, with `.includeInactive` so the layout does not reflow while the device is being folded.
Only a *vertical* region that crosses the view counts: unfolded in portrait the fold is horizontal
and cuts a scrolling list, there is nothing to avoid. Then lay out an even number of columns, the
same count each side of the fold, and widen the central gutter by the fold's thickness. Measure on
the padded view, as in pattern 2, so the insets you add do not change what you measure.

```swift
/// nil without a vertical fold across the view: iPhone, iPad, folded Duo, unfolded in portrait, or
/// a system older than iOS 27.1. `nonisolated`: see apis.md, Swift 6 isolation.
private nonisolated struct FoldGeometry: Equatable, Sendable {
    var width: CGFloat
    var minX: CGFloat
    var maxX: CGFloat

    init?(_ proxy: GeometryProxy) {
        guard #available(iOS 27.1, *) else { return nil }
        let width = proxy.size.width
        let regions = proxy.reservedRegions(kind: .division, options: .includeInactive)
        guard let region = regions.first(where: { region in
            region.frame.height > region.frame.width
                && region.frame.minX > 0 && region.frame.maxX < width
        }) else { return nil }
        self.width = width.rounded()
        minX = region.frame.minX.rounded()
        maxX = region.frame.maxX.rounded()
    }
}

/// Even grid centred on the fold: as many columns each side, central gutter widened by the fold.
private struct FoldGridLayout {
    let columns: [GridItem]
    let leadingInset: CGFloat
    let trailingInset: CGFloat

    init(fold: FoldGeometry, minimum: CGFloat, spacing: CGFloat) {
        let middle = (fold.minX + fold.maxX) / 2
        // The shorter side sets the half width; the other side keeps a mirror margin (on the
        // Duo in landscape, the status column's).
        let half = min(middle, fold.width - middle)
        leadingInset = middle - half
        trailingInset = fold.width - middle - half

        let gutter = spacing + (fold.maxX - fold.minX)
        let perSide = max(1, Int((half - gutter / 2 + spacing) / (minimum + spacing)))
        var columns = Array(repeating: GridItem(.flexible(), spacing: spacing), count: perSide * 2)
        columns[perSide - 1] = GridItem(.flexible(), spacing: gutter)
        self.columns = columns
    }
}

// in the view
@State private var fold: FoldGeometry?
let foldLayout = fold.map { FoldGridLayout(fold: $0, minimum: 100, spacing: 16) }

LazyVGrid(columns: foldLayout?.columns ?? adaptiveColumns, spacing: 16) { … }
    .padding(.leading, foldLayout?.leadingInset ?? 0)
    .padding(.trailing, foldLayout?.trailingInset ?? 0)
    .onGeometryChange(for: FoldGeometry?.self) { FoldGeometry($0) } action: { fold = $0 }
```

Measured, Duo unfolded in landscape (951 × 669 pt, fold at x = 475.5): the adaptive grid showed 7
columns and the fold crossed the 4th; with the fold read, 6 columns of the same 105 pt width, 3 a
side, and a 448–504 pt gutter centred on the fold. Give the section title the same insets so it
centres on the grid, not on the screen. Everywhere else `FoldGeometry` is nil and the adaptive grid
is untouched: iPhone portrait pixel-identical before and after.

Two more things to handle:

- **Few items.** A grid that narrows to its items centres them on the safe area, so a single or
  middle item lands on the fold. Cap the columns per side at `ceil(count / 2)`.
- **Anything that spans the width crosses the fold too**: a control above the grid, a centred
  floating button. Give it the grid's insets and split it at the gutter, or move it off-centre (a
  corner).

Measure the fold on a view that is always on screen, not on the grid: an empty state has no grid.

If the project cannot build with the 27.1 SDK yet, **report the grid, do not drop it**: list it as a
deferred finding with what blocks it. Once it uses the fold API, the project no longer builds with an
older Xcode (`apis.md`, the SDK-gating trap).

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

A `List` or a `Form` is the exception to the frame: capping its frame also caps what scrolls, and the
rows look centred in a narrower list. Keep the whole width scrolling with content margins (iOS 17;
the frame stays as the iOS 16 fallback):

```swift
private struct ReadableListColumn: ViewModifier {
    @State private var geometry: ListGeometry?

    func body(content: Content) -> some View {
        if #available(iOS 17.0, *) {
            content
                .contentMargins(.leading, leadingMargin, for: .scrollContent)
                .contentMargins(.trailing, trailingMargin, for: .scrollContent)
                .onGeometryChange(for: ListGeometry.self) {
                    ListGeometry(width: $0.size.width.rounded(),
                                 leading: $0.safeAreaInsets.leading.rounded(),
                                 trailing: $0.safeAreaInsets.trailing.rounded())
                } action: { geometry = $0 }
        } else {
            content.frame(maxWidth: 640).frame(maxWidth: .infinity)
        }
    }

    // nil while the list is narrower than the column: the system margins stay, nothing moves.
    private var side: CGFloat? {
        guard let geometry, geometry.width > 640 else { return nil }
        return ((geometry.width - 640) / 2).rounded()
    }
    private var leadingMargin: CGFloat? { side.map { $0 + (geometry?.leading ?? 0) } }
    private var trailingMargin: CGFloat? { side.map { $0 + (geometry?.trailing ?? 0) } }
}

private nonisolated struct ListGeometry: Equatable, Sendable {
    var width: CGFloat, leading: CGFloat, trailing: CGFloat
}
```

Why the insets are added: content margins count from the **screen edge**, under the safe area, and
the system keeps the larger of the margin and the inset. Without them the column centres on the
screen, not on the safe area: on the unfolded Duo the rows ran 84 pt further, under the status
column.

A pinned footer takes the same column on its *content*, with the background applied after it, so the
background still bleeds (pattern 14).

---

## 13. A hard-coded home-indicator clearance

**Symptom** — a page ends with `.ignoresSafeArea(edges: .bottom)` and its pinned buttons carry
`.padding(.bottom, 34)`. It looks right on every current iPhone, and it is two of the things Apple's
checklist forbids: a stored bar height, and a page opting out of the safe area so a background can
reach the edge.

**Cause** — the footer's background has to bleed to the bottom edge, and the only way the author found
was to let the whole page ignore the inset, then add the home indicator's height back by hand.

**Fix** — the page keeps its safe area; only the background ignores it (pattern 14); the footer gets
`.minimumBottomMargin(34)` (pattern 7's modifier on the other edge), so the buttons still clear the
edge on an iPhone without a home indicator.

```swift
footer
    .minimumBottomMargin(34)
    .background(Color.footerBackground.shadow(…).mask(…)
        .ignoresSafeArea(edges: [.horizontal, .bottom]))
// and drop .ignoresSafeArea(edges: .bottom) from the page
```

Detection: `padding\(\.bottom, *(34|20)\)` with an `ignoresSafeArea` nearby.

---

## 14. A composed background that stops at the status column

**Symptom** — a footer's grey background reaches the bottom edge but stops at the side status column:
a white band under the column on the unfolded Duo in landscape, and beside the notch on an iPhone in
landscape.

**Cause** — `Color` bleeds into safe areas on its own; `Color.x.shadow(…).mask(…)` is a composed view
and behaves like an `Image` (pattern 11). Any modifier on the colour turns the automatic bleed off.

**Fix** — `.ignoresSafeArea(edges: .horizontal)` on the composed background, plus `.bottom` once the
page no longer ignores it (pattern 13). The content above keeps its insets.

Detection: `\.background\(\s*Color\.[A-Za-z]+\s*\.(shadow|mask|opacity|blur)`.

---

## 15. A bottom panel that becomes a side pane

**Symptom** — a custom keypad, a tool palette or a result panel sits under a list; unfolded in
landscape it takes the whole bottom of a 669 pt display and the list shows three rows.

**Fix** — when the content is wider than tall, place the panel beside the list. `AnyLayout` keeps
the same children, so identity survives a fold or a rotation and typed input is kept:

```swift
let layout = isWide ? AnyLayout(HStackLayout(spacing: 0)) : AnyLayout(VStackLayout())
layout {
    VStack { header; list }
    Keypad(sideBySide: isWide, paneWidth: geometry.rightPaneWidth)
}
.onGeometryChange(for: ContentGeometry.self) { ContentGeometry(proxy: $0) } action: { geometry = $0 }

// isWide = geometry.size.width > geometry.size.height   (the content's own size, not the device)
// rightPaneWidth: the physical middle of the window, insets included, so the split lands on the fold
//   (size.width + insets.leading + insets.trailing) / 2 - insets.trailing
```

Three things move with the panel: its bleed, `.ignoresSafeArea(edges: sideBySide ? .bottom : .all)`;
its height cap, `.infinity` beside the list instead of the phone's fixed height; and a 1 pt rule on
the fold edge.

**Behaviour follows the layout.** A `DragGesture` that hides the panel when the list scrolls exists
to free the phone's screen; beside the list the panel hides nothing, and hiding it on every scroll
becomes the defect. Gate the gesture by the same `isWide` rather than by the
device. Detection: `simultaneousGesture(DragGesture` that sets a visibility flag.

---

## 16. A list that pushes a detail

**Symptom** — on iPad and the unfolded Duo, a list (settings, library, favourites) fills the width,
then pushes a second full-width screen: two taps and a lot of empty space for what fits side by side.

**Fix** — a master/detail in regular width only; in compact width keep the list and its push, so the
phone is unchanged. **Do not place a collapsing `NavigationSplitView` inside a pushed screen of the
app's `NavigationStack`: collapsed, it renders empty** (iOS 27.1, iPhone portrait: nothing but the
back button).

```swift
@Environment(\.horizontalSizeClass) private var horizontalSizeClass
@State private var selection: Item?

if horizontalSizeClass == .regular {
    NavigationSplitView {
        List(selection: $selection) { rows(isSplit: true) }   // rows carry .tag(item)
            .toolbar { titleItem }                             // see "title" below
    } detail: {
        if let selection {
            DetailScreen(item: selection, onChange: { reload() })
                .id(selection)   // its .task loads once per identity
        }
    }
} else {
    List { rows(isSplit: false) }   // tap pushes the detail, as before
}
```

What the system gives for free: the column divider lands on the fold by itself (no fold code), and
two columns on an iPad Air 11" in portrait and on an iPhone Max in landscape.

What to handle:

- preselect the first item **only when split**: collapsed, a split view shows "the last column that
  displays useful information", so a preselection would open the detail instead of the list;
- a `.principal` toolbar title is centred on the whole screen and lands on the fold: put it in the
  sidebar column's toolbar;
- changes made in the detail must refresh the list (an `onChange` callback plus a revision counter for
  rows that cache a computed value);
- `.toolbar(removing: .sidebarToggle)` (iOS 17) when the list is the only way to choose a detail;
- `navigationSplitViewColumnWidth` was ignored on the iPad Air 11" in portrait (320 pt column, names
  on two lines): accept it, or move the row's secondary text under the name.


---

## 17. An overlay window attached to the wrong scene

**Symptom** — a toast or a modal drawn in an extra `UIWindow` (level above `.alert`) never shows, or
shows on the other display of the Duo.

**Cause** — `UIApplication.shared.connectedScenes.first` picks *a* scene; on a two-display device or
in multi-window it is not necessarily the one showing the app, and a retry timer hides the race.

**Fix** — take the scene from the view that shows the app:

```swift
struct WindowSceneReader: UIViewRepresentable {
    let onScene: (UIWindowScene) -> Void
    func makeUIView(context: Context) -> SceneReportingView { … isUserInteractionEnabled = false … }
    func updateUIView(_ view: SceneReportingView, context: Context) { view.onScene = onScene }

    final class SceneReportingView: UIView {
        var onScene: ((UIWindowScene) -> Void)?
        override func didMoveToWindow() {
            super.didMoveToWindow()
            if let scene = window?.windowScene { onScene?(scene) }
        }
    }
}

// root view: .background(WindowSceneReader { overlay.attach(to: $0) })
// attach(to:) creates the window once, then re-attaches it if the scene changes.
```

Give shared overlay components the same care as screens: a shared toast component sat 4 pt from the top
of the unfolded Duo until it got pattern 7's minimum margin.

Observed, not fixed: a window at `.alert + 1` may take status-bar control from third-party full-screen
presenters. An ad SDK logged "Status bar could not be hidden for full screen ad" and its close button
ended under the status column, unreachable. Check every full-screen SDK presentation on the Duo.

