# SwiftUI Handbook

## Declarative UI, State, Architecture, Layout & Core UI

---

# 1. The Bigger Picture: Declarative UI

Before learning SwiftUI, it helps to understand that SwiftUI belongs to a broader shift in UI development.

Traditional UI frameworks are largely **imperative**.

You create objects, place them into a hierarchy, and explicitly tell them what to change:

```text
Create view
    ↓
Configure view
    ↓
Add view to hierarchy
    ↓
User interacts
    ↓
Find affected view
    ↓
Mutate view
```

Declarative frameworks invert that relationship:

```text
Application State
        ↓
     UI description
        ↓
   Framework renders UI
        ↑
    User interaction
        ↓
  State changes
        ↓
   UI description changes
```

You describe **what the UI should look like for a given state** rather than manually maintaining every visual mutation.

Three important ecosystems use this model:

```text
Apple       → SwiftUI
Android     → Jetpack Compose
Web         → React
```

The syntax differs, but the architectural ideas are remarkably similar.

---

# 2. SwiftUI, Jetpack Compose and React

## SwiftUI

SwiftUI is Apple's declarative UI framework.

```swift
struct ProfileView: View {
    var body: some View {
        VStack {
            Text("Alex")
            Button("Follow") {
                // update state
            }
        }
    }
}
```

The `body` describes the UI.

SwiftUI reevaluates the relevant view description when the state that the view depends on changes.

Apple's model-data documentation and Observation framework are built around connecting application data to views so that UI responds to changes.

---

## Jetpack Compose

Compose uses composable functions instead of SwiftUI's `View` protocol.

```kotlin
@Composable
fun ProfileScreen() {
    Column {
        Text("Alex")

        Button(onClick = { /* update state */ }) {
            Text("Follow")
        }
    }
}
```

The same fundamental principle applies:

```text
State → Composable → UI
          ↑
       events
```

Compose explicitly describes its UI as immutable and says that when state changes, affected parts of the UI tree are recomposed. Its architecture guidance also recommends unidirectional data flow: state flows down, events flow up.

---

## React

React uses components and JSX.

```jsx
function Profile() {
    const [following, setFollowing] = useState(false);

    return (
        <button onClick={() => setFollowing(!following)}>
            {following ? "Following" : "Follow"}
        </button>
    );
}
```

React's `useState` gives a component state plus a setter that requests another render with the new state. React's documentation also emphasizes finding the minimal representation of state and keeping a single owner for each piece of state.

---

# 3. The Common Declarative UI Mental Model

The three frameworks can be thought of like this:

| Concept            | SwiftUI           | Jetpack Compose                    | React                         |
| ------------------ | ----------------- | ---------------------------------- | ----------------------------- |
| UI unit            | `View`            | `@Composable`                      | Component                     |
| UI description     | `body`            | Composable function                | JSX / returned tree           |
| Local state        | `@State`          | `remember { mutableStateOf(...) }` | `useState`                    |
| Passed-in data     | Properties        | Parameters                         | Props                         |
| Binding/event flow | `Binding`         | State + callbacks                  | Props + callbacks             |
| Observable model   | `@Observable`     | `State`, `StateFlow`, etc.         | External/store state patterns |
| UI update          | View invalidation | Recomposition                      | Render/update                 |
| Shared environment | `Environment`     | `CompositionLocal`                 | Context                       |

The terminology differs, but the engineering questions are mostly the same:

```text
What is state?
Who owns it?
Who can change it?
Who needs to observe it?
How does an event modify it?
Which UI depends on it?
```

That is the real foundation of modern declarative UI.

---

# 4. Declarative vs Imperative Thinking

### Imperative UIKit-style thinking

```swift
label.text = user.name
label.textColor = .blue
button.isHidden = !user.isLoggedIn
```

You are telling the UI **how to mutate itself**.

### Declarative SwiftUI thinking

```swift
VStack {
    Text(user.name)

    if user.isLoggedIn {
        Button("Sign Out") {
            signOut()
        }
    }
}
```

You describe the desired UI.

The framework determines the necessary update.

This has a major architectural consequence:

> State becomes more important than individual UI objects.

Instead of asking:

```text
"How do I update this label?"
```

you ask:

```text
"What state should the label represent?"
```

---

# 5. The SwiftUI View

A SwiftUI view is a description of UI.

```swift
struct WelcomeView: View {
    var body: some View {
        Text("Welcome")
    }
}
```

The important part is:

```swift
var body: some View
```

A view can compose other views:

```swift
struct DashboardView: View {
    var body: some View {
        VStack {
            HeaderView()
            SummaryCard()
            RecentActivityView()
        }
    }
}
```

Think of a screen as a tree:

