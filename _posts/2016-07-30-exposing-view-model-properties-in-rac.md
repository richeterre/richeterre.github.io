---
layout: post
title: "Exposing view model properties in ReactiveCocoa"
date: 2016-07-30 00:20:00 +0200
summary: "Providing view model data for the view layer to bind to is a vital part of MVVM. Here's how you can do this using ReactiveCocoa's native Swift API."
---

_This is the eighth post in my series on [MVVM with ReactiveCocoa 3/4 in Swift][mvvm-reactivecocoa3-swift]._

In an MVVM architecture, it makes a lot of sense to expose view model properties in such a way that code interacting with your view model, typically the view layer, can observe their changes. In the early days of ReactiveCocoa (RAC), this often meant using regular `NSString*` or `BOOL` properties and turning them into signals in the view's `bindViewModel` method, typically through a `RACObserve` macro.

Since the release of ReactiveCocoa 3.0 with its native Swift API, this approach no longer works, at least without [sacrificing type safety][rac-swift-objc-bridging]. Instead, RAC comes with several types that we can use to publish our app's internal state to the view layer, most notably `Signal` and `MutableProperty`.

The __signal__ represents a stream that consumers can subscribe to _without creating side effects_. Imagine somebody tuning into an FM radio broadcast. The radio station, unlike an internet streaming provider, won't know that this is happening. It just merrily sends out its content, and this is essentially what `Signal` does too. (A `SignalProducer`, by contrast, also performs side effects when creating its signals.)

__Mutable properties__ are basically wrappers around the regular properties we all know. They hold the value itself, but also give us a signal or signal producer to subscribe to and receive updates on its value changing over time. The difference is that the signal will only send value changes _that happen after subscribing_, whereas the signal producer creates us a signal that _sends the current value immediately_, followed by all changes later on.

But hold on – didn't I just mention that signal producers always perform a side effect when subscribed to? In fact, this puzzled me for a long time after first encountering RAC's mutable properties, until I learned that the signal producer's [only side effect is the subscription itself][neilpa-subscription-comment]. It's a bit of a trick RAC's creators used to give us a value immediately after subscription. :relaxed:

## An example

As usual, we'll take a look at the [SwiftGoal][swiftgoal] example app to see how we can use the above types in a real project. This time we'll focus on the ["Outputs" section][swiftgoal-manage-players-vm-v1.3-outputs] of `ManagePlayersViewModel`, the class that backs a screen where the user can add players and select them for a match.

```swift
// ManagePlayersViewModel.swift

// Outputs
let title: String
let contentChangesSignal: Signal<PlayerChangeset, NoError>
let isLoading: MutableProperty<Bool>
let alertMessageSignal: Signal<String, NoError>
let selectedPlayers: MutableProperty<Set<Player>>
let inputIsValid = MutableProperty(false)
```

As you can see, we're essentially dealing with three kinds of properties here:

* The view model's title never changes and therefore modeled as a `String`. "Binding" to this value means simply assigning it to the view's `title` property.
* Content changes and alert messages are exposed to the view as __signals__ carrying two different types: Each content change is represented by a `PlayerChangeset`, whereas error messages are modeled as plain vanilla strings.
* The remaining three are __mutable properties__ wrapping a boolean flag that tells whether content is loading, a set of `Player` models selected by the user, and another, initially `false` flag for input validation when creating a new player.

If you're curious how other classes (mostly `ManagePlayersViewController`) bind to these exposed properties, check out the [code][swiftgoal-manage-players-vc-v1.3-bindings-1] that [does][swiftgoal-manage-players-vc-v1.3-bindings-2] exactly [that][swiftgoal-edit-match-vm-v1.3-bindings].

## Signal or mutable property?

Looking at the above outputs, you may notice that there is a qualitative difference between the three approaches. Consider the way their value _behaves_ over time. The title is easy – it never changes. But whether our data is loading, what players are selected, or whether the player form input is valid – those are all examples of __continuous data__, which can be queried anytime. We could ask our view model at any random moment: "What players are currently selected?" and it would always have an answer for us.

The content changes and error messages, on the other hand, are __discrete data__. They represent singular events that happen, get noticed and handled by the view, but cannot be returned upon request. Asking the view model "What is your current content change?" doesn't make any sense. (And because SwiftGoal presents errors as modal alerts to the user, modeling error messages as continuous values didn't make much sense to me either. In an app where errors stay on screen as long as the problem persists, I would have picked a mutable property instead.)

## Conclusion

_As a guideline, ask yourself whether you're dealing with continuous or discrete data before deciding on a type for your view model properties._

For continuous data, mutable properties work better. This is especially true if you're moving in the borderlands between reactive and imperative programming, such as when implementing `UITableViewDataSource`. In those cases, being able to ask the mutable property for its value _whenever you need it_ is a good thing. I'm actually [using this technique][swiftgoal-manage-players-vm-v1.3-datasource] in the view model presented earlier.

For discrete events, consider using a signal. It doesn't hold its value, so you can't be tempted to mess around with it in non-reactive ways. Just react to whatever it sends you, and be grateful for every piece of state you don't have to manage yourself. 😇

[mvvm-reactivecocoa3-swift]: {% post_url 2015-08-12-mvvm-with-reactivecocoa-3-in-swift %}

[rac-swift-objc-bridging]: https://github.com/ReactiveCocoa/ReactiveCocoa/blob/master/Documentation/ObjectiveCBridging.md
[neilpa-subscription-comment]: https://github.com/ReactiveCocoa/ReactiveCocoa/issues/2061#issuecomment-108022754
[swiftgoal]: https://github.com/richeterre/SwiftGoal
[swiftgoal-manage-players-vm-v1.3-outputs]: https://github.com/richeterre/SwiftGoal/blob/v1.3/SwiftGoal/ViewModels/ManagePlayersViewModel.swift#L21-L27
[swiftgoal-manage-players-vc-v1.3-bindings-1]: https://github.com/richeterre/SwiftGoal/blob/v1.3/SwiftGoal/Views/ManagePlayersViewController.swift#L55-L91
[swiftgoal-manage-players-vc-v1.3-bindings-2]: https://github.com/richeterre/SwiftGoal/blob/v1.3/SwiftGoal/Views/ManagePlayersViewController.swift#L160-L163
[swiftgoal-edit-match-vm-v1.3-bindings]: https://github.com/richeterre/SwiftGoal/blob/v1.3/SwiftGoal/ViewModels/EditMatchViewModel.swift#L89
[swiftgoal-manage-players-vm-v1.3-datasource]: https://github.com/richeterre/SwiftGoal/blob/v1.3/SwiftGoal/ViewModels/ManagePlayersViewModel.swift#L121-L124
