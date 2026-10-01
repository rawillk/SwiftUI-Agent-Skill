# iPhone Duo (foldable) layout

Everything here is `anyAppleOS 27.1` — the iOS 27.1 SDK in Xcode 27.1. It applies to
**every** app, not just ones that opt in: an app built against the 27.1 SDK is expected
to adapt, and an app built against an older one is letterboxed at a conventional iPhone
aspect ratio. There is no key that keeps the old fixed-size behaviour while building
against the new SDK, so "we'll adopt this later" is not a position an app can hold once
it rebuilds.


## The rule that outranks the rest

**Lay out for the available space, not for a named pose.** The device has a handful of
recognisable physical configurations, but they are a hardware story — there is no
"pose" to switch on, and an app that branches its layout on the hinge will be wrong the
moment the user holds it some other way. Size, aspect ratio and the reserved regions are
the inputs; the hinge is not one.

Apple's own framing: move, resize and adapt *only when it makes the experience better*.


## What already adapts, for free

Flag code that reimplements any of this by hand:

- `NavigationStack`, `NavigationSplitView` and `TabView` adapt to the fold, including
  `NavigationSplitView` column widths and margins.
- `List` and `ScrollView` adapt.
- Alerts, action sheets, menus, popovers and sheets are repositioned clear of a reserved
  region by the system. Do not hand-dodge the fold for a presentation.

So the audit starts at the custom layout: centered single-column content, hand-placed
overlays, floating buttons, anything positioned with a hard-coded offset.


## Reserved regions

A reserved region is an area the layout has to accommodate. Read them from a
`GeometryProxy` — via `GeometryReader` or `onGeometryChange()`:

```swift
public func reservedRegions(
    kind: ReservedRegion.Kind,
    options: ReservedRegion.QueryOptions = [],
    layoutDirectionBehavior: LayoutDirectionBehavior = .mirrors
) -> [ReservedRegion]
```

Two kinds, and the distinction decides what you do about one:

- `.division` — the **fold**. It *divides* the area into two usable areas.
- `.occlusion` — an obstruction such as the inner camera. It does not divide anything; it
  *obscures*.

Each region carries `frame`, `margins`, `isActive` and an `id`.

- `margins` is the required clearance *around* the region. Honouring `frame` alone is not
  enough — content pushed flush against a fold still reads as crossing it.
- A division region is inactive (and zero-width) when the device is flat. Query the active
  set for frame arithmetic, but pass `options: [.includeInactive]` for structural
  decisions that shouldn't flip as the device opens — preferring an even number of columns
  whenever a division region exists at all, for instance, rather than re-columnising on
  every hinge movement.
- `layoutDirectionBehavior` defaults to `.mirrors`, so the regions come back already
  mirrored for right-to-left. Don't mirror them again.

Treat reserved regions like any other area the layout already accommodates — iPadOS window
controls are the closest existing analogue. Reach for this API for the highest-priority
hand-laid-out controls and leave the rest to standard containers.


## `ArrangementView`

The new container for a two-view relationship. It takes a primary and a secondary view and
decides how they relate from the size, the aspect ratio and the active fold:

```swift
@available(anyAppleOS 27.1, *)
public struct ArrangementView<Primary, Secondary>: View {
    public init(
        @ContentBuilder primary: () -> Primary,
        @ContentBuilder secondary: () -> Secondary
    )
}
```

Note it builds with `@ContentBuilder`, not `@ViewBuilder`.

Pick the style from the relationship the app already expresses — `.automatic` is the
default, and `arrangementViewStyle()` sets the others:

- **`.split`** — what an `HStack`/`VStack` was saying. The two views share the bounds and
  neither is ever obscured; use it for a main-detail relationship. It splits horizontally
  when the container is wider than tall and vertically when taller than wide, and shifts
  the views so neither lands on the fold. Restrict with `.split.axes(.horizontal)`. If the
  permitted axis is unusable it shows a single view rather than cramming both.
- **`.overlay`** — what a `ZStack` was saying: a real foreground/background relationship
  where partial obscuring is acceptable. Folded, it pulls the two views apart so each gets
  its own side of the fold. Read `@Environment(\.overlayArrangementZIndex)` to respond.

