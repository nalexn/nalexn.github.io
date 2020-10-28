---
layout: post
title: "Who said we cannot unit test SwiftUI views?"
date: 2019-11-21 8:30:00 +0300
description: "The story behind creating a Unit Testing framework"
tags: [iOS,swift,test,xcode,exception,traverse,instance,variable,subview,child,view]
comments: true
sharing: true
published: true
img: swiftui_unit_testing_01.jpg
---

Viktor Chernomyrdin, a Russian politician from the ‘90s, once said:

> Such has never happened before, and here it is again...

This reminds me of the situation we find ourselves in with SwiftUI: We have a brand-new, exciting technology — but with stability issues, an incomplete API, and scarce documentation. *Such has never happened before, and here it is again*.

Anyway, things are not as bad as they could be, and teams have already started adopting SwiftUI in production projects. Still, one of the main arguments against using it in production is a complete inability to unit test the UI.

<div style="max-width:600px; display: block; margin-left: auto; margin-right: auto;"><img src="{{ site.url }}/assets/img/clean_swiftui_02.jpg"></div>

A function of state should be really straightforward to test — with one **if**. We need to have access to the function’s output.

Views in SwiftUI are nested inside one another, forming a statically typed hierarchy of structs without an API to inspect the view’s content.

One day Apple may release its unit testing tool for SwiftUI, but who knows whether/when this will happen.

