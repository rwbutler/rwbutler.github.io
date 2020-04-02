---
layout: post
title:  "How to Debounce UIButton Taps"
subtitle: "Does your QA like to button bash? Here’s a solution."
---

<div align="center">
    <img src="https://github.com/rwbutler/rwbutler.github.io/raw/master/img/how-to-debounce-uibutton-taps.png" alt="MacBook Keyboard">
</div>

_We’re all familiar with the situation — we’ve implemented the greatest new feature our app has ever seen. It gets to QA and the first thing they do is rapidly bash the button that pushes the next UIViewController five times in a row._

Mechanical switches typically operate using two metallic contacts which when the switch is depressed come into contact within one another thus completing a circuit. These contacts are usually made of a springy metal which may bounce apart from one another once or several times before coming into final contact with one another. This effect can result in several transitions from low to high power state occurring which in a logic circuit may potentially register as multiple button taps. In order to guard against this, the signal state can be sampled at a low rate until a state change can be reliably said to have occurred. This process is known as _debouncing_.

The term has been appropriated particularly in the JavaScript and reactive programming communities to refer to the process of observing a stream of successive events until a steady state is reached at which point an event representing that steady state (i.e. the final state) is dispatched. If this idea were to be applied to a stream of button taps then the button tap event would only be dispatched at some point after the last button tap had occurred. This would typically be implemented by introducing a delay in emitting the button tap event such that the event is only emitted if no subsequent button taps were made during the delay period.

It is frequently referenced in relation to _throttling_ because the two operations are somewhat similar in nature. Both involve sampling a stream of events — with throttling we are interested in emitting a sampled version (the sample rate or delay is usually passed as a parameter to the throttling function) of the original event stream. This might be implemented by emitting the first or the last button tap events every delay period for example every 0.5 seconds. With debouncing we are interested in sampling the original event stream and only emitting an event once a steady state has been reached i.e. once the value no appears to be changing.
An example of where this could be useful is the implementation of search suggestions. We might generate excessive traffic on an API if we were to send a request to a search suggestions API each time the user types a new letter into a text field however if we were to throttle the stream events we might only dispatch the request to the API every couple of seconds rather than on each letter press. If we wanted to implement pre-loading of search results such that results appeared without having to hit a done button or equivalent we might debounce the stream of input events such that we dispatch a request to the search API at some delay time after we stop receiving new input.

In terms of debouncing a stream of button taps, it wouldn’t make sense to wait until after the last button tap to dispatch the button tapped event as this would make the UI appear unresponsive. In this case, accepting the first button tap and then ignoring subsequent button taps allows the UI to remain responsive without potentially causing incorrect program behaviour through the execution of the original button tap handler code multiple times.

We can implement such behaviour on a UIButton in Swift fairly simply though an extension on UIControl as follows:

```swift
public extension UIControl {
    @objc static var debounceDelay: Double = 0.5
    @objc func debounce(delay: Double = UIControl.debounceDelay, siblings: [UIControl] = []) {
        let buttons = [self] + siblings
        buttons.forEach { $0.isEnabled = false }
        let deadline = DispatchTime.now() + delay
        DispatchQueue.main.asyncAfter(deadline: deadline) {
            buttons.forEach { $0.isEnabled = true }
        }
     }
}
```

In the above code sample, we simply set the button’s `isEnabled` property to `false` for a period of 0.5 seconds after the initial tap and then set it back to `true` after the delay period elapses. We put the extension on UIControl rather than UIButton as the [`isEnabled`](https://developer.apple.com/documentation/uikit/uicontrol/1618217-isenabled) property is actually defined on UIControl rather than UIButton and allows the code to be used more generally. We also set a global delay of 0.5s such that if this function is invoked within a button’s handler function then all subsequent button taps will be ignored for this period by default — however, if we wish to override this delay on a per button basis then we may pass the overridden delay value to the `debounce` function as a parameter.

One last thing to note is we also allow an array of sibling UIControls to be passed as a parameter to the function in case we wish to temporarily debounce other controls at the same time as the original button. For example, say that we have two buttons A and B whereby tapping button A results in UIViewController A being pushed whilst tapping button B results in UIViewController B being pushed. If we rapidly tapped button A followed by button B (which might happen if a delay in pushing UIViewController A occurred due to a delay introduced by poor signal strength and a logic error) whilst only debouncing button A then at best we would end up in a situation whereby UIViewController A would be pushed followed by UIViewController B. At worst such a scenario could result in an application crash. Therefore we allow sibling controls to be simultaneously debounced in order to avoid scenarios whereby button handlers for distinct buttons being invoked simultaneously could result in unintended interactions.

Functional Reactive Programming (FRP) frameworks such as [RxSwift](https://github.com/ReactiveX/RxSwift) frequently provide implementations of debouncing and throttling functions including Apple’s [Combine](https://developer.apple.com/documentation/combine) framework introduced with iOS 13. The [Combine implementation of debounce](https://developer.apple.com/documentation/combine/anypublisher/3204205-debounce), emits an event after the input stream has reached a steady state which would be exactly what we would need in the example above involving making an API request for search results following data entry into a text field.

```swift
func debounce<S>(for dueTime: S.SchedulerTimeType.Stride, scheduler: S, options: S.SchedulerOptions? = nil) -> Publishers.Debounce<AnyPublisher<Output, Failure>, S> where S : Scheduler
```

In terms of debouncing button taps however the use of this function wouldn’t be ideally suited since as mentioned above, there would be a delay between tapping the button and the UI reacting to the tap.

The Swift extension above is much more in-line with [Combine’s throttle implementation](https://developer.apple.com/documentation/combine/passthroughsubject/3204657-throttle) which can emit either the first or last event within the specified time interval. It accepts a parameter `latest` which if set to `false` will cause the first event within a time interval to be emitted which seems more in-line with our desired behaviour of allowing the button and the UI to remain responsive whilst still ignoring subsequent button bashing.

```swift
func throttle<S>(for interval: S.SchedulerTimeType.Stride, scheduler: S, latest: Bool) -> Publishers.Throttle<PassthroughSubject<Output, Failure>, S> where S : Scheduler
```

## Summary

By applying a concept prevalent in web development (particularly JavaScript) and reactive programming using a simple Swift extension or FRP library function we can guard against unintended consequences in our application and hopefully see fewer tickets coming back from QA as a result. 😎

<hr/>

_The extension on UIControl can be found open-sourced on [GitHub](https://github.com/rwbutler/TailorSwift/blob/master/TailorSwift/Classes/UIControlAdditions.swift) under MIT license along with a [sample app](https://github.com/rwbutler/TailorSwift) in order to allow you to try it out. Note how each button tap in the sample application does not result in ‘Button tapped!’ being printed. Instead due to button taps being debounced, the text is printed no more than once every 0.5s._
