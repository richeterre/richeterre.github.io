---
layout: post
title: "How getting rid of singletons boosts testability"
date: 2015-09-06 17:58:00 +0200
summary: "The MVVM architecture can help bring your app to a much more testable state. But to really reap its benefits, you need to look beyond the good ol' singleton pattern."

---

_This is the fourth post in my series on [MVVM with ReactiveCocoa 3/4 in Swift][mvvm-reactivecocoa3-swift]._

## The problem with singletons

In the past, I used to write a lot of code like this to retrieve my model objects:

```swift
let matches = Store.sharedInstance().fetchMatches()
```

(Of course, `Store` could also be called differently. Common choices are `APIClient`, `NetworkManager` or, in this case, `MatchStore`.)

At some point I learned about reactive programming and realized that `fetchMatches()` should probably be made asynchronous. It should also starts its work whenever someone is interested in the result. ReactiveCocoa 3 offers [signal producers][rac-signalproducer] to solve this problem:

```swift
let matchesSignalProducer = Store.sharedInstance().fetchMatches()

matchesSignalProducer.start(next: { [weak self] matches in
    self?.matches = matches
})
```

In an MVVM app such as [SwiftGoal][swiftgoal], the above code would reside in a view model. One of the architecture's "selling points" is that your app logic becomes more testable, because it resides in view models that define inputs and outputs. This is, after all, the essence of testing: _Does the class yield the correct output for a given input?_

Suppose we want to write this simple test:

1. Make the view model active. (Input)
1. Check that it yields the correct amount of matches. (Output)

It turns out that our `Store` being a singleton becomes quite a blocker here. How are we supposed to provide a known amount of matches to the view model for testing? Maybe we could set the number somehow from our testing code, but won't that break tests we may have for other cases, e.g. when there aren't any matches?

## Enter dependency injection

As I described in an [earlier post][anatomy-of-an-mvvm-ios-app], we can clean up our app's spaghetti-like dependency graph by passing references to instances that a class relies on, preferably during initialization. Swift is particularly nice here, because we can easily declare a designated initializer to make this requirement explicit:

```swift
// MatchesViewModel.swift

init(store: Store) {
    self.store = store
    // …
}
```

Now it's perfectly clear for any consumer of this view model that it requires a `Store` instance to work with. In our test suite, we can simply subclass `Store` with a `MockStore`, override the `fetchMatches()` method, and return a known amount of matches that will allow us to write simple, but powerful tests like the one described above. We can even set a flag to check whether the method was called correctly.

```swift
class MockStore: Store {
    // …
    var didFetchMatches = false

    override fetchMatches() -> SignalProducer<[Match], NSError> {
        didFetchMatches = true
        return SignalProducer.value([match1, match2])
    }
}

class MatchesViewModelSpec: QuickSpec {
    override func spec() {
        it("fetches a list of matches after becoming active") {
            let mockStore = MockStore()
            let matchesViewModel = MatchesViewModel(store: mockStore)
            matchesViewModel.active = true

            expect(mockStore.didFetchMatches).toEventually(beTrue())
            expect(matchesViewModel.numberOfMatches()).toEventually(equal(2))
        }
    }
}
```

## Growing confidence

There is another huge benefit to this approach, which is that you can't get your app into an inconsistent state quite so easily. Our `Store` has a base URL, which it uses to construct its network requests, and requires this at – you guessed it – initialization. In [SwiftGoal][swiftgoal], there is actually a feature that lets the user modify the base URL in the app's settings at any time to connect to a different server.

Imagine the mayhem that would ensue if we had a singleton store and changed that base URL under everyone's nose! Your user might have a new "Create Match" dialog open, and hitting "Save" would create the match on a different server than was used to select the players in the first place. Quite a recipe for broken app state :grimacing:

Instead, the app delegate (which has [good enough reason to be a singleton][anatomy-of-an-mvvm-ios-app-app-delegate]) notices when the setting changes, and initializes a whole new hierarchy with a fresh store and view models that depend on it. This top-down approach allows the view models to be confident that their store's base URL doesn't change suddenly and unnoticedly.

[mvvm-reactivecocoa3-swift]: {% post_url 2015-08-12-mvvm-with-reactivecocoa-3-in-swift %}
[rac-signalproducer]: https://github.com/ReactiveCocoa/ReactiveCocoa/blob/v3.0-RC.1/Documentation/DesignGuidelines.md#the-signalproducer-contract
[swiftgoal]: https://github.com/richeterre/SwiftGoal
[anatomy-of-an-mvvm-ios-app]: {% post_url 2015-08-18-anatomy-of-an-mvvm-ios-app %}
[anatomy-of-an-mvvm-ios-app-app-delegate]: {% post_url 2015-08-18-anatomy-of-an-mvvm-ios-app %}#app-delegate
