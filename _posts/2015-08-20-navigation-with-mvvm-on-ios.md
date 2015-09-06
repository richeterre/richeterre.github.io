---
layout: post
title: "Navigation with MVVM on iOS"
date: 2015-08-20 11:10:00 +0200
summary: "How to pass data when transitioning between views, without breaking the MVVM pattern."
---

_This is the third post in my series on [MVVM with Reactive Cocoa 3 in Swift][mvvm-reactivecocoa3-swift]._

One of the things that puzzled me the most when beginning to explore MVVM on iOS was how to do navigation between view controllers. The [SwiftGoal][swiftgoal] app, for instance, may have a list of matches that the user has already created. But everyone makes mistakes, and so we want to allow changes by modally presenting an "Edit Match" screen when the user taps on a match while in table edit mode.

The tap is first registered in the view layer, triggering our `MatchesViewController`'s `tableView:didSelectRowAtIndexPath:` method. So now that we have the index path, maybe we could use it to grab the correct `Match` object, and somehow assign it to a property on the view model of a newly created `EditMatchViewController`?

After considering this for a while, I realized that although technically possible, it would violate the MVVM architecture in several ways:

* The core task of the view model is to expose only the required parts of a model to the view, in a format suitable for presentation. Having `MatchesViewModel` provide a full `Match` object to the `MatchesViewController` is pretty overkill and makes the view model's [data source methods][matchesviewmodel-data-source] altogether pointless.
* To pass that `Match` object, the `EditMatchViewController` would have to expose its view model, and even worse, the internal state of that view model, to our `MatchesViewController`! Letting random objects swap out the match currently being edited wouldn't exactly increase our confidence in the code.

Ideally, we want to keep the view controller's view model in a private, immutable and non-optional property. That way, we can use it with confidence, knowing that it isn't `nil`, hasn't suddenly changed under our nose, and is in fact wholly invisible to the outside world:

{% highlight swift %}

// EditMatchViewController.swift

private let viewModel: EditMatchViewModel

{% endhighlight %}

Since we now _must_ have the view model ready at initialization, we can make it part of the designated initializer,

{% highlight swift %}

// EditMatchViewController.swift

init(viewModel: EditMatchViewModel) {
    self.viewModel = viewModel
    // …
}

{% endhighlight %}

and move the responsibility for _creating_ that `EditMatchViewModel` to our `MatchesViewModel`, while _assigning_ in the view controller:

{% highlight swift %}

// MatchesViewController.swift
// (code adjusted for readability)

let editMatchVM = self.viewModel.editViewModelForIndexPath(indexPath)
let editMatchVC = EditMatchViewController(viewModel: editMatchViewModel)
let editMatchNC = UINavigationController(rootViewController: editMatchVC)
self.presentViewController(editMatchNC, animated: true, completion: nil)

{% endhighlight %}

Note how we keep the view controller stupid by never even letting it know what exactly goes into the `EditMatchViewModel`! All it cares about is receiving an instance of the right type to inject into the newly created `EditMatchViewController`, which it then presents to the user. (Of course, this method also works with push navigation and other presentation modes, as the actual transition happens completely in the view controller.)

You may already have guessed what's happening behind the scenes. The view model knows exactly how to get a `Match` with the given index path, and what to do with it:

{% highlight swift %}

// MatchesViewModel.swift
// (code adjusted for readability)

func editViewModelForIndexPath(indexPath: NSIndexPath) -> EditMatchViewModel {
    let match = matchAtIndexPath(indexPath)
    return EditMatchViewModel(store: store, match: match)
}

{% endhighlight %}

It's as simple as that, and [as the codebase shows][matchesviewmodelspec], really testable too.

[mvvm-reactivecocoa3-swift]: {% post_url 2015-08-12-mvvm-with-reactivecocoa-3-in-swift %}
[swiftgoal]: https://github.com/richeterre/SwiftGoal
[matchesviewmodel-data-source]: https://github.com/richeterre/SwiftGoal/blob/v1.0/SwiftGoal/ViewModels/MatchesViewModel.swift#L94-L117
[matchesviewmodelspec]: https://github.com/richeterre/SwiftGoal/blob/v1.0/SwiftGoalTests/ViewModels/MatchesViewModelSpec.swift#L144-L157
