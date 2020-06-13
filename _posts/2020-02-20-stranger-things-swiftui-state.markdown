---
layout: post
title: "Stranger things around SwiftUI's state"
date: 2020-02-20 10:30:00 +0300
description: "How do @Published, @State, @Binding, ObserveableObject, @ObservedObject, and @EnvironmentObject relate to each other?"
tags: [observableObject,observedObject,published,state,binding,property wrapper,wrappedValue,projectedValue]
comments: true
sharing: true
published: true
img: forest_001.jpg
---

Like many other developers, I began my practical acquaintance with SwiftUI from Apple's [wonderful tutorial](https://developer.apple.com/tutorials/swiftui/), which, however, instilled me a false belief that SwiftUI is a breeze to learn.

As it appeared to me later, SwiftUI has many tricky topics that can be a challenge to figure out quickly; even experienced developers [say](https://twitter.com/a_grebenyuk/status/1229064810962259968?s=20) it feels like learning everything from scratch.

In this article, I collected my top confusing aspects of SwiftUI related to state management. It'd saved me hours of painful troubleshooting if I had this article on hands...

Let's get started!

# @State, @Published, @ObservedObject, etc.

In the beginning, I perceived these `@Something` as a bunch of brand new *language attributes*, like `weak` or `lazy`, introduced specifically for SwiftUI.

So I was confused why these new "keywords" allow the variable to morph depending on the prefix:

`value`, `$value`, and `_value` are representing three completely different things!

I lacked the understanding that these `@Things` are nothing more than just a few structs declared in the SwiftUI framework and are not part of the language.

What is indeed part of the language is the feature of Swift 5.1: [property wrappers](https://github.com/apple/swift-evolution/blob/master/proposals/0258-property-wrappers.md).

Only after [reading](https://docs.swift.org/swift-book/LanguageGuide/Properties.html) about property wrappers I could finally understand that there is absolutely no mystery behind `@State` or `@Published`. These wrappers grant "superpowers" to the original variables, such as *nonmutating mutability* for `@State` or *Rx-ness* for `@Published`.

Got confused even more? No worries - I'll explain that in a minute.

The generalized picture appeared very clear: when we have a variable "attributed" with SwiftUI's `@Something`, for example, `@State var value: Int = 0`, the Swift compiler generates three (!) variables for us (two of which are computed variables):

`value` - the wrapped original value (`wrappedValue`) of the type we declared, `Int` in our example.

`$value` - an "additional" `projectedValue` that has an arbitrary type defined by the property wrapper we used. `@State`'s `projectedValue` has the type `Binding<Value>`, so for the example above we get `Binding<Int>`

`_value` - references the property wrapper struct itself. This might be useful in the view initialization:

```swift
struct MyView: View {
    @Binding var flag: Bool
    
    init(flag: Binding<Bool>) {
        self._flag = flag
    }
}
```

# Classifying the projected values

Let's go over the most common `@Things` used in SwiftUI and see what's their `projectedValue` are:

* `@State` - `Binding<Value>`
* `@Binding` - `Binding<Value>`
* `@ObservedObject` - `Binding<Value>` (*)
* `@EnvironmentObject` - `Binding<Value>` (*)
* `@Published` - `Publisher<Value, Never>`

(*) technically, we get an intermediary value of type `Wrapper`, which turns a `Binding<Value>` once we specify the keyPath to the actual value inside the object.

So, as you can see, the majority of the property wrappers in SwiftUI, namely responsible for the view's state, are being "projected" as `Binding`, which is used for passing the state between the views.

The only wrapper that diverges from the common course is `@Published`, but:

1. It's declared in Combine framework, not in SwiftUI
2. It serves a different purpose: making the value *observable*
3. It is never used for a view's variable declaration, only inside `ObservableObject`

Consider this pretty common scenario in SwiftUI, where we declare an `ObservableObject` and use it with `@ObservedObject` attribute in a view:

```swift
class ViewModel: ObservableObject {
    @Published var value: Int = 0
}

struct MyView: View {
    @ObservedObject var viewModel = ViewModel()
    
    var body: some View { ... }
}
```

`MyView` can refer to `$viewModel.value` and `viewModel.$value` - both expressions are correct. Quite confusing, isn't it?

These two expressions ultimately represent values of different types: `Binding` and `Publisher`, respectively.

Both have a practical use:

```swift
var body: some View {
    OtherView(binding: $viewModel.value)     // Binding
        .onReceive(viewModel.$value) { value // Publisher
            // do something that does not
            // require the view update
        }
}
```

# Schrödinger's @State

We all know that a `struct` contained in another *non-mutable* `struct` cannot be changed.

Most of the time in SwiftUI, we have to work with an immutable `self`, for example, inside a callback for a `Button`. From that context, every instance variable, including the `@State` struct is also immutable.

So can you explain why this code is perfectly legit then?

```swift
struct MyView: View {
    @State var counter: Int = 0
    
    var body: some View {
        Button(action: {
            self.counter += 1 // mutating an immutable struct!
        }, label: { Text("Tap me!") })
    }
}
```

Why are we allowed to mutate the value inside the `@State` even though it is an immutable struct, just like `self`?

Here is a [detailed explanation](https://forums.swift.org/t/why-i-can-mutate-state-var-how-does-state-property-wrapper-work-inside/27209) of how SwiftUI handles the value mutation in this scenario, but I want to note one important fact: **SwiftUI uses hidden external state storage** for keeping the actual values of `@State` variables.

`@State` is a proxy: it has an internal variable `_location` for accessing that external storage.

Here is an interview question from me: what will be the value printed out in this case:

```swift
func test() {
    var view = MyView()
    view.counter = 10
    print("\(view.counter)")
}
```

The code above is really straightforward; sanity tells us that the value should be 10.

But it's not. The output is 0.

The trick here is that the view is not always connected to that state store: SwiftUI does plug it in when the view needs a redraw or receives a SwiftUI-originated callback, but plugs it out afterward.

In the same manner, there is no guarantee that the `@State` mutation inside a `DispatchQueue.main.async` would work: sometimes it does. Still, if you introduce a delay, the store might get disconnected at the moment the closure is executed, and the state mutation won't take effect.

The traditional async dispatch is unsafe for a SwiftUI view - you're playing with fire when you do this.

# Phantom state updates

After using RxSwift and ReactiveSwift for several years, I was taking it for granted that the data streams are easily connectable to the view's properties through reactive bindings.

And this was a shocking experience trying to make SwiftUI and Combine work together.

These frameworks feel very foreign to each other: one does not simply connect a `Publisher` to a `Binding`, or turn a `CurrentValueSubject` into an `ObservableObject`.

There is just a couple of ways these frameworks can be connected for interoperation.

The first touchpoint is `ObservableObject` - a protocol [declared](https://developer.apple.com/documentation/combine/observableobject) in Combine but used extensively with SwiftUI views.

The second one is the `.onReceive()` view modifier, the only API that allows you to connect an arbitrary data Publisher with the view.

So my next big point of confusion was related to this modifier. Consider this example:

```swift
struct MyView: View {
    
    let publisher: AnyPublisher<String, Never>
    
    @State var text: String = ""
    @State var didAppear: Bool = false
    
    var body: some View {
        Text(text)
            .onAppear { self.didAppear = true }
            .onReceive(publisher) {
                print("onReceive")
                self.text = $0
            }
    }
}
```

This view just shows the String value produced by the `Publisher`, and it raises the `didAppear` flag when the view appears on the screen. As simple as that.

Now, can you tell me how many times will the `print("onReceive")` be triggered in these two use cases:

```swift
struct TestView: View {

    let publisher = PassthroughSubject<String, Never>()    // 1
    let publisher = CurrentValueSubject<String, Never>("") // 2
    
    var body: some View {
        MyView(publisher: publisher.eraseToAnyPublisher())
    }
}
```

Let's take the scenario with `PassthroughSubject` first.

If your answer is 0, you're correct. `PassthroughSubject` has never received a value; thus there is nothing to submit for the `onReceive` closure.

The second one is more tricky. Seriously, take your time to analyze what's going on.

When the view is created, `onReceive` modifier subscribes to the `Publisher`, providing an unlimited "demand" for values (the [notion](https://developer.apple.com/documentation/combine/subscribers/demand) from Combine).

Since the `CurrentValueSubject` does have the initial value `""` it immediately pushes that value to its new subscriber, triggering the `onReceive` callback.

After that, when the view is about to be displayed on screen for the first time, SwiftUI calls its `onAppear` callback, which, in our example, tweaks the view's state by setting the `didAppear` to `true`.

What happens next? Right! The `onReceive` closure gets called *again*! Why is that?

When `MyView` mutated its state inside `onAppear`, SwiftUI had to create a new view for comparing them after the state change! This is required for proper diffing of the view hierarchy.

Upon the second creation, the view traditionally got subscribed to the `Publisher`, and that guy joyfully pushed the current value.

The right answer is 2.

Can you imagine my confusion when I was debugging these "fantom" updates delivered to `onReceive` while I was specifically [trying to filter out](https://nalexn.github.io/swiftui-observableobject/) the duplicated updates?

The last question for you: which text will be displayed if inside `onAppear` we set `self.text = "abc"`?

If you didn't know the story above, the logical answer would be "abc", but now you are armed with the knowledge: no matter where and when you assign a new value to the `text`, the `onReceive` callback will always be chasing you, erasing the value you just assigned with the most recent one from the `CurrentValueSubject`.

## Do you have more?

What's your most confusing part of the SwiftUI? Was it related to the state, or something else? Let's discuss this in [Twitter](https://twitter.com/nallexn)!
