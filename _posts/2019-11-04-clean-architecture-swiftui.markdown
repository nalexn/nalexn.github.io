---
layout: post
title: "Clean Architecture for SwiftUI"
date: 2019-11-04 14:30:00 +0300
description: "Are VIPER, RIBs, MVVM, VIP, or MVC suitable for a SwiftUI project?"
tags: [MVVM,viewmodel,iOS,swift,SwiftUI,Combine,design,pattern,redux,unidirectional,data,flow,model,state,management]
comments: true
sharing: true
published: true
img: clean_swiftui_01.jpg
---
Can you imagine, UIKit is 11 years old! Ever since the release of the iOS SDK in 2008 we were building our apps with it. And throughout this time the developers were in a relentless search for the best architecture to use for their apps. It all started with **MVC**, but later we witnessed the rise of **MVP**, **MVVM**, **VIPER**, **RIBs**, and **VIP**.

But something has happened recently. This "something" is so significant, that the majority of the architectural patterns used for iOS will soon become history.

I'm talking about SwiftUI. It's not going anywhere. Like it or not, this is the future of iOS development. And it's a game-changer in terms of the challenges we face when designing the architecture.

# What are the conceptual changes?

UIKit was an **imperative, event-driven** framework. We could reference each view in the hierarchy, update its appearance when the view is loaded or as a reaction on an event (a touch-down on the button or a new data becoming available for display in UITableView). We used callbacks, delegates, target-actions for handling these events.

