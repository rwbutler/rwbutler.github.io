---
layout: post
title:  "Animated Gradients on iOS"
subtitle: "Sequencing animation between gradients with varying numbers of colors and more."
---

<div align="center">
    <img src="https://github.com/rwbutler/rwbutler.github.io/raw/master/img/above-clouds.png" alt="Above Clouds">
</div>

_I recently created a game called Heartbreaker as an Easter Egg for our app. The game had a gradiented background and whilst at the time of making the game there wasn’t enough time to animate the background gradient, it was a challenge I was keen to return to when done. This article is intended to provide an overview of animating gradients on iOS as well as illustrating some of the challenges encountered along the way._

<br/>

<div align="center">
    <img src="https://github.com/rwbutler/rwbutler.github.io/raw/master/img/heartbreaker.gif" alt="Heartbreaker">
</div>

<br/>

<div align="center">
_Heartbreaker_
</div>

# Creating a gradient layer

A gradiented background can be added to a view’s layer fairly easily. To do so we need to instantiate a [`CAGradientLayer`](https://developer.apple.com/documentation/quartzcore/cagradientlayer) instance and then add it the view’s layer as a subview:

```swift
let gradientLayer = CAGradientLayer()
gradientLayer.colors = [UIColor.red, UIColor.blue]
gradientLayer.startPoint = CGPoint(x: 0.0, y: 0.0)
gradientLayer.endPoint = CGPoint(x: 1.0, y: 1.0)
gradientLayer.frame = CGRect(origin: CGPoint.zero, size: view.bounds)
view.layer.addSublayer(gradientLayer)
```

The `colors` property represents the color stops that will make up the gradient. We needn’t limit ourselves to two colors here - it is possible to add a large number of colors creating a multicolored gradient.

The `startPoint` and `endPoint` coordinate values must be within the range 0.0 to 1.0 with (0.0, 0.0) being the top-left corner of the layer and the point (1.0. 1.0) representing the bottom right. Therefore the gradient defined above will be diagonal gradient blending from red in the top-left corner to blue in the bottom-right.

# Animating a gradient

Performing an animation between two gradients containing the same number of color stops is relatively straightforwards by creating a `CABasicAnimation` and adding it to the `CAGradientLayer` we created earlier as follows:

```swift
let colorsAnimation = CABasicAnimation(keyPath: #keyPath(CAGradientLayer.colors))
colorsAnimation.fromValue = gradientLayer.colors
colorsAnimation.toValue = newColors
colorsAnimation.duration = 5.0
colorsAnimation.delegate = self
colorsAnimation.fillMode = .forwards
colorsAnimation.isRemovedOnCompletion = false
gradientLayer.add(colorsAnimation, forKey: "colors")
```

In the above example, `fromValue` is an array containing the colors to animate from whilst `toValue` is an array containing the colors to animate to. Also note that these must be arrays of `CGColor` not `UIColor`. Luckily converting from one to the other is relatively straightforwards as follows:

```swift
let newColors = [UIColor.purple, UIColor.orange].map { $0.cgColor }
```

We use a fill mode of [`.forwards`](https://developer.apple.com/documentation/quartzcore/camediatimingfillmode/1427658-forwards) to ensure that the when complete the animation remains in its final state rather than reverting back to the initial state. If setting the fill mode to `.forwards` it is also necessary to set the `isRemovedOnCompletion` property to `false` since if we remove the animation from the layer then the final state of the animation will no longer be visible.

One final thing to note that if a string is supplied as the `keyPath` when instantiating a CABasicAnimation, this needn’t be the same string as the one you use as a key when adding the animation to the CAGradientLayer. A `keyPath` string indicates the property of the CAGradientLayer to be animated whilst the key used when adding the animation to the CAGradientLayer is just an identifier to use to refer to the animation later on e.g. for removing the animation at some point in the future or for working out which animation completed in a CAAnimationDelegate. In fact, with Swift 4 it is better practice to make use of the `#keyPath()` syntax e.g. `#keyPath(CAGradientLayer.colors)` (as in the example above) which is checked by the compiler to ensure that the property supplied is a valid one.

Notice that we have also assigned self to the `delegate` property of the animation. Here, we are setting the view controller containing this code to be the animation’s delegate which means that the view controller containing this code must conform to the [`CAAnimationDelegate`](https://developer.apple.com/documentation/quartzcore/caanimationdelegate) protocol. This can easily be achieved by creating an extension declaring conformance to the protocol:


```swift
extension ViewController: CAAnimationDelegate {
    func animationDidStop(_ anim: CAAnimation, finished flag: Bool) {
        if flag {
            ...
        }
}
```


The protocol actually declares two functions: `animationDidStart()` and `animationDidStop(finished:)`. If we provide an implementation of `animationDidStop(finished:)` then if we wish to repeat the animation or run another animation after the first has finished we can do so here. Note that we should only do this if the value of flag is `true`. This value is supplied to let us know whether or not the animation completed. It is possible for this function to be called with a value of `false` for example, if the animation was removed prior to completion.

Something worth noting from the [Apple developer documentation](https://developer.apple.com/documentation/quartzcore/cagradientlayer) is that most of the properties of CAGradientLayer are animatable including the `startPoint`, `endPoint`, `locations` and `colors`.

We can animate several of these properties simultaneously with the use of [`CAAnimationGroup`](https://developer.apple.com/documentation/quartzcore/caanimationgroup) or [`CATransaction`](https://developer.apple.com/documentation/quartzcore/catransaction).

An animation group has most of the same properties as `CABasicAnimation` so we can shift many of the property values we would have set on `CABasicAnimation` to the `CAAnimationGroup`. Here’s an example of how we might go about animating multiple `CAGradientLayer` properties simultaneously:

```swift
let locationsAnimation = CABasicAnimation(keyPath: #keyPath(CAGradientLayer.locations))
locationsAnimation.fromValue = gradientLayer.locations
locationsAnimation.toValue = newLocations
let colorsAnimation = CABasicAnimation(keyPath: #keyPath(CAGradientLayer.colors))
colorsAnimation.fromValue = gradientLayer.colors
colorsAnimation.toValue = newColors
let startPointAnimation = CABasicAnimation(keyPath: #keyPath(CAGradientLayer.startPoint))
startPointAnimation.fromValue = gradientLayer.startPoint
startPointAnimation.toValue = newStartPoint
let endPointAnimation = CABasicAnimation(keyPath: #keyPath(CAGradientLayer.endPoint))
endPointAnimation.fromValue = gradientLayer.endPoint
endPointAnimation.toValue = newEndPoint
let animationGroup = CAAnimationGroup()
animationGroup.animations = [colorsAnimation, startPointAnimation, endPointAnimation, locationsAnimation]
animationGroup.duration = 5.0
animationGroup.delegate = self
animationGroup.fillMode = CAMediaTimingFillMode.forwards
animationGroup.isRemovedOnCompletion = false
animationGroup.timingFunction = CAMediaTimingFunction(name: .linear)
gradientLayer.colors = newColors
gradientLayer.locations = newLocations
gradientLayer.startPoint = newStartPoint
gradientLayer.endPoint = newEndPoint
gradientLayer.add(animationGroup, forKey: "animation-group")
```

We have four animations in this example — `colorsAnimation`, `locationsAnimation`, `startPointAnimation` and `endPointAnimation`. We can add them to the animation group by setting the group’s animations property:

```swift
animationGroup.animations = [colorsAnimation, startPointAnimation, endPointAnimation, locationsAnimation]
```

The group can then be added to the gradient layer as you would a CABasicAnimation.

A couple of things to note in the above example include the fact that before adding the animation we set the properties of the gradient layer to the new values — this is good practice so that once the animation is complete even if we hadn’t used `.forwards` as the fill mode, the layer would be the correct final state.

We have also specified a [timing function](https://developer.apple.com/documentation/quartzcore/camediatimingfunction) for the animation which defines how the animation unfolds over time. In this case, we have opted for `.linear` to produce as smooth and continuous an animation as possible but [other options exist](https://developer.apple.com/documentation/quartzcore/camediatimingfunction/predefined_timing_functions) including `.easeIn` which causes the animation to start slow then speed up, `.easeOut` which causes the animation to slow towards the end and `.easeInEaseOut` which start slow, speeds up then slows down again at the end.

# Complex animation

There are a few difficulties which arise when attempting to produce more complex gradient animations such as animating from a gradient containing two colors to a gradient containing three colors. Unless the number of colors in the seconds gradient matches the number of colors in the first, the gradient with two colors will be shown and the animation will not be performed.

In order to ease the process of animating between gradients containing varying numbers of colors we can use AnimatedGradientView, which finds the animation containing the largest number of color stops and then equalizes the number of colors in each animation in order that the animation between the gradients may be performed. The framework also re-calculates the color stop locations as part of animation between two gradients containing a different number of color stops. It is open source, [hosted on GitHub](https://github.com/rwbutler/AnimatedGradientView), written in Swift and available through both Cocoapods and Carthage.

<br/>

<div align="center">
    <img src="https://github.com/rwbutler/rwbutler.github.io/raw/master/img/animatedgradientview.gif" alt="Animated Gradient View">
</div>

# AnimatedGradientView

AnimatedGradientView is designed to make animating gradients as easy as possible. To get started, instantiate an AnimatedGradientView as would any other UIView by defining the frame for the view. You may also define the UIView in a XIB or storyboard setting its `Custom Class` to `AnimatedGradientView` and then creating an outlet for it in your view controller. Then, define your `animationValues` and add your newly created `AnimatedGradientView` as a subview of your view controller’s view.

The `animationValues` property is an array containing values in a tuple which together constitute an animation frame. The framework will automatically animate from the first set of values in this array to the next and so on. On reaching the end of the array, the framework will, by default, loop the sequence of animations, animating from the last frame back to the start. This can be disabled by setting the `autoRepeat` property to `false`.

In order to define an animation frame, three things are required, the colors which make up the gradient, the direction of the gradient as well as the type:

```swift
let animatedGradient = AnimatedGradientView(frame: view.bounds)
animatedGradient.animationValues = [(colors: ["#2BC0E4", "#EAECC6"], .up, .axial),
(colors: ["#833ab4", "#fd1d1d", "#fcb045"], .right, .axial),
(colors: ["#003973", "#E5E5BE"], .down, .axial),
(colors: ["#1E9600", "#FFF200", "#FF0000"], .left, .axial)]
gradientView.addSubview(animatedGradient)
```

## Colors

You might wonder why the color stops for the gradients are defined using Strings rather than UIColors. This is because developers defining gradients frequently wish to define the colors in terms of their hexadecimal values, especially if using a site such as [UIGradients](https://uigradients.com/).

If you wish to use one of the standard color definitions provided by UIColor for example `UIColor.red` it is possible to do so by simply specifying `"red"` as part of the colors array. It is also possible to define colors in terms of their RGB values by specifying each of the RGB components as a value between 0 and 255 e.g. `"rgb(255, 0, 0)"`.

From iOS 11 onwards it is also possible define colors as part of an asset catalog and then access these colors using the `UIColor(named:)` initializer. AnimatedGradientView supports colors defined in this manner so if you defined a color named _turquoise_ in your asset catalog then by supplying the string `"turquoise"` as part of your colors array, the color will be interpreted and included as part of the gradient color stops.

## Directions

AnimatedGradientView supports the following gradient directions:

- `up`
- `up right`
- `right`
- `down right`
- `down`
- `down left`
- `left`
- `up left`

So, if gradient with red and blue color stops is defined with an upwards direction then the red color stop will be at the bottom of the gradient, transitioning upwards into the blue color stop.

## Types
The final parameter to be supplied as part of the animation values array is the type of gradient desired. Options include:

- `axial` — This is your standard linear gradient blending between color stops from the start point to the end point.

<br/>

<div align="center">
    <img src="https://github.com/rwbutler/rwbutler.github.io/raw/master/img/axial.png" alt="Axial Gradient">
</div>

<br/>

<div align="center">
_Axial_
</div>

- `radial` — The gradient appears to radiate outwards from the start point (at the center) towards the end point in a circular fashion blending between the color stops from the start point to end point as with a linear gradient.

<br/>

<div align="center">
    <img src="https://github.com/rwbutler/rwbutler.github.io/raw/master/img/radial.png" alt="Radial Gradient">
</div>

<br/>

<div align="center">
_Radial_
</div>

`conic` — This type of gradient is only available from iOS 12 onwards. It is similar to a radial gradient in that the start point represents the center of the circle and the end point represents a point on the outer edge. However, whilst a radial gradient blends between color stops from the start point to the end point (from the center to the outer edge), a conic gradient places the color stops along the outer edge of the circle blending between the color stops from 0 degrees to 360 degrees.

<br/>

<div align="center">
    <img src="https://github.com/rwbutler/rwbutler.github.io/raw/master/img/conic.png" alt="Conic Gradient">
</div>

<br/>

<div align="center">
_Conic_
</div>

A word of warning — whilst many of the properties of `CAGradientLayer` are animatable, the `type` property is not so although the example application illustrates the gradient animation shifting through different types of gradients this should be employed with caution as switching from one type of gradient to another could cause artefacts to appear. You should run your animation to check it renders well if switching gradient types. It is recommended that you stick to a single gradient type or switch types infrequently.

## Locations

Setting `animationValues` on the AnimatedGradientView is actually shorthand for setting the `animations` property which is an array of `AnimatedGradientView.Animation` objects. Were we to use the longhand form it would look something like this:

```swift
let animatedGradient = AnimatedGradientView(frame: view.bounds)
animatedGradient.animations = [
AnimatedGradientView.Animation(colorStrings: ["#833ab4", "#fd1d1d", "#fcb045"], direction: .up, locations: [0.0, 0.5, 1.0], type: .axial),
AnimatedGradientView.Animation(colorStrings: ["#FEAC5E", "#C779D0", "#fcb045"], direction: .upRight, locations: [0.0, 0.5, 1.0], type: .axial)
]
gradientView.addSubview(animatedGradient)
```

Note that when instantiating a `AnimatedGradientView.Animation`, we may optionally supply an array of locations (an array of NSNumber). If supplied, these will be used for animations rather than AnimatedGradientView calculating these values for you. This can be useful should you not want equally spaced locations for color stops but instead wish to define custom spacing between stops.

## Manual Animation

By default, animations begin automatically if an array of `animationValues` or `animations` is supplied to the AnimatedGradientView when the `autoAnimate` property is `true`. If you prefer to start and stop animations manually simply set `autoAnimate` to `false` and then use the `startAnimating()` function to start animating between gradients and `stopAnimating()` to stop.

# Summary

Gradients can easily be rendered in iOS by creating a CAGradientLayer and adding it to the desired view’s layer as a sublayer. Animating between an initial gradient and another can be achieved using a CABasicAnimation so long as the number of color stops and locations in both gradients match. We can animate multiple properties of CAGradientLayer simultaneously by using a CAAnimationGroup or a CATransaction.

Gradient animations involving more than two gradients can be easily animated using AnimatedGradientView by supplying an `animationValues` array. AnimatedGradientView makes it easy to define gradients in terms of hexadecimal or RGB color values or even asset catalog-defined color names. AnimatedGradientView also performs the calculations required allowing us to animate between gradients with a different number of color stops or locations in each.

## Challenges

- Attempting to animate from a gradient containing a certain number of colors to another gradient containing a different number of colors or from a gradient containing a certain number of locations to another gradient containing a different number of locations will result in no animation being performed.
- Out of the box it can be verbose to chain multiple gradient animations together.
- A gradient cannot be defined using hexadecimal values.

## Advantages of AnimatedGradientView

- Performs the calculations to animate between gradients with a different number of color stops or locations.
- Concise syntax for defining a sequence of gradients.
- Colors may defined in terms of hexadecimal or RGB values or even using predefined color names.
- Can be used to create either a static gradient or an animation one.
- Gradient direction is easily configurable.
- Support for `.axial`, `.radial` and (on iOS 12) `.conic` gradients.
- Auto-animate the sequence of gradient animations or manually control the start / stop of animations.
- Loop animation sequences using the `autoRepeat` property (set to true by default).

<hr/>

_AnimatedGradientView can be found open-sourced on [GitHub](https://github.com/rwbutler/AnimatedGradientView) under MIT license and is compatible with both [Cocoapods](https://cocoapods.org/pods/AnimatedGradientView) and Carthage._

<br/>

<div align="center">
    [<img src="https://github.com/rwbutler/AnimatedGradientView/raw/master/docs/images/animated-gradient-view-large-logo.png" alt="AnimatedGradientView Logo"/>](https://github.com/rwbutler/AnimatedGradientView)
</div>