```text
DashboardView
│
├── HeaderView
│
├── SummaryCard
│
└── RecentActivityView
```

This is conceptually similar to the component trees used in React and the composition trees used by Compose.

---

# 6. Composition Is the Core Skill

SwiftUI is not primarily about writing huge screens.

It is about composing small descriptions of UI.

For example:

```swift
struct ProductCard: View {
    let product: Product

    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            Text(product.name)
                .font(.headline)

            Text(product.price)
                .foregroundStyle(.secondary)

            Button("Buy") {
                buy(product)
            }
        }
        .padding()
    }
}
```

Then:

```swift
ForEach(products) { product in
    ProductCard(product: product)
}
```

A professional SwiftUI codebase should therefore resemble:

```text
Feature
│
├── Screen
│
├── Components
│   ├── Card
│   ├── Row
│   └── Header
│
└── Model / State
```

rather than a single monolithic view.

---

# 7. State: The Central Concept

**State is information whose value can change over time and whose changes affect the application or its UI.**

Examples:

```text
isLoggedIn
username
selectedTab
searchText
products
loading state
error state
navigation path
selected item
```

The important distinction is that **not every value is state**.

For example:

```swift
let products: [Product]
```

may be input data.

Meanwhile:

```swift
@State private var searchText = ""
```

is changing UI state.

A useful rule is:

> Store the smallest set of values that must actually change. Derive everything else.

For example, don't normally store:

```swift
items
itemCount
```

when:

```swift
itemCount == items.count
```

can simply be computed.

This principle appears across declarative frameworks. React explicitly recommends a minimal representation of UI state, while Compose emphasizes state as the source of truth for the UI.

---

# 8. State Ownership

One of the most important architectural questions is:

> Who owns this state?

Suppose an editor has:

```swift
name
email
```

and only the editor needs them.

Local state is appropriate:

```swift
@State private var name = ""
@State private var email = ""
```

But suppose a parent screen and two child views both need access to the selected account.

That state should move to a common owner.

Conceptually:

```text
             Account State
                  │
        ┌─────────┴─────────┐
        ↓                   ↓
   Profile View        Settings View
```

The objective is **one source of truth for each piece of mutable state**.

React calls this ownership/single-source-of-truth principle and recommends lifting state to the closest common owner. Compose similarly recommends hoisting state to the lowest common ancestor that needs to read and write it.

---

# 9. `@State`

`@State` is primarily for state owned by a view.

```swift
struct CounterView: View {
    @State private var count = 0

    var body: some View {
        VStack {
            Text("\(count)")

            Button("Increment") {
                count += 1
            }
        }
    }
}
```

The key relationship is:

```text
CounterView
    owns
       ↓
    count
       ↓
     UI
```

Use `@State` for local, view-scoped state.

It is not a replacement for an application-wide model or persistence system.

---

# 10. `@Binding`

A binding is a read/write connection to state owned elsewhere.

Parent:

```swift
struct SettingsView: View {
    @State private var enabled = true

    var body: some View {
        NotificationSetting(enabled: $enabled)
    }
}
```

Child:

```swift
struct NotificationSetting: View {
    @Binding var enabled: Bool

    var body: some View {
        Toggle("Notifications", isOn: $enabled)
    }
}
```

Conceptually:

```text
Parent owns state
       ↓
     Binding
       ↓
Child edits state
```

The `$` exposes the projected binding value.

This is analogous to React's pattern of passing a value and callback together, while Compose generally makes the same relationship explicit through state parameters and event callbacks.

---

# 11. Observable Models

When state belongs to a screen, feature, or application model rather than an individual control, you need an observable model.

There are two important SwiftUI generations to understand.

```text
Older SwiftUI
    ↓
ObservableObject
@Published
@StateObject
@ObservedObject
    ↓
Modern Swift
    ↓
Observation
@Observable
Observable
@Bindable
```

Understanding both is important because large existing applications still use the older model.

---

# 12. `ObservableObject`

The older Combine-based model looks like this:

```swift
import Combine

final class ProfileModel: ObservableObject {
    @Published var name = ""
    @Published var isLoading = false
}
```

The model conforms to:

```swift
ObservableObject
```

and individual properties that should trigger object-level change publication are commonly marked:

```swift
@Published
```

A SwiftUI view can observe it:

```swift
struct ProfileView: View {
    @ObservedObject var model: ProfileModel

    var body: some View {
        Text(model.name)
    }
}
```

Historically this meant a number of pieces had to line up:

```text
ObservableObject
       +
@Published properties
       +
SwiftUI observer
```

Apple's migration material describes the modern Observation approach as a simplification of this older `ObservableObject` model.

---