Sizing, applied to the child views:

```swift
.splitArrangementLayoutRatio(_ ratio: CGFloat?)
.splitArrangementLayoutRatio(minHorizontal:idealHorizontal:maxHorizontal:
                             minVertical:idealVertical:maxVertical:)
.splitArrangementLayoutSize(minWidth:idealWidth:maxWidth:
                            minHeight:idealHeight:maxHeight:)
.splitArrangementFixedLayoutSize(horizontal: Bool = true, vertical: Bool = true)
.overlayArrangementEdge(_ edge: VerticalEdge?)    // also a HorizontalEdge? overload
```

Two hard don'ts:

- **No navigation container inside an `ArrangementView`.** It provides no navigation
  infrastructure, so a `NavigationSplitView` nested in one is a mistake, not a composition.
- **No `ArrangementView` inside a `List` or `ScrollView`.** It needs bounds to divide, and
  a scrollable container doesn't give it any.


## The hinge is for interaction, never layout

```swift
.onHingeChange(isEnabled: Bool = true) { oldContext, newContext in
    // newContext.hinge: DeviceHinge?
    //   .angle  -> Angle
    //   .status -> .closed | .partiallyOpen | .fullyOpen
}
```

This exists for interactions and effects that *are about* the fold — a control that
responds to the angle, a parallax, a shutter. **Flag any layout decision made from
`status` or `angle`**, including the tempting `switch` over the three statuses that picks
a layout. That is the pose-driven mistake above, written in a supported API.


## The vertical toolbar

A toolbar can run down a side of the display, which breaks the assumption that toolbar
items are laid out horizontally:

```swift
.toolbarVerticalBehavior(_:)             // .automatic | .disabled
.toolbarVerticalCompressionBehavior(_:)  // .automatic | .prefersToolbarItems | .prefersTabBar
```

and per item, on `ToolbarContent`/`CustomizableToolbarContent`:

```swift
.axisBehavior(_:)   // .automatic | .horizontalOnly | .verticalPreferred
```

Read `@Environment(\.toolbarVerticalEdge)` — a `HorizontalEdge?`, so `.leading`,
`.trailing`, or `nil` when the toolbar isn't vertical. An item that only makes sense in a
row (a segmented control, a wide search field) wants `.horizontalOnly`.


## Container margins

`.container` is the guide that accounts for reserved regions:

```swift
.contentMargins(for: .container, edges: .all, alignment: nil)        // View
proxy.contentMargins(for: .container, edges: .all) -> EdgeInsets     // GeometryProxy
```


## What to flag in review

- A centered single-column layout with no story for a wide or divided container. Ask
  whether it becomes two columns, or which element displaces.
- Fixed frames, and any `UIScreen.main.bounds` read (already banned in `design.md`, and now
  actively wrong).
- **Assuming symmetric safe areas.** Leading and trailing insets can differ on this device;
  code that reads one and applies it to both is broken.
- An `if`/`else` on the horizontal size class that swaps the *whole* hierarchy. The branches
  are different views, so local `@State` inside them is destroyed on every transition —
  and the fold makes those transitions routine rather than rare. Lift shared state above
  the branch, or better, let one hierarchy adapt.
- Continuous scrolling content being displaced between regions. Articles, feeds, documents
  and lists already adapt by scrolling; moving them across the fold interrupts continuity.

When displacement *is* right: move an element on its own if it can adapt independently,
move elements together when they work as a group, and keep the travel short — an element
moved far from its origin loses its relationship to it.


## Testing

The simulator ships a device type for it:

```bash
xcrun simctl list devicetypes | grep Duo
# iPhone Duo (com.apple.CoreSimulator.SimDeviceType.iPhone-Duo)
```

Pose controls are toolbar buttons along the bottom of the Simulator window, and
holding Option reveals a slider for a precise hinge angle. There is no `simctl`
equivalent — `simctl ui` exposes only appearance, contrast and content size — so folding
cannot be scripted in a UI test the way dark mode can.

Fold and unfold **while interacting with the app**, not just at rest. The failures that
matter are state loss, interrupted animations, and views that jump to the wrong position
mid-transition, and all three need the transition to happen under load to show up.