So I decided to build [ViewInspector](https://github.com/nalexn/ViewInspector), a library that allows for inspecting the SwiftUI view hierarchy at runtime.

By now, the framework has evolved into a fully-fledged tool that allows you to inspect and unit test SwiftUI hierarchy for iOS, macOS, and tvOS.

You can dig into the hierarchy and read the actual state values on any SwiftUI view:

```swift
func testVStackOfTexts() throws {
    let view = VStack {
        Text("1")
        Text("2")
        Text("3")
    }
    let values = try view.inspect().map { try $0.text().string() }
    XCTAssertEqual(values, ["1", "2", "3"])
}
```

You can also simulate the user interaction by programmatically triggering system-controls callbacks:

```swift
let button = try view.inspect().hStack().button(1)
try button.tap()

let list = try view.inspect().list()
try list[5].view(RowItemView.self).callOnAppear()
```

And it's possible to extract and evaluate the state of any custom view that uses `@Binding`, `@State`, `@ObservedObject` or `@EnvironmentObject`:

```swift
let sut = try view.inspect().tabView().navigationView()
    .overlay().anyView().view(CustomView.self).actualView()
XCTAssertTrue(sut.viewModel.isUserLoggedIn)
```
## SwiftUI internals

I should start with a note that I'm a developer and not a hacker: I hardly ever had to reverse-engineer something, explore how jailbreak works, or mess with assembly code ever since my studying at a university.

But SwiftUI appeared to be a black box I had difficulties in getting it to work properly. There was some minor issue I couldn't get around, and after many hours of trial and error, I gave up searching the web for the solution and decided to look inside that box for getting any clue.

I simply put a breakpoint and ran `po dump(view)` to see what's kept inside a simple `Text("Hello, world!")` view. To my surprise, there were plenty of information available in runtime:

```swift
"view" of type Text
  ↳ "modifiers" of type Array<Modifier>
      ↳ value = []
  ↳ "storage" of type Storage
      ↳ "verbatim" of type String
          ↳ value = "Hello, world!"
```

I knew that `dump` uses reflection, a public API available in Swift. That meant I could get access to these values as well. What if this trick could work for cracking every SwiftUI view?

I had a gut feeling it just cannot be that simple, there had to be a wall that I won't be able to get through with using just reflection, but I was curious how far I can dig.

And as it turned out, there were many pitfalls waiting for me on the way:

1. All types in reflection are erased to `Any`
2. Computed properties, such as `var body: some View`, are not available in reflection
3. Generic private structs and function types which are tricky to cast the value to
4. Initializing a struct which all `init` methods are private
5. SwiftUI dependency injection through `Environment`
6. Property wrappers, such as `@State`, with an elusive storage for values
7. Significant variations of the hierarchy after a tiny tweak of the input. For example, `Text("Hi")` vs `Text(hiValue)`
8. Overall obscurity and lack of information about the private structures

In this piece, I want to share abnormal use cases and hacky tricks I had to appeal to when building this library using just the standard capabilities of Swift language.

## Creating a struct without calling `init()`

There is one interesting SwiftUI view that provides information about the view's container size: `GeometryReader`

```swift
GeometryReader { geometry in
    Text("Hello, world!")
        .padding(geometry.size.width * 0.1)
}
```

The reflection showed that this view does not store the contained view directly. Instead, it holds a closure for building the enclosed views. The closure takes one parameter - the `GeometryProxy` value.

This means that the only way to obtain the `Text` view in the example above is to call that closure with a `GeometryProxy`.

Ok, fortunately, `GeometryProxy` is a public struct... But it has no public initializers!

How can we construct a value without a factory?

Value types, as opposed to objects, don't require storing pointers to the parent class for self-identification, meaning that they remain functional without `isa` pointers inside... I had one crazy idea in my mind, and I decided to try!

At first, I wanted to find out the number of bytes that `GeometryProxy` takes. Swift provides `MemoryLayout` for this purpose:

```swift
MemoryLayout<GeometryProxy>.size
>> 48
```

There are two options where to allocate the memory - on the stack and on the heap.

The latter is more flexible, as you can just specify the number of bytes you need:

```swift
let pointer = UnsafeMutableRawBufferPointer
                .allocate(byteCount: 48, alignment: 8)
```

But dynamic memory requires manual deallocation with `deallocate()` and is slower than allocation on the stack, so I decided to go with the first option, which is more exotic.

I needed to declare a value type that would take the same amount of bytes: 48. I called `MemoryLayout` for a `Double`, and expectedly got the following:

```swift
MemoryLayout<Double>.size
>> 8
```

So if I declared a struct, for example, that kept 6 Doubles, its total memory size should be 48:

```swift
struct Allocator {
    let data: (Double, Double, Double, 
               Double, Double, Double) = (0, 0, 0, 0, 0, 0)
}
MemoryLayout<Allocator>.size
>> 48
```

Great! The last step was to cast the types:

```swift
let proxy = unsafeBitCast(Allocator(), to: GeometryProxy.self)
```

> It's alive!! Alive!!!

Of course, there were no guarantees the fake `GeometryProxy` would work correctly, as the inner variables may not expect to be zeros, but fortunately, this worked well:

```swift
proxy.size
>> CGSize(0, 0)
```

I had an idea to find the position of bytes responsible for storing the CGSize and initialize them with the custom value, but reflection showed that `size`, just like all the other public vars on `GeometryProxy` are computed, so there was no way to achieve this.

So after I called the factory closure on the `GeometryReader` with this "Frankenstein" struct, I got the contained views with no issues! Of course, the layout of the views is screwed, but at least, the values like the string on `Text` could be safely extracted.

## Casting to an unknown generic type

Another notable case was with `ForEach`. To explore the internals I made a simple setup with an array of strings transformed to `Text` views:

```swift
let array = ["0", "1", "2"]
let view = ForEach(array, id: \.self) { Text($0) }
```

`dump` showed the following:

```swift
"view" of type ForEach<Array<String>, String, Text>
  ↳ "data" of type Array<String>
      ↳ value = ["0", "1", "2"]
  ↳ "content" of type (String) -> Text
  ↳ "idGenerator" of type WritableKeyPath<String, String>
      ↳ value = WritableKeyPath<String, String>
  ↳ "contentID" of type Int
      ↳ value = 0
```

So I could extract the `Text` views using the content builder closure `content: (String) -> Text` by providing it with an element of `data: [String]` array.

All I needed was to cast `data` and `content` to correct types from reflection's default type `Any`:

```swift
func extractContentOfForEach(view: Any) -> [Any] {
    let mirror = Mirror(reflecting: view)
    let data: Any = mirror.descendant("data")
    let content: Any = mirror.descendant("content")
    if let array = data as? [String],
       let builder = content as? (String) -> Text {
       // works for the case with String and Text
    }
}
```

Hardcoding types `String` and `Text`, of course, wouldn't work for an arbitrary `ForEach`, so I needed to get the types from elsewhere.

A naive attempt to obtain the type dynamically with `type(of: value)` did not make the compiler happy - it needs to know the types in compile time. Basically this is not a valid code: `let casted = value as? type(of: value)`

Ok, the Type information should be known at compile time. From where could we get it?

The first workable solution I came up with was to provide the types from the caller side:

```swift
func extract<Element,Content>(view: Any, 
                              element: Element.Type,
                              content: Content.Type) -> [Any] {
    let mirror = Mirror(reflecting: view)
    let data: Any = mirror.descendant("data")
    let content: Any = mirror.descendant("content")
    if let array = data as? [Element],
       let builder = content as? (Element) -> Content {
           return array.map { builder($0) }
       }
    return []
}
```

I didn't like this approach because it was bulky and inconvenient to use, so I appealed to the following hack.

I've declared a type-erased middleware protocol and extended the `ForEach` to conform to that protocol. The trick is that in the extension of the `ForEach` we have the inner type information required for the content extraction:

```swift
protocol ForEachContentProvider {
    func extractContent() -> [Any]
}

extension ForEach: ForEachContentProvider {

    func extractContent() throws -> [Any] {
	    let mirror = Mirror(reflecting: view)
	    let data: Any = mirror.descendant("data")
	    let content: Any = mirror.descendant("content")
	    // Types Data and Content are known in this context
	    if let array = data as? [Data.Element],
	       let builder = content as? (Data.Element) -> Content {
	           return array.map { builder($0) }
	       }
	    return []
    }
}
```

So the original extraction function now just needed to cast the `view: Any` to the middleware protocol and call `extractContent()`. Since `ForEach` now conforms to that protocol, the cast succeeds and the extraction works as expected:

```swift
func extractContentOfForEach(view: Any) -> [Any] {
    if let forEach = view as? ForEachContentProvider {
        return forEach.extractContent()
    }
    return []
}
```

## SwiftUI's native Environment injection

SwiftUI has a very handy dependency injection mechanism through `@ObservedObject`, `@EnvironmentObject` and `@Environment` attributes.

While there was no practical problem with supporting `@ObservedObject` in the inspection framework, I had to spend quite some time trying to figure out how to inject `@EnvironmentObject`.

When a view receives traditional DI injection through `.environmentObject(...)` it gets wrapped into a view of type `ModifiedContent`. This type of view is widely used throughout SwiftUI for applying various tweaks to the view, such as `.padding()`, `.blur(radius:)`, etc.

`ModifiedContent` is quite transparent - one of its attributes, called `content`, provides the enclosed view, which could be easily extracted.

The problem is with the other attribute: `modifier`, which usually refers to the value of a "semi-private" type, such as `_PaddingLayout`. I called them semi-private because Xcode recognizes these types if you paste them in the source code, but their symbols are excluded from the public headers: if you control-click and select "Jump to Definition", Xcode won't be able to locate them.

For some types, Xcode Autocomplete shows a few instance vars, for example, `_PaddingLayout` has `var edges: Edge.Set` and `var insets: EdgeInsets?`

So going back to the problem of injecting `@EnvironmentObject`: the view gets wrapped in a `ModifiedContent` which `modifier` has the type `_EnvironmentKeyWritingModifier<InjectedObject?>`.

That modifier has no public methods, and here is what reflection shows for it, when we inject an object of type `InjectedObject`:

```swift
"modifier" of type _EnvironmentKeyWritingModifier<InjectedObject?>
   ↳ "keyPath" of type WritableKeyPath
       ↳ value = WritableKeyPath<EnvironmentValues, InjectedObject?>
   ↳ "value" of type InjectedObject?
       ↳ value = InjectedObject(...)
```

It does keep the reference to the `InjectedObject`, and also has a `WritableKeyPath` for `EnvironmentValues`.

Those `EnvironmentValues` are very mysterious. So far I know that both `@EnvironmentObject` and `@Environment` are using it for storing the values used by the SwiftUI views, but my experiments showed that the `EnvironmentValues` are provided to the view hierarchy only at the render time – and withdrawn after!

Try running the following code:

```swift
struct ContentView: View {
    @EnvironmentObject var object: InjectedObject
    
    var body: some View {
        DispatchQueue.main.async {
            print("\(self.object.flag)")
        }
        return Text(object.flag ? "Flag is on" : "Flag is off")
    }
}
```

... and you'll see that asynchronous reading of `@EnvironmentObject` outside of the rendering cycle is prohibited - you'll get the same crash as if you never provided `InjectedObject` in `.environmentObject(...)` call.

## Design decisions behind the inspection framework

I wanted to make the library safe and convenient to use. All I had was just an idea how the syntax on the caller side should look like: it should be chained calls like `view.anyView.hStack.button`

It was clear that each intermediate element should return a statically typed value which would restrict the available options: there is no sense of calling `.tap()` for `AnyView`, or `.hStack` on a `Text`.

One of the options was to create an object-oriented hierarchy of classes, but after using functional and protocol-oriented programming for a few years I developed a strong allergy to OOP 🧐

So I decided to use a unified struct `InspectableView` and encapsulate the polymorphic behavior in its Generic parameter `View`:

```swift
struct InspectableView<View> {
    let view: Any
}
```

At first, I thought I'll be using SwiftUI views as the `View` parameter, but quickly realized that most of the SwiftUI views have generic parameters as well, and constructions like `InspectableView<HStack<VStach<Text>>>` would be too cumbersome and fragile to operate.

Instead, I've created an empty `struct ViewType { }` that served as the base namespace for future view types: `ViewType.Button` being a representative for the `Button` view, for example.

I thought that the user of the library could falsely assume they can substitute SwiftUI views in that parameter. In order to help them quickly identify that this is the wrong path, I've put a restriction on the generic type to conform to a simple protocol `KnownViewType`, which SwiftUI views don't conform to by default:

```swift
protocol KnownViewType { }

struct InspectableView<View> where View: KnownViewType {
    let view: Any
}

struct ViewType { }

extension ViewType {    
    struct Button: KnownViewType { }
}
```

Now it was all ready to start building the polymorphic behavior with generics.

The views in SwiftUI can either contain a single view (`AnyView`), a collection of views (`HStack`), or no other views (`Text`).

In order to encapsulate this behavior, I defined two protocols: `SingleViewContent` and `MultipleViewContent`

```swift
protocol SingleViewContent {
    static func content(view: Any, envObject: Any) -> Any?
}

protocol MultipleViewContent {
    static func content(view: Any, envObject: Any) -> [Any]?
}
```

Now any `ViewType` was able to adopt the content extraction strategy based on its nature:

```swift
extension ViewType.AnyView: SingleViewContent {
    static func content(view: Any, envObject: Any) -> Any? {
        ...
    }
}

extension ViewType.HStack: MultipleViewContent {
    static func content(view: Any, envObject: Any) -> [Any]? {
        ...
    }
}
```

For views like `Text` that don't have a contained view, its companion `ViewType` simply opts out of conforming to either of these protocols.

Now every `ViewType` could declare its strategy of extracting the content.

The last piece of the puzzle was to add methods, such as `.hStack` for extraction FROM the parent.

This one was easy - I just extended `InspectableView where View: SingleViewContent` with a method named after the type of view intended for extraction, allowing such views to continue the chain with `.hStack`, for example:

```swift
public extension InspectableView where View: SingleViewContent {
    
    var hStack: InspectableView<ViewType.HStack>? {
        ...
        return InspectableView<ViewType.HStack>(extractedView)
    }
}
```

A similar extension is defined for `MultipleViewContent` as well.

Finally, for types like `ViewType.Button`, I could add exclusive support of the methods like `.tap()`

```swift
extension InspectableView where View == ViewType.Button {
    func tap() {
        ...
    }
}
```

With this approach `InspectableView` obtained a controlled set of methods available for particular `ViewType`, eliminating the possible logical errors when working with the view extraction library.

---

That's the story behind creating the [ViewInspector](https://github.com/nalexn/ViewInspector) framework. If you have a SwiftUI project you want to cover with Unit Tests - consider trying it out! I'm accepting pull requests and general feedback!