# Data flow, shared state, and property wrappers

It is important that SwiftUI body code and logic code be kept separate in order to make code easier to read, write, and maintain. That usually means placing code into methods rather than inline in the `body` property, but often also means carving functionality out into separate `@Observable` classes.

These rules help ensure code is efficient and works well in the long term.


## Shared state

- `@Observable` classes must be marked `@MainActor` unless the project has Main Actor default actor isolation. Flag any `@Observable` class missing this annotation.
- All shared data should use `@Observable` classes with `@State` (for ownership) and `@Bindable` / `@Environment` (for passing).
- Strongly prefer not to use `ObservableObject`, `@Published`, `@StateObject`, `@ObservedObject`, or `@EnvironmentObject` unless they are unavoidable, or if they exist in legacy/integration contexts when changing architecture would be complicated.


## Local state

- `@State` should be marked `private` and only owned by the view that created it.
- **As of Xcode 27, `@State` is a macro rather than a property wrapper**, and the change is back-deployed to iOS 17 / macOS 14 and aligned releases — so it applies whatever the deployment target, as soon as the project builds with Xcode 27. It is source-compatible "with exceptions", and the exceptions are what to flag:
  - A property must not have both a default value at its declaration *and* an assignment in `init`. The declaration's value is discarded, and referring to `self` in `init` now errors. Remove the default value: `@State private var page: StickerPage` then assign in `init`.
  - `@State` no longer composes with other property wrappers.
  - Type inference through `@State` in generic contexts is weaker, so an explicit type annotation is sometimes now required.
  - The memberwise initializer is no longer synthesized for a type with a `private` `@State` property.
  - The upside is that it fixes the oldest `@State` trap: the stored value is now initialized lazily and **only once** for the lifetime of the view, so `@State private var model = ExpensiveModel()` no longer builds a throwaway instance on every `body` re-evaluation.
- If a view stores a class instance that contains expensive-to-recompute data, e.g. `CIContext`, it can be stored using `@State` even though it is not an observable object. This effectively uses `@State` as a cache – storing something persistently, but not doing any change tracking on it since it's not an observable object.


## Bindings

- Strongly prefer to avoid creating bindings using `Binding(get:set:)` in view body code. It is much cleaner and simpler to use a binding provided by `@State`, `@Binding` or similar, then use `onChange()` to trigger any effects.
- If the user needs to enter a number into a `TextField`, bind the `TextField` to a numeric value such as `Int` or `Double`, then use its `format` initializer like this: `TextField("Enter your score", value: $score, format: .number)`. Apply either `.keyboardType(.numberPad)` (for integers) or `.keyboardType(.decimalPad)` (for floating-point numbers) as appropriate. Using the modifier alone is *not* sufficient.


## Working with data

- Prefer to make structs conform to `Identifiable` rather than using `id: \.someProperty` in SwiftUI code.
- Never attempt to use `@AppStorage` inside an `@Observable` class, even if marked `@ObservationIgnored` – it will *not* trigger view updates when a change happens.


## SwiftData

- If you only need the number of items matching a query, consider `ModelContext.fetchCount()` with a fetch descriptor. This will *not* live update if the data changes unless something else triggers the update, such as `@Query`, so it should be used carefully.

For more help with SwiftData, suggest the [SwiftData Pro agent skill](https://github.com/twostraws/swiftdata-agent-skill).

## If the project uses SwiftData with CloudKit

- Never use `@Attribute(.unique)`.
- Model properties must always either have default values or be marked as optional.
- All relationships must be marked optional.
