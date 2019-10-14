---
layout: post
title: "CancelBag: a DisposeBag for Combine subscriptions"
date: 2019-10-04 14:30:00 +0300
description: "A container for holding Cancellable (including AnyCancellable) returned by Combine subscriptions. Inspired by DisposeBag from RxSwift"
tags: [Cancellable,AnyCancellable,iOS,swift,SwiftUI,RxSwift,functionBuilder]
comments: true
sharing: true
published: true
img: bag_001.jpg
---

With the announce of [SwiftUI](https://developer.apple.com/documentation/swiftui/) and [Combine](https://developer.apple.com/documentation/combine) frameworks on WWDC2019 Apple has clearly outlined their vision on the future of the software development for their platforms, and that future is going to be *Reactive*.

Just like it happened in the past with ARC for Objective-C, Autolayout, or Swift, there is only a couple years lag between the announce of the new Apple's technology, and it's pervasive use. So if one was still reluctant about learning functional reactive programming, now they are left with a choice to board the Rocket now or stay in the fading world of object-oriented UIKit.

However, Apple has made a great job of making the transition as seamless as possible for developers: Combine is a masterpiece of simplicity, among other reactive frameworks.

It provides really concise and programmer-friendly API that's just enough for basic handling of the "values over time".

## Houston, we have a problem…

As you'd expect from the conciseness of the API, there always will be someone who's missing his [favorite fancy operator](https://rxmarbles.com/) from a reactive framework they're used to.

I understand that. But it feels like Combine is missing something fundamental, and the suggested alternative seems cumbersome and old-fashioned: I'm talking about the **memory management of the subscriptions**.

[RxSwift](https://github.com/ReactiveX/RxSwift) provides us with [DisposeBag](https://github.com/ReactiveX/RxSwift/blob/master/RxSwift/Disposables/DisposeBag.swift); it's competitor, [ReactiveSwift](https://github.com/ReactiveCocoa/ReactiveSwift), has a counterpart: [Lifetime](https://github.com/ReactiveCocoa/ReactiveSwift/blob/master/Sources/Lifetime.swift).

But what about Combine? The subscription returns a token of type [AnyCancellable](https://developer.apple.com/documentation/combine/anycancellable) that we cannot put anywhere except for storing in an instance variable:

```swift
class ValueConsumer {

    private var subscription1: AnyCancellable?
    private var subscription2: AnyCancellable?
    private var subscription3: AnyCancellable?
    private let mySubject = CurrentValueSubject<Int, Never>(0)
    private var myVar: Int = 0

    func consumeValues(publisher1, ...) {
        subscription1 = publisher1
            .sink { print("New value: \($0)") }
        subscription2 = mySubject
            .subscribe(publisher2)
        subscription3 = publisher3
            .assign(to: \. myVar, on: self)
    }
}
```

--

> Does Combine need a CancelBag after all?

– The question asked by [Michael Long](http://twitter.com/michaellong) in [his blog post](https://medium.com/better-programming/swift-5-1-and-combine-memory-management-a-problem-14a3eb49f7ae), for me has a clear answer: "Yes, Combine lacks it."

Here is how the code above could be refactored if we had a container for subscriptions:

```swift
class ValueConsumer {

    private var cancelBag = CancelBag()
    
    private let mySubject = CurrentValueSubject<Int, Never>(0)
    private var myVar: Int = 0

    func consumeValues(publisher1, ...) {
        publisher1
            .sink { print("New value: \($0)") }
            .cancel(with: cancelBag)
        subscription2 = mySubject
            .subscribe(publisher2)
            .cancel(with: cancelBag)
        subscription3 = publisher3
            .assign(to: \. myVar, on: self)
            .cancel(with: cancelBag)
    }
}
```
Even though this looks cleaner, the number of lines stayed about the same.

But how about taking one step further and implement [Variadic DisposeBag](https://medium.com/@michaellong/rxswifty-and-his-variadic-disposebag-1682ecceaf41)?


```swift
class ValueConsumer {

    private var cancelBag = CancelBag()
    
    private let mySubject = CurrentValueSubject<Int, Never>(0)
    private var myVar: Int = 0

    func consumeValues(publisher1, ...) {
        cancelBag.collect {
            publisher1
                .sink { print("New value: \($0)") }
            mySubject
                .subscribe(publisher2)
            publisher3
                .assign(to: \. myVar, on: self)
        }
    }
}
```

The `collect` function above is using the new [`@functionBuilder`](https://blog.vihan.org/swift-function-builders/) attribute available in Swift 5.1, the same one that allows view containers from [SwiftUI](https://developer.apple.com/documentation/swiftui/), such as `VStack`, to take an array of elements without any (,) separators.

So now we can collect all the subscriptions tokens without explicitly calling `.cancel(with: cancelBag)` or storing it in a variable!

The full gist for the CancelBag implementation (~50 lines of code) can be [found on Github](https://gist.github.com/nalexn/9b53421f2900631176d7617e12eaa359).
