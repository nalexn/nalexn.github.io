---
layout: post
title: "Fighting state redundancy in Model-View-ViewModel"
date: 2019-09-29 17:30:00 +0600
description: "A better way to represent the state of loading resource in MVVM"
tags: [mvvm,model,view,viewmodel,view-model,state,frp,rxswift,functional,reactive,ios,swift]
comments: true
sharing: true
published: true
img: state_002.jpg
---

One of the most common practical problems in mobile apps is loading displayable data from the server, where the data can be anything from user's feed or a list of podcasts to a profile picture or a streaming video.

Apps show various spinners and bars to indicate the loading process, all for inducing user's patience and improving their experience.

However, this seemingly trivial problem has hidden pitfalls for a programmer, where a naive implementation can lead to writing excessive code and fixing sudden bugs.

Let's take a simple example where we need to load and display a list of podcasts using [Model-View-ViewModel](https://en.wikipedia.org/wiki/Model-view-viewmodel) pattern for the screen structure with [RxSwift](https://github.com/ReactiveX/RxSwift) for UI binding.

### Model

An immutable struct for holding basic info about a podcast record:

```swift
struct Podcast: Equatable {
    let title: String
    let previewImage: UIImage
    let podcastURL: URL
}
```

The networking layer is represented with this tiny protocol; I'll omit the factual networking code for simplicity of the example:

```swift
protocol PodcastsService {
    func loadPodcasts(completion: (Result<[Podcast], Error>) -> Void)
}
```

### ViewModel

For the list of podcasts, we'll need to have an observable array of `Podcast` structures. For that, we wrap `[Podcast]` in the `BehaviorRelay` class from `RxSwift`, which is basically an **observable property** that always holds a value:

```swift
class PodcastsViewModel {

    private let service: PodcastsService
    
    let podcasts = BehaviorRelay<[Podcast]>(value: [])
    
    func loadPodcasts() {
        service.loadPodcasts { [weak self] result in
            let records = result.value ?? []
            self?.podcasts.accept(records)
        }
    }
}
```

As you can see, we provided the ViewModel with access to the networking layer through a reference to `PodcastsService`.

The array of `Podcast` records is initially empty, but `loadPodcasts()` function allows the user of the ViewModel to query the podcasts at the right time, and as the request completes it updates the list of podcasts.

### View

A simple TableViewCell for displaying the Podcast info:

```swift
class PodcastCell: UITableViewCell {

    func populate(podcast: Podcast) {
        textLabel?.text = podcast.title
        imageView?.image = podcast.previewImage
    }
}
```
And finally the ViewController that shows the list of podcasts:

```swift
class PodcastsViewController: UIViewController {

    @IBOutlet weak var tableView: UITableView!
    var viewModel: PodcastsViewModel!
    var disposeBag = DisposeBag()
    
    func viewDidLoad() {
        super.viewDidLoad()
        let cellIdentifier = String(describing: PodcastCell.self)
        tableView.register(PodcastCell.self, forCellReuseIdentifier: cellIdentifier)
        viewModel.podcasts
            .bind(to: tableView.rx.items(cellIdentifier: cellIdentifier, cellType: PodcastCell.self)) { (row, country, cell) in
                cell.populate(podcast: podcast)
            }
            .disposed(by: disposeBag)
        viewModel.loadPodcasts()
    }
}
```

Considering that the `viewModel` is instantiated and injected to the `ViewController` from the outside, now we have a fully functioning `MVVM` module that automatically loads and displays the list of podcasts.

The only problem is that the app is currently not user-friendly. It doesn't have an indicator for the loading process; neither does it show an error that can possibly occur in the networking layer.

Let's update the ViewModel to address the challenge:

```swift
class PodcastsViewModel {

    private let service: PodcastsService
    
    let podcasts = BehaviorRelay<[Podcast]>(value: [])
﹢  let isLoading = BehaviorRelay<Bool>(value: false)
﹢  let onError = PublishRelay<Error> = PublishRelay()
    
    func loadPodcasts() {
﹢      isLoading.accept(true)
        service.loadPodcasts { [weak self] result in
﹢          self?.isLoading.accept(false)
﹢          if let error = result.error {
﹢              self?.onError.accept(error)
﹢          }
            let records = result.value ?? []
            self?.podcasts.accept(records)
        }
    }
}
```
We've added `isLoading` boolean property for tracking the loading status, and a `onError` PublishRelay for reporting an error.

Now we need to update the UI and bind it with the status updates:

