---
layout: post
title: "Callbacks, Part 3: Promise, Event, and Stream (Functional Reactive Programming)"
date: 2018-04-12 21:30:00 +0600
description: "Comparison of callback techniques in iOS, pros & cons"
tags: [promise,frp,functional,reactive,programming,ios,development]
redirect_from:
  - blog/2018/04/12/callbacks-part-3-promise-event-stream
comments: true
sharing: true
published: true
img: tools_002.jpg
---

This article continues the series of posts about callback techniques in [Cocoa](https://en.wikipedia.org/wiki/Cocoa_(API)), their comparison & benchmarking. This one is about `Promise`, `Event`, and `Stream`, their pros & cons. Happy reading!

[Callbacks, Part 1: Delegation, NotificationCenter, and KVO]({% post_url 2018-01-28-callbacks-part-1-delegation-notificationcenter-kvo %})

[Callbacks, Part 2: Closure, Target-Action, and Responder Chain]({% post_url 2018-02-08-callbacks-part-2-closure-target-action-responder-chain %})

---

# Intro

What's the most challenging problem for you when writing an async code? For me, it's to keep the async code as readable as the sync code is (considering your sync core is readable 😜). Unfortunately, neither Swift nor Objective-C have the support of the [async-await syntax](https://msdn.microsoft.com/ru-ru/library/mt674882.aspx) like C# has, so we have to deal with the routine of the Closure callbacks, which is truly the best tool among standard solutions we have in our disposal.

As discussed in the [previous post]({% post_url 2018-02-08-callbacks-part-2-closure-target-action-responder-chain %}#Closure_disadvantages), when we use Closures as callbacks the code can quickly become hardly readable, because the only way we can chain callbacks is to nest one into another, forming the [pyramid of doom](<https://en.wikipedia.org/wiki/Pyramid_of_doom_(programming)>).

In fact, this construction of a few callbacks nested in each other can cause even more inconvenience:

* _Tedious error handling_: for each callback, we have to check for error and manually route the execution flow to either next callback in case there was no error, or to error handling code otherwise. A lot of boilerplate code.
* _No support for cancellation_: again, you have to write it on your own, store somewhere the state and check the state of the operation in each callback. More code to write!
* _Hard to manage parallel tasks_: you have two operations working in parallel which in the end need to launch another operation awaiting the first two. With traditional callbacks, you'd need to share the state between the callbacks and cross-check the state in callbacks. A lot of ugly and error-prone code.

So, after using standard callback techniques through the length and breadth I came to a conclusion that their drawbacks are so significant that I needed to look for better tools, and these are [_Promise_](#Promise), [_Event_](#Event) and [_Stream_](#Stream).

# <a name="Promise">[Promise](#Promise)</a>

Although [the concept](https://en.wikipedia.org/wiki/Futures_and_promises) of _Promises_ (aka _Futures_) is ancient as dinosaurs, it has been rediscovered by the JS community not so long ago and then rumors reached Cocoa developers as well.

Promises mitigate all the aforementioned problems: they have straightforward error handling, they support cancellation, can organize parallel tasks and make the code to look more sync-like.

So, how do Promises work?

Just like a [BlockOperation](https://developer.apple.com/reference/foundation/blockoperation), Promises use `Closures` for encapsulation of the client code, which is triggered at an appropriate time.

When we need to chain a few async tasks to perform one after the other, instead of nesting callbacks, Promises allow us to put the callbacks' code in a natural order one block after the other, forming a _chain_ rather than a _pyramid_.

This makes the code much easier to read and maintain.

Consider an example: _we need to send two subsequent network requests: at first load a `user` and then using its `id` send another request for user's `posts`. While requests are working, we need to display a "Loading..." message and hide it when both requests finish or when either fail with an error. In the error's case, we should display it's description._

A lot of business logic, huh? But see how gracefully this can be coded with Promises:

```swift
firstly {
    // Triggering the loading indication and making the first API call
    showLoadingIndicator()
    return urlSession.get("/user")
}.then { user in
    // Notice that we use 'user.id', which was just loaded
    return urlSession.get("/user/\(user.id)/posts")
}.always {
    // The 'always' promise performs even if either of prior network requests failed
    hideLoadingIndicator()
}.then { posts in
    // The 'then' promise only performs if all the preceding promises succeeded
    // So here we can use "posts" loaded from the second request
    display(posts: posts)
}.catch { error in
    // Error handling in one place
    showErrorMessage(error)
}
```

As you can see, there are a few chained async operations written in a natural order of how they are performed. The code is more declarative and explanatory than with nested Closure callbacks.

Pure `Cocoa` doesn't provide us with a native citizen for Promises, so the code above uses [PromiseKit](https://promisekit.org/) library, but there are also [BrightFutures](https://github.com/Thomvis/BrightFutures) and [Bolts](https://github.com/BoltsFramework/Bolts-ObjC), you can choose your favorite.

## <a name="Promise_advantages">[Advantages of using Promises](#Promise_advantages)</a>
* Great alternative to traditional Closure callbacks, addresses many of their problems:
  * Chaining async callbacks rather then nesting them improves the readability of the code.
  * Straightforward error handling, all errors are captured and forwarded for handling in one place.
  * Ability to recover from errors and continue the original flow.
  * Cancellation support (PromiseKit and Bolts).
  * Awaiting of multiple promises before the chain continues can be coded without significant efforts.
* Automatically catches all the _exceptions_ thrown by the client code and pacify them to simple `Errors`, which are handled just like other regular errors. This is an additional safety barrier for you if you work with something _explosive_, for example parsing a volatile JSON from some unstable web API.
* Promises are designed to be executed only _once_ and then self-destruct. In certain situations, this is a very helpful feature, since this adds clarity to the code where it's important to guarantee the callback cannot be called the second time.
* Because promises self-destruct after execution, it's harder to leak memory referencing `self` or another object from inside a Closure.

## <a name="Promise_disadvantages">[Disadvantages of using Promises](#Promise_disadvantages)</a>
* Until you master your skills of using the Promises, every now and then the compiler would complain it cannot understand the code you just wrote with the promises. And don't expect to see any meaningful explanation from the Swift compiler. So you have to be prepared to spend some time getting used to the syntax. This is definitely harder to use for newbies who are still not confident with Closures.
* As mentioned in the advantages, promises are resolved once and then dismissed. This also implies you [cannot easily use](https://promisekit.org/docs/cookbook/wrapping-delegation/) Promises as an alternative to callbacks intended to be called _multiple_ times, such as with _delegate_ or _NotificationCenter_.
* The cancellation isn't that tasteful as the error handling. Depending on realization, you'd have to check the cancellation status in each promise (Bolts) or handle a special type of `Error` in the error handling code (PromiseKit).
* The syntax still isn't that great as it could be with [async-await](https://gist.github.com/lattner/31ed37682ef1576b16bca1432ea9f782#part-1-asyncawait-beautiful-asynchronous-apis), which is still not supported as of Swift version 5.0. 😒
* Promises are __100 times__ slower than any other callback technique ([I've done benchmarking](https://github.com/nalexn/PerformanceTestTools)). This is because each consequent promise in the chain has to be scheduled through `dispatch_async` and performed _asynchronously_, which is a necessary evil: otherwise, promises can unintentionally cause [deadlocks](https://en.wikipedia.org/wiki/Deadlock). This problem has its own name - ["releasing Zalgo"](https://blog.izs.me/post/59142742143/designing-apis-for-asynchrony). So it's ok to use Promises for the networking layer, where the performance drop won't be noticed, but I would think twice before using Promises elsewhere in the app.
* Debugging the promises is a pain. As you learned from the previous point, each consequent promise in the chain is _always_ performed asynchronously. That means our favorite step-over debugging is simply not possible, you have to put breakpoints all over the place to be able to follow the execution flow.
* Any crash reporting tool, such as [Crashlytics](https://try.crashlytics.com/), are almost useless with Promises because when a code scheduled through `dispatch_async` crashes, your printed call stack trace is almost empty. You'd expect to see the full promise chain in the stack, but instead, there will be only the last promise that crashed, leaving you with no clue on where the source of the problem was.

# <a name="Event">[Event](#Event)</a>

If _Promises_ can be used as a replacement for Closure callbacks, _Events_ are a great altrnative to standard _delegation_, _target-action_ and _NotificationCenter_ discussed in [two]({% post_url 2018-01-28-callbacks-part-1-delegation-notificationcenter-kvo %}) [previous]({% post_url 2018-02-08-callbacks-part-2-closure-target-action-responder-chain %}) posts.

Isn't it cool that those three different APIs each having many drawbacks can be thrown away and replaced with one, very simple yet functional API? _Event_ is a truly great alternative when you realize the problem of using the standard tools, but not yet willing to use _Streams_ from _Functional Reactive Programming_.

The concept of Event can be found in C#, where it has [support on the language level](https://msdn.microsoft.com/en-us/library/awbftdfh.aspx), but in Cocoa, we have to add a third-party library for _Event_. This can be [Signals](https://github.com/artman/Signals), or [EmitterKit](https://github.com/aleclarson/emitter-kit).

What's really charming about Events is the simplicity of implementation - Event libraries are usually just about couple hundred lines of code, as opposed to Promise or FRP frameworks, where the size varies from [5,000](https://github.com/mxcl/PromiseKit) to [55,000](https://github.com/ReactiveX/RxSwift) lines of code.

Events provide a generic mechanism for sending notifications to one or many recipients. The notification can carry any set of parameters of any types (thanks to Generics) and are delivered to subscription Closures.

Take a look at the example:

```swift
class DataProvider {

    // These public variables are points for subscription
    let dataSignal = Signal<(data: Data, error: Error)>()
    let progressSignal = Signal<Float>()

    ...

    func handle(receivedData: Data, error: Error) {
        // Whenever we want to notify subscribers we trigger the signal with the payload data to deliver
        progressSignal.fire(1.0)
        dataSignal.fire(data:receivedData, error:error)
    }
}
```
As you can guess from the code above, the `DataProvider` is going to be the source of notifications. Now see how the other modules can subscribe for and handle the notifications:

```swift
class DataConsumer {

    init(dataProvider: DataProvider) {
        // 'progress' updates will be sampled to us every 0.5 second
        dataProvider.progressSignal.subscribe(on: self) { progress in
            // handle progress
        }.sample(every: 0.5)
        // one time subscription for tuple (data, error)
        dataProvider.dataSignal.subscribeOnce(on: self) { (data, error) in
            // handle data or error
        }
    }
}
```

## <a name="Event_advantages">[Advantages of using Events](#Event_advantages)</a>
* Supports multiple recipients.
* Has ability to transmit any set of data with a safe static type checking. This includes a `Void` event for no data in the notification.
* Automatic cancellation of the subscription for deallocated subscribers (depends on the library).
* Very lean infrastructure:
  * We describe all the information about the notification (name + data type) and simultaneously create the subscription point with one line of code
  * The subscription point is at the same time the delivery hub: sending a notification is again just one line of code
* Low coupling: both the subscription point and the transmitted data format are declared in one place with lowest possible [connascence of name](<https://en.wikipedia.org/wiki/Connascence_(computer_programming)#Static_Connascences>).
* The lightweight of the libraries. [Less code – fewer bugs](https://blog.codinghorror.com/the-best-code-is-no-code-at-all/).
* Because of inner simplicity of the notification delivery code, we have
  * Meaningful call stacks if the client code crashes
  * Easier debugging than with Promises or Streams
* Trivial API that's really easy to understand and start using to full extent than Promises or Streams
* Library specific features, such as:
  * one-time subscription
  * delayed notifications
  * filtering and time sampling of notifications
  * delivery of notifications to specific `OperationQueue`

## <a name="Event_disadvantages">[Disadvantages of using Events](#Event_disadvantages)</a>
* Two or more Events cannot be naturally combined in any manner. Any business logic depending on multiple Events has to be coded separately. (In the meanwhile Promises and Streams are combinable by design).
* Events can only be used for transmitting the data in one direction. We cannot pull data with them, as we could with [delegation]({% post_url 2018-01-28-callbacks-part-1-delegation-notificationcenter-kvo %}#delegate_advantages) or [Closure]({% post_url 2018-02-08-callbacks-part-2-closure-target-action-responder-chain %}#Closure_advantages).
* Just as with any other tool utilizing Closures for client code encapsulation, you should beware retain cycles. Be sure to use constructions like `[weak self]`.

# <a name="Stream">[Stream](#Stream)</a>

"Asynchronous data stream" or just _Stream_ - is the main concept behind _Functional Reactive Programming_ frameworks, such as [ReactiveSwift](https://github.com/ReactiveCocoa/ReactiveSwift) or [RxSwift](https://github.com/ReactiveX/RxSwift). FRP is a fairly big topic, so I'll give a short reference here and focus on its Pros and Cons, while you can read a more detailed introduction [somewhere else](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754).

We just talked about _Event_, so in this case, the _Stream_ is the "Event on steroids":

* It remembers all the values that have been sent through during its lifetime
* It can be combined with other streams in [many fancy ways](https://rxmarbles.com/)
* It has significant incline towards writing functional-style code
* It has deep integration with Cocoa classes (via supplementary frameworks¹)
* It has more generic use cases than just _observation_

¹ When combined with [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa) or [RxCocoa](https://github.com/ReactiveX/RxSwift/tree/master/RxCocoa), Streams form a fully-fledged toolkit that replaces many things we got used to in Cocoa, including previously discussed delegation, target-action, and other techniques. More than that, most of the code boilerplate we had in our apps, such as when working with `UITableView`, now can be replaced with literally one-line-of-code UI bindings and data streams manipulated by functional operators.

Although the functional code is preferred and highly encouraged by those libraries, the developer can choose to which extent he wants to dive into the functional programming world - at a bare minimum we can use _Streams_ just like _Events_ - for notification delivery, writing the callback closure in imperative style.

Below is an example from [ReactiveSwift](<https://github.com/ReactiveCocoa/ReactiveSwift>), which demonstrates how compact could be the code that sends search requests as the user enters the text, keeping only the latest request alive:

```swift
let searchResults = searchStrings
    .flatMap(.latest) { (query: String?) -> SignalProducer<(Data, URLResponse), AnyError> in
        let request = self.makeSearchRequest(escapedQuery: query)
        return URLSession.shared.reactive.data(with: request)
    }
    .map { (data, response) -> [SearchResult] in
        let string = String(data: data, encoding: .utf8)!
        return self.searchResults(fromJSONString: string)
    }
    .observe(on: UIScheduler())
```

If you never worked with FRP libraries, you might not know what `flatMap` does, or why you even need a `SignalProducer` here and not a `Signal`. For a developer joining the FRP world, this is the main challange - a lot of unknown functions, operators and even bizarre code formatting to get used to.

From a developer's standpoint, _Stream_ is the most challenging "callback" technique to learn among those discussed in this post' series; but as the outcome, you get significantly bigger opportunities.

And again, if you want to learn more about FRP, here are a [couple](https://blog.scottlogic.com/2015/04/24/first-look-reactive-cocoa-3.html) [usefull](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754) links for you to get started, and a [comparison](<https://www.raywenderlich.com/126522/reactivecocoa-vs-rxswift>) of ReactiveSwift and RxSwift if you want to decide which one to use in your project.

## <a name="Stream_advantages">[Advantages of using Streams](#Stream_advantages)</a>
* Generally universal tool that can be used wherever you need to "call back" to other programming entities. The stream is able to replace all the traditional callback techniques available in Cocoa, and does it gracefully.
* Has a very strong community, so you won't be left alone with your problem when using Stream.
* Encourages writing code in a functional style rather than in imperative, which mitigates a [callback hell](https://www.quora.com/What-is-callback-hell) problem and generally improves the code [cohesion](https://en.wikipedia.org/wiki/Cohesion_(computer_science)).
* Stream libraries have Cocoa extensions with UI bindings that help with reducing the amount of code we write for updating the UI with app's state changes.
* Stream can be naturally combined with other streams to address a problem of having many dependent async operations.
* The Stream borrows many advantages from _Events_ and _Promises_:
  * Ability to transmit any set of data with the strict type checking by the compiler.
  * Support for multiple observers.
  * Ability to easily control the lifetime of the subscriptions.
  * Cancellation that automatically propagates and stops all related operations.
  * Error handling in one place.
  * Ability to recover from errors and continue execution.
  * Delayed notifications
  * Filtering and time sampling of notifications
  * Delivery of notifications to specific `OperationQueue`


## <a name="Stream_disadvantages">[Disadvantages of using Streams](#Stream_disadvantages)</a>
* Considerably higher required skill-level than with other techniques. This is not a tool like Event, which works out of the box for you, you need to learn (a lot) how to cook it first.
* Many sources of confusion even for seasoned developers:
  * Hot & cold signals (streams). You need to understand the difference as this dramatically impacts how they should be used. In __RxSwift__ Hot & Cold signals are syntactically indistinguishable, which can lead to hard-to-find bugs.
  * [Long list](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/api/core/observable.md) of [exotic functions](https://rxmarbles.com/) applicable to the Stream makes developers look up for the definition and carefully read about the semantics of the method in order to avoid misusage.
  * It is impossible to predict the behavior of the Stream - how many events (and of which type) it will generate. You cannot guarantee the Stream will send one or many _value_ events before it completes. An example is a networking request - the client code may expect to receive the response once, but instead, there can be multiple _value_ events if the request streams the data, or if the networking code automatically requests next pages of a paginated list. With __RxSwift__ this is also impossible to declare a Stream that cannot generate an _error_ event, which means you always have to implement error handlers or expose your app for potential bugs if you choose to ignore errors.
* This has never been easier to massively leak the memory ever since the times we had manual reference counting in Objective-C (`[[object retain] autorelease]`, remember them?). Even when using `weak` and `unowned` for every reference inside the Closures, each time you start a Stream subscription you should explicitly limit its lifetime using `DisposeBag` from __RxSwift__ or `Lifetime` from __ReactiveSwift__, or you risk leaking this subscription (and possibly other objects too) and ultimately ruining the app performance. If you prefer using `unowned` over `weak`, be prepared for crashes as well.
* FRP frameworks encourage the use of custom operators borrowed from other functional languages. This often looks unnatural in Swift and adds ambiguity, unless you have a few years of Haskell programming in your background.
* Heavyweight frameworks with the core functionality ([15,000](https://github.com/ReactiveCocoa/ReactiveSwift) and [55,000](https://github.com/ReactiveX/RxSwift) lines of code for two most popular FRP frameworks). This not only increases the size of your app but also extends the app's launch time, just as with other dynamically loaded frameworks.