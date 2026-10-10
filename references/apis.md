# APIs and availability

| API | Availability | Use |
|---|---|---|
| `onGeometryChange(for:of:action:)` | iOS 16+ (back-deployed) | measure a size or inset; round to whole points to avoid thrash during live resize |
| `ViewThatFits(in:)` | iOS 16+ | pick a layout that fits; judges children by their *ideal* size |
| `AnyLayout`, `HStackLayout`, `VStackLayout` | iOS 16+ | switch axis without rebuilding the subtree (identity, focus, animation survive) |
| `containerCornerOffset(_:sizeToFit:)` | iOS 26 | move a custom corner control clear of iPadOS window controls |
| `safeAreaBar(edge:)` | iOS 26 | bar content that reserves its space; `safeAreaInset(edge:)` below that target |
| `GeometryProxy.reservedRegions(kind:options:layoutDirectionBehavior:)` → `[ReservedRegion]` | **iOS 27.1 SDK** | `kind: .division` = the fold, `.occlusion` = e.g. camera; `options: .includeInactive` returns the fold even when flat, so layouts don't reflow mid-fold. `ReservedRegion` has `frame`, `margins`, `isActive` |
| `DeviceHinge` (`status`: `.closed` / `.partiallyOpen` / `.fullyOpen`, `angle`), `DeviceHingeContext` | **iOS 27.1 SDK** | pose-dependent behaviour |
| `@Environment(\.toolbarVerticalEdge)` | **iOS 27.1 SDK** | which edge holds the system's vertical bar (`nil` where there is none) |
| UIKit `reservedRegionsOfKind:` / `reservedRegions(kind:options:)` | iOS 27.1 SDK | UIKit counterpart; no change notification, read from `layoutSubviews` |
| `contentMargins(_:_:for:)` | iOS 17 | readable column in a `List`/`Form` that keeps the whole width scrolling; counts from the **screen edge**, so add each side's safe-area inset (pattern 12) |
| `NavigationSplitView` | iOS 16 | master/detail in regular width only; collapsed inside a pushed screen it renders empty (pattern 16) |
| `navigationSplitViewColumnWidth(_:)` / `(min:ideal:max:)` | iOS 16 | ignored on an iPad Air 11" in portrait with the 27.1 RC: do not rely on it |
| `toolbar(removing: .sidebarToggle)` | iOS 17 | drops the button that hides the sidebar column |
| `@Environment(\.horizontalSizeClass)` | iOS 14 | `.regular` ⇒ two columns; with the vertical class, the only size-class decision this skill makes |
| `UIViewRepresentable` + `didMoveToWindow` → `window?.windowScene` | any | the scene of the view that shows the app, for an overlay window (pattern 17) |

SwiftUI exposes the fold directly from the 27.1 SDK: no `UIViewRepresentable` probe is needed for it.
A representable is still the way to learn which `UIWindowScene` shows a view (pattern 17).

## Swift 6 isolation and `onGeometryChange`

The value you measure must be `Sendable`. In a module built with `SWIFT_DEFAULT_ACTOR_ISOLATION =
MainActor` (the default for new targets since Xcode 26), a plain `struct` is main-actor isolated and
the compiler refuses it: "main actor-isolated conformance … cannot satisfy conformance requirement
for a 'Sendable' type parameter". Declare measured value types `nonisolated struct Foo: Equatable,
Sendable`; an optional of it works too (`for: FoldGeometry?.self`). Scalars (`CGFloat`) need nothing.

## The SDK-gating trap

`#available(iOS 27.1, *)` is a **runtime** check: it does not let the code compile with an older SDK.
Two Xcodes can ship the same Swift compiler, so `#if compiler(>=…)` cannot tell them apart either;
`#if canImport(SwiftUICore, _version: …)` can, but `_version` is unofficial and moves between builds.

So: build and archive with an Xcode that carries the API, or defer the work and leave the layout on its
tested behaviour. Once the API is used, `#available` inside the measuring code is the only guard left,
and the project no longer builds with an older Xcode. If the fold is never reported, the layout must
fall back to the behaviour it had without it.

After switching Xcode, build from a fresh derived-data folder: precompiled modules keep the old
Xcode's path and fail with "malformed or corrupted precompiled file".

## Inset rules worth repeating

- A `GeometryProxy.size` is already inset; subtracting insets from it counts them twice.
- Inside `.ignoresSafeArea`, proxies report 0: measure outside it.
- `safeAreaPadding(.top)` consumes the inset for the views inside it.
- Never read insets from `UIApplication`, a `UIWindowScene` found by enumeration, or `keyWindow`.
- `contentMargins` count from the screen edge, not from the safe area: add the inset of each side.
- With a navigation bar, the top inset is the bar's height (82 pt measured on the unfolded Duo); the
  side status column adds nothing at the top. Bar-less screens get ~0 (pattern 7).
