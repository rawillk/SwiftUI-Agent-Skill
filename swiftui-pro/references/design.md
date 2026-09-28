# Design

## Creating a uniform design in this app

Prefer to place standard fonts, sizes, colors, stack spacing, padding, rounding, animation timings, and more into a shared enum of constants, so they can be used by all views. This allows the app’s design to feel uniform and consistent, and be adjusted easily.


## Requirements for flexible, accessible design

- Never use `UIScreen.main.bounds` to read available space; prefer alternatives such as `containerRelativeFrame()`, or `visualEffect()` as appropriate, or (if there is no alternative) `GeometryReader`.
- Prefer to avoid fixed frames for views unless content can fit neatly inside; this can cause problems across different device sizes, different Dynamic Type settings, and more. Giving frames some flexibility is usually preferred.
- Apple’s minimum acceptable tap area for interactions on iOS is 44x44. Ensure this is strictly enforced.


## Standard system styling

- Strongly prefer to use `ContentUnavailableView` when data is missing or empty, rather than designing something custom.
- When using `searchable()`, you can show empty results using `ContentUnavailableView.search` and it will include the search term they used automatically – there’s no need to use `ContentUnavailableView.search(text: searchText)` or similar.
- If you need an icon and some text placed horizontally side by side, prefer `Label` over `HStack`.
- Prefer system hierarchical styles (e.g. secondary/tertiary) over manual opacity when possible, so the system can adapt to the correct context automatically.
- When using `Form`, wrap controls such as `Slider` in `LabeledContent` so the title and control are laid out correctly.
- `LabeledContent` also works outside `Form` for any title-value display; it might be necessary to define a custom `LabeledContentStyle` for consistent layout across views.
- When using `RoundedRectangle`, the default rounding style is `.continuous` – there is no need to specify it explicitly.


## Ensuring designs work for everyone

- Use `bold()` instead of `fontWeight(.bold)`, because using `bold()` allows the system to choose the correct weight for the current context.
- Only use `fontWeight()` for weights other than bold when there's an important reason - scattering around `fontWeight(.medium)` or `fontWeight(.semibold)` is counterproductive.
- Avoid hard-coded values for padding and stack spacing unless specifically requested.
- Avoid UIKit colors (`UIColor`) in SwiftUI code; use SwiftUI `Color` or asset catalog colors.
- The font size `.caption2` is extremely small, and is generally best avoided. Even the font size `.caption` is on the small side, and should be used carefully.


## Resizability (iOS 27)

- iPhone apps are resizable from iOS 27, as iPad and Mac apps already were, which matters for iPhone Mirroring and for iPhone apps running on iPad. Treat every window as resizable at runtime.
- Consequently, size decisions must come from the size classes or the container, not from the device: flag layout that branches on `UIDevice.current.userInterfaceIdiom` to choose sizes, columns or spacing. (Idiom is still the right question for genuinely device-shaped behaviour, and for the known SwiftUI quirk that a `NavigationSplitView` sidebar reports a compact horizontal size class on iPad.)
- Xcode 27's Live Previews have resize handles, so this is testable without running on a device — resize the preview rather than trusting a fixed canvas.