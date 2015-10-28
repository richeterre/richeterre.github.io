---
layout: post
title: "Migrating to Swift 2 and ReactiveCocoa 4"
date: 2015-10-22 00:30:00 +0200
summary: "The Swift language is moving fast, and Xcode 7 requires Swift 2. Here's what you need to do to bring ReactiveCocoa 4 with Swift 2 support to your project."
---

*__Update, 28 Oct 2015:__ Adjusted the post for ReactiveCocoa 4's alpha 3 pre-release.*

---

_This is the sixth post in my series on [MVVM with ReactiveCocoa 3/4 in Swift][mvvm-reactivecocoa3-swift]._

With Xcode 7's public release out of the door, and myself being in the lucky position of working on a bleeding-edge project at the moment, I boldly went ahead and threw Xcode 6 off my development machine.

Because Xcode 7 requires Swift 2, this also meant that [SwiftGoal][swiftgoal] would need to receive some updating love for me to be able to work on it in the future. A few days ago I set aside some time to make the jump, and if you're planning to do the same, this post should give you an idea of what awaits you. Spoiler: It's not too bad :relaxed:

The whole process took me about an hour. Let's dive in!

## Updating the frameworks

SwiftGoal currently has four dependencies:

* [Argo][argo] for JSON parsing
* [SnapKit][snapkit] for making Auto Layout constraints in code
* [DZNEmptyDataSet][dznemptydataset] for handling the "no data yet" case
* [ReactiveCocoa][reactivecocoa] for bringing the app to life

plus the [Quick][quick]/[Nimble][nimble] testing combo, listed in a separate [private Cartfile][private-cartfile].

All of the above support Swift 2 in their latest releases, with the exception of ReactiveCocoa, whose official release still targets Xcode 6 and Swift 1.2. So we need to explicitly specify its [alpha pre-release][rac-4-alpha-3] with Swift 2 support in the `Cartfile`:

    github "dzenbot/DZNEmptyDataSet"
    github "ReactiveCocoa/ReactiveCocoa" "v4.0.0-alpha.3"
    github "SnapKit/SnapKit"
    github "thoughtbot/Argo"

Running `carthage update --platform iOS` will then bring our locally compiled frameworks up to date. It also lets us get rid of some dependencies:

* Thanks to improved associated enum values in Swift 2, ReactiveCocoa [no longer depends][box-removal] on the Box framework.
* Argo [no longer requires][runes-removal] the Runes framework.

As Carthage requires us to add frameworks to the project manually, we also need to do the inverse when removing a framework: Unlink it from the build target and remove it from the input of the `copy-frameworks` script under Build Phases.

## Swift syntax changes

After updating the frameworks, half of my code was painted red with build errors. I started off by fixing those caused by changes to the Swift standard library. For instance, collection operators like `enumerate` and `count` are now implemented as __protocol extensions__, leading to the new syntax

{% highlight swift %}

[1, 2, 3].enumerate()
[1, 2, 3].count
["one, two"].joinWithSeparator(", ")

{% endhighlight %}

replacing the old way of passing the array as a parameter to the function.

Another big addition to Swift 2 is __error handling__, which frees us from the outdated concept of having to pass an error pointer. For instance, `NSJSONSerialization`'s method `JSONObjectWithData:options:error` has lost its trailing `error` parameter. To deserialize received backend data into JSON, I use the new `try?` syntax that returns `nil` upon failure:

{% highlight swift %}

if let json = try? NSJSONObjectWithData(data, options: []) {
    // Parse JSON into model
}

{% endhighlight %}

Swift's somewhat confusing hash (`#`) syntax to explicitly name the first function parameter has been removed; this is now achieved by repeating the parameter name instead, or simply making it part of the function name as I did:

{% highlight swift %}

func createPlayerWithName(name: String) -> SignalProducer<Bool, NSError> {
    // Make request that creates the player server-side
}

{% endhighlight %}

And finally, the pesky but `required init?(coder aDecoder: NSCoder)` is now a failable initializer, which is reflected by the newly inserted question mark.

## New ReactiveCocoa syntax

Swift 2 introduced a number of concepts such as protocol extensions and error handling. From ReactiveCocoa's perspective, this turned out to be both a [blessing][burn-pipe-gt] (protocol extensions) and a [curse][angry-justin] (error handling), and has led to a number of breaking API changes. Here is a list of things that I needed to fix and that you are likely to encounter in your own ReactiveCocoa projects, too:

The most notable change is the replacement of ReactiveCocoa 3's ubiquitous `|>` operator by the more familiar __dot syntax__. Find & Replace was my friend here.

__Sinks__ have been [replaced][observers-replacing-sinks] by __observers__ that also bring along some dot syntax goodness. Instead of defining a sink of type `SinkOf<Event<Bool, NoError>>` and passing it to the free function `sendNext()` alongside its next value, we now write

{% highlight swift %}

let observer = Observer<Bool, NoError>()
observer.sendNext(true)

{% endhighlight %}

The __`flatMap` operator__ now requires the argument label `transform:` for the mapping function parameter. Xcode actually shows a helpful error message about this.

On the __subscribing__ side, `start(next: {})` for signal producers and `observe(next: {})` for signals have become `startWithNext({})` and `observeNext:({})`, respectively.

To __set the value__ of a `MutableProperty`, instead of `isActive.put(true)` we now do a regular assignment: `isActive.value = true`

The __`collect` operator__, which buffers sent signal values into an array and sends the latter, is now called as a function: `collect()`.

When pattern-matching ReactiveCocoa events, we can now __directly access their associated value__ – no need to unbox it anymore. So instead of

{% highlight swift %}

switch event {
    case let .Next(boxedValue):
        let value = boxedValue.value
        // Handle unboxed value
    case let .Error(boxedError):
        let error = boxedError.value
        // Handle unboxed error
}

{% endhighlight %}

we can just write

{% highlight swift %}

switch event {
    case let .Next(value):
        // Handle value
    case let .Failed(error)
        // Handle error
}

{% endhighlight %}

Note how the __`.Error` case has been renamed to `.Failed`__ to better reflect the fact that signals can only ever send one such event before terminating.

And finally, because of the naming conflict with Swift's new error handling syntax, __`catch` has been renamed to `flatMapError`__. It continues to work exactly as before.

## Testability

In Xcode 7, it is finally no longer necessary to make all your methods `public` to be able to test them. Instead, in your test suite you can import your module alongside your testing frameworks like so:

{% highlight swift %}

import Quick
import Nimble
@testable import SwiftGoal

{% endhighlight %}

That way, all your APIs marked as `internal` (which is the default) will become available in your test suite, yet won't pollute the global namespace.

## Oh, and one more thing…

After fixing all the above issues, the project finally compiled again. But when firing up the app, I was greeted by Apple's newest shenanigan:

> The resource could not be loaded because the App Transport Security policy requires the use of a secure connection.

This is all well and good when dealing with a remote backend, but in this case I was running the app against a local [Goalbase][goalbase] installation. For our humble `localhost`, an unencrypted connection should do just fine, so I [added an exception domain for it][transport-security-commit] to the target's `Info.plist` file like this:

![Transport Security exception for localhost in SwiftGoal's Info.plist]({{ site.baseurl }}/resources/swiftgoal-transport-security.png)

And that's it for today. What remains now is to wait for ReactiveCocoa 4's public release, but as far as I can tell, the alpha version works just fine – the only caveat is that the API contract may still change in future pre-releases.

Let's build great stuff! :rocket:

[mvvm-reactivecocoa3-swift]: {% post_url 2015-08-12-mvvm-with-reactivecocoa-3-in-swift %}
[swiftgoal]: https://github.com/richeterre/SwiftGoal
[argo]: https://github.com/thoughtbot/Argo
[snapkit]: https://github.com/SnapKit/SnapKit
[dznemptydataset]: https://github.com/dzenbot/DZNEmptyDataSet
[reactivecocoa]: https://github.com/ReactiveCocoa/ReactiveCocoa
[quick]: https://github.com/Quick/Quick
[nimble]: https://github.com/Quick/Nimble
[private-cartfile]: https://github.com/Carthage/Carthage/blob/master/Documentation/Artifacts.md#cartfileprivate
[rac-4-alpha-3]: https://github.com/ReactiveCocoa/ReactiveCocoa/releases/tag/v4.0.0-alpha.3
[box-removal]: https://github.com/ReactiveCocoa/ReactiveCocoa/commit/be78f5feabcf1d211c176f34a3cdad173feb904c
[runes-removal]: https://github.com/thoughtbot/Argo/commit/073ea9f81b8ad167b784bd7fa2da5d84a5b2a15b
[burn-pipe-gt]: https://github.com/ReactiveCocoa/ReactiveCocoa/commit/11952aae5013d82b3365e4c06b9e36a462ed8d7d
[angry-justin]: https://twitter.com/jspahrsummers/status/608066730250924032
[observers-replacing-sinks]: https://github.com/ReactiveCocoa/ReactiveCocoa/pull/2442
[goalbase]: https://github.com/richeterre/goalbase
[transport-security-commit]: https://github.com/richeterre/SwiftGoal/commit/fb28015f8b82dbdc24fb1951361d5c5172629e27