# 13. What Does `@Published` Actually Mean?

`@Published` is a Combine property wrapper.

```swift
final class ProfileModel: ObservableObject {
    @Published var name = ""
}
```

Conceptually it creates a publisher associated with changes to `name`.

When the value changes:

```text
name changes
    ↓
publisher emits
    ↓
ObservableObject change propagation
    ↓
SwiftUI observes
    ↓
dependent UI updates
```

This is important:

> `@Published` is not simply a "SwiftUI state keyword."

It belongs to the Combine observation/publisher model.

This distinction becomes particularly useful when migrating older code to Observation.

---

# 14. `@StateObject` vs `@ObservedObject`

Another common source of confusion in older SwiftUI code:

### `@StateObject`

A view creates and owns the observable object.

```swift
@StateObject private var model = ProfileModel()
```

Conceptually:

```text
View
 └── owns lifetime of model
```

### `@ObservedObject`

The model is supplied by somebody else.

```swift
@ObservedObject var model: ProfileModel
```

Conceptually:

```text
Parent
  └── owns model
       ↓
     Child
```

Apple's current model-data documentation still documents these types because they are relevant to existing `ObservableObject` architectures.

---

# 15. Modern Observation

Modern Swift provides the **Observation** framework.

The key API is:

```swift
@Observable
```

Example:

```swift
import Observation

@Observable
final class ProfileModel {
    var name = ""
    var isLoading = false
}
```

There is no need to write:

```swift
ObservableObject
```

and there is no need to annotate every property with:

```swift
@Published
```

The macro generates the machinery needed for observation at compile time. Apple describes Observation as a type-safe mechanism for tracking changes in observable instances, with `@Observable` providing `Observable` conformance through a macro.

---

# 16. Why `@Observable` Is Different

This:

```swift
@Observable
final class ProfileModel {
    var name = ""
    var age = 30
}
```

is conceptually much closer to:

```text
"Make this type observable."
```

than:

```text
"Make these individual properties publish changes."
```

Swift's Observation machinery tracks which properties are actually accessed by an observer.

For example:

```swift
struct ProfileHeader: View {
    let model: ProfileModel

    var body: some View {
        Text(model.name)
    }
}
```

This view depends on:

```text
model.name
```

not necessarily every property of `ProfileModel`.

Apple's Observation presentation specifically describes SwiftUI tracking property accesses during `body` evaluation, allowing more targeted update behavior.

---

# 17. Modern SwiftUI with `@Observable`

A simple model:

```swift
import Observation

@Observable
final class CounterModel {
    var count = 0

    func increment() {
        count += 1
    }
}
```

A view can own it with `@State`:

```swift
struct CounterView: View {
    @State private var model = CounterModel()

    var body: some View {
        VStack {
            Text("\(model.count)")

            Button("Increment") {
                model.increment()
            }
        }
    }
}
```

This is an important modern pattern:

```text
@State
   ↓
owns observable model instance
   ↓
@Observable model
   ↓
SwiftUI observes properties used by the view
```

---

# 18. `@Bindable`

Modern Observation also introduces an elegant solution for bindings into observable models.

Suppose:

```swift
@Observable
final class ProfileModel {
    var name = ""
    var notificationsEnabled = true
}
```

An editor can use:

```swift
struct ProfileEditor: View {
    @Bindable var model: ProfileModel

    var body: some View {
        Form {
            TextField("Name", text: $model.name)

            Toggle(
                "Notifications",
                isOn: $model.notificationsEnabled
            )
        }
    }
}
```

`@Bindable` creates bindings to mutable properties of an `Observable` model. Apple documents it specifically for this purpose.

This gives a useful modern pattern:

```text
Observable model
       ↓
    @Bindable
       ↓
TextField / Toggle / Picker
```

---

# 19. `@ObservationIgnored`

Observation can also be selectively disabled.

```swift
@Observable
final class AppModel {
    var username = ""

    @ObservationIgnored
    var cache = SomeCache()
}
```

`@ObservationIgnored` tells the Observation system not to track that property.

This is useful for implementation details that should not participate in UI observation.

---

# 20. State Management: The Practical Decision Tree

When choosing a state mechanism, ask:

### Is it simple state owned by this view?

Use:

```swift
@State
```

### Does a child need to edit that state?

Use:

```swift
@Binding
```

### Is it a modern observable model?

Use:

```swift
@Observable
```

with appropriate ownership, often:

```swift
@State
```

### Does a child need bindings to the observable model?

Use:

```swift
@Bindable
```

### Are you maintaining older Combine-based SwiftUI?

You will encounter:

```swift
ObservableObject
@Published
@StateObject
@ObservedObject
```

### Is the value an application-wide dependency?

