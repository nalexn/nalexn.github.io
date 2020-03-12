---
layout: post
title: "Why I quit using the ObservableObject"
date: 2019-12-20 8:30:00 +0300
description: "There is a better option for Redux-style state-managed apps."
tags: [iOS,Swift,SwiftUI,Combine,Redux,Publisher,ObservedObject,EnvironmentObject,Snapshot,EquatableView,objectWillChange]
comments: true
sharing: true
published: true
img: observable_001.jpg
---

"Single source of truth" has become a buzzword in the iOS community after WWDC 2019 [Data Flow Through SwiftUI](https://developer.apple.com/videos/play/wwdc2019/226/) session.

SwiftUI framework was designed to encourage building the apps in the single-source-of-truth style, be that Redux-like centralized app state or ViewModels serving the data only to their views.

It feels natural to use `@ObservedObject` or `@EnvironmentObject` for the state management of such views, as the `ObservableObjects` fit the design really well.

But there is a little problem.

As I've been [exploring](https://nalexn.github.io/anyview-vs-group/) how SwiftUI works under high load, I discovered severe degrade of the performance proportional to the number of views being subscribed on the state update.

We can have a couple of thousands of views on the screen with just one being subscribed - the update is rendered lightning-fast, even for a view deep inside the hierarchy.

But it's sufficient to have just a few hundreds of views **subscribed** on the same update (and not being factually changed) - and you'll notice a significant performance drawdown.

This means if we're building a large SwiftUI app based on a Redux-like centralized state, we're likely to be in big trouble!

## Wrapping every view in EquatableView

At first glance, `EquatableView` looks like the perfect candidate for solving this problem.

It allows for writing a custom diffing strategy for the views, specifically, comparing the state instead of comparing the `body`.

But even if the [mystical undocumented behavior](https://swiftui-lab.com/equatableview/) of `EquatableView` is addressed someday in the future, we still won't be able to compare views that reference mutating state objects, such as `EnvironmentObject` or `ObservedObject`.

Let me explain why. Consider this simple example:

```swift
class AppState: ObservableObject {
    @Published var value: Int = 0
}

struct CustomView: View {
    @EnvironmentObject var appState: AppState
    
    var body: some View {
        Text("Value: \(appState.value)")
    }
}
```
We're opting out of the default SwiftUI diffing strategy by conforming to `Equatable`:

```swift
extension CustomView: Equatable {
    static func == (lhs: Self, rhs: Self) -> Bool {
        return lhs.appState.value == rhs.appState.value
    }
}
```
...and wrapping the view in `EquatableView`:

```swift
CustomView().equatable().environmentObject(AppState())
```

Now everything should be good, right?

You run the code and see that things got only worse:
now the view freezes in its initial state and never gets redrawn.

So, what's going on?

While `lhs` and `rhs` are two distinct instances of the `CustomView` struct, both copies are referencing the same shared object.

Because `AppState` is a [reference type](https://developer.apple.com/swift/blog/?id=10), SwiftUI won't copy it upon mutation, so you're basically comparing object instance to itself.

The `==` func always returns `true`, telling SwiftUI that our view does not require re-calculation. Never.

## Snapshotting the state in the view

Ok, since we cannot rely on comparing references to the `ObservableObjects`, how about storing the snapshot of the previous state in the view when it receives an update?

Something like this:

```swift
struct CustomView: View {
    @EnvironmentObject var appState: AppState
    private var prevValue: Int
    
    var body: some View {
        self.prevValue = appState.value
        return Text("Value: \(appState.value)")
    }
}

extension CustomView: Equatable {
    static func == (lhs: Self, rhs: Self) -> Bool {
        return lhs.prevValue == rhs.appState.value
    }
}
```

In the `==` func we are comparing the `prevValue` to the updated `appState.value`, so this should work fine...

But it doesn't. The reason - this simply won't compile. `body` is immutable, so we're not allowed to set `prevValue` in it.

There is a workaround to this problem - we can create a reference type wrapper for storing the `prevValue`, but this all starts to be really cumbersome and smelly. In addition to that, the `==` func is [not always called](https://swiftui-lab.com/equatableview/), making this approach worthless.

How about something more elegant?

## Filtering the state updates

Prior to the release of SwiftUI and Combine frameworks I had a chance to try Redux state management on a large-scale UIKit app, and the combination of [ReSwift](https://github.com/ReSwift/ReSwift) with [RxSwift](https://github.com/ReactiveX/RxSwift) worked really well.

The problem of massive state updates was not so apparent, but I still used filtering of the updates in the pipeline to reduce the load:

```swift
BehaviorRelay(value: AppState()) // produces all state updates
    .map { $0.value1 } // removing all unused state values
    .distinctUntilChanged() // remove duplicated "value1"
    .bind(to: ...)
```

There is a function `distinctUntilChanged` is RxSwift (aka `skipRepeats` in ReactiveSwift and `removeDuplicates` in Combine) that allows for dropping the unnecessary update events when neither of the values used in the specific view got changed.

This approach would work for SwiftUI as well, but data bindings produced by `@State`, `@ObservedObject` and `@EnvironmentObject` don't have this filtering functionality.

To me, it was surprising that `Publisher` from Combine is incompatible with `Binding` from SwiftUI, and there are literally just a couple of weird ways to connect them.

## Wrapping the ObservableObject in ObservableObject

<div style="max-width:500px; display: block; margin-left: auto; margin-right: auto;"><img src="{{ site.url }}/assets/img/xzibit_01.png"></div>

If we want to stick with the `ObservableObject`, there is literally no way to skip an event from the inner `objectWillChange` publisher, because SwiftUI subscribes to it directly.

What we can do is to wrap the `ObservableObject` in another `ObservableObject` that does the filtering under the hood.

We can make this wrapper generic and highly reusable. `@dynamicMemberLookup` attribute allows the client code to interact with the wrapper just like with the genuine object.

You can find the implementation of `Deduplicated` on Github [gist](https://gist.github.com/nalexn/ace9ddd07db5a6e150163712e20c6235), but here is the conceptual part:

```swift
object.objectWillChange
    .delay(for: .nanoseconds(1), scheduler: RunLoop.main)
    .compactMap { _ in makeSnapshot() }
    .prepend(makeSnapshot())
    .removeDuplicates()
    .dropFirst()
    .sink { [weak self] _ in
        self?.objectWillChange.send()
    }
```

It is observing the `objectWillChange` of the original object, takes the snapshot of `AppState` by only including the values used in the specific view and then removes the duplicated values. In the end, if the snapshot is different than the previous one, it triggers `objectWillChange` on the wrapper object used by the view.

On the consumer's side, what we had:

```swift
struct CustomView: View {
    @EnvironmentObject var appState: AppState
    
    var body: some View {
        Text("Value: \(appState.value)")
    }
}

let view = CustomView()
    .environmentObject(appState)
```

...can be reworked in this manner:

```swift
struct CustomView: View {
    @EnvironmentObject var appState: Deduplicated<AppState, Snapshot>
    
    var body: some View {
        Text("Value: \(appState.value)")
    }
}

let view = CustomView()
    .environmentObject(appState
        .deduplicated { Snapshot(value: $0.value) })
        
extension CustomView {
    struct Snapshot {
        let value: Int
    }
}
```

And that's it! Now `AppState` can generate tons of updates, but only the ones containing different `.value` will be forwarded to the view.

I should note two downsides of this approach:

1. State update is delivered asynchronously due to the `.delay` call. This could be a gate for numerous bugs related to race conditions, but we're not in UIKit. If another update comes out of the blue, the view will just get re-calculated twice and end up reflecting the most recent state values.
2. The `Snapshot` type must be unique for each screen consuming `Deduplicated` as an `@EnvironmentObject`. This is required because otherwise there might be a conflict when injecting multiple objects with `.environmentObject(_:)` modifiers.

## Using Publisher instead of the ObservableObject

You know what, I've had enough of `ObservableObject`!

Seriously, the currently available API for `Binding` provides absolutely no control over the values flow.

You constantly need to bridge between it and Publishers from Combine, which are naturally used for networking and other asynchronous business logic.

I cannot see any use case where `@ObservedObject` or `@EnvironmentObject` would outpace the solution I'm about to propose when dealing with a centralized app state.

Let's dive in:

```swift
struct ContentView: View {
    
    // The local view's state encapsulated in one container:
    @State private var state = ViewState()
    
    // The app's state injection
    @Environment(\.injected) private var injected: AppState.Injection
    
    var body: some View {
        Text("Value: \(state.value)")
            .onReceive(stateUpdate) { self.state = $0 }
    }
    
    // The state update filtering
    private var stateUpdate: AnyPublisher<ViewState, Never> {
        injected.appState.map { $0.viewState }
            .removeDuplicates().eraseToAnyPublisher()
    }
}

// Container for the local view state encapsulation
private extension ContentView {
    struct ViewState: Equatable {
        var value: Int = 0
    }
}

// Convenient mapping from AppState to ViewState
private extension AppState {
    var viewState: ContentView.ViewState {
        return .init(value: value1)
    }
}
```

This approach has many benefits:

1. We're not only filtering the updates but also limiting the access to the values defined in the local `ViewState` struct.
2. We still benefit from using the native dependency injection with `@Environment` being used instead of `@EnvironmentObject`.
3. The same `injected` container can be extended for injecting services.
4. The updates are delivered synchronously.
5. The code is more concise and clear than before.
6. Easily scalable. No need to update the root dependency injection when adding a new view.

Just to complete the example with the full code, here is the `AppState` and its injection:

```swift
struct AppState {
    var value1: Int = 0
    var value2: Int = 0
    var value3: Int = 0
}

extension AppState {
    struct Injection: EnvironmentKey {
        let appState: CurrentValueSubject<AppState, Never>
        static var defaultValue: Self {
            return .init(appState: .init(AppState()))
        }
    }
}

extension EnvironmentValues {
    var injected: AppState.Injection {
        get { self[AppState.Injection.self] }
        set { self[AppState.Injection.self] = newValue }
    }
}

let injected = AppState.Injection(appState: .init(AppState()))
let contentView = ContentView().environment(\.injected, injected)
```

I've tried several ways to implement the reversed data flow from the standard SwiftUI views back to the `AppState`. The one that finally worked well was wrapping the `Binding` submitted to the SwiftUI's view into the middleware that forwards the values to the `AppState`:

```swift
extension Binding where Value: Equatable {
    func dispatched(to state: CurrentValueSubject<AppState, Never>,
                    _ keyPath: WritableKeyPath<AppState, Value>) -> Self {
        return .init(get: { () -> Value in
            self.wrappedValue
        }, set: { newValue in
            self.wrappedValue = newValue
            state.value[keyPath: keyPath] = newValue
        })
    }
}

var body: some View {
    ...
    .sheet(isPresented: viewState.showSheet.dispatched(
                            to: appState, \.routing.showSheet),
               content: { ... })
}
```

There are some ways how this whole solution can be improved syntactically (most notably by using `keyPaths`), but conceptually it is a more performant alternative to both `@EnvironmentObject` and `@ObservedObject`.

For the latter, you'd be injecting `CurrentValueSubject` as the `init` parameter of the view, in place of the object.

You can refer to the [sample project](https://github.com/nalexn/clean-architecture-swiftui) where I use the described approach.

If you're trying to wrap your head around SwiftUI's `ObservableObject`, `@ObservedObject` and other fancy constructions, I'd highly recommend you my other article ["Stranger things around SwiftUI's state"](https://nalexn.github.io/stranger-things-swiftui-state/).