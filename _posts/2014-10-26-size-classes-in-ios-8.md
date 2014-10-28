---
layout: post
title: Working With Size Classes in Code With UITraitCollection
---

With the introduction of different screen sizes in iOS 8 we now lean more heavily then ever before on AutoLayout to decide where our views belong given different screen sizes. Thankfully, Apple has given us some great tools to write one interface that will expand or contract given different phones and screen sizes. 

## Size Classes

[Apple defines Size Classes](https://developer.apple.com/library/ios/recipes/xcode_help-IB_adaptive_sizes/chapters/AboutAdaptiveSizeDesign.html) like this:

  > A size class identifies a relative amount of display space for the height and for the width. Each dimension can be either compact, for example, the height of an iPhone in landscape orientation, or regular, for example, the height or width of an iPad. Because much of the layout of an app does not need to change for any available screen size, there is an additional value, any.

Going into the different Size Classes themselves is outside of the scope of this blog post but it boils down to a 2X2 grid of size classes.

|           | H Regular                 | H Compact                        |
|-----------|---------------------------|----------------------------------|
| V Regular | iPad Portrait/Landscape   | 5.5 inch Landscape               |
| V Compact | 3.5"/4.0"/4.7" Landscape  | 3.5", 4.0", 4.7" Portrait iPhone |

Now there is not concept of "rotation", simply a change in the size classes being used. If you take a look at Interface Builder there is the size class selector at the bottom. This allows you to set different AutoLayout constraints for different size classes in IB. I really don't like setting AutoLayout constraints in AutoLayout because I find it confusing, limiting and *very* hard to teach around. Whenever I am teaching I like to go from the explicit case to the implicit case. This allows students to properly understand what is going on before the "magic" that implicit definitions bring set in. Because of this, I start my teaching of AutoLayout with code. We start very early on with creating instances `NSLayoutConstraint` and adding them to the views. There is no ambiguity on what is going on and students gain a much deeper understanding of AutoLayout then if they used Interface Builder. As I prepared for the lecture though, I was curious about how to access the current Size Class from code and to respond to rotation events in the new iOS 8 world.

## What Size Class Am I?

After extensive searching and searching I finally discovered that all of the size class information is kept in a [UITraitCollection](https://developer.apple.com/library/IOs/documentation/UIKit/Reference/UITraitSet_ClassReference/index.html) object. There isn't a ton of documentation on this sadly. If you take a look at the [UIViewController](https://developer.apple.com/library/ios/Documentation/UIKit/Reference/UIViewController_Class/index.html) documentation you'll note there is no mention of `UITraitCollection`. Looking closer we notice that `UIViewController` conforms to the [UITraitEnvironemnt](https://developer.apple.com/library/ios/documentation/uikit/reference/UITraitEnvironment_Ref/index.html) protocol. Hark! We found the location of the `UITraitCollection` property named `traitCollection`.

On `viewDidLoad` for your `UIViewController` take a look at `self.traitCollection`. In it we see all of the visual properties of our device! Here is an example one:

```
<UITraitCollection: 0x7f833b623e80; 
_UITraitNameUserInterfaceIdiom = Phone,
_UITraitNameDisplayScale = 2.000000,
_UITraitNameHorizontalSizeClass = Compact,
_UITraitNameVerticalSizeClass = Regular,
_UITraitNameTouchLevel = 0,
_UITraitNameInteractionModel = 1>
```

From this we can discover that the device I'm using is a retina (@2x from the `_UITraitNameDisplayScale`) phone that is currently in the Compact Horizontal, Regular Vertical size class. That means this is a 3.5", 4.0" or 4.7" phone in portrait mode. Great, now depending on different Size Classes we can apply different AutoLayout constraints on `viewDidLoad`.

## Handling Rotation

Before Size Classes we had a lot of code handling the rotation of the phone in the `willAnimateRotationToInterfaceOrientation:duration:` methods in our View Controllers. Apple has changed this with Size Classes. They've removed the concept of rotation. When the phone rotates, all that changes is the current Size Class. I love this change. There is no reason to think about how your phone has rotated. All that really matters is the dimensions of the screen are different. This Size Class change is caught in the `traitCollectionDidChange:` method. You will be given the previous `UITraitCollection` as an argument and are able to access the current `UITraitCollection` through the `traitCollection` property.