Consider:

```swift
@Environment
```

or dependency injection rather than automatically turning everything into global state.

---

# 21. State Flow

A well-structured SwiftUI feature often looks like:

```text
                 ┌───────────────┐
                 │ Observable    │
                 │ Model         │
                 └───────┬───────┘
                         │
                  state flows down
                         ↓
                 ┌───────────────┐
                 │ Screen        │
                 └───────┬───────┘
                         │
              ┌──────────┴──────────┐
              ↓                     ↓
        Child View A          Child View B
              │                     │
              └──────────┬──────────┘
                         ↓
                     Events
                         ↓
                  Model methods
                         ↓
                  State changes
```

This is essentially **unidirectional data flow**.

```text
State ↓
Events ↑
```

Compose explicitly recommends this pattern, and React's component model similarly relies on data flowing down and events updating the owner of state.

---

# 22. Model UI State Explicitly

For production applications, avoid creating a loose collection of unrelated Boolean flags when the screen really represents distinct states.

Instead of:

```swift
var isLoading = false
var hasError = false
var hasContent = false
```

consider a state model:

```swift
enum ScreenState {
    case loading
    case loaded([Product])
    case failed(Error)
}
```

Then:

```swift
switch model.state {
case .loading:
    ProgressView()

case .loaded(let products):
    ProductList(products: products)

case .failed:
    ErrorView()
}
```

The advantage is that the possible UI states become explicit.

```text
Loading
   ↓
Loaded

Loading
   ↓
Failed

Loaded
   ↓
Loading
```

This becomes especially useful as screens become more complex.

---

# 23. Architecture

Declarative UI does not eliminate architecture.

It changes where architecture lives.

A useful production model is:

```text
┌──────────────────────────┐
│          View            │
│     presentation/UI      │
└────────────┬─────────────┘
             │
             ↓
┌──────────────────────────┐
│   Observable Model       │
│ state + UI behavior      │
└────────────┬─────────────┘
             │
             ↓
┌──────────────────────────┐
│       Services           │
│ API / database / auth    │
└────────────┬─────────────┘
             │
             ↓
┌──────────────────────────┐
│       Data Layer         │
└──────────────────────────┘
```

This isn't a requirement to use one particular architecture such as MVVM.

The more important principle is separation of responsibilities.

---

# 24. MVVM in SwiftUI

A traditional SwiftUI MVVM arrangement might look like:

```text
ProductView
     ↓
ProductViewModel
     ↓
ProductService
     ↓
API
```

For example:

```swift
@Observable
final class ProductViewModel {
    var products: [Product] = []
    var isLoading = false
    var errorMessage: String?

    private let service: ProductService

    init(service: ProductService) {
        self.service = service
    }

    func load() async {
        isLoading = true
        defer { isLoading = false }

        do {
            products = try await service.fetchProducts()
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}
```

And the view:

```swift
struct ProductListView: View {
    @State private var model: ProductViewModel

    init(service: ProductService) {
        _model = State(
            initialValue: ProductViewModel(service: service)
        )
    }

    var body: some View {
        Group {
            if model.isLoading {
                ProgressView()
            } else {
                List(model.products) { product in
                    Text(product.name)
                }
            }
        }
        .task {
            await model.load()
        }
    }
}
```

The view describes presentation; the model owns the screen's changing state and operations.

---

# 25. Layout Fundamentals

SwiftUI's basic layout vocabulary is small but powerful.

```text
VStack
HStack
ZStack
Spacer
frame
padding
alignment
ScrollView
List
Form
Grid
```

The system is based on relationships rather than absolute coordinates.

---

# 26. `VStack`

Vertical composition:

```swift
VStack(spacing: 12) {
    Text("Title")
    Text("Subtitle")
    Button("Continue") {}
}
```

Visual model:

```text
Title
Subtitle
Continue
```

---

# 27. `HStack`

Horizontal composition:

```swift
HStack {
    Text("Settings")

    Spacer()

    Image(systemName: "chevron.right")
}
```

Visual model:

```text
Settings                       >
```

---

# 28. `ZStack`

Layered composition:

```swift
ZStack {
    Image("background")
        .resizable()
        .scaledToFill()

    Text("Welcome")
        .font(.largeTitle.bold())
}
```

Useful for:

```text
Overlays
Badges
Image captions
Floating controls
Backgrounds
Loading overlays
```

---

# 29. Alignment and Spacing

Prefer relationships:

```swift
VStack(alignment: .leading, spacing: 8) {
    Text("Account")
    Text("Manage your profile")
}
```

instead of coordinates.

Think:

```text
alignment = relationship
spacing   = relationship
padding   = relationship
frame     = constraint
```

