---
layout: post
title: "Technical design"
date: 2019-12-05 8:30:00 +0300
description: "Verifying the rumors about SwiftUI performance bottlenecks"
tags: [iOS,swift,anyview,performance,issue,bottleneck,ConditionalView,Group,equatable,EquatableView]
comments: true
sharing: true
published: false
img: comparison_01.jpg
---


In this article I'll elabotare on the Why's: the motivation in the technical design for clean architecture.

I decided to structure the article is the following format - at first I declare a concept / term / principle that is an intrinsic part of a good software product, and then describe Why it is worth time and efforts to adhere to.

## Separation of concerns

The [separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns) is one of the most fundamental principles of the software developent. Two out of five SOLID principles (Single Responsibility and Interface Segregation) are direct derivations from this concept.

The principle is simple: don't write your program as a one solid block, instead, break up the code into modules that are finalized tiny pieces of the program each able to complete a simple distinct job.

The application of this principle on the micro level instructs us to re-structure the long complex function into a function calling multiple smaller functions containing refactored portions of the original code ([method extraction](https://refactoring.guru/extract-method)).

Each such function should be responsible for completing an intermediate step of the original algorithm with a clear objective reflected in the function's name. You can refer to Martin Fowler's [reasoning](https://martinfowler.com/bliki/FunctionLength.html) on this topic.

At a bit higher level, this principle tells us to group the functionality under self-contained modules, each responsible for the fulfillment of a single set of tasks that have clear logical correlation.

This is a non-trivial problem of [combinatorial optimization](https://en.wikipedia.org/wiki/Combinatorial_optimization) with two opposite processes going on: estrangement of less-closely related functionality and inclusion of features surving the same distinct purpose. 

1. Each module should **include** functions with [high cohesion](https://en.wikipedia.org/wiki/Cohesion_(computer_science)) between them.
2. The modules should **limit** the way it interoperates with other modules ([loose coupling](https://en.wikipedia.org/wiki/Coupling_(computer_programming))).

The module doesn't have to be a Class - it can be any container supported by the language able to encapsulate coherent functionality.

### Why separation of concerns is essential

Adhering to this principle helps to improve numerous characteristics of the code base:

1. Better code clarity. It is much easier to understand what is going on in the program when each module has a consize and clear API with logically scoped set of methods.
2. Better code reusability ([DRY principle](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)). The main benefit of reusing the code is reduced maintenance costs. Whenever you need to extend the functionality or fix a bug - it's much less painful to do when you're certain it appears in one place only.
3. Better testability. Independent modules with properly scoped functionality and isolation from the rest of the app are a breeze to test. You don't need to set up the entire environment to see how your module works - it is sufficient to replace neighboring real modules with dummy mocks or fake data sources. This way you can test the module as the black box by verifying just the output, or as the white box by also seeing which methods are being called on the connected modules ([BDD](https://en.wikipedia.org/wiki/Behavior-driven_development)).
4. Faster project evolution. Whether it's a new feature or an update of the existing one, isolation of the modules helps with scoping out the areas of the program that may be affected by the change, thus speeding up the development.
5. It is easier to organize simultaneous development by multiple engineers. They just need to agree on which module they are working on to make sure they don't interfere with each other. Only the update of a module's API can be a flag for explicit notifying other developers, while most of the changes can be added without immediate attention from the other contributors. When coupled with a good test coverage, the parallel development becomes as efficient as cumulative productivity of each individual engineer working solely ([it is usually slower](https://en.wikipedia.org/wiki/The_Mythical_Man-Month)).

Such benefits come at a cost - the engineers need to write more code and spend more time on designing the software infrastructure.

But the required time drastically goes down as the engineer gains more experience - they can go straight to the optimal solution instead of investigating, experimenting, debugging and re-writing the code over and over.

https://habr.com/en/post/423889/

## Layered design of the software


As we've seen how well is the impact of applying SoC for functions and modules, could it be used at a higher level of abstraction, at the application level?


The separation of concerns applied on the level of the programming functions helps us to refactor them in a more readable and maintainable structure.

The same principle allows us to group the functionality under modules for easier development.

If we take this 

You saw how the concept of separation of concerns brings clarity to the code inside programming functions and on the application level.

If we can take it one step further, we'll end up grouping the modules by relation to specific "layer" of the application - a very high level segregation of the entities based on their duty.

For the client applications there is a tendency to distinguish three layers:

* User Interface Layer
* Business Logic Layer
* Data Access Layer

When we start to think about modules' interoperation from this perspective, we can clearly see whether we should allow a module should ideally be fully isolated from the other - the same [loose coupling]

Overengineering

Don't try to solve non existing problems

THings that will likely happen

This may be a bit bold statement, but from my observation the backend development always goes slower than the front end.

backend is the enemy of the mobile.


Single Responsibility Pronciple from SOLID

# Repository

Persistence layer.

Isolation of the business logic from direct access to external data is essential for a good software.

There are multiple reasons why you don't want to embed direct read-write operations in your business logic:

1. You may accidentally corrupt the valuable data as you run an unfinished algorithm
2. The access to the real data may be slow (large file sizes for local resources, slow network / test server when accessing remote resources)
3. The external data may not be available (local database is empty and needs to be pre-populated, the server is down or there is no Internet for any reason)

You need to have a quick and relyable way to run your business logic in any curcumstances with no penalties to your productivity.

Thus, it should be possible to use another module that would mimic the behavior of providing access to external resources by returning lightweight 


Remember, a mobile enginner: the backend might seem you're allies, but when something happens on the backend, the angry users (and project managers) will blame you - as your app will always seem to the source of the problem.

Whenever you're writing an algorithm that manupulates with some data - it is the business logic of your app.

You should be able to quickly 

You don't want to test your new algorithm on the real database of the users. Not only the data may be corrupted after your first attempt to run your code, but also this might be slow

The only responsibility of the Repository is providing access to some external data. This can be a local data on the disk or a backend service.



###Why to have it?

