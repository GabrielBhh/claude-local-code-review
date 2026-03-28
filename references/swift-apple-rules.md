# Swift, SwiftUI & Apple Liquid Glass Rules

## Swift Language Best Practices

| Issue | Violation | Fix |
|---|---|---|
| Force-unwrap without justification | `let val = optional!` | `guard let val = optional else { return }` |
| `try!` swallowing errors | `let data = try! load()` | `let data = try load()` with proper catch |
| `try?` hiding errors silently | `let result = try? parse(data)` | Use `try` with `catch`, or log the failure |
| Retain cycle in closure | `NetworkManager.fetch { result in self.update() }` | `{ [weak self] result in self?.update() }` |
| Strong self in escaping closure | `completion = { self.doWork() }` | `completion = { [weak self] in self?.doWork() }` |
| `AnyObject` / `Any` overuse | `func process(_ item: Any)` | Use proper protocols or generics |
| Missing `@MainActor` on UI state | `@Published var title: String` in class updating UI | `@MainActor @Published var title: String` |
| `DispatchQueue.main.async` | `DispatchQueue.main.async { self.title = val }` | `@MainActor func updateTitle(_ val: String)` |
| `Notification.Name` as raw string | `NotificationCenter.post(name: Notification.Name("myEvent"), ...)` | `extension Notification.Name { static let myEvent = Notification.Name("myEvent") }` |
| Synchronous network on main thread | `URLSession.shared.data(from: url)` without `await` | `await URLSession.shared.data(from: url)` in async context |
| Ignoring `Sendable` conformance | Passing non-Sendable across actor boundaries | Make type `Sendable` or use `@unchecked Sendable` with justification |

---

## SwiftUI Best Practices

| Issue | Violation | Fix |
|---|---|---|
| `@State` for shared/injected state | `@State var viewModel = ViewModel()` | `@StateObject var viewModel = ViewModel()` |
| `@StateObject` in non-owning view | Passing VM down but recreating it | Use `@ObservedObject` in child views |
| Missing `@Published` on mutated properties | `var name: String = ""` in `ObservableObject` | `@Published var name: String = ""` |
| `ObservableObject` when `@Observable` is better | iOS 17+ target, class-based VM | `@Observable class ViewModel { var name = "" }` |
| `body` exceeding ~60 lines | Monolithic `body` | Extract subviews into separate `View` structs |
| `onAppear` for async work | `.onAppear { Task { await load() } }` | `.task { await load() }` |
| Hardcoded color | `.foregroundColor(Color(red: 0.1, green: 0.2, blue: 0.3))` | `.foregroundStyle(.primary)` or semantic color |
| Hardcoded font size | `.font(.system(size: 17))` | `.font(.headline)` or `.font(.body)` (Dynamic Type) |
| `GeometryReader` overuse | Using `GeometryReader` for simple spacing | Use `.frame()`, `Spacer()`, layout modifiers instead |
| Missing accessibility label on interactive elements | `Image(systemName: "heart")` button | `.accessibilityLabel("Like post")` |
| Missing `#Preview` macro | No preview for view | `#Preview { MyView() }` |
| `@State private` missing `private` | `@State var isLoading = false` | `@State private var isLoading = false` |

---

## Apple Liquid Glass (iOS 26 / macOS Tahoe / visionOS 2+)

Liquid Glass is the new translucent material system introduced in iOS 26. It replaces manual blur/vibrancy implementations with a unified `.glassEffect()` modifier that automatically adapts to content, color scheme, and system context.

### Critical Violations

| Issue | Violation | Fix |
|---|---|---|
| Deprecated blur API | `UIVisualEffectView(effect: UIBlurEffect(style: .systemMaterial))` | Delete `UIViewRepresentable` wrapper; use `.glassEffect()` modifier directly |
| `NSVisualEffectView` (macOS) | `NSViewRepresentable` wrapping `NSVisualEffectView` | Use `.glassEffect()` — works cross-platform |
| Hardcoded opaque background on glass surface | `.background(Color.white.opacity(0.9))` | Remove `.background(...)` — let `.glassEffect()` handle the surface |
| Wrong button style in glass context | `.buttonStyle(.bordered)` on glass panel | `.buttonStyle(.glass)` |
| Custom blur conflicting with system material | `ZStack { BlurView() ... }` | Single `.glassEffect()` on the container |

### Warnings

| Issue | Violation | Fix |
|---|---|---|
| Floating panel missing glass | Custom sheet/card with solid background | Add `.glassEffect()` to the container |
| Toolbar not using glass | Custom toolbar with `.background(.regularMaterial)` | `.glassEffect()` replaces material modifiers in iOS 26 |
| Text on glass without vibrancy | `.foregroundStyle(.primary)` on text overlaid on glass | `.foregroundStyle(.secondary)` or `.glassEffect` with vibrancy label style for contrast |
| Missing availability guard | `.glassEffect()` without `#available(iOS 26, *)` check | Wrap in `if #available(iOS 26, *) { ... } else { ... }` with `.ultraThinMaterial` fallback |
| Hardcoded dark/light color instead of adaptive | `Color.white` / `Color.black` on glass | Use `.primary`, `.secondary`, or `Color(.label)` |
| `GlassEffectContainer` missing for grouped elements | Multiple `.glassEffect()` views that should merge | Wrap in `GlassEffectContainer` so borders merge correctly |
| Ignoring `colorScheme` environment | Manual color switching based on `@Environment(\.colorScheme)` | Liquid Glass adapts automatically — remove manual override |

### iOS 26 Liquid Glass HIG Checklist

Apply `.glassEffect()` to:
- [ ] Floating panels, cards, and sheets
- [ ] Sidebars and navigation drawers
- [ ] Toolbars and tab bars (when custom)
- [ ] Modal overlays and popups
- [ ] Any surface that floats above dynamic content (maps, photos, video)

Do NOT apply `.glassEffect()` to:
- Static backgrounds with no content beneath
- List rows (use system list styles)
- Full-screen solid-color views

### Code Examples

**Deprecated blur (violation):**
```swift
struct BlurView: UIViewRepresentable {
    func makeUIView(context: Context) -> UIVisualEffectView {
        UIVisualEffectView(effect: UIBlurEffect(style: .systemUltraThinMaterial))
    }
    func updateUIView(_ view: UIVisualEffectView, context: Context) {}
}

// Used as:
ZStack {
    BlurView()
    content
}
```

**Correct Liquid Glass pattern:**
```swift
content
    .glassEffect()
```

**Floating card with availability guard:**
```swift
struct ProfileCard: View {
    var body: some View {
        VStack { content }
            .background {
                if #available(iOS 26, *) {
                    Color.clear.glassEffect()
                } else {
                    .ultraThinMaterial
                }
            }
    }
}
```

**Button in glass context:**
```swift
// Violation:
Button("Follow") { ... }
    .buttonStyle(.bordered)

// Fix:
Button("Follow") { ... }
    .buttonStyle(.glass)
```

**Grouped glass elements (GlassEffectContainer):**
```swift
GlassEffectContainer {
    HStack {
        Button("Like") { ... }.glassEffect()
        Button("Share") { ... }.glassEffect()
    }
}
// Borders between adjacent glass elements merge automatically
```