That mindset leads to much more resilient interfaces.

---

# 30. Flexible Layout

Prefer:

```swift
.frame(maxWidth: .infinity)
```

over:

```swift
.frame(width: 390)
```

For example:

```swift
Button("Continue") {
    continueAction()
}
.frame(maxWidth: .infinity)
```

The first adapts naturally to different devices and containers.

Hard-coded device dimensions tend to break with:

```text
iPad
landscape
Split View
Stage Manager
Dynamic Type
localization
different device sizes
```

---

# 31. Safe Areas

SwiftUI normally manages safe areas automatically.

For a background extending edge-to-edge:

```swift
ZStack {
    Color.blue
        .ignoresSafeArea()

    Text("Welcome")
}
```

Use edge-to-edge behavior intentionally.

A common pattern is:

```text
Background → edge-to-edge
Content    → respects safe area
```

rather than ignoring the safe area for everything.

---

# 32. ScrollView

Use `ScrollView` for custom scrollable layouts.

```swift
ScrollView {
    VStack(spacing: 20) {
        ForEach(items) { item in
            ItemCard(item: item)
        }
    }
    .padding()
}
```

For large collections:

```swift
ScrollView {
    LazyVStack {
        ForEach(items) { item in
            ItemRow(item: item)
        }
    }
}
```

Lazy containers allow child content to be created on demand as it becomes relevant to the scrolling viewport. Apple recommends choosing ordinary or lazy containers based on the actual layout and performance characteristics rather than defaulting to lazy containers everywhere.

---

# 33. Grids

Two-dimensional layouts:

```swift
let columns = [
    GridItem(.flexible()),
    GridItem(.flexible())
]

LazyVGrid(columns: columns, spacing: 16) {
    ForEach(products) { product in
        ProductCard(product: product)
    }
}
```

Good for:

```text
Photo galleries
Product catalogues
Dashboards
Feature tiles
```

---

# 34. List

`List` provides a platform-standard list experience.

```swift
List {
    Section("Account") {
        Text("Profile")
        Text("Security")
        Text("Notifications")
    }
}
```

Dynamic:

```swift
List(products) { product in
    Text(product.name)
}
```

Use `List` when standard list semantics and behavior are desirable.

Use `ScrollView` plus stacks/grids when you need more custom composition.

---

# 35. Form

`Form` is particularly useful for preferences and data entry:

```swift
Form {
    Section("Account") {
        TextField("Name", text: $name)
        TextField("Email", text: $email)
    }

    Section("Preferences") {
        Toggle("Notifications", isOn: $notifications)
    }
}
```

It provides platform-aware presentation and behavior appropriate to forms.

---

# 36. Core Text UI

### `Text`

```swift
Text("Hello")
    .font(.title)
    .foregroundStyle(.secondary)
```

Common styles:

```swift
.font(.largeTitle)
.font(.title)
.font(.headline)
.font(.body)
.font(.caption)
```

### `Label`

```swift
Label("Favorites", systemImage: "star.fill")
```

### `Image`

Asset:

```swift
Image("avatar")
```

SF Symbol:

```swift
Image(systemName: "person.circle.fill")
```

Resizable:

```swift
Image("avatar")
    .resizable()
    .scaledToFill()
    .clipShape(Circle())
```

---

# 37. Buttons

Basic:

```swift
Button("Save") {
    save()
}
```

Role:

```swift
Button("Delete", role: .destructive) {
    delete()
}
```

Custom:

```swift
Button {
    share()
} label: {
    Label("Share", systemImage: "square.and.arrow.up")
}
```

Built-in styles:

```swift
.buttonStyle(.bordered)
.buttonStyle(.borderedProminent)
.buttonStyle(.plain)
```

Prefer semantic buttons over attaching gestures to arbitrary visual elements.

---

# 38. Text Input

```swift
@State private var name = ""

TextField("Name", text: $name)
```

Secure input:

```swift
@State private var password = ""

SecureField("Password", text: $password)
```

The control receives a binding so that:

```text
User types
   ↓
Binding writes state
   ↓
State changes
   ↓
UI reflects state
```

---

# 39. Toggles

```swift
@State private var enabled = true

Toggle(
    "Notifications",
    isOn: $enabled
)
```

The binding is the important part:

```text
Toggle ↔ Boolean state
```

---

# 40. Pickers

```swift
Picker("Theme", selection: $theme) {
    Text("System").tag(Theme.system)
    Text("Light").tag(Theme.light)
    Text("Dark").tag(Theme.dark)
}
```

A picker is another example of the general declarative pattern:

```text
Current selection
       ↓
      UI
       ↑
User selects
       ↓
Binding updates state
```

