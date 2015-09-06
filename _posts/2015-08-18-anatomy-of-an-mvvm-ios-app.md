---
layout: post
title: "Anatomy of an MVVM iOS app"
date: 2015-08-18 00:55:00 +0200
summary: "A high-level overview of the different parts that make up an MVVM app and how they relate to each other, spiced with a few code examples."
---

_This is the second post in my series on [MVVM with Reactive Cocoa 3 in Swift][mvvm-reactivecocoa3-swift]._

Before tackling some of the questions presented in the [previous post]({% post_url 2015-08-12-mvvm-with-reactivecocoa-3-in-swift %}), let's take a closer look at the different parts of the [SwiftGoal][swiftgoal] app!

Its basic idea is quickly explained: After adding some pals and yourself as players, you can add your played matches along with the results, edit and delete them if needed, and see player rankings on a separate tab.

## App delegate

A good place to start exploring an app's codebase is usually the app delegate, which is a singleton. This actually makes sense, as it represents something that exists only once: your application running on a given user's device.

The view hierarchy is set up in `application:didFinishLaunchingWithOptions:` in a manner similar to this:

{% highlight swift %}

let store = Store(baseURL: baseURL)

let matchesViewModel = MatchesViewModel(store: store)
let matchesViewController = MatchesViewController(viewModel: matchesViewModel)

let rankingsViewModel = RankingsViewModel(store: store)
let rankingsViewController = RankingsViewController(viewModel: rankingsViewModel)

let tabBarController = UITabBarController()
tabBarController.viewControllers = [matchesViewController, rankingsViewController]
self.window?.rootViewController = tabBarController

{% endhighlight %}

There's a veritable chain of dependency injection going on here! First, a __store__ is created with the base URL. It is then passed to two different __view models__, which in turn are injected into their respective __views__. I found that this approach has some benefits over singletons à la `Store.sharedStore()`, such as letting us change the base URL without messing up the app's internal state, and far easier unit testing. This is covered in more detail in a [later post]({% post_url 2015-09-06-how-getting-rid-of-singletons-boosts-testability %})!

## Models

The model layer consists of simple containers, or "bags of data", often representing real-world entities. In Swift they can nicely be implemented as structs, which gives them value-type semantics. This is a good thing, because it will keep them thread-safe and lets us pass them onwards without introducing needless coupling. Instead of modifying _mutable, long-lived_ model instances, we can represent change over time by sending _immutable, short-lived_ model values on a signal.

SwiftGoal has the following models:

* __Player__, to represent human players and store their name
* __Match__, to store the home and away players of matches and their results
* __Ranking__, to each represent a player and their current rating
* __Changeset__, to indicate the _location_ of changes to a table view, using arrays of `NSIndexPath` for insertions, deletions and modifications

## Stores

The store layer is responsible for vending model instances and thus knows how to best retrieve them. In SwiftGoal, this basically means querying various endpoints on a remote server (indicated by the `baseURL` parameter when creating a `Store`), but the other layers don't need to know about that. If we wanted to add some form of caching for models, this would be the place to put it.

As stores tend to deal with asynchronous activities such as network requests, their API cannot instantly return anything. Instead, the data's future presence is represented by ReactiveCocoa 3's new `SignalProducer` type:

{% highlight swift %}

// Store.swift

func fetchMatches() -> SignalProducer<[Match], NSError> {
    // …
}

{% endhighlight %}

No actual work is done until the returned `SignalProducer` is started, which is a job for the …

## View models

The view models are where the magic happens: They receive some form of input from the view that owns them, do their stuff which often involves a `Store`, and produce output by updating their output properties and signals, which in turn are then observed by the view.

As an example, here are the `MatchesViewModel`'s inputs and outputs, beautifully self-documented by Swift's type system in ReactiveCocoa 3:

{% highlight swift %}

// Inputs
let active = MutableProperty<Bool>
let refreshSink: SinkOf<Event<Void, NoError>>

// Outputs
let title: String
let contentChangesSignal: Signal<Changeset, NoError>
let isLoading: MutableProperty<Bool>
let alertMessageSignal: Signal<String, NoError>

{% endhighlight %}

Just looking at these few lines, which are conveniently at the top of the file, give you an excellent idea of what this view model does. The view needs to tell it

* whether it is active (which usually means "on screen")
* when it wants to refresh data (such as when the user pulls down the table view)

The view model provides

* a static title for the view, typically shown in the navigation bar
* a signal of `Changeset`s that indicate what parts of the table view need reloading
* whether it is currently loading data (e.g. for showing an activity indicator)
* a signal of alert messages to present to the user (e.g. network failure)

Note how there is no actual model data transmitted here, only the _locations_ that change! The view can then request formatted data for each index path by calling methods like this one:

{% highlight swift %}

// MatchesViewModel.swift

public func homePlayersAtIndexPath(indexPath: NSIndexPath) -> String {
    let match = matchAtIndexPath(indexPath)
    return separatedNamesForPlayers(match.homePlayers)
}

{% endhighlight %}

## Views

At the top of our food chain are the views, which are as powerful as they are stupid: On one hand, they are the owners of their view models, holding a strong reference to them; on the other, they merely visualize the output of the view model.

This is done by declaring the relationships between the view model's outputs and the view's behavior in the `bindViewModel` method:

{% highlight swift %}

// MatchesViewController.swift

private func bindViewModel() {
    self.title = viewModel.title

    viewModel.active <~ isActiveSignal

    viewModel.contentChangesSignal
        |> observeOn(UIScheduler())
        |> observe(next: { [weak self] changeset in
            self?.tableView.beginUpdates()
            self?.tableView.deleteRowsAtIndexPaths(changeset.deletions, withRowAnimation: .Left)
            self?.tableView.insertRowsAtIndexPaths(changeset.insertions, withRowAnimation: .Automatic)
            self?.tableView.endUpdates()
        })

    viewModel.isLoading.producer
        |> startOn(UIScheduler())
        |> start(next: { [weak self] isLoading in
            if !isLoading {
                self?.refreshControl?.endRefreshing()
            }
        })

    // …
}

{% endhighlight %}

(It's important to observe all UI-related signals on the `UIScheduler` representing the main thread. Failing to do so can lead to strange behavior, such as table views suddenly refreshing upon scrolling.)

In MVVM, most view code looks like the above – it simply describes what should happen in the UI when a view model's output changes. The domain logic needed to generate these outputs resides in the view model.

[mvvm-reactivecocoa3-swift]: {% post_url 2015-08-12-mvvm-with-reactivecocoa-3-in-swift %}
[swiftgoal]: https://github.com/richeterre/SwiftGoal
