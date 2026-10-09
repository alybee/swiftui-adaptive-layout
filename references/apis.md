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

SwiftUI exposes the fold directly from the 27.1 SDK: no hidden `UIViewRepresentable` probe is needed.

## The SDK-gating trap

`#available(iOS 27.1, *)` is a **runtime** check. It does not help when the code must still compile
with an older SDK — for example when releases are archived with the stable Xcode while the 27.1 SDK is
only in beta. Both Xcodes can ship the **same Swift compiler**, so `#if compiler(>=…)` cannot tell them
apart.

`#if canImport(SwiftUICore, _version: <x.y.z>)` can (the module's `user-module-version` differs between
SDKs), but `_version` is unofficial and the number can move between beta and release. Prefer deferring
the work until the shipping Xcode carries the SDK, and leave the grid on its tested behaviour until
then. If the fold is never reported, the layout must fall back to the behaviour it had without it.

## Inset rules worth repeating

- A `GeometryProxy.size` is already inset; subtracting insets from it counts them twice.
- Inside `.ignoresSafeArea`, proxies report 0: measure outside it.
- `safeAreaPadding(.top)` consumes the inset for the views inside it.
- Never read insets from `UIApplication`, a `UIWindowScene` found by enumeration, or `keyWindow`.