---

# 41. Slider

```swift
@State private var volume = 0.5

Slider(
    value: $volume,
    in: 0...1
)
```

The UI does not maintain a separate "volume slider state" from your model.

The binding connects the control directly to the source of truth.

---

# 42. Stepper

```swift
@State private var quantity = 1

Stepper(
    "Quantity: \(quantity)",
    value: $quantity,
    in: 1...10
)
```

---

# 43. DatePicker

```swift
@State private var date = Date()

DatePicker(
    "Date",
    selection: $date,
    displayedComponents: [.date]
)
```

---

# 44. ProgressView

Indeterminate:

```swift
ProgressView()
```

Determinate:

```swift
ProgressView(
    value: progress,
    total: 100
)
```

Model the loading state rather than merely showing a spinner from arbitrary locations in the view tree.

---

# 45. Navigation

Modern SwiftUI navigation is built around `NavigationStack`.

```swift
NavigationStack {
    List(products) { product in
        NavigationLink(product.name, value: product)
    }
    .navigationDestination(for: Product.self) { product in
        ProductDetailView(product: product)
    }
    .navigationTitle("Products")
}
```

The conceptual model is:

```text
NavigationStack
      │
      ├── Product A
      │      └── Detail
      │
      └── Product B
             └── Detail
```

Navigation can also be modeled explicitly as state, which makes deep links and programmatic navigation easier to reason about.

---

# 46. Sheets

For secondary presentation:

```swift
@State private var showingEditor = false

Button("Edit") {
    showingEditor = true
}
.sheet(isPresented: $showingEditor) {
    EditorView()
}
```

Data-driven presentation:

```swift
.sheet(item: $selectedProduct) { product in
    ProductEditor(product: product)
}
```

The second form often communicates intent more clearly because the presented item itself is the source of truth.

---

# 47. Alerts

```swift
@State private var showError = false

.alert(
    "Something went wrong",
    isPresented: $showError
) {
    Button("OK", role: .cancel) {}
}
```

---

# 48. Confirmation Dialogs

```swift
.confirmationDialog(
    "Choose an action",
    isPresented: $showActions
) {
    Button("Archive") {
        archive()
    }

    Button("Delete", role: .destructive) {
        delete()
    }

    Button("Cancel", role: .cancel) {}
}
```

These presentation APIs fit the same declarative pattern:

```text
State
 ↓
presentation exists / does not exist
```

---

# 49. Environment

Some information naturally belongs to the surrounding UI context.

```swift
@Environment(\.colorScheme)
private var colorScheme
```

Observable models can also be made available through the environment:

```swift
@State private var appModel = AppModel()

var body: some View {
    ContentView()
        .environment(appModel)
}
```

A descendant can read it:

```swift
@Environment(AppModel.self)
private var appModel
```

Environment is useful for shared contextual data and dependencies that many descendants require.

It should not become a dumping ground for every piece of application state.

---

# 50. Async Work

SwiftUI integrates naturally with Swift concurrency.

```swift
struct ProductListView: View {
    @State private var products: [Product] = []

    var body: some View {
        List(products) { product in
            Text(product.name)
        }
        .task {
            products = await loadProducts()
        }
    }
}
```

A more scalable architecture moves the operation into the model:

```swift
.task {
    await model.load()
}
```

The view describes the lifecycle relationship; the model performs the business operation.

---

# 51. Architecture by Responsibility

A useful production boundary is:

```text
View
│
├── renders
├── binds controls
├── handles presentation
└── sends user events
         ↓
Observable Model
│
├── owns screen state
├── coordinates UI behavior
└── calls services
         ↓
Services
│
├── networking
├── authentication
├── persistence
└── external systems
```

The exact names do not matter.

The boundaries do.

---

# 52. Feature-Based Project Structure

A scalable project might look like:

```text
App
│
├── Features
│   ├── Home
│   │   ├── HomeView.swift
│   │   ├── HomeModel.swift
│   │   └── Components/
│   │
│   ├── Profile
│   │   ├── ProfileView.swift
│   │   ├── ProfileModel.swift
│   │   └── Components/
│   │
│   └── Settings
│
├── Core
│   ├── Networking
│   ├── Persistence
│   └── Authentication
│
├── DesignSystem
│   ├── AppButton
│   ├── AppCard
│   └── AppTypography
│
└── Resources
```

Organizing around **features** often scales better than placing every view, model, and service from the entire application into separate global folders.

---

# 53. Component Design

A reusable SwiftUI component should expose configuration rather than application internals.

Good:

```swift
struct PrimaryButton: View {
    let title: String
    let action: () -> Void

    var body: some View {
        Button(title, action: action)
            .buttonStyle(.borderedProminent)
    }
}
```

