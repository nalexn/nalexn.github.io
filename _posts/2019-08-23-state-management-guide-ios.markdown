---
layout: post
title: "The Complete Guide to the State Management in iOS"
date: 2019-08-23 21:30:00 +0600
description: "Common problems when working with the program state and the ways to address them. ReSwift - REDUX for iOS"
tags: [reswift,redux,ios,swift,development,unidirectional,data,flow,flux]
comments: true
sharing: true
published: true
img: state_001.jpg
---

This guide takes a deep dive into the design of state management for UIKit-based production apps. SwiftUI is a game-changer in this regard; however, most of the concepts and statements in this article remain true even for the new Apple's UI framework. State management for SwiftUI-based apps is well covered in [these](https://medium.com/better-programming/understanding-swiftui-data-flow-79429a49ae35) [wonderful](https://medium.com/@ZoeWave/swiftui-19da16c1af0) [articles](https://medium.com/@alexobenauer/demystifying-data-flow-through-swiftui-697017aba7e0).

---

There are many challenges in software development, but there is one beast that tends to screw things up much more often than the others: the problem of app's state management and data propagation.

So what can go wrong with the [state](https://en.wikipedia.org/wiki/State_(computer_science)), which is simply a data intended for reading and writing?

### Common problems when working with the state

1. **[Race condition](https://searchstorage.techtarget.com/definition/race-condition)** - unsynchronized access to the data in a concurrent environment. This leads to hard-to-find bugs, such as unexpected or incorrect results of computation, data corruption and even crashes of the app.
2. **[Unexpected side effect](https://en.wikipedia.org/wiki/Side_effect_(computer_science))**. When multiple entities in the program share the state by keeping references to a single state value, a mutation of the state initiated by one of the entities may come unexpected for the other. This is usually a result of poor design with unrestricted data accessibility. The aftermath varies from glitches in the UI to the [dead-locks](https://en.wikipedia.org/wiki/Deadlock) and crashes.
3. **[Connascence of Values](https://en.wikipedia.org/wiki/Connascence#Connascence_of_Values_(CoV))**. When multiple entities in the program share the state by storing their own copies of the state, a mutation of the local copy of the state does not automatically affect the other copies. This requires writing additional code for renewing the values when either copy gets updated. Failure to do it correctly makes the state copies go out of sync, which usually leads to an erroneous data displayed to the user with subsequent corruption of the app's state when the user or system itself interacts with the outdated data.
4. **[Connascence of Type](https://en.wikipedia.org/wiki/Connascence#Connascence_of_Type_(CoT))** in languages with [dynamic typing](https://stackoverflow.com/questions/1517582/what-is-the-difference-between-statically-typed-and-dynamically-typed-languages) is when a variable during its lifetime is changing not only the value but also the type. Even though there are [practical applications](https://softwareengineering.stackexchange.com/questions/115520/should-i-reuse-variables) of this technique, in general, it is considered a [bad practice](https://softwareengineering.stackexchange.com/questions/187332/is-changing-the-type-of-a-variable-partway-through-a-procedure-in-a-dynamically). It makes the algorithm much harder to follow and understand and increases the chances of human error when maintaining such code. Even seasoned programmers run the risk to unintentionally change the type by assigning the wrong variable by mistake. The outcome of such an error depends on the language, but you can tell that nothing good can happen.
5. **[Connascence of Convention](https://en.wikipedia.org/wiki/Connascence#Connascence_of_Meaning_(CoM)_or_Connascence_of_Convention_(CoC))**. A misinterpretation of the value that was accidentally put in place of another parameter with the same primitive type. For example, if we have `UserID` and `BlogID` both represented as `String` type, it is possible to accidentally pass `UserID` to a function expecting `BlogID`. An incorrect value might be used in a server call, or stored in a local app state, either way - it is an erroneous situation. A solution to this could be using `struct` wrappers for primitive values, which allows the compiler to distinguish the types and warn about the type mismatch.
6. **[Memory leak](https://en.wikipedia.org/wiki/Memory_leak)**. Just like any other resource in the program, when handled incorrectly, the state can stay in memory even after it is no longer intended to be used. Leaking big chunks of memory (hundreds of megabytes allocated for images, for example) can eventually lead to significant free memory deficit with subsequent crashing. When the state leaks, we probably lose a few KBs of memory at most, but who knows _how many times_ our program will leak it? The aftermath is slower performance and crashing.
7. **[Limited testability](https://en.wikipedia.org/wiki/Software_testing)**. The state plays an essential role in [unit testing](https://en.wikipedia.org/wiki/Unit_testing). Sharing of the state by value or by reference [couples programming entities](https://en.wikipedia.org/wiki/Coupling_(computer_programming)), making their algorithms dependent on each other. The infelicitous design of state management in the program can make tests less effective or even impossible to be written for poorly designed modules.

---
There are always two questions arising for a developer when a new piece of state is introduced: _"Where to store the state data?"_ and _"How to notify the other entities in the app about the state updates?"_. Let's cover each one in detail and see if there is a silver bullet to the problem.

## 1. Where to store the state data?

We can have a local variable in a method or instance variable in the class, or a global variable accessible from everywhere - the main difference between them is the scope of the program that can read or write it.

When deciding _where_ to store the new variable,
we need to consider the main characteristic of the state - its locality, or the accessibility scope.

A general rule of thumb is to __always aim for the smallest accessibility scope possible__. A local variable defined within a method is preferred over a global variable, not only for avoiding unexpected data mutations from other scopes that we didn't notice at first glance, but also for improving the testability of the modules that use that data.

The local screen's state, which is not shared with other screens, can safely be stored in the screen module itself. This only depends on the [architecture](https://medium.com/ios-os-x-development/ios-architecture-patterns-ecba4c38de52) you use for your screen module. For example, in the case of [Model-View-Controller](https://ru.wikipedia.org/wiki/Model-View-Controller), this is the `ViewController`, for [Model-View-ViewModel](https://en.wikipedia.org/wiki/Model-view-viewmodel) it would be the `Model`.

Things get much more complicated when we have the data meant to be transmitted or shared by multiple modules. There are two main scenarios:

* We need to pass some state to a subsequent screen and hear back when it fulfills its duties and possibly get some data back.
* We have a state shared by the entire app. This could be any data that can be displayed by multiple screens that do not represent a parent-child form of ownership, like in the previous point. The data can be read or updated, and every party should gracefully handle the changes.

In this article, I cover both cases, and it's logical to start from the first one. Just so you don't get lost, the second case is fully covered down below in the section __Shared state management__, just keep reading.

Ok, the interaction between two subsequent screens (parent-child). In order to achieve [loose coupling](https://en.wikipedia.org/wiki/Loose_coupling) between the modules, we need to make sure that the data transfer we design does not introduce an unnecessary dependency by disclosing needless details about the parties passing or receiving the data. _The less they know about each other - the better._

For passing data forward, we have a practically standard technique - the [dependency injection](https://en.wikipedia.org/wiki/Dependency_injection) of the value itself or a reference to an entity with the read access to that data.

The opposite movement of the data back to the caller side is a bit more tricky, and we can naturally transition to answering the second question about the state:

## 2. How to notify the other entities in the app about the state updates?

In my previous articles I've already covered possible answers to this question, only among the standard tools we have:

* [delegate](https://nalexn.github.io/callbacks-part-1-delegation-notificationcenter-kvo/#delegate)
* [NotificationCenter](https://nalexn.github.io/callbacks-part-1-delegation-notificationcenter-kvo/#NotificationCenter)
* [KVO](https://nalexn.github.io/callbacks-part-1-delegation-notificationcenter-kvo/#KeyValueObserving)
* [Closure](https://nalexn.github.io/callbacks-part-2-closure-target-action-responder-chain/#Closure_Block)
* [Target-Action](https://nalexn.github.io/callbacks-part-2-closure-target-action-responder-chain/#Target-Action)
* [Responder Chain](https://nalexn.github.io/callbacks-part-2-closure-target-action-responder-chain/#Responder_chain)

There are endless amounts of ways how a developer can use any of these techniques alone or in combination with each other, not to mention the full freedom in choosing the naming for functions and parameters.

If we also introduce [Promise](https://nalexn.github.io/callbacks-part-3-promise-event-stream/#Promise), [Event](https://nalexn.github.io/callbacks-part-3-promise-event-stream/#Event) or [Stream of values](https://nalexn.github.io/callbacks-part-3-promise-event-stream/#Stream) in our project, the choice may blow one's mind (and their program).

<div style="width:400px; display: block; margin-left: auto; margin-right: auto;"><img src="{{ site.url }}/assets/img/john-travolta-01.gif"></div>

### So which method to use for propagating the state changes?

Over the past few years I formed the following rules I follow when choosing the data propagation methods (you may have different preferences):

* **[delegate](https://nalexn.github.io/callbacks-part-1-delegation-notificationcenter-kvo/#delegate)**. Even though this technique maintains its popularity in the iOS community, I'm a supporter of [the idea that closures](https://medium.cobeisfresh.com/why-you-shouldn-t-use-delegates-in-swift-7ef808a7f16b) are generally more flexible and convenient replacement for a `delegate`. They serve the same purpose but the use of `Closures` results in writing less boilerplate code with a _much higher_ [cohesion](https://en.wikipedia.org/wiki/Cohesion_(computer_science)) at the same time.
* **[Target-Action](https://nalexn.github.io/callbacks-part-2-closure-target-action-responder-chain/#Target-Action)**. Pretty much the same comments as for `delegate`. The only reason when I would still use it is when I subclass `UIControl` or `UIGestureRecognizer`, because `Target-Action` is naturally supported by them.
* **[Closure](https://nalexn.github.io/callbacks-part-2-closure-target-action-responder-chain/#Closure_Block)**. This is my to-go choice for the simplest cases of interaction between two modules. If only there are any complications, such as subsequent asynchronous tasks also with `Closure` callbacks, or when I need to notify more than one module - I start to look at `Promise`, `Event` or `Stream`.
* **[Promise](https://nalexn.github.io/callbacks-part-3-promise-event-stream/#Promise)** is my favorite tool for handling a chain of asynchronous tasks, such as subsequent backend API calls. `Stream of values` also can handle this, but `Promise` offers much less complicated API and suits for engineering teams avoiding `Rx` and other reactive tools for any reason.
* **[Event](https://nalexn.github.io/callbacks-part-3-promise-event-stream/#Event)** is a lightweight yet powerful implementation of the Observer pattern, a great alternative to `NotificationCenter` and `KVO`. Whenever you need to forecast a notification with or without data - this tool provides a handy subscription point that's safe to work with and easy to test. Can be used as an observable property - a variable always carrying a value that also offers "KVO"-style change observation for any number of subscribers.
* **[Stream of values](https://nalexn.github.io/callbacks-part-3-promise-event-stream/#Stream)**. This is a universal tool that can replace both `Promise` and `Event` while also offer `UI bindings` and other perks. Beware though! I'm a huge fan of [Functional Reactive Programming](https://en.wikipedia.org/wiki/Functional_reactive_programming) myself, but still not too many people _fully_ understand its concepts and can _properly_ use this tool in practice. On an interview with one big retailer store, their team manager secretly shared with me his concern that they have to hire _exclusively_ senior engineers __only because__ their app is written fully in `RxSwift` and __cannot be supported__ by engineers of lower skill levels (they tried hiring them). This is mind-blowing how a tool intended for making development easier in practice turned things the opposite way! At another interview, this time at a top-3 bank in Russia, they said among all 10+ product teams that they have it is _strictly forbidden_ to use reactive programming, exactly for the same reason.
* **[NotificationCenter](https://nalexn.github.io/callbacks-part-1-delegation-notificationcenter-kvo/#NotificationCenter)**. In my projects I'm using a custom [SwiftLint](https://github.com/realm/SwiftLint) rule to _prevent_ the use of `NotificationCenter ` for custom needs. I believe this is [one of the most harmful tools](https://davidnix.io/post/stop-using-nsnotificationcenter/) you can use in your app for the data distribution. Not only does it rely on using a [singleton](https://stackoverflow.com/questions/12755539/why-is-singleton-considered-an-anti-pattern) (which makes unit testing harder), but it also introduces wormholes for the data to flow in an uncontrolled manner. *It's a harbinger of the coding hell!* The only case where I still use them is for observing notifications from Apple's frameworks where they didn't provide an alternative API. If you need a non-broken implementation of the Observer pattern – consider using `Event` or `Stream of values`
* **[KVO](https://nalexn.github.io/callbacks-part-1-delegation-notificationcenter-kvo/#KeyValueObserving)**. A very powerful technique that I use as the last resort when I have to hear back from a closed class and there is no other way to observe its state' changes. We all should be thankful for `KVO` – we wouldn't have reactive UIKit extensions without it (stream of values). The `Event` is a more practical alternative when you need an observable property.
* **[Responder Chain](https://nalexn.github.io/callbacks-part-2-closure-target-action-responder-chain/#Responder_chain)**. Since it cannot carry data in the notification, this technique is pretty much useless for our purpose. However, imagine we have a reference to the state and only need a trigger for a UI refresh, this technique would still be a bad choice. It builds an implicit and very fragile notification channel which is hard to test and maintain.

## Shared state management

OK, we have some data that needs to be accessible by multiple modules in our app. Can we make it a global variable or a singleton and call it a day? Yes, if we're on a few-hours hackathon and plan to throw the project away after it ends...

The mobile apps today have to deal with increasingly complicated business problems that involve not only feature-rich and responsive UI closely tied with the underlying data processing, but also complex state management for the data received from the server or created locally, cached in RAM or in database.

Every project has to determine a unified strategy for storing the data and its propagation through the system; failure to do it early inevitably leads to loss of control over the data flows, data consistency and a basic understanding of how the app functions.

Considering the common problems with the state I listed above, let me give some _recommendations_ on how to organize the shared state in the app.

### Single Source of Truth

You have to make a choice whether to cache the state in multiple places or store in one. In my opinion, the state complying with the principle of [Single Source of Truth](https://en.wikipedia.org/wiki/Single_source_of_truth) is more practical, since you don't need to worry about the outdated data - once you change the state - the old value is gone and cannot pop from somewhere else and ruin your user's experience.

Imagine we're developing a simple TODO list app. We store the list in the local database and also on the backend. When the app launches we show the local copy while running a request to the backend for obtaining a list possibly updated from another device.

If we don't care much about the user experience, we can freeze the UI until the request is complete, but we go ahead and allow the user to check-uncheck the tasks from the list and modify it in every other way.

The race condition between the user editing the local copy and the networking request completing after a delay would force us to implement the logic for merging the document edits when there is a conflict.

For this purpose, we can introduce a local database wrapper and rely on it as a single source of truth when it comes to providing the up-to-date list of tasks. This way we can avoid unnecessary complications of the program.

Once you unify the place to store the state, it becomes much easier to extend it or refactor as the project evolves: if later we decide that we need to add a separate editing screen for the task, in that screen we can safely read from that wrapper and be sure it always provides the newest data, regardless of who and when updated it.

### Restricted State Mutation

You need to put a barrier for the outer code not to be able to change the value and run away. What if there are other parties that need to know the value has changed? The responsibility of notifying the others should not lay on the caller side, because it's very easy to forget to add the required code in every place the data is mutated.

On the other hand, if we introduce a wrapper for the mutation operation, then we can send the notifications from that wrapper, as well as perform other operations with the data if needed.

In our example of the TODO list, that wrapper can be a facade that conceals both access to the local database as well as the backend calls, leaving the client code with a neat API telling nothing about where the data originated from and providing a simple gateway for the data mutation.

### Unidirectional data flow

This is another restriction you can implement in your app that will greatly improve the clarity and stability of the whole system.

Let's suppose we didn't put the backend API call behind the facade and originate the networking call directly from the main `ViewController`.

In this case, we would have two sources from where the data may come from - first is still functioning facade, another is the request completion callback, and we have to update the UI for both cases independently.

Back in the days, I worked on the app where the common practice was to send a `Notification` through `NotificationCenter` for every case of data mutation. There was one notification for a single record update and another notification for the whole list update. And of course, the networking callbacks were also providing either a list of a single record - and the UI had to handle data coming from 4 sources! Can you imagine writing tests for such structure?

In real apps the data flows can quickly multiply, requiring a consistent effort from developers on updating the existing functionality, and the number of such places can grow exponentially along with the evolution of the app.

The approach with implementing the [unidirectional data flow](https://flaviocopes.com/react-unidirectional-data-flow/) allows us to construct a unified channel for handling the data updates once and pretty much forget about its existence as we move on with the development.

All of this greatly contributes to the [principle of least astonishment](https://en.wikipedia.org/wiki/Principle_of_least_astonishment), making it easier for everyone on the project to quickly locate the state, all the possible conditions when it is mutated and the channels of the data distribution.

There are many ways to comply with all three patterns at once. One example is [Redux](https://github.com/reduxjs/redux) library, initially created for JavaScript world that later inspired the iOS community to build their own: [ReSwift](https://github.com/ReSwift/ReSwift).

As you design your shared state management with these three concepts and decide to use `Stream of values` in your project, you can easily utilize [binding the UI with the state](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/Examples.md#simple-ui-bindings), making the entire app super responsive to any state changes with clear declarative code for UI updates.

### Obtaining reference to the shared state

I would not recomment creating a global access point (a singleton object or a global variable) for the state, be that ReSwift's state or anything else. A much more practical approach that would ensure the best decoupling and isolation of modules using the shared state is aforementioned [dependency injection](https://en.wikipedia.org/wiki/Dependency_injection). For this purpose, you can utilize DI libraries like [typhoon](https://github.com/appsquickly/typhoon) or inject the instance by assigning the reference directly from the entity that creates the new module. For the latter this could be `AppDelegate` assigning the dependencies for the `RootViewController`, or some kind of `Builder` or `Router` that creates new `ViewController` and injects the dependencies right after.

---
> “Everything should be made as simple as possible, but no simpler.” - Albert Einstein

On one hand, we all need to aim for simplicity of the code we write, but there are shortcuts engineers should not take, and one of them is neglecting the design of solid state management in their apps.