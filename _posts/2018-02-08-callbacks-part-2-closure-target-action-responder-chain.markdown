---
layout: post
title: "Callbacks, Part 2: Closure, Target-Action, and Responder chain"
description: "Comparison of callback techniques in iOS, pros & cons"
tags: [closure,block,target,action,target-action,responder,chain]
date: 2018-02-08 21:30:00 +0600
comments: true
sharing: true
published: true
img: tools_003.jpg
---

This article continues the series of posts about callback techniques in [Cocoa](<https://en.wikipedia.org/wiki/Cocoa_(API)>), their comparison & benchmarking. If you're familiar with the concepts of `Closure`, `Target-Action` and `Responder chain`, you might want to skip the introduction and go straight to the __Pros & Cons__ of each. Happy reading!

[Callbacks, Part 1: Delegation, NotificationCenter, and KVO]({% post_url 2018-01-28-callbacks-part-1-delegation-notificationcenter-kvo %})

[Callbacks, Part 3: Promise, Event, and Stream (Functional Reactive Programming)]({% post_url 2018-04-12-callbacks-part-3-promise-event-stream %})


# <a name="Closure_Block">[Closure (Block)](#Closure_Block)</a>

Although this series of the posts is called _"Callbacks in Cocoa"_, the word "callback" is usually used as a synonym for _Closure_ (or _Block_).

Closure and Block are two names for the same phenomenon used in Swift and Objective-C respectively.

The [documentation](https://developer.apple.com/library/prerelease/content/documentation/Swift/Conceptual/Swift_Programming_Language/Closures.html) says that the _"Closures are self-contained blocks of functionality that can be passed around..."_.

If you think about it, regular _objects_ of any class are also "self-contained blocks of functionality" that can be "passed around" by the reference, but Closure is its own beast.

Just like objects, a Closure has associated "functionality", that is some code to be executed, and also Closure can capture _variables_ and operate with them as would an object do with its own variables.

The key difference is that the functionality of an object is declared by its class implementation, which is a standalone structure, but for Closure, we write its implementation in place of use, where it's passed somewhere else as a parameter.

Although a Closure has characteristics of an object, to a bigger extent it is a _function_. Because this function is declared without association to a type, it is also called [anonymous function](https://en.wikipedia.org/wiki/Anonymous_function).

The easiest way to grasp the concept is to see an example, and if we are to solve the _Pizzeria_ problem from the [previous post]({% post_url 2018-01-28-callbacks-part-1-delegation-notificationcenter-kvo %}#Pizzeria_Problem), we could end up with the following code:

```swift
// The Closure type can be pre-defined, and since it's an anonymous function, its declaration
// looks like a function signature. In our case - a void function that takes a Pizza
typealias PizzaClosure = (Pizza) -> Void

class Pizzaiolo {

  // "An instance" of the PizzaClosure can be treated just as an object
  private var completion: PizzaClosure?

  // PizzaClosure will be passed in as a parameter
  func makePizza(completion: PizzaClosure) {
    // Save the 'completion' to be able to call it later
    self.completion = completion
    // Cooking the pizza
    ...
  }
  
  private func onPizzaIsReady(pizza: Pizza) {
    // Calling the closure in order to "deliver" the pizza
    self.completion?(pizza)
    self.completion = nil
  }
}

class Customer {

  func askPizzaioloToMakeAPizza(pizzaiolo: Pizzaiolo) {
    // We declare the body of the closure in a place where it is passed as an input parameter
    pizzaiolo.makePizza(completion: { pizza in
      // We're inside the closure! And we've got the pizza!
      // This code gets executed when pizzaiolo calls the closure we passed to him
    })
  }
}
```
## <a name="Closure_advantages">[Advantages of using Closure (Block)](#Closure_advantages)</a>

* [Loose coupling](https://en.wikipedia.org/wiki/Loose_coupling) - the callee and caller must agree only on the signature of the closure, which is the weakest [connascence of name](<https://en.wikipedia.org/wiki/Connascence_(computer_programming)#Static_Connascences>).
* [High cohesion](<https://en.wikipedia.org/wiki/Cohesion_(computer_science)>) of the business logic - the action and the response to the action can be declared together in one place.
* Capturing of the variables from the client's context helps with state management because we don't need to store the variables in the class and face potential access problems, such as the [race condition](https://en.wikipedia.org/wiki/Race_condition).
* Closures are [as fast](https://github.com/nalexn/PerformanceTestTools) as calling a regular function.
* Just like with [`delegation`]({% post_url 2018-01-28-callbacks-part-1-delegation-notificationcenter-kvo %}#delegate), Closure can have non-`void` return type, so it also can be used as a `DataSource` for asking for data from the client code
* Full type safety. Using the static code analysis the IDE is able to check the signature of the closure and verify types of parameters passed in.

## <a name="Closure_disadvantages">[Disadvantages of using Closure (Block)](#Closure_disadvantages)</a>
* Closures and blocks have quite [complex syntax](https://fuckingclosuresyntax.com/)
* For newcomers the concept of [anonymous function](https://en.wikipedia.org/wiki/Anonymous_function) (or [lambda](https://stackoverflow.com/questions/16501/what-is-a-lambda-function)) is harder to understand than most constructions from imperative programming
* Because _Swift_ and _Objective-C_ use [automatic reference counting](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/AutomaticReferenceCounting.html) memory management, closures can cause [memory leaks](https://en.wikipedia.org/wiki/Memory_leak) when they form so-called [retain cycle](https://stackoverflow.com/questions/24042949/block-retain-cycles-in-swift). We should always be careful when using Closures and use `[weak ...]` modificator in cases where the cyclic capture takes place
* When used as _callbacks_ for async operations, in practice the client code can often result in several nested in each other callbacks, which is called the [pyramid of doom](<https://en.wikipedia.org/wiki/Pyramid_of_doom_(programming)>), which is hard to read and support. Moreover, when the code flow inside the callbacks has branching (`if` - `else`) the pyramid size grows exponentially.
* Arguments of the closure or block cannot have argument labels. Because of that, the client code should assign names to each argument, which may lead to a wrong interpretation of the parameters after refactoring of the closure signature.
* [Obj-C] Blocks must be checked for `NULL` before being called, or the app will crash.

# <a name="Target-Action">[Target-Action (NSInvocation)](#Target-Action)</a>

Target-action on its own is a [design pattern](https://developer.apple.com/library/content/documentation/General/Conceptual/Devpedia-CocoaApp/TargetAction.html), but conceptually it's a variation of the [command](https://en.wikipedia.org/wiki/Command_pattern). Traditionally this callback technique is used in Cocoa for handling user interactions with the UI but is not restricted to this use case.

The client code provides two parameters in order to receive a callback in the future - an object and a function. In this context, the object is `target`, and the function is `action`.

The callback then is fulfilled by calling the `action` function on the `target` object.

It's not a trivial case for a programming language when an object and its function are provided separately, so this technique requires support from the language.

For example, Swift does not allow us to do the trick of calling an arbitrary method on an arbitrary object, so we'd need to appeal to Objective-C if we wanted something like that to work in Swift project.

On the other hand, there are a few ways how we can perform the _action_ on the _target_ in Objective-C (thanks to volatile Objective-C runtime). In this example, I'm using the class [NSInvocation](https://developer.apple.com/reference/foundation/nsinvocation), which suits our needs the best.

Implementation of the [_Pizzeria_]({% post_url 2018-01-28-callbacks-part-1-delegation-notificationcenter-kvo %}#Pizzeria_Problem) problem with target-action would look like this:

```objc
@interface Pizzaiolo : NSObject

// The target and the action are sent separately
- (void)makePizzaForTarget:(id)target selector:(SEL)selector;

@end

@interface Customer: NSObject

- (void)askPizzaioloToMakePizza:(Pizzaiolo *)pizzaiolo;

@end

// ------ 

@implementation Pizzaiolo

- (void)makePizzaForTarget:(id)target selector:(SEL)selector
{
  NSInvocation *invocation = [self invocationForTarget:target selector:selector];
  // Storing the reference to the invocation object to be used later
  self.invocation = invocation;

  // making pizza
  ...
}

- (void)onPizzaIsReady:(Pizza *)pizza
{
  // Argument indices start from 2, because indices 0 and 1 indicate the hidden arguments self and _cmd
  if (self.invocation.methodSignature.numberOfArguments > 2) {
    [self.invocation setArgument:&pizza atIndex:2];
  }
  [self.invocation invoke];
}

// A helper method that constructs NSInvocation for given target and action
- (NSInvocation *)invocationForTarget:(id)target selector:(SEL)selector
{
  NSMethodSignature *signature = [target methodSignatureForSelector:selector];
  if (signature == nil) return nil;
  NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:signature];
  invocation.target = target;
  invocation.selector = selector;
  return invocation;
}

@end

@implementation Customer

- (void)askPizzaioloToMakePizza:(Pizzaiolo *)pizzaiolo
{
  [pizzaiolo makePizzaForTarget:self selector:@selector(takePizza:)];
}

- (void)takePizza:(Pizza *)pizza
{
  // Yeah! Objective "pizza" completed
}

@end
```
## <a name="Target-Action_advantages">[Advantages of using Target-Action](#Target-Action_advantages)</a>
* In some aspects target-action is similar to [`delegation`]({% post_url 2018-01-28-callbacks-part-1-delegation-notificationcenter-kvo %}#delegate), but has two great advantages over it:
  * The name of the callback function is not predefined and is to be provided by the client code. This greatly decouples the caller and the callee.
  * It supports multiple recipients with individual methods to be called for each of them.
* `NSInvocation` provides a way to get a non-`void` result of invocation, so this technique can be used for requesting data (but I would not recommend to do it - read the disadvantages).
* Static code analyzer is able to perform at least shallow check of the method name provided as _action_. It cannot tell if the _target_ is actually implementing the _action_.

## <a name="Target-Action_disadvantages">[Disadvantages of using Target-Action](#Target-Action_disadvantages)</a>
* Complex infrastructure. For target-action we need to implement everything ourselves:
  * Constructing `NSInvocation` the right way
  * Managing a collection of `NSInvocation` in order to support multiple targets and actions.
* Compiler is unable to check that the receiver can handle the selector it provides as an action. This often leads to crashes at runtime after inattentive code refactoring.
* `NSInvocation` doesn't automatically nullify reference to the target which got deallocated, so the app can crash at the runtime when the callback is triggered. Thus every target should explicitly remove itself from target-action.
* For the client code it is not clear which method signature should the _action_ have (how many parameters and of which type). Passing around arguments is quite error-prone. The same applies to the returned type. Static code analyzer won't be able to warn of any type discrepancies.
* Shares with [`delegate`]({% post_url 2018-01-28-callbacks-part-1-delegation-notificationcenter-kvo %}#delegate) the problem of distributed business logic ([low cohesion](<https://en.wikipedia.org/wiki/Cohesion_(computer_science)>)) and promotion of the [massive view controller](https://www.smashingmagazine.com/2016/05/better-architecture-for-ios-apps-model-view-controller-pattern/).

# <a name="Responder_chain">[Responder chain](#Responder_chain)</a>
Although _responder chain_ is the core mechanism in `UIKit` originally designed for distribution of touch and motion input events for the views and view controllers, it certainly can be used as a callback technique and in some cases is a great alternative to the `delegation`.

On its core responder chain is a beautiful application of the [chain of responsibility](https://en.wikipedia.org/wiki/Chain-of-responsibility_pattern) design pattern. The central role in the responder chain is allocated to the [`UIResponder`](https://developer.apple.com/reference/uikit/uiresponder) class.

In Cocoa pretty much every class related to the UI is a direct or indirect descendant of the `UIResponder` class, thus the responder chain is truly pervasive in the app and includes every _view_ on the screen, their _view controllers_, and even the `UIApplication` itself.

A great feature of the responder chain is that it is maintained automatically unless the developer wants to implement a custom `UIResponder` (which I cannot foresee when might be needed).

So, suppose you have this nontrivial view hierarchy:

`UIApplication` – `UIWindow` – __`ViewController A`__ – `ViewController` – `View` – __`Subview B`__

The responder chain allows the `Subview B` send a message to the `ViewController A` with ease. The `ViewController A` should just be the first in the chain who implements the selector that `Subview B` is sending. And if no one including `UIApplication` can handle the selector - it just gets discarded and the app does not crash.

Not every `UIResponder` can send a message in the pipe - in Cocoa, there are only two classes which can do it: `UIControl` and `UIApplication`, and `UIControl` in fact [uses UIApplication's API](https://developer.apple.com/reference/uikit/uiapplication/1622946-sendaction) for that.

It's possible to send actions on behalf of `UIControl`...

```swift
let control = UIButton()
// UIControl should be added to the view hierarchy
view.addSubview(control)
control.sendAction(#selector(handleAction(sender:)), to: nil, for: nil)
```

... but normally you'd use `UIApplication`

```swift
// Triggering callback with UIApplication

UIApplication.shared.sendAction(#selector(handleAction(sender:)), to: nil, from: self, for: nil)
```

As written in the [docs](https://developer.apple.com/reference/uikit/uiapplication/1622946-sendaction), if we provided non-`nil` UIResponder object in the `from` parameter (this could be our current view or view controller), the responder chain would start traversing from this element, otherwise, when the `from` parameter is `nil`, the traverse starts at the _first responder_.

For the needs of calling back through a few layers of nested child view controllers, we should always specify the `from` parameter, because _first responder_ does not necessarily belong to our subviews hierarchy.

## <a name="Responder_chain_advantages">[Advantages of using Responder chain](#Responder_chain_advantages)</a>
* Supported by every UIView and UIViewController in the project out of the box.
* Responder chain is managed automatically as the view hierarchy changes.
* Natively supported by the _Interface Builder_ - without writing code at all you can choose to deliver action from a specific `UIControl` to the _first responder_ in the chain.
* Action delivery is crash-safe. If there were no responder that agreed to step in and handle the action - nothing bad happens.
* Sometimes this is the easiest way to hear back from a _view_ or _view controller_ residing deeply in the view hierarchy. Normally you'd need to forward the callback through all the view controllers manually with a chain of delegates, and this implies writing custom classes for each UIViewController or UIView on the way. This technique truly saves us from writing a lot of code.

## <a name="Responder_chain_disadvantages">[Disadvantages of using Responder chain](#Responder_chain_disadvantages)</a>
* Responder chain is for `UIResponder` descendants only. Objects of arbitrary classes are not supported.
* You cannot send any useful data with the action. The parameters for the action are predefined as `sender: Any` and `event: UIEvent?`.
* It is impossible to get a non-`void` result of the call
* Responder chain is a [simplex channel](https://en.wikipedia.org/wiki/Simplex_communication) and you cannot choose the direction.
* Just like any binding from _Interface Builder_, the connection may break as you refactor the code. There are [some tools](https://github.com/fastred/IBAnalyzer) that can help with detecting the incorrect state.
* It is quite confusing for the newcomers why they have to use obscure `responder chain` for such _simple_ thing as showing the on-screen keyboard in iOS


Continuation: [Callbacks, Part 3: Promise, Event, and Stream (Functional Reactive Programming)]({% post_url 2018-04-12-callbacks-part-3-promise-event-stream %})