Avoid embedding business logic:

```swift
// Avoid making a generic UI component
// responsible for networking, authentication,
// persistence, etc.
```

A component should generally answer:

```text
"What should this UI look like?"
```

rather than:

```text
"How does the whole feature work?"
```

---

# 54. Accessibility

Production SwiftUI should be designed semantically.

Example:

```swift
Image("profile")
    .accessibilityLabel("Profile photo")
```

For a value:

```swift
Text("75%")
    .accessibilityLabel("Battery level")
    .accessibilityValue("75 percent")
```

Prefer:

```swift
Button("Delete") {
    delete()
}
```

rather than:

```swift
Text("Delete")
    .onTapGesture {
        delete()
    }
```

Standard controls carry semantic information that assistive technologies can understand.

Also test:

```text
VoiceOver
Dynamic Type
high contrast
different text lengths
localization
reduced motion
iPad layouts
landscape
```

---

# 55. Animation

SwiftUI animations are usually state-driven.

```swift
@State private var expanded = false

var body: some View {
    VStack {
        if expanded {
            Text("Additional information")
        }

        Button("Toggle") {
            withAnimation {
                expanded.toggle()
            }
        }
    }
}
```

The mental model is:

```text
Old state
   ↓
State transition
   ↓
New UI description
   ↓
Animation
   ↓
Visual transition
```

Rather than manually moving UI objects from point A to point B, describe the new state and let SwiftUI animate the transition.

---

# 56. Persistence

For simple user preferences:

```swift
@AppStorage("selectedTheme")
private var selectedTheme = "system"
```

For scene-specific UI restoration:

```swift
@SceneStorage("selectedTab")
private var selectedTab = 0
```

Neither should be treated as a general database.

For substantial application data, use an appropriate persistence technology and keep that concern outside the view layer.

---

# 57. Testing the State Model

Declarative UI becomes particularly powerful when UI state is explicit.

Instead of testing:

```text
tap button
wait
find label
check label
```

you can reason about:

```text
Given Loading
When request succeeds
Then state = Loaded(products)
```

and:

```text
Given Loaded(products)
When delete succeeds
Then state = Loaded(updatedProducts)
```

This encourages testable business logic independent of the rendering layer.

---

# 58. Common SwiftUI Mistakes

### Treating `@State` as a global data store

Keep state close to where it is owned and consumed.

### Duplicating state

Avoid:

```swift
model.items
view.items
```

when both represent the same source of truth.

### Storing derived data

Prefer:

```swift
var itemCount: Int {
    items.count
}
```

over maintaining a second mutable count.

### Using hard-coded dimensions

Prefer flexible layout constraints.

### Building everything with `GeometryReader`

Start with stacks, alignment, frames, grids, and standard layout APIs.

### Making views responsible for networking

Move API/database operations into services or feature models.

### Keeping all feature state in environment

Environment is useful, but explicit ownership and dependencies are usually easier to reason about.

### Making generic components too intelligent

A button should generally be a button, not an entire business layer.

---

# 59. A Complete Modern Example

Consider a product screen.

```swift
import SwiftUI
import Observation

@Observable
final class ProductListModel {
    var products: [Product] = []
    var isLoading = false
    var errorMessage: String?

    private let service: ProductService

    init(service: ProductService) {
        self.service = service
    }

    func load() async {
        isLoading = true
        errorMessage = nil

        defer {
            isLoading = false
        }

        do {
            products = try await service.fetchProducts()
        } catch {
            errorMessage = "Unable to load products."
        }
    }
}
```

Then:

```swift
struct ProductListView: View {
    @State private var model: ProductListModel

    init(service: ProductService) {
        _model = State(
            initialValue: ProductListModel(service: service)
        )
    }

    var body: some View {
        NavigationStack {
            content
                .navigationTitle("Products")
                .task {
                    await model.load()
                }
        }
    }

    @ViewBuilder
    private var content: some View {
        if model.isLoading {
            ProgressView()
        } else if let error = model.errorMessage {
            ContentUnavailableView(
                "Unable to Load Products",
                systemImage: "exclamationmark.triangle",
                description: Text(error)
            )
        } else {
            List(model.products) { product in
                ProductRow(product: product)
            }
        }
    }
}
```

The architecture is:

```text
ProductListView
       ↓
ProductListModel
       ↓
ProductService
       ↓
API
```

The UI knows how to render states.

The model knows how the screen behaves.

The service knows how data is retrieved.

---

# 60. The Three Most Important Comparisons

## SwiftUI vs Jetpack Compose

Both are native declarative UI systems.

```text
SwiftUI:
View + State + Binding + Observation

Compose:
Composable + State + Events + StateFlow/ViewModel
```

