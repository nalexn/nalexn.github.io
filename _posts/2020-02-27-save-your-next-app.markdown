---
layout: post
title: "Save your next app from rebuilding from scratch"
date: 2020-02-27 19:30:00 +0300
description: "Implicit technical requirements that should not be neglected"
tags: [ios,swift,best practice,app,efficiency,testability,static code analysis,swiftlint,advanced,networking,dependencies,accessibility,localization,theming,logging,logs,analytics,crash reporting]
comments: true
sharing: true
published: true
img: sunrise_001.jpg
---

Imagine you start off a new project. There is no code written yet, and you have absolute freedom to how to build it.

You begin with gathering the business requirements and ultimately end up talking about the product features. Technical requirements for a mobile app usually default to general traits like "responsive UI", "smooth animations", "stable work", etc.

In this article, I've collected a list of implicit technical requirements that are often overlooked in the beginning but may impose costly refactoring or even a rebuild of the project later on.

> "Programmers can add features steadily to well-designed software."
> Kent Beck, [Responsive Design](https://magazines.pragprog.com/2009/pragpub-2009-09.pdf)

We cannot foresee the future and prepare for every pivot of the project, but we can design the system to be pliable for certain amendments. Let's get started.

# Programmatic navigation

I put this one first because this feature has an extreme combination of these two factors:

1. It is often overlooked at the start of the project
2. It has devastating consequences when integrated into the pre-existent codebase

Deep linking from a push notification, spotlight search, or quick actions are considered as "nice to have" features at best. It is often not required for the MVP, and the manager may explicitly tell you not to work on this for now.

But even without the deep linking your app might still require to do some nontrivial routing, such as opening a different tab followed by presenting a modal screen, or dismissing two modal screens stacked one above the other.

There are numerous ways how these local navigation events can be coded: using a bunch of delegates forwarding the command one to another, utilizing the Responder Chain, Closure callbacks or even by broadcasting a Notification through NotificationCenter, but almost every solution applied to an unprepared system will be an astronomical code smell with numerous hacks like async dispatch.

The ideal case is to have your app be ready to execute the command of opening a specific screen regardless of what screen is shown right now, but this requires an advanced screen coordinator tailored for the project's routing model.

A "good enough" solution to this problem is the use of standard [Coordinator](https://www.hackingwithswift.com/articles/71/how-to-use-the-coordinator-pattern-in-ios-apps) pattern for UIKit apps or [centralized navigation state](https://nalexn.github.io/swiftui-deep-linking/) for SwiftUI.

It doesn't add much overhead to the codebase but makes your app's routing much more flexible and testable, so I tend to include this in almost every project except little ones.

Testability is another quite significant reason why you want to have a decoupled programmatically-controlled navigation in your project.

# Manageable data flows

The functionality of the apps tends to grow over time. An increasing number of cases in the business logic often leads to the multiplication of the data flows in the app: there could be several screens operating with the same set of data, and the concern of synching the changes between the screens may become a big problem.

In the beginning, if we didn't design a clear, unified data flow, the related complexity of maintaining the control over the state and data would be growing exponentially with the number of entities we add.

In order to address this problem, we can appeal to the "single source of truth" principle, where we explicitly avoid state duplication and design a subscription-based mechanism for the UI refreshes.

[WWDC '19: Data flow through SwiftUI](https://developer.apple.com/videos/play/wwdc2019/226/) explains how this concept is used in SwiftUI, while in UIKit we need to put additional efforts in designing the state update distribution channels.

Unidirectional data flow, used in [Clean Swift](https://clean-swift.com/) and [Redux](https://github.com/ReSwift/ReSwift), has proven to be an extremely scalable solution for managing the state with a growing app's complexity.

I elaborate more on this topic in the [state management guide](https://nalexn.github.io/state-management-guide-ios/) for UIKit, and [Clean Architecture for SwiftUI](https://nalexn.github.io/clean-architecture-swiftui/).

# Testability

There are different points of view regarding the usefulness of tests. Tests can be the top priority in [TDD](https://www.guru99.com/test-driven-development.html) workflow, just as they could be fully neglected for a prototype or an MVP, especially in a case of a tight timeline.

In a scenario when you choose not to write the tests, you still should structure the project so the tests could be added later if you want to.

Testability is the trait that does not only allow the presence of the tests but ultimately contributes to the reusability of the components and clearer [separation of concerns](https://nalexn.github.io/separation-of-concerns/) in the project.

A way to achieve testability in the project:

* Break up the modules with more than one responsibility ([SRP](https://en.wikipedia.org/wiki/Single_responsibility_principle))
* [Avoid](https://www.swiftbysundell.com/articles/avoiding-singletons-in-swift/) using any global variables, references to objects (including Singletons), and [inject the dependencies](https://ilya.puchka.me/dependency-injection-in-swift/) instead.

# Static code analysis

One of the first things I do for a new project is configuring the static code analysis tools, such as **SwiftLint** and **cpd**.

[SwiftLint](https://github.com/realm/SwiftLint) is an absolute must-have. Whether you work solo or in a team, this tool allows you to adhere to a unified code style and avoid overcomplicated or syntactically excessive code.

The best part - it warns you early on as you write the code. This makes the code review more productive and greatly contributes to the clarity and readability of the codebase. I often add custom rules when I want to forbid the use of certain APIs ([NotificationCenter](https://objcsharp.wordpress.com/2013/08/28/why-nsnotificationcenter-is-bad/), for example).

[Copy-paste-detector](https://pmd.github.io/latest/pmd_userdocs_cpd.html) (cpd) helps with identifying not just an obviously duplicated code, but also the code chunks that have a very similar structure, which are good candidates for refactoring into a reusable method or entity, so it turns out really helpful even when you unintentionally duplicated some functionality.

# Networking

Networking is a fairly big topic, so I'll just list what can potentially become needed as the project evolves:

* [Chaining](https://nalexn.github.io/callbacks-part-3-promise-event-stream/#Promise_advantages) several requests without nested callbacks
* Handling malformed responses from the server
* [Proper authentication](https://developer.apple.com/documentation/foundation/url_loading_system/handling_an_authentication_challenge) and handling of the 401 Unauthorized
* Response mocking for unit testing or offline demo mode
* Option to cancel a running request or perform an automatic retry

I prefer writing a custom lightweight wrapper around standard URLSession instead of using any third-party tools like Alamofire or Moya. The custom wrapper can be easier adapted for the project needs, while capabilities of the plain URLSession are incredibly underrated.

Alamofire and other libraries are wrappers around URLSession anyway, so why to bring in a heavyweight dependency in the project if you can easily go without it?

# Dependencies

Since I touched the topic of the third-party dependencies in the project, here is an advice for you: always think twice before adding any new dependency. There are many reasons why you want to have as few of these as possible in your project:

* Often times the frameworks are overengineered to cover a broader range of use cases
* Massive dynamic libraries and frameworks [increase](https://engineering.fb.com/data-infrastructure/messenger/) the launch time of the app
* Your app bloats up in size.
* Xcode has to spend more time on the compilation, DerivedData folder becomes a gigantic dump of the build artifacts
* Third-party code often has a mediocre quality, poor test coverage, and bugs
* You cannot expect every dependency to be migrated to the new Swift or iOS version quickly. Many libraries become abandoned over time
* If you find that the framework is missing even a tiny bit of functionality you need, you can try submitting a pull request - but not all maintainers respond in a timely manner, so you may be stuck
* Frameworks may frequently revisit their APIs with every major release. Migration to the new version can be very labor-intensive

# Memory and battery consumption

A quote from the [Energy Efficiency Guide](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/EnergyGuide-iOS/index.html) from Apple:

> Even small inefficiencies in apps add up, significantly affecting battery life, performance, and responsiveness. As an app developer, **you have an obligation** to make sure your app runs as efficiently as possible.

The truth is, as long as the app works, memory and battery efficiency is almost never considered important by the product managers or developers, but it should - these parameters directly affect the user experience and ultimately separate good apps from the best.

I bet you've been in the situation when the app you minimized just for 10 seconds re-launches when you call it back, losing all the text you typed.

This is the result of the app neglecting the [memory consumption policy](https://developer.apple.com/documentation/uikit/app_and_environment/scenes/preparing_your_ui_to_run_in_the_background): the more memory it retains while backgrounded, the more are the chances of iOS terminating the app for reclaiming the necessary memory resources.

What you want to do is to handle the `applicationDidEnterBackground` and free up caches and other resources that can be restored from the copy on the disk.

# State restoration

Of course, even if you freed up all the memory possible preparing for the background mode, there is no guarantee the system won't kill the app.

A way to mitigate the problem is to [restore the state](https://developer.apple.com/documentation/uikit/view_controllers/preserving_your_app_s_ui_across_launches) when the user comes back.

You need to be cautious, though, and respect the user's session expiration and abort the restoration early, showing the sign-in UI as required.

The state restoration feature is crucial for the apps where users are spending a fair amount of time, and their progress is valuable for them: frustration from the lost data may drive the uninstalls and bad reviews for your app.

# Accessibility

[Accessibility](https://developer.apple.com/design/human-interface-guidelines/accessibility/overview/introduction/) helps people with limitations use your app. Although you're not always tasked to explicitly support it, every app responds to the global settings, such as the increased text size, which can easily break your UI.

You don't want this to happen, so it's best to test the app against various settings that users can adjust under **Settings - Accessibility**.

If you're working on a SwiftUI app, you can use the [tool](https://github.com/nalexn/EnvironmentOverrides) I've created specifically for this purpose:

<div style="max-width:255px; display: block; margin-left: auto; margin-right: auto;"><img src="https://github.com/nalexn/blob_files/blob/master/images/EnvironmentOverrides.gif?raw=true"></div>

# Localization

Most of the time, developers are asked to build the app just for the target market with one localization. A sudden new requirement that can catch the developer off guard is the need to [localize](https://developer.apple.com/internationalization/) already built app for a few more languages.

But you can be prepared for this.

In UIKit, it takes just a little discipline to never use `"Plain text string"` in the project and always wrap them in the `NSLocalizedString`, like so:

```swift
NSLocalizedString("Text", comment: "")
```

After that, you can run an [automated tool](https://github.com/SwiftGen/SwiftGen) that collects all the strings in your project under a `Localizable.strings` file that you can work with.

In SwiftUI, it's a little bit easier because `Text` view considers the strings localizable by default.

Another case where localization might not take the desired effect automatically is various formatters, such as `DateFormatter` or `NumberFormatter`.

All the formatters have a property `locale` that you need to configure appropriately, either by setting to `Locale.current` or `Locale(identifier: ...)`

Essentially, this is all that's required from you.

You can utilize the same tool I mentioned above for testing how the app adjusts for different localizations.

# Theming

You should always refrain from copy-pasting hardcoded values for colors, font names, and other appearance-related parameters throughout your project, because one day, you may be tasked to do a slight redesign.

Tweaking one color or font for the entire project shouldn't involve changing more than a couple of lines of code, and the most convenient way to achieve this is to use the [UIAppearance](https://developer.apple.com/documentation/uikit/uiappearance) for configuring the global theme settings.

In SwiftUI, you use the view modifiers, such as `.accentColor(...)`, applying them to the root view for changing the appearance for the entire hierarchy.

Considering that the apps now should ideally support alternative [Dark Mode](https://developer.apple.com/documentation/xcode/supporting_dark_mode_in_your_interface), it's just easier to do the things the right way from the outset and save yourself hours of tedious refactoring later on.

For the detailed instructions, you can refer to the official guides (links I left above).

# Logs, analytics, and crash reporting

Another "optional" requirement that pops up down the road is the need to troubleshoot what's going on with the app for your users.

Be that a crash or another unexpected erroneous behavior of the app - you won't have a debugger attached to the program when this happens.

The crash reporting tool is a must-have for any app in production. Not only does it allow you to know how stable your app is, but it also provides you with information extremely helpful for troubleshooting, including the call stack trace and the statistics of how often the crash happens for the users.

There are numerous solutions on the market, but my favorite free one is [Firebase Crashlytics](https://firebase.google.com/docs/crashlytics) from Google.

The crash reporting platforms often come with an option to gather usage analytics, so look up if that's supported and log the key user's actions, such as visiting a screen where you offer paid features or products.

While analytics is mainly useful for gathering business-related statistics, you may still want to log more thoroughly for troubleshooting purposes.

A while ago, I wrote an [article](https://nalexn.github.io/to_log_or_not_to_log/) dedicated to the problem of designing the logging system that is not a burden to maintain.

And finally, remember to exclude any sensitive information from the logging and analytics, as this [might turn](https://krebsonsecurity.com/2019/03/facebook-stored-hundreds-of-millions-of-user-passwords-in-plain-text-for-years/) substantial reputational and legal trouble for your company.

# Conclusion

As you can see, there are quite a lot of hidden requirements that may or may not become apparent, so please, don't overengineer:

> "The temptation is to put these design ideas in the system now because you just know you’ll need them *eventually*. Over-designing early leads to delaying feedback from real usage of the system, makes adding features more complicated, and makes adapting the design more difficult."
> Kent Beck, [Responsive Design](https://magazines.pragprog.com/2009/pragpub-2009-09.pdf)

While I totally agree with this statement, most of the practices I collected in this article require minimal effort and discipline, plus have almost no potential of causing any harm.

Doing things right from the outset is not always possible, but these techniques may save you hours of unnecessary work.