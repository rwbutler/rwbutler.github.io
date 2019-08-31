---
layout: post
title:  "Console Video Game-Style Cheat Codes for iOS Apps"
subtitle: "How to implement D-pad style cheat codes in an iOS app."
---

<div align="center">
    <img src="https://github.com/rwbutler/rwbutler.github.io/raw/master/img/console-video-game-style-cheat-codes-for-ios-apps.png" alt="SNES Video Game Console">
</div>

_Recently I was thinking about how I would go about implementing an Easter egg within an iOS app and how it could be found by the user. If you’re not familiar with the concept of an Easter egg — an [Easter egg](https://en.wikipedia.org/wiki/Easter_egg_(media)) is a fun feature hidden within an application that has to be discovered by the user._

In the early days of console gaming it was common for video games to contain [cheat codes](https://en.wikipedia.org/wiki/Cheating_in_video_games#Cheat_codes) to unlock Easter eggs or other features of the game such as a level select screen or [_god mode_](https://en.wikipedia.org/wiki/Glossary_of_video_game_terms#God_mode). Usually these were created by the developers as a means to jump to a particular level etc without having to play the game all the way through. As nearly all consoles at this time were played using D-Pad controllers (consisting of a directional-pad with up, down, left and right arrows along with (usually) four colored / lettered buttons e.g. A, B, C, D), a cheat code had to be input using a combination of these inputs.

A popular example of 8-bit era video game cheat codes is the Konami code which gets its name from the video game developer Konami whose games frequently contained the code _up, up, down, down, left, right, left, right, B, A_ for unlocking a cheat particular to that game. Over time, this code and many other variations have been adopted in video games by other developers and even by DVDs and websites to unlock Easter eggs.

Games and media that don’t rely on a D-pad controller for input have found creative means to allow users to input cheat codes. For example, some games in the Dance Dance Revolution series allow cheats to be inputted via the dance pad whilst certain DVDs accept a cheat code through the DVD player remote.

To implement a cheat code such as the Konami code in an iOS app, we can equate directional swipes to D-pad arrow presses whilst A, B, C button presses can make use of the iOS keyboard for data input.

# Cheats

Cheats is an open-source framework designed to allow iOS developers to define retro video game-style cheat codes of their own which an app can recognize in order to unlock a hidden feature / Easter egg. It is written using Swift 4 and available through Cocoapods, Carthage and Swift Package Manager.

<br/>

<div align="center">
    <img src="https://github.com/rwbutler/rwbutler.github.io/raw/master/img/cheats-sample-app.gif" alt="Cheats Sample App">
</div>

<br/>

<div align="center">
_Cheats Sample App_
</div>

# Usage

As an example, a simple cheat code sequence can be defined as follows:

```swift
let actionSequence: [CheatCode.Action] = [.swipe(.up), .swipe(.down), .swipe(.left), .swipe(.right), .keyPress("a"), .keyPress("b")]
```

This sequence allows a user to _swipe up, swipe down, swipe left, swipe right_ (on a view) and then brings up the keyboard allowing the _A and B key presses_ to be input. With a sequence of actions defined, we can define a `CheatCode` which along with an array of `CheatCode.Action` objects requires a completion closure which will be invoked each time the `CheatCode` changes state.

```swift
let cheat = CheatCode(actions: actionSequence) { cheatCode in
    switch cheatCode.status() {
        case .matched: // correct
            print("Cheat unlocked!")
        case .matching: // correct *so far*
            print("Further actions required to unlock cheat.")
        case .notMatched: // incorrect
            print("Cheat incorrect.")  
        case .reset: // initial state or sequence reset
            print("Cheat sequence reset")
    }
}
```

The closure is invoked returning the `CheatCode` instance as a parameter allowing the `CheatCode.State` to be queried using the `state()` method. The `CheatCode` object is essentially a state machine transitioning between states as inputs matching the defined sequence are entered. The states it can take are:

- `.matching` as the input sequence partially matches the full sequence of actions required to unlock the cheat.
- `.matched` once the full sequence of actions required to unlock the cheat has been entered.
- `.notMatched` when the user makes a mistake inputting the cheat code. After entering this state, the `reset()` method must be invoked to return the state machine to the `.reset` state.
- `.reset` — this is the initial state prior to any actions being performed by the user. It is also the state the state machine returns to after the `reset()` method is called and user’s inputted action sequence is cleared e.g. after getting the cheat code wrong. It allows the user to try again at entering the correct cheat code following a mistake.

Actions are the building blocks of cheat code sequences. Available actions are:

- `keyPress` - For whenever a key on the keyboard is pressed.
- `shake` - When the user shakes the device.
- `swipe` - In the directions up, down, left and right.
- `tap` - Specified with the number of taps required.

Using these actions we can easily implement a cheat code sequence such as the Konami code or one of a number of variations. A sample app is distributed as part of the framework allowing cheat code sequences to be tried out.

<br/>

<div align="center">
    <img src="https://github.com/rwbutler/rwbutler.github.io/raw/master/img/cheats-sample-app-2.png" alt="Example app illustrating the Cheats framework">
</div>

<br/>

<div align="center">
_Example app illustrating the Cheats framework_
</div>

# UIKit

In order to provide easy integration of the state machine with UIKit, Cheats also provides a gesture recognizer for recognizing cheat code sequences. It can easily be instantiated with a `CheatCode` instance and then added to a `UIViewController`'s view. Whilst the entire framework is available through Cocoapods and Carthage note that only the `CheatCode` state machine component is available through Swift Package Manager as it currently doesn’t support the iOS platform.

```swift
let gestureRecognizer = CheatCodeGestureRecognizer(cheatCode: cheatCode, target: self, action: #selector(actionPerformed(_:)))
view.addGestureRecognizer(gestureRecognizer)
```

# Summary

The Cheats framework provides a means of defining actions which when fed to the `CheatCode` state machine can be used to recognize a retro video game-style cheat code entered by a user. This can be used to unlock a hidden feature or Easter egg in an app when the `CheatCode` enters the `.matched` state.

To make using the framework with UIKit straightforward, a gesture recognizer has been provided for recognizing gestures as the user inputs them as part of the cheat code sequence.

In summary, with the `Cheats` framework recognizing a cheat code such as the Konami code can be achieved with only a few lines of code. To experiment with other cheat codes, try modifying the `CheatCode.Action` sequence in the sample app.

<hr/>

_Cheats can be found open-sourced on [GitHub](https://github.com/rwbutler/Cheats) under MIT license and is compatible with both [Cocoapods](https://cocoapods.org/pods/Cheats) and Carthage._

<br/>

<div align="center">
    [<img src="https://github.com/rwbutler/Cheats/raw/master/docs/images/cheats-logo.png" alt="Cheats Logo"/>](https://github.com/rwbutler/Cheats)
</div>
