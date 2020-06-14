---
layout: post
title: "Callbacks, Part 1: Delegate, NotificationCenter, and KVO"
description: "Comparison of callback techniques in iOS, pros & cons"
tags: [delegate,delegation,notification,center,key,value, observing,kvo]
date: 2018-01-28 21:30:00 +0600
comments: true
sharing: true
published: true
img: tools_001.jpg
---

This article kicks off the series of posts about callback techniques in [Cocoa](<https://en.wikipedia.org/wiki/Cocoa_(API)>), their comparison & benchmarking. If you're familiar with the concepts of `delegation`, `NotificationCenter` and `Key-Value Observing`, you might want to skip the introduction and go straight to the __Pros & Cons__ of each. Happy reading!

[Callbacks, Part 2: Closure, Target-Action, and Responder Chain]({% post_url 2018-02-08-callbacks-part-2-closure-target-action-responder-chain %})

[Callbacks, Part 3: Promise, Event, and Stream (Functional Reactive Programming)]({% post_url 2018-04-12-callbacks-part-3-promise-event-stream %})

# <a name="delegate">[delegate](#delegate)</a>

`delegate` is the most common type of [loose](https://en.wikipedia.org/wiki/Loose_coupling) connection between entities is Cocoa, but nonetheless, this approach can be used in other languages and platforms as this is basically a [delegation design pattern](https://en.wikipedia.org/wiki/Delegation_pattern).

The idea is simple - instead of interacting with an object through its real API we can instead omit the details of which exact type we talk to and instead use `protocol` with a subset of methods that we need. We define the protocol ourselves and only list those methods we're going to call at our opponent. A real object then would need to implement this protocol, and probably other methods - but we won't know about it - and that's good! [Loose coupling](https://en.wikipedia.org/wiki/Loose_coupling), remember?

Let's see an example. Suppose we have a <a name="Pizzeria_Problem">Pizzeria</a>, with a man who makes the pizza (__pizzaiolo__), and a __customer__ who orders the pizza. The `pizzaiolo` should not know all the details about who exactly is the `customer` (could it be Queen Elizabeth, or Snoop Dogg, or just a dog...), the only thing that matters is the customer's ability to take the pizza when it is ready. This is how we would implement this problem with `delegate` in Swift:

```swift

// The protocol the Pizzaiolo is using for "calling back" to customer
protocol PizzaTaker {
  func take(pizza: Pizza)
}

class Pizzaiolo {
  // We store a reference to a PizzaTaker rather than to a Customer
  var pizzaTaker: PizzaTaker?

  // The method a customer can call to ask Pizzaiolo to make a pizza
  func makePizza() {
    ...
    // As soon as the pizza is ready the `onPizzaIsReady` function is called
  }

  private func onPizzaIsReady(pizza: Pizza) {
    // Pizzaiolo provides the pizza to a PizzaTaker
    pizzaTaker?.take(pizza: pizza)
  }
}

// Class Customer implements the protocol PizzaTaker - the only requirement
// which allows him to get a pizza from Pizzaiolo
class Customer: PizzaTaker {

  // Some details related to Customer class which Pizzaiolo should not know about
  var name: String
  var dateOfBirth: Date

  // Implementation of the PizzaTaker protocol
  func take(pizza: Pizza) {
    // yummy!
  }
}
```
## <a name="delegate_advantages">[Advantages of using delegate](#delegate_advantages)</a>

* Loose coupling. Indeed, with delegate pattern, we have interacting parties decoupled from each other without exposure of who exactly they are. We can easily implement the protocol `PizzaTaker` in other class `Dumpster` and make all fresh pizzas end up in a trash can instead of being eaten, without `Pizzaiolo` even knowing something has changed.
* IDE (such as Xcode) is able to check the connection for correctness with static analysis. This is a great advantage because connections between objects can break when you refactor things, and IDE will point you out the problem even before you try to build the project.
* Ability to get a non-`void` result from calling the `delegate`. Unlike many other callback techniques, with the delegate, you can not only _notify_ but also _ask for data_. In Cocoa, you can often see `DataSource` protocols, which stand exactly for this purpose.
* Speed. Since calling a method on a delegate is nothing more than the direct call of a function, with `delegate` you can achieve absolutely best performance among other callback techniques, which have to spend time on more sophisticated delivery of the call.

## <a name="delegate_disadvantages">[Disadvantages of using delegate](#delegate_disadvantages)</a>

* Single recipient. Even with our Pizzeria example above, `pizzaiolo` is simply unable to serve multiple pizzas at a time for more than one `customer`, and moreover if another `customer` comes while the pizza for the previous `customer` is still being cooked, things will totally mess up: the new `customer` will __overwrite__ the `pizzaTaker` reference on `pizzaiolo` to himself and the previous `customer` will never get his pizza! Awful service...
* Distribution of closely related business logic (aka [low cohesion](<https://en.wikipedia.org/wiki/Cohesion_(computer_science)>)). Since all the callback methods have to be implemented as separate functions inside the recipient we cannot use anonymous functions and get _the action_ and _the callback for the action_ nested near each other. So sometimes it's hard to reason about under which conditions the callback is called.
* Cumbersome infrastructure. In order to use `delegate` we need to do quite a few steps:
  *  define a new protocol
  *  define a weak property for the delegate
  *  implement the protocol in the target type
  *  assign the delegate reference.
  *  [Obj-C] check if the delegate can handle the message with `respondsToSelector:`
* Promotes the emergence of [Massive View Controller](https://www.smashingmagazine.com/2016/05/better-architecture-for-ios-apps-model-view-controller-pattern/). Because of the popularity of this pattern in Cocoa your classes quickly become a convention of delegates if you do nothing so split them out.


# <a name="NotificationCenter">[NotificationCenter](#NotificationCenter)</a>

`NotificationCenter` (or `NSNotificationCenter` for Objective-C) is a class in Cocoa which provides the [publish – subscribe](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) functionality, and as you can guess from its name, it deals with _Notifications_. The notifications it sends around are objects of `Notification` class, which can carry an optional payload, but more often are used just as notices. The `NotificationCenter` itself is a _data bus_, it doesn't send notifications on its own, only when someone _askes_ it to send one. The main feature of this pattern is that the `sender` and `recipients` (there can be many) do not talk directly, like with `delegate` pattern. Instead, they both talk to `NotificationCenter` - the sender calls method `postNotification` on the `NotificationCenter` to send a notification, while recipients opt-in for receiving the notifications by calling `addObserver` on `NotificationCenter`. They can later opt-out with `removeObserver`. Important note though is that `NotificationCenter` does not store notifications for future subscribers - only present subscribers receive the notifications.

The `NotificationCenter` also provides a global access point `defaultCenter`, however, this class is not implemented as a pure [Singleton](https://en.wikipedia.org/wiki/Singleton_pattern) - you can create your own instances of `NotificationCenter` (and you probably should).

Implementation of the same case with _Pizzeria_:

```swift

// First step is to declare new notification type - to be identified by sender and recipients
extension NSNotification.Name {
  static let PizzaReadiness = NSNotification.Name(rawValue: "pizza_is_ready")
}

class Pizzaiolo {

  func makePizza() {
    ...
  }
  
  private func onPizzaIsReady(pizza: Pizza) {
    // Pizzaiolo notifies all interested parties that the pizza is ready:
    NotificationCenter.default.post(name: NSNotification.Name.PizzaReadiness, object: self, userInfo: ["pizza_object" : pizza])
  }
}

class Customer {
  
  // If a customer wants to get a pizza he needs to register as an observer at NotificationCenter
  func startListeningWhenPizzaIsReady() {
    // A customer subscribes for all notifications of type NSNotification.Name.PizzaReadiness
    NotificationCenter.default.addObserver(self, selector: #selector(pizzaIsReady(notification:)), name: NSNotification.Name.PizzaReadiness, object: nil)
  }
  
  // The customer should opt-out of notifications when he's not interested in them anymore
  func stopListeningWhenPizzaIsReady() {
    NotificationCenter.default.removeObserver(self, name: NSNotification.Name.PizzaReadiness, object: nil)
  }
  
  dynamic func pizzaIsReady(notification: Notification) {
    if let pizza = notification.userInfo?["pizza_object"] as? Pizza {
      // Got the pizza!
    }
  }
}
```

## <a name="NotificationCenter_advantages">[Advantages of using NotificationCenter](#NotificationCenter_advantages)</a>

* Multiple recipients. `NotificationCenter` transparently delivers a `Notification` to all subscribers, could there be one, or thousand, or none - this class will take care of the delivery for you. What's also notable - the `sender` cannot know how many subscribers he has. Not to say it's good or bad, it's just how this tool is designed.
* Loose coupling. When using `NotificationCenter` the only thing that couples `sender` and `receivers` together is the name of the notification they use for their communication. If you include a payload to the notification, both parties would need to have symmetric boxing and unboxing of the data.
* Global access point. When you don't need (or just don't care about) the [dependency injections](https://en.wikipedia.org/wiki/Dependency_injection) in your code, the singleton-style method `defaultCenter` allows you to connect two random objects together with ease.

## <a name="NotificationCenter_disadvantages">[Disadvantages of using NotificationCenter](#NotificationCenter_disadvantages)</a>

* Global access point. Yes, this is also a disadvantage. Any globally accessible stuff breaks the testability of your code and even `singleton` as a design pattern [many people consider](https://stackoverflow.com/questions/137975/what-is-so-bad-about-singletons) as an anty-pattern.
* Inability to step-in with a debugger. The only way you can debug `NotificationCenter` is by placing breakpoints
* Non-obvious control flow. If you are trying to understand the business logic of the program and see a place where a `Notification` is sent - the only way to continue exploration is by finding all the recipients manually with a _text search_ in the entire project - because they can be anywhere!
* Transferring data is very error-prone because of boxing and unboxing with a `Dictionary` (`NSDictionary`). Even when used from type-safe Swift, the compiler cannot check the types as well as the structure of the `Dictionary`.
* Recipients must unsubscribe when they are about to deallocate or the app may crash. This requirement has been removed only in iOS 8, but this will haunt iOS developers in their nightmares for years.
* The sender cannot get a non-`void` result, as opposed to the `delegate` and `closure`
* Third-party libs may rely on the same notifications as your code and interfere with each other. Great example is the `NSManagedObjectContextDidSaveNotification` from __CoreData__ framework - every party should [properly handle](https://stackoverflow.com/questions/17568667/ios-googleanalytic-continuously-produce-an-exception/27739971#27739971) this notification or the app can crash.
* There is no control over who is eligible of sending the particular notification. A junior developer on your team may come up to send a system notification like `UIApplicationWillEnterForegroundNotification` to fix a weird bug in his code, and the entire system can get screwed up. Funny, huh?
* When overused, `NotificationCenter` can turn your project into hell because of the previous points

# <a name="KeyValueObserving">[Key Value Observing](#KeyValueObserving)</a>

There is a [dedicated post]({{ site.url }}/kvo-guide-for-key-value-observing) I've written about KVO in Swift 5 and Objective-C.

`KVO` is a traditional [observer](https://en.wikipedia.org/wiki/Observer_pattern) pattern built-in any `NSObject` out of the box. With `KVO` observers can be notified of any changes of a `@property` values. It leverages _Objective-C runtime_ for automated notifications dispatch, and because of that for `Swift` class, you'd need to opt into _Objective-C dynamism_ by inheriting `NSObject` and marking the `var` you're going to observe with modifier `dynamic`. The observers should also be `NSObject` descendants because this is enforced by the `KVO` API.

Sample code for `Key-Value Observing` a property `value` of the class `ObservedClass`:

```swift
class ObservedClass : NSObject {
  @objc dynamic var value: CGFloat = 0
}

class Observer {
  var kvoToken: NSKeyValueObservation?
    
  func observe(object: ObservedClass) {
    kvoToken = object.observe(\.value, options: .new) { (object, change) in
      guard let value = change.new else { return }
        print("New value is: \(value)")
      }
    }
    
  deinit {
    kvoToken?.invalidate()
  }
}
```
## <a name="KeyValueObserving_advantages">[Advantages of using Key-Value Observing](#KeyValueObserving_advantages)</a>
* Observation pattern in a few lines of code. Normally you would need to implement it yourself, but in `Cocoa` you have this feature coming standard for every `NSObject`.
* Multiple observers - there is no limitation on the number of subscribers.
* There is no need to change the source code of the class under observation
* Because of the aforementioned you can observe objects of _any_ class (including those from system frameworks, to which sources we don't have access thus cannot modify)
* Very low coupling ([connascence of name](<https://en.wikipedia.org/wiki/Connascence_(computer_programming)#Static_Connascences>)) - the observed party doesn't even know it's being observed, the observing party's knowledge is bounded by the name of the `@property`
* Notification can be configured to deliver not only the most recent value of the observed `@property` but also the previous value.

## <a name="KeyValueObserving_disadvantages">[Disadvantages of using Key-Value Observing](#KeyValueObserving_disadvantages)</a>
* One of the worst APIs across Cocoa. This point could be easily broken up in several - [so](https://www.mikeash.com/pyblog/key-value-observing-done-right.html) [bad](https://khanlou.com/2013/12/kvo-considered-harmful/) [it](https://ianthehenry.com/2014/5/4/kvo-101/) [is](https://nshipster.com/key-value-observing/). Good for us there are [nicer](https://github.com/facebook/KVOController) [alternative](https://github.com/postmates/PMKVObserver) implementations of the `KVO`.
* _keyPath_ used for subscription is a string, and it cannot be statically verified. Luckily this has a solution in Swift (`#keyPath` directive), but in Objective-C, if the observed party changes the name of the `@property` - there won't be a compiler warning about this, so the app will just crash in runtime.
* Each observer has to explicitly unsubscribe on `deinit` - otherwise crash is unavoidable.
* We have to call the `super` implementation of `observeValueForKeyPath` callback function to make sure we don't break the implementation of the superclass.
* Relatively slow performance. Even when compiled with `-Os` optimization, for `Objective-C` one `KVO` notification is _30 times_ slower than a function call, for `Swift` it is _200 times_ slower. I have used [this project](https://github.com/nalexn/PerformanceTestTools) for benchmarking.

Continuation: [Callbacks, Part 2: Closure, Target-Action, and Responder Chain]({% post_url 2018-02-08-callbacks-part-2-closure-target-action-responder-chain %})