Now, it is all gone. SwiftUI is a **declarative, state-driven** framework. We cannot reference any view in the hierarchy, neither can we directly mutate a view as a reaction to an event. Instead, we mutate the state bound to the view. Delegates, target-actions, responder chain, KVO, — [the entire zoo of callback techniques](https://nalexn.github.io/callbacks-part-1-delegation-notificationcenter-kvo/) have been replaced with closures and bindings.

Every view in SwiftUI is a struct that can be created many times faster than an analogous UIView descendant. That struct keeps references to the state that it feeds to the function `body` for rendering the UI.

<div style="max-width:600px; display: block; margin-left: auto; margin-right: auto;"><img src="{{ site.url }}/assets/img/clean_swiftui_02.jpg"></div>

So a view in SwiftUI is just a programming function. You provide it with the input (state) — it draws the output. And the only way to change the output is to change the input: we cannot touch the algorithm (the body function) by adding or removing subviews — all the possible alterations in the displayed UI have to be declared in the body and cannot be changed in runtime.

In terms of the SwiftUI we’re not adding or removing subviews, but enabling or disabling different pieces of the UI in the predefined flowchart algorithm.

# MVVM is the new standard architecture

SwiftUI comes with MVVM built-in.

In the simplest case, where the `View` does not rely on any external state, its local `@State` variables take the role of the `ViewModel`, providing the subscription mechanism (`Binding`) for refreshing the UI whenever the state changes.

For more complex scenarios, `Views` can reference an external `ObservableObject`, which in this case can be a distinct `ViewModel`.

One way or another, the way SwiftUI views work with the state very much resembles the classical MVVM (unless we introduce a more complex graph of programming entities).

<div style="max-width:800px; display: block; margin-left: auto; margin-right: auto;"><img src="{{ site.url }}/assets/img/clean_swiftui_03.jpg"></div>

> And well, you don't need a ViewController anymore.

Let's consider this quick example of the MVVM module for a SwiftUI app.

**Model**: a data container

```swift
struct Country {
    let name: String
}
```

**View**: a SwiftUI view

```swift
struct CountriesList: View {
    
    @ObservedObject var viewModel: ViewModel
    
    var body: some View {
        List(viewModel.countries) { country in
            Text(country.name)
        }
        .onAppear {
            self.viewModel.loadCountries()
        }
    }
}
```

**ViewModel**: an `ObservableObject` that encapsulates the business logic and allows the `View` to observe changes of the state

```swift
extension CountriesList {
    class ViewModel: ObservableObject {
        @Published private(set) var countries: [Country] = []
        
        private let service: WebService
        
        func loadCountries() {
            service.getCountries { [weak self] result in
                self?.countries = result.value ?? []
            }
        }
    }
}
```

In this simplified example, when the `View` appears on the screen, the `onAppear` callback calls `loadCountries()` on the `ViewModel`, triggering the networking call for loading the data inside `WebService`. `ViewModel` receives the data in the callback and pushes the updates through `@Published` variable `countries`, observed by the `View`.

<div style="max-width:900px; display: block; margin-left: auto; margin-right: auto;"><img src="https://github.com/nalexn/blob_files/blob/master/images/swiftui_arc_002_d.png?raw=true" alt="Diagram"/></div>
<div style="width:0px; height:20px; display: block;"></div>

Although this article is dedicated to Clean Architecture, I was receiving many questions about the application of MVVM in SwiftUI, so I took the original [sample project](https://github.com/nalexn/clean-architecture-swiftui) and ported it to MVVM in a [separate branch](https://github.com/nalexn/clean-architecture-swiftui/tree/mvvm). You can compare the two and choose which suits your needs best. The project's key features:

* Vanilla SwiftUI + Combine implementation
* Decoupled Presentation, Business Logic, and Data Access layers
* Full test coverage, including the UI (thanks to the [ViewInspector](https://github.com/nalexn/ViewInspector))
* Redux-like centralized AppState as the single source of truth
* Programmatic navigation (deep links support)
* Simple yet flexible networking layer built on Generics
* Handling of the system events (blurring the view hierarchy when the app is inactive)

# Under the hood, SwiftUI is based on ELM

Just watch a couple minutes from this talk "MCE 2017: Yasuhiro Inami, Elm Architecture in Swift" from 28:26

<div style="position: relative; width: 100%; padding-bottom: 56.25%; height: 0;">
<iframe width="560" height="315" style="position: absolute; top:0; left: 0; width: 100%; height: 100%;" src="https://www.youtube.com/embed/U805TqsDIV8?start=1706" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

That guy had a WORKING prototype of SwiftUI in 2017!

Does it feel like we're on a reality show where SwiftUI, a half-orphan kid, has just got to know who his father is?

Anyways, what interests us is whether we can use any other ELM concepts for making our SwiftUI apps better.

I followed the [ELM Architecture](https://guide.elm-lang.org/architecture/) description on the ELM language's web site and... found nothing new. SwiftUI is based on the same essences as ELM:

> * Model — the state of your application
> * View — a way to turn your state into HTML
> * Update — a way to update your state based on messages

We saw this somewhere, didn't we?

<div style="max-width:600px; display: block; margin-left: auto; margin-right: auto;"><img src="{{ site.url }}/assets/img/clean_swiftui_04.jpg"></div>

We already have the `Model`, the `View` gets generated automatically from the `Model`, the only thing we can tweak is the way `Update` in delivered. We can go **REDUX** way and use the `Command` pattern for state mutation instead of letting SwiftUI’s views and other modules write to the state directly.
Although I preferred using REDUX in my previous UIKit projects (ReSwift ❤), it’s questionable whether it’s needed for a SwiftUI app — the data flows are already under control and are easily traceable.

# Coordinator is history

Coordinator (aka Router) was an essential part of VIPER, RIBs and MVVM-R architectures. Allocation of a separate module for screen navigation was well justified in UIKit apps – the direct routing from one ViewController to another led to their tight coupling, not to mention the coding hell of deep linking to a screen deeply inside the ViewController's hierarchy.

Unlike with the views in UIKit, we cannot take a SwiftUI view and ask it to layout subviews, or render onto an image context. A SwiftUI view is absolutely useless without the render engine, which also owns the state, and the views only receive a reference to that state at render time, [even when they use a local @State](https://nalexn.github.io/stranger-things-swiftui-state/)).

**A SwiftUI View is nothing more than a drawing algorithm**. That's why it's very difficult to extract routing off the SwiftUI view: **routing is an integral part of this algorithm**.

We should not fight its nature. Instead, we should structure the program so that major pieces of this drawing algorithm would reside in separate views, with business logic extracted in a plain-struct modules that are easy to test in isolation.

I believe that SwiftUI made the router (aka coordinator) needless.

Every view that alters the displayed hierarchy, be that `NavigationView`, `TabView` or `.sheet()`, now uses `Binding` to control what's displayed.

`Binding` is an "unpossessed" form of a state variable - you can read and write it, but the factual value belongs to another module.

When the user selects a tab in the `TabView`, you don't get a callback. You simply cannot. What happens instead, is that the `TabView` unilaterally changes the value through `Binding` to "displayedTab = .userFavorites".

The programmer can also assign a value to that `Binding` at any time - and the `TabView` will obey immediately.

The programmatic navigation in SwiftUI is fully controlled by the state through `Bindings`. I dedicated a [separate article]({{ site.url }}/swiftui-deep-linking/) to this problem.

# Are VIPER, RIBs and VIP applicable for SwiftUI?

There are a lot of great ideas and concepts we can borrow from these architectures, but ultimately the canonical implementation of either one doesn't make sense for the SwiftUI app.

First, as you already know, there is no more practical need to have a `Router`.

Secondly, the completely new design of the data flow in SwiftUI coupled with native support of view-state bindings shrank the required setup code to the degree that `Presenter` becomes a goofy entity doing nothing useful.

Along with the decreased number of modules in the pattern, we figure out that we don’t need `Builder` either. So basically, the whole pattern just falls apart, as **the problems it aimed to solve don't exist anymore**.

SwiftUI introduced its own set of challenges in the system’s design, so the patterns we had for UIKit have to be re-designed from the ground up.

There are [attempts](https://theswiftdev.com/2019/09/18/how-to-build-swiftui-apps-using-viper/) to stick with the beloved architectures no matter what, but please, don’t.

# Clean Architecture

Let's refer to [Uncle Bob's Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html), the progenitor of VIP.

> By separating the software into layers, and conforming to The Dependency Rule, you will create a system that is intrinsically testable, with all the benefits that imply.

Clean Architecture is quite liberal about the number of layers we should introduce because this depends on the application domain.

But in the most common scenario for a mobile app we'll need to have three layers:

* Presentation layer
* Business Logic layer
* Data Access layer

So if we distilled the requirements of the Clean Architecture through the peculiarity of SwiftUI, we'd come up with something like this:

<div style="max-width:720px; display: block; margin-left: auto; margin-right: auto;"><img src="https://github.com/nalexn/blob_files/blob/master/images/swiftui_arc_001_d.png?raw=true" alt="Diagram"/></div>

There is a [demo project](https://github.com/nalexn/clean-architecture-swiftui) I've created to illustrate the use of this pattern. The app talks to the [restcountries.eu](https://restcountries.eu/) REST API to show the list of countries and details about them.

# AppState

AppState is the only entity in the pattern that requires to be an object, specifically, an `ObservableObject`. [Alternatively](https://nalexn.github.io/swiftui-observableobject/), it can be a struct wrapped in a `CurrentValueSubject` from Combine.

Just like with **Redux**, AppState works as the single source of truth and keeps the state for the entire app, including user's data, authentication tokens, screen navigation state (selected tabs, presented sheets) and system state (is active / is backgrounded, etc.)

AppState knows nothing about any other layer and does not contain any business logic.

An example of the [AppState](https://github.com/nalexn/clean-architecture-swiftui/blob/master/CountriesSwiftUI/Injected/AppState.swift) from the Countries demo project:

```swift
class AppState: ObservableObject, Equatable {
    @Published var userData = UserData()
    @Published var routing = ViewRouting()
    @Published var system = System()
}
```

# View

This is the usual SwiftUI's view. It may be stateless or have local `@State` variables.

No other layers know about the View layer existence, so there is no need to hide it behind a protocol.

When the view is instantiated, it receives
`AppState` and `Interactor` through the SwiftUI's standard dependency injection of a variable attributed with `@Environment`, `@EnvironmentObject` or `@ObservedObject`.

Side effects are triggered by the user's actions (such as a tap on a button) or view lifecycle event `onAppear` and are forwarded to the `Interactor`.

```swift
struct CountriesList: View {
    
    @EnvironmentObject var appState: AppState
    @Environment(\.interactors) var interactors: InteractorsContainer
    
    var body: some View {
        ...
        .onAppear {
            self.interactors.countriesInteractor.loadCountries()
        }
    }
}
```

# Interactor

`Interactor` encapsulates the business logic for the specific `View` or a group of views. Together with the `AppState` forms the Business Logic layer, that's fully independent of the presentation and the external resources.

It is fully stateless and only refers to the `AppState` object, injected as a constructor parameter.

Interactors should be "facaded" with a protocol so that the `View` could talk to a mocked `Interactor` in tests.

Interactors receive requests to perform work, such as obtaining data from an external source or making computations, but they never return data back directly, such as in a closure.

Instead, they forward the result to the `AppState` or a `Binding` provided by the View.

The `Binding` is used when the result of work (the data) is owned locally by one View and does not belong to the central `AppState`, that is, it doesn't need to be persisted or shared with other screens of the app.

[CountriesInteractor](https://github.com/nalexn/clean-architecture-swiftui/blob/master/CountriesSwiftUI/Interactors/CountriesInteractor.swift) from the demo project:

```swift
protocol CountriesInteractor {
    func loadCountries()
    func load(countryDetails: Binding<Loadable<Country.Details>>, country: Country)
}

// MARK: - Implemetation

struct RealCountriesInteractor: CountriesInteractor {
    
    let webRepository: CountriesWebRepository
    let appState: AppState
    
    init(webRepository: CountriesWebRepository, appState: AppState) {
        self.webRepository = webRepository
        self.appState = appState
    }

    func loadCountries() {
        appState.userData.countries = .isLoading(last: appState.userData.countries.value)
        weak var weakAppState = appState
        _ = webRepository.loadCountries()
            .sinkToLoadable { weakAppState?.userData.countries = $0 }
    }

    func load(countryDetails: Binding<Loadable<Country.Details>>, country: Country) {
        countryDetails.wrappedValue = .isLoading(last: countryDetails.wrappedValue.value)
        _ = webRepository.loadCountryDetails(country: country)
            .sinkToLoadable { countryDetails.wrappedValue = $0 }
    }
}
```

# Repository

`Repository` is an abstract gateway for reading / writing data.
Provides access to a single data service, be that a web server or a local database.

I have a [dedicated article](https://nalexn.github.io/separation-of-concerns/) explaining why extracting the Repository is essential.

For example, if the app is using its backend, Google Maps APIs and writes something to a local database, there will be three Repositories: two for different web API providers and one for database IO operations.

The repository is also stateless, doesn't have write access to the `AppState`, contains only the logic related to working with the data. It knows nothing about `View` or `Interactor`.

The factual Repository should be hidden behind a protocol so that the `Interactor` could talk to a mocked `Repository` in tests.

[CountriesWebRepository](https://github.com/nalexn/clean-architecture-swiftui/blob/master/CountriesSwiftUI/Repositories/CountriesWebRepository.swift) from the demo project:

```swift
protocol CountriesWebRepository: WebRepository {
    func loadCountries() -> AnyPublisher<[Country], Error>
    func loadCountryDetails(country: Country) -> AnyPublisher<Country.Details.Intermediate, Error>
}

// MARK: - Implemetation

struct RealCountriesWebRepository: CountriesWebRepository {
    
    let session: URLSession
    let baseURL: String
    let bgQueue = DispatchQueue(label: "bg_parse_queue")
    
    init(session: URLSession, baseURL: String) {
        self.session = session
        self.baseURL = baseURL
    }
    
    func loadCountries() -> AnyPublisher<[Country], Error> {
        return call(endpoint: API.allCountries)
    }

    func loadCountryDetails(country: Country) -> AnyPublisher<Country.Details, Error> {
        return call(endpoint: API.countryDetails(country))
    }
}

// MARK: - API

extension RealCountriesWebRepository {
    enum API: APICall {
        case allCountries
        case countryDetails(Country)
        
        var path: String { ... }
        var httpMethod: String { ... }
        var headers: [String: String]? { ... }
    }
}
```

Since WebRepository takes URLSession as a constructor parameter, it is very easy to test it by [mocking the networking calls](https://github.com/nalexn/clean-architecture-swiftui/blob/master/UnitTests/NetworkMocking/RequestMocking.swift) with a custom `URLProtocol`

# Final thoughts

[The demo project](https://github.com/nalexn/clean-architecture-swiftui) now has **97% test coverage**, all thanks to the Clean Architecture's "dependency rule" and segregation of the app on multiple layers.

It offers fully setup persistence layer with CoreData, deep linking from a Push Notification, and other non-trivial yet practical examples.

<div style="max-width:800px; display: block; margin-left: auto; margin-right: auto;"><img src="https://github.com/nalexn/blob_files/blob/master/images/countries_preview.png?raw=true" alt="Diagram"/></div>