```swift
class PodcastsViewController: UIViewController {

    @IBOutlet weak var tableView: UITableView!
﹢  @IBOutlet weak var errorMessageLabel: UILabel!
﹢  @IBOutlet weak var indicatorView: UIActivityIndicatorView!
    var viewModel: PodcastsViewModel!
    var disposeBag = DisposeBag()
    
    func viewDidLoad() {
        super.viewDidLoad()
        // ...
﹢      viewModel.isLoading.bind(to: indicatorView.rx.isAnimating).disposed(by: disposeBag)
﹢      viewModel.onError.subscribe(onNext: { [weak self] error in
﹢          self?.errorMessageLabel?.text = error.localizedDescription
﹢      }).disposed(by: disposeBag)
﹢      viewModel.podcasts.map { $0.count > 0 }
﹢        .bind(to: errorMessageLabel.rx.isHidden).disposed(by: disposeBag)
    }
}
```

OK, now we're showing `UIActivityIndicatorView` when `viewModel.isLoading` reports that the podcasts are loading, and there is a `UILabel` for showing errors emitted by `viewModel.onError`

Note, that in the lines 16-17 we also had to explicitly hide the `errorMessageLabel` when `podcasts` has any records, because otherwise, the error message would stay on the screen even if the podcasts request succeeds after it initially errored out.

You can already sense a code smell. For such a simple use case we've already started fighting the state inconsistency.

Let's take a deep breath in and out, and think for a minute. What are the possible states that our screen actually can be?

1. Podcasts have not yet been queried
2. Podcasts are being loaded
3. Podcasts have failed to load
4. Podcasts were loaded successfully

That's right, just **four** cases! The superposition of the state values we have right now (`podcasts` being empty or not, `isLoading` and `onError`) gives us `2*2*2 = 8` possible cases. **This state redundancy is the reason why we had to implement a fix for conflicting values of `podcasts` and `onError`**. In fact, this means there are other combinations we didn't consider yet, which can lead to unwanted effects, such as simultaneous display of the _error message_ and the _loading indicator_.

As we've identified the problem, we can refactor the current implementation to purge the state redundancy by introducing `enum` with those four cases:

```swift
enum Loadable<Value> {
    case notRequested
    case isLoading
    case loaded(Value)
    case failed(Error)
}
```
This neat `enum` allows us to rework the `ViewModel` in the following way:

```swift
class PodcastsViewModel {

    private let service: PodcastsService
    
    let podcasts = BehaviorRelay<Loadable<[Podcast]>>(value: .notRequested)
    
    func loadPodcasts() {
        podcasts.accept(.isLoading)
        service.loadPodcasts { [weak self] result in
            switch result {
            case let .success(value):
                self?.podcasts.accept(.loaded(value))
            case let .failure(error):
                self?.podcasts.accept(.failed(error))
            }
        }
    }
}
```
With just one variable we were able to represent all the states we had, but now – without the state redundancy.


Refactoring the ViewController:

```swift
class PodcastsViewController: UIViewController {

    @IBOutlet weak var tableView: UITableView!
    @IBOutlet weak var errorMessageLabel: UILabel!
    @IBOutlet weak var indicatorView: UIActivityIndicatorView!
    var viewModel: PodcastsViewModel!
    var disposeBag = DisposeBag()
    
    func viewDidLoad() {
        super.viewDidLoad()
        // ...
﹢      viewModel.podcasts.map { $0.isLoading }
            .bind(to: indicatorView.rx.isAnimating).disposed(by: disposeBag)
﹢      viewModel.podcasts.map { $0.error?.localizedDescription }
            .bind(to: errorMessageLabel.rx.text).disposed(by: disposeBag)
﹢      viewModel.podcasts.map { $0.value ?? [] }
            .bind(to: tableView.rx.items(cellIdentifier: cellIdentifier, cellType: PodcastCell.self)) { (row, country, cell) in
                cell.populate(podcast: podcast)
            }.disposed(by: disposeBag)
    }
}
```
For convenient mapping the `Loadable<[Podcast]>` in the code above we can use an extension for the `Loadable`:

```swift
extension Loadable {
    var isLoading: Bool {
        switch self {
        case .isLoading: return true
        default: return false
        }
    }
    var value: Value? {
        switch self {
        case let .loaded(value): return value
        default: return nil
        }
    }
    var error: Error? {
        switch self {
        case let .failed(error): return error
        default: return nil
        }
    }
}
```
Great! Now our state - UI binding is much more clear and doesn't require an excessive code for fixing visual defects.

If we want to display the previously loaded list while performing the list refresh, we can extend the `isLoading` case with a parameter for holding the value.

```swift
enum Loadable<Value> {
    case notRequested
﹢  case isLoading(prevValue: Value?)
    case loaded(Value)
    case failed(Error)
}

extension Loadable {
    var value: Value? {
        switch self {
        case let .loaded(value): return value
﹢      case let .isLoading(prevValue): return prevValue
        default: return nil
        }
    }
    // ...
}
```

The final implementation of the `Loadable`, as well as a few more perks for it can be found in the [gist on Github](https://gist.github.com/nalexn/df65131de76301130c8e179d02cec939); however, this was just an example of how to improve the clarity and stability of the reactive code by eliminating the state redundancy in the ViewModel.
