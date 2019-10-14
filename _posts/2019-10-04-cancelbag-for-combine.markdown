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

I always thought that despite great flexibility and universality of RxSwift, its APIs are heavily overloaded and redundant, which made it hard to adopt in production by many engineering teams around the globe.

Combine, on the other hand, provides really concise and programmer-friendly API that's just enough for basic handling of the "values over time". Given the power to extend existing frameworks, I anticipate the community to create hundreds of tiny extensions, helpers and wrappers that will make Combine even more convenient to use. One of such extensions I propose in this article.

## The problem

When trying the Combine for the first time, I immediately stumbled upon an issue: the memory management of the subscriptions.

Consider this example of [subscription mechanism](https://developer.apple.com/documentation/combine/receiving_and_handling_events_with_combine) in Combine framework:

```swift
import Combine

let subject = PassthroughSubject<Int, Never>()
let subscription = subject
    .sink { value in
        print("New value: \(value)")
    }
subject.send(7)
// "New value: 7" is printed out

subscription.cancel()
subject.send(8)
// Nothing is printed out
```

The act of subscribing with the method `sink` returns a value of type `AnyCancellable`. This value is a **token** that manages the lifetime of the subscription.

We can either call `cancel()` or allow the token to get deallocated for voiding the subscription.

This means that to *prevent immediate termination* of the subscription in a programming module we need to retain the token and store it in an instance variable:

```swift
import Combine

class ValueProvider {
    // A value we want to observe:
    let value = CurrentValueSubject<Int, Never>(0)
}

class ValueConsumer {

    // A variable to keep the subscription alive:
    private var subscription: AnyCancellable?

    func consumeValues(from valueProvider: ValueProvider) {
        // Subscribing on value updates and retaining the returned Cancellable token:
        subscription = valueProvider.value
            .sink { value in
                print("New value: \(value)")
            }
    }
}

let provider = ValueProvider()
let consumer = ValueConsumer()
consumer.consumeValues(from: provider)
```

If we needed to subscribe to the second source of values, we'd have to define another instance variable for the new token as well:

```swift
class ValueConsumer {

    private var subscription1: AnyCancellable?
    private var subscription2: AnyCancellable?

    func consumeValues(from valueProvider1: ValueProvider, valueProvider2: ValueProvider) {
        subscription1 = valueProvider1.value
            .sink { value in
                print("New value1: \(value)")
            }
        subscription2 = valueProvider2.value
            .sink { value in
                print("New value2: \(value)")
            }
    }
}
```

Can you imagine how much boilerplate code this implies for production apps, where modules can carry a dozen subscriptions that define the presentation state and ultimately power the UI?

**So, the problem here is that Apple didn't provide a convenient container for storing the subscriptions tokens, when subscriptions' lifetime has to be bound to the lifetime of the observer.**

## The Solution

> Does Combine need a CancelBag after all?

– The question asked by Michael Long in [his blog post](https://medium.com/better-programming/swift-5-1-and-combine-memory-management-a-problem-14a3eb49f7ae), for me has a clear answer: "Yes, Combine lacks it."

Using the [**CancelBag.swift**](https://gist.github.com/nalexn/9b53421f2900631176d7617e12eaa359) (~40 lines of code), we can refactor the code above either in a traditional way [DisposeBag](https://github.com/ReactiveX/RxSwift/blob/master/RxSwift/Disposables/DisposeBag.swift) is employed in RxSwift:

```swift
class ValueConsumer {

﹢  private let cancelBag = CancelBag()

    func consumeValues(from valueProvider1: ValueProvider, valueProvider2: ValueProvider) {
        valueProvider1.value
            .sink { value in
                print("New value1: \(value)")
            }
﹢          .cancel(with: cancelBag)
        valueProvider2.value
            .sink { value in
                print("New value2: \(value)")
            }
﹢          .cancel(with: cancelBag)
    }
}
```

...or take one step further and utilize the new [Variadic DisposeBag](https://medium.com/@michaellong/rxswifty-and-his-variadic-disposebag-1682ecceaf41) approach:

```swift
func consume(value1: ..., value2: ...) {
    cancelBag.insert {
        value1
            .sink { print("New value1: \($0)") }
        value2
            .sink { print("New value2: \($0)") }
    }
}
```
The `insert` function above is using the new **@functionBuilder** syntax available in Swift 5.1, the same one that allows view containers from [SwiftUI](https://developer.apple.com/documentation/swiftui/), such as `VStack`, to take an array of elements without any (,) separators.

Of course, this also works with all subscription APIs in **Combine** framework that return `Cancellable` token, including [value binding](https://developer.apple.com/documentation/combine/publisher/3235801-assign) with `assign(to:)` and `subscribe(_:)`:

```swift
func consume(value1: ..., value2: ..., value3: ...) {
    cancelBag.insert {
        value1
            .sink { print("New value1: \($0)") }
        value2
            .sink { print("New value2: \($0)") }
        myVariable
            .subscribe(value3)
        Timer.publish(every: 1, on: .main, in: .default)
            .assign(to: \MyViewModel.date, on: viewModel)
    }
}
```

In a case when the lifetime of the subscription is shorter than the lifetime of the parent we can reset the `CancelBag` to terminate the observation:

```swift
class Cell: UITableViewCell {
    private var cancelBag = CancelBag()
    
    func prepareForReuse() {
        super.prepareForReuse()
        cancelBag = CancelBag()
    }
}
```

I believe that using the `CancelBag` makes the code easier to read and support. Subscriptions can be grouped and put in different `CancelBags` based on their logical association, which ultimately contributes to the clarity of the code.

The `CancelBag` gist is [available on Github](https://gist.github.com/nalexn/9b53421f2900631176d7617e12eaa359).