Compose's official architecture guidance similarly separates UI rendering from state holders and recommends unidirectional data flow.

---

## SwiftUI vs React

React's conceptual model is:

```text
Props
State
Components
Events
Render
```

SwiftUI's equivalent vocabulary is approximately:

```text
Input properties
State
Views
Bindings/actions
View description
```

React explicitly treats props as data passed to a component and state as its changing memory; SwiftUI separates input values, bindings, and state ownership in a similar conceptual way.

The implementation details are different, particularly because SwiftUI is integrated tightly with Swift's type system, property wrappers, Observation, and Apple's native UI platforms.

---

# 61. The Observation Transition

For someone joining an existing SwiftUI project, expect to see both architectures.

### Legacy

```swift
final class UserModel: ObservableObject {
    @Published var name = ""
}
```

with:

```swift
@StateObject
@ObservedObject
```

### Modern

```swift
@Observable
final class UserModel {
    var name = ""
}
```

with modern ownership/binding patterns such as:

```swift
@State
@Bindable
@Environment
```

The modern model removes a considerable amount of boilerplate and makes observable models look more like ordinary Swift types. Apple introduced this approach with the Observation framework and provides migration guidance from `ObservableObject` to `Observable`.

---

# 62. Professional SwiftUI Workflow

When implementing a feature:

## Step 1 — Define the state

Write down everything that can change.

```text
searchText
products
loading
error
selectedProduct
```

## Step 2 — Remove derived state

Ask:

```text
Can this value be calculated from existing state?
```

If yes, don't store it independently.

## Step 3 — Establish ownership

Ask:

```text
Who owns this state?
Who needs to read it?
Who needs to mutate it?
```

## Step 4 — Model the UI states

For example:

```text
Loading
Loaded
Empty
Error
```

## Step 5 — Build the layout

Start structurally:

```text
NavigationStack
    ↓
ScrollView / List
    ↓
Stacks / Grid
    ↓
Components
```

## Step 6 — Add bindings and events

Connect controls to the source of truth.

## Step 7 — Add presentation

```text
sheet
alert
confirmationDialog
navigationDestination
```

## Step 8 — Add visual design

```text
Typography
Spacing
Materials
Shapes
Colors
Icons
```

## Step 9 — Test state transitions

Test the model and the UI separately where practical.

## Step 10 — Test adaptability

Always consider:

```text
iPhone
iPad
Landscape
Dark Mode
Dynamic Type
Accessibility
Localization
Empty state
Loading state
Error state
Slow network
```

---

# 63. The Mental Model to Keep

The syntax can be remembered later.

The core system should be understood now:

```text
                 ┌──────────────┐
                 │    STATE     │
                 └──────┬───────┘
                        │
                        ↓
                 ┌──────────────┐
                 │     VIEW     │
                 │  describes   │
                 │     UI       │
                 └──────┬───────┘
                        │
                        ↓
                 ┌──────────────┐
                 │   RENDERED   │
                 │      UI      │
                 └──────┬───────┘
                        │
                    user event
                        │
                        ↓
                 ┌──────────────┐
                 │     MODEL    │
                 │ / state owner│
                 └──────┬───────┘
                        │
                        └──────────→ new state
```

And the architectural principles are:

```text
One source of truth
        +
Clear state ownership
        +
Unidirectional data flow
        +
Composable views
        +
Semantic controls
        +
Observable models
        =
Maintainable declarative UI
```

---

# 64. Final Takeaway

Learning SwiftUI effectively is less about memorizing hundreds of modifiers and more about understanding a small set of ideas extremely well:

**Declarative UI**
Describe the UI for the current state instead of manually mutating views.

**Composition**
Build screens from small, focused views.

**State ownership**
Every mutable value should have a clear source of truth.

**Binding**
Create explicit read/write connections between controls and their state.

**Observation**
Make model changes visible to the UI.

**Unidirectional flow**
State moves toward the UI; user events move toward the state owner.

**Layout relationships**
Use stacks, alignment, spacing, constraints and flexible sizing rather than coordinates.

**Architecture**
Keep presentation, state, business logic, services and persistence appropriately separated.

For modern SwiftUI development, the important evolution to internalize is:

```text
UIKit / imperative UI
        ↓
SwiftUI / declarative UI
        ↓
ObservableObject + @Published
        ↓
Observation + @Observable
        ↓
State-driven, observable application architecture
```

Once that mental model is solid, `@State`, `@Binding`, `@Observable`, `@Bindable`, `NavigationStack`, `List`, `Form`, `ScrollView`, and the rest of SwiftUI become tools within a coherent system rather than isolated APIs.
