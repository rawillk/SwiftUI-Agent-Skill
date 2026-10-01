# Using modern SwiftUI API

- Always use `foregroundStyle()` instead of `foregroundColor()`.
- Always use `clipShape(.rect(cornerRadius:))` instead of `cornerRadius()`.
- Always use the `Tab` API instead of `tabItem()`.
- Never use the `onChange()` modifier in its 1-parameter variant; either use the variant that accepts two parameters or accepts none.
- Do not use `GeometryReader` if a newer alternative works: `containerRelativeFrame()`, `visualEffect()`, or the `Layout` protocol. Flag `GeometryReader` usage and suggest the modern alternative.
- When designing haptic effects, prefer using `sensoryFeedback()` over older UIKit APIs such as `UIImpactFeedbackGenerator`.
- Use the `@Entry` macro to define custom `EnvironmentValues`, `FocusValues`, `Transaction`, and `ContainerValues` keys. This replaces the legacy pattern of manually creating a type conforming to (for example) `EnvironmentKey` with a `defaultValue`, then extending `EnvironmentValues` with a computed property.
- Strongly prefer `overlay(alignment:content:)` over the deprecated `overlay(_:alignment:)`. For example, use `.overlay { Text("Hello, world!") }` rather than `.overlay(Text("Hello, world!"))`.
- Never use `.navigationBarLeading` and `.navigationBarTrailing` for toolbar item placement; they are deprecated. The correct, modern placements are `.topBarLeading` and `.topBarTrailing`.
- Prefer to rely on automatic grammar agreement when dealing with English, French, German, Portuguese, Spanish, and Italian. For example, use `Text("^[\(people) person](inflect: true)")` to show a number of people.
- You can fill and stroke a shape with two chained modifiers; you do *not* need an overlay for the stroke. The overlay was required previously, but this is fixed in iOS 17 and later.
- When referencing images from an asset catalog, prefer the generated symbol asset API when the project is configured to use them: `Image(.avatar)` rather than `Image("avatar")`.
- When targeting iOS 26 and later, SwiftUI has a native `WebView` view type that replaces almost all uses of hand-wrapped `WKWebView` inside `UIViewRepresentable`. To use it, make sure to include `import WebKit`.
- `ForEach` over an `enumerated()` sequence should not convert to an array first. Use `ForEach(items.enumerated(), id: \.element.id)` directly.
- When hiding scroll indicators, use `.scrollIndicators(.hidden)` rather than `showsIndicators: false` in the initializer.
- Never use `Text` concatenation with `+`.

For example, the usage of `+` here is bad and deprecated:

```swift
Text("Hello").foregroundStyle(.red)
+
Text("World").foregroundStyle(.blue)
```

Instead, use text interpolation like this:

```swift
let red = Text("Hello").foregroundStyle(.red)
let blue = Text("World").foregroundStyle(.blue)
Text("\(red)\(blue)")
```


## Using ObservableObject

If using `ObservableObject` is absolutely required – for example if you are trying to create a debouncer using a Combine publisher – you should always make sure `import Combine` is added. This was previously provided through SwiftUI, but that is no longer the case.


## New in the iOS 27 SDK (Xcode 27)

These apply when the project targets iOS 27 or later, except where noted.

- `PreviewProvider` and its associated modifiers are now formally **deprecated**, not merely legacy. Migrate to `#Preview` while it is still only a deprecation.
- `swipeActions()` is no longer exclusive to `List`. Applying `swipeActionsContainer()` to a `LazyVStack`, `LazyVGrid` or a custom `Layout` inside a `ScrollView` lets its rows carry swipe actions, coordinated across the container so only one is open at a time. Flag hand-rolled `DragGesture` row-swipe implementations and third-party swipe-row packages.
- Drag-reordering is now declarative: `reorderable()` on the `ForEach`, plus `reorderContainer(for:)` on the container, which hands back a `ReorderDifference` to apply to the model. This reaches `LazyVGrid` and watchOS, where `onMove(perform:)` was never available. Prefer it to a hand-built drag-and-drop reorder.
- Toolbars gained overflow control: `visibilityPriority()` says which items survive a squeeze, `ToolbarOverflowMenu` groups the ones that should collapse into an overflow menu, `.topBarPinnedTrailing` pins an item so it never collapses, and `toolbarMinimizeBehavior(.onScrollDown, for: .navigationBar)` shrinks the bar as content scrolls. Flag manual `if` branching on size class to hide toolbar items.
- `Tab(role: .prominent)` singles out one tab — a cart, a compose action — for prominent placement, instead of faking it with an overlay button on top of the tab bar.
- `AsyncImage` now caches by default, honouring the server's HTTP cache headers. `AsyncImage(request:)` takes a `URLRequest` so the cache policy can be set, and `asyncImageURLSession()` supplies a session with a configured `URLCache`. Flag image-caching dependencies and hand-written `URLCache` wrappers that exist only to add this.
- `appearsActive` is an environment value: read it to quiet a window's chrome when it is not the active one, rather than tracking scene phase by hand.
- Document-based apps have a new protocol family: `Document`, with `ReadableDocument` (`readableDocumentTypes`, `DocumentReader`) and `WritableDocument` (`writableDocumentTypes`, `snapshot(contentType:)`, `writer(configuration:)`) and a nonisolated `write(snapshot:to:previous:progress:)`, plus `DocumentGroupLaunchScene`, `NewDocumentButton` and `DocumentCreationSource` for a custom creation screen. It offers direct disk access and snapshot diffing, so prefer it to `FileDocument`/`ReferenceFileDocument` in new code.
- `@ContentBuilder` unifies SwiftUI's result builders behind one builder and one initializer path, which is where a good part of Xcode 27's type-checking speedup comes from. It is built on top of `ViewBuilder`, so it is **available at any deployment target**, and unlike `ViewBuilder` it does not constrain its contents to `View` — that makes it the right builder for your own content-producing helpers and SwiftUI-like DSLs.
- **iOS 27.1** adds the iPhone Duo surface: `ArrangementView`, `ReservedRegion` and `GeometryProxy.reservedRegions(kind:options:layoutDirectionBehavior:)`, `onHingeChange()`, the vertical-toolbar modifiers, and `contentMargins(for: .container)`. All of it is `anyAppleOS 27.1`, a minor-version floor above the rest of this section. See `references/duo.md`.
- The Liquid Glass refresh in this release needs no code changes. One thing worth flagging: menu items only show their icons when the label style allows it, so `labelStyle(.titleAndIcon)` may be needed on a `Menu`'s content.