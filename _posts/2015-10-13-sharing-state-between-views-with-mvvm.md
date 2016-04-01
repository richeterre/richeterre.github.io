---
layout: post
title: "Sharing state between views with MVVM"
date: 2015-10-13 00:58:00 +0200
summary: "No view is an island, and often they like to depend on each other in intricate ways. Luckily, view models offer some nice ways to share state without losing your sanity."
---

_This is the fifth post in my series on [MVVM with ReactiveCocoa 3/4 in Swift][mvvm-reactivecocoa3-swift]._

The web offers plenty of MVVM tutorials, which I found a great resource to get started on the topic. However, they tend to feature view classes living in isolation, each backed by their own view model, and stop short of addressing cases where two or more `UIViewController` instances need to share some real-time information between each other. Such scenarios may involve, for instance, item selection or validation of user input.

As before, let's look at the open-source [SwiftGoal project][swiftgoal] for some examples!

### Example 1: Bindings between view models

 First, let's consider the screen where the user can select the home or away players while creating or editing a match. The view model that handles player selection, `ManagePlayersViewModel`, needs to communicate this choice back to the `EditMatchViewModel` that created it. I described this mechanism of spawning view models within other view models in my [earlier post on navigation with MVVM][mvvm-navigation].

Intuitively, we would probably solve this with delegation: The `EditMatchViewModel` would implement a `ManagePlayersViewModelDelegate` protocol, assign itself as the delegate when creating the other view model, and be called whenever the player selection changes. But this approach would cause there to be two separate arrays of selected players (one in each view model), and some imperative code (the delegate method call) to keep the two in sync. Wouldn't it be better if we could just __declare the relationship__ once, and let machines handle the details? Being able to write software in such a _declarative_ style is one of the key advantages of functional reactive programming.

The most natural place to set up the relationship between the two view models is where one gets created by the other:

```swift
// EditMatchViewModel.swift

public func manageHomePlayersViewModel() -> ManagePlayersViewModel {
    let homePlayersViewModel = ManagePlayersViewModel(
        store: store,
        initialPlayers: homePlayers.value,
        disabledPlayers: awayPlayers.value
    )
    self.homePlayers <~ homePlayersViewModel.selectedPlayers

    return homePlayersViewModel
}
```

In the above code excerpt, we set up the binding between `self.homePlayers` and the `selectedPlayers` array right between creating the `ManagePlayersViewModel` and returning it to the calling view controller.

### Example 2: Shared view models

For another example, let's look at what happens in the app after the user proceeds to the "Manage Players" screen that is backed by the aforementioned `ManagePlayersViewModel`. Perhaps a player that needs to be selected for a match hasn't been added to the service yet. The user thus hits the "+" button to create the player.

Presenting the simple form (which has a single field for the new player's name) asks for a modal transition, as the app expects the user to either submit new data or cancel. Again, traditionally we would use delegation to communicate back from the modally presented form to the presenting view controller, which would then refresh its content. But in MVVM we'd have to back each view with its own view model, despite the two being conceptually very close to each other.

Instead, we can __share a single view model__ between the two views to represent their shared state. The below code example illustrates the creation of the modal player form within its presenting `ManagePlayersViewController`. In fact, the modal form is so simple that we can implement it with a simple `UIAlertController` holding a text field:

```swift
// ManagePlayersViewController.swift
// (code adjusted for readability)

let newPlayerVC = UIAlertController(
    title: "New Player",
    message: nil,
    preferredStyle: .Alert
)

// Add Cancel and Save actions
newPlayerVC.addAction(UIAlertAction(title: "Cancel", style: .Cancel, handler: nil))
let saveAction = UIAlertAction(
    title: "Save",
    style: .Default,
    handler: { [weak self] _ in
        self?.viewModel.saveAction.apply().start()
    }
)
newPlayerVC.addAction(saveAction)

// Allow saving only with valid input
viewModel.inputIsValid.producer.start(next: { isValid in
    saveAction.enabled = isValid
})

// Add field for user input
newPlayerVC.addTextFieldWithConfigurationHandler { textField in
    textField.placeholder = "Player name"
}

// Forward text input to view model
if let nameField = newPlayerVC.textFields?.first as? UITextField {
    viewModel.playerName <~ nameField.signalProducer()
}
```

Note how the bindings for the player name field and the save action, which becomes enabled only for valid input, are set up between the modal form and the now shared `ManagePlayersViewModel`. The view model's `saveAction` kicks off the new player's creation, and triggers a refresh signal upon success. That signal is observed by the `ManagePlayersViewController` to reload its table view accordingly.

### Making the right choice

The above examples demonstrated two different ways of sharing state between views – either by setting up a relationship between two view models, or by sharing a common view model. But how to choose between them?

My approach to this is to look at each view and ask myself the following: _Is the view independent enough to warrant a view model of its own?_

In the first case, we're dealing with two views that hold quite independent data sets: One constitutes a match, the other a list of available players. The relationship between matches and players is a "many-to-many" one and as such rather loose; setting up two separate view models and have a binding to represent their relationship feels like a natural choice.

In the second case, however, the modally presented form is a mere UI detail. It could easily be replaced by, say, a special table view cell that the user fills in to add the player. Both views also operate on the same data: an array of players. And if you want to add validation for player uniqueness, it does get a lot easier when your view model already contains the data to validate against. Here, a shared view model probably is a better fit.

[mvvm-reactivecocoa3-swift]: {% post_url 2015-08-12-mvvm-with-reactivecocoa-3-in-swift %}
[swiftgoal]: https://github.com/richeterre/SwiftGoal
[mvvm-navigation]: {% post_url 2015-08-20-navigation-with-mvvm-on-ios %}
