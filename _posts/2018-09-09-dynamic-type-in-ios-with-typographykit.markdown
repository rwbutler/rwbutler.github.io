---
layout: post
title:  "Dynamic Type in iOS with TypographyKit"
---

Dynamic Type is a feature that was introduced in iOS 7 allowing users to change the default font size used across iOS. It is intended predominantly to support visually impaired users but in practice there are many iOS users who simply prefer a smaller / larger reading size for a variety of reasons. The setting can either be updated from the Settings app in Display & Brightness -> Text Size or from General -> Accessibility -> Larger Text. If the former setting is used only seven font sizes are available to select whereas from the latter section there exists a Larger Accessibility Sizes switch which if enabled, allows the user to select from an additional five larger font sizes.

Apps from the App Store do not automatically support Dynamic Type (unlike on Android) without some additional developer effort. In iOS 7 Apple provided six [UIFontTextStyles](https://developer.apple.com/documentation/uikit/uifont/textstyle), the idea being that each label on a screen would be assigned a text style e.g. headline text or body text which would allow the app developer to retrieve the preferred font for that style of text as follows:

```swift
UIFont.preferredFont(forTextStyle: .body)
```

As the font size slider in the Settings app is adjusted by the user, the point size of the font returned by this method updates accordingly.

At the time of iOS 7 implementing Dynamic Type was challenging since app developers were required to classify each UIKit element showing text into one of the available UIFontTextStyle categories in order to retrieve the preferred font for that category. Furthermore, the preferred font returned would be the system font i.e. Helvetica Neue Light (on versions of iOS prior to 9; the default in iOS 9 onwards being San Francisco) which meant that developers could not use custom fonts whilst supporting Dynamic Type (more on this later). For these reasons, Dynamic Type was often not supported in apps from the App Store.

If a user updated the font size preference in the Settings app, apps which were already open that did support Dynamic Type would typically not update until the app was closed and relaunched by the user (depending on where the call to UIFont.preferredFont(forTextStyle:) was made). Each time the user updates the Dynamic Type preference using the slider in the Settings app, the value of the property [UIApplication.shared.preferredContentSizeCategory](https://developer.apple.com/documentation/uikit/uiapplication/1623048-preferredcontentsizecategory) is updated returning a new [UIContentSizeCategory](https://developer.apple.com/documentation/uikit/uicontentsizecategory) value i.e. each notch on the slider corresponds to a UIContentSizeCategory value.

In order to update the app’s font size without having to close and re-open the app, a developer could observe the (UIContentSizeCategory) [didChangeNotification](https://developer.apple.com/documentation/uikit/uicontentsizecategory/1622948-didchangenotification) using NotificationCenter and then call UIFont.preferredFont(forTextStyle:) which would return a font with an updated size for each of the UIKit elements showing text onscreen. In practice, observing this notification in every UIViewController and updating every textual UIKit element onscreen programmatically usually resulted in a UIViewController bloated with a lot of additional code to support Dynamic Type.

It was for the reasons above that [TypographyKit](https://github.com/rwbutler/TypographyKit) was created. TypographyKit allows apps to use custom fonts whilst supporting Dynamic Type from iOS 9+ without a large amount of additional code. If using Cocoapods, TypographyKit can be incorporated into your Xcode project by adding the following line to the project Podfile then running the ‘pod install’ command:

```
pod "TypographyKit"
```

For those using Carthage, TypographyKit can be installed by adding the line below to the project’s Cartfile, running ‘carthage update --platform iOS’ to build the framework then manually linking the built framework for the relevant Xcode project targets:

```
github "rwbutler/TypographyKit"
```

The next step is to add a TypographyKit configuration file to your project (be sure to add this to the Copy Bundle Resources build phase). The file may be in either Property List or JSON format and named either [TypographyKit.plist](https://github.com/rwbutler/TypographyKit/blob/master/Example/TypographyKit/TypographyKit.plist) or [TypographyKit.json](https://github.com/rwbutler/TypographyKit/blob/master/Example/TypographyKit/TypographyKit.json) respectively. A TypographyKit configuration specifies the the typography styles (i.e. UIFontTextStyles) to be used in your app as well as how each should look. It also specifies how fonts should scale with changes in UIContentSizeCategory:

```json
{
    "typography-colors": {
         "background": "lighter royal-blue",
         "gold": "#FFAB01",
         "royal-blue": "#08224C",
         "text": "gold"
     },
     "typography-kit": {
         "minimum-point-size": 10,
         "maximum-point-size": 100,
         "point-step-size": 2,
         "point-step-multiplier": 1
     },
     "ui-font-text-styles": {
         "heading": {
            "font-name": "Avenir-Medium",
            "point-size": 36,
            "text-color": "text",
            "letter-case": "regular"
        },
        "paragraph": {
            "font-name": "Avenir-Medium",
            "point-size": 16,
            "text-color": "text",
            "letter-case": "regular"
        }
    }
}
```

TypographyKit configuration files can be hosted remotely and so can be thought of as almost akin to a CSS file for an app.

The ‘typography-kit’ section specifies how your fonts should scale with changes in UIContentSizeCategory the default being an increase in two points for each increase in UIContentSizeCategory (i.e. each notch on the text size slider in Settings). You may optionally specify a maximum and / or minimum point size here to prevent fonts ever scaling above / below sensible values you define.

The ‘ui-font-text-styles’ section specifies typography styles for your app. For each style the ‘font-name’ (which may be either a system or a custom font) and the default ‘point-size’ for that font i.e. the size of text when the text size slider in Settings is set to the middle notch (equating to UIContentSizeCategory.large) should be specified. Optionally, a text style color and letter case may be defined as part of your typography style such that a style might consist of the font Avenir-Medium, a default size of 36 points in blue and uppercased.

It is worth noting a few things here — the value of the ‘font-name’ should be the PostScript name of the font which can be retrieved using the [Font Book](https://support.apple.com/en-gb/guide/font-book/welcome/mac) application if you are unsure of it. If using a custom font, that font must be defined as part of your app’s Info.plist under the ‘[Fonts provided by application](https://developer.apple.com/documentation/uikit/text_display_and_fonts/adding_a_custom_font_to_your_app)’ as before prior to use e.g.

```
<key>UIAppFonts</key>
<array>
<string>MyCustomFont.otf</string>
</array>
```

The value of ‘text-color’ may be a simple color value such as ‘blue’, a hex value prefixed by a # e.g. ‘#08224C’ or an RGB color value (with color components in the range 0–255) e.g. ‘rgb(255, 255, 255)’. It is recommended that you assign colors to be used as part of your typography styles a name in the ‘typography-colors’ section of your configuration. Advantages of this include:

- The same color can be used programmatically with an extension on UIColor so that the same color shade is applied consistently throughout your app:

```
extension UIColor {
 static let textColor: UIColor = TypographyKit.colors["text-color"]!
}
```

- iOS 11 introduced the UIColor(named:) initializer allowing a color defined in an asset catalog to be instantiated. TypographyKit can polyfill this functionality back to iOS 9 provided the color is also defined in the TypographyKit.json and a UIColor extension (as above) used to access the color. On iOS 11, this will call UIColor(named:) and on prior iOS versions use the color definition in your TypographyKit.json.
- [Palette for TypographyKit](https://github.com/rwbutler/TypographyKitPalette) is able to generate a color palette for use in Interface Builder such that colors defined in configuration will also be available to select when editing Storyboards and XIBs in IB.
- If TypographyKit is hosted remotely, your app’s color palette can be updated after release.
- Recursive color definitions are supported such that you may set your text color to ‘gold’ in TypographyKit.json where gold is defined in the configuration as “#FFAB01”. This easier for developers to read than simply setting “text-color”: “#FFAB01”.

Further information on color configuration is available in the blog post [Remotely Configured Colour Palettes in TypographyKit](https://medium.com/@rwbutler/remotely-configured-colour-palettes-in-typographykit-e565c927e2b4).

Letter cases which may be applied to your typography style include:

- regular (text remains unchanged — this is the default)
- capitalized (first letter of each word is capitalized)
- kebab (words joined by a hyphen)
- lower (lowercase)
- lower-camel (camel case with each word beginning with a lowercase letter e.g. lowerCamelCase)
- macro (uppercase with words joined with an underscore e.g. MACRO_CASE)
- snake (lowercase with spaces between words removed)
- upper (uppercase)
- upper-camel (camel case with each word beginning with an uppercase letter e.g. UpperCamelCase)

Having defined the typography styles for an app, these styles may be applied to the UIKit (UIButton, UILabel, UITextField and UITextView are currently supported) elements in your storyboard by setting a Key Path value of String type named ‘fontTextStyleName’ to the name of your style as defined in your TypographyKit configuration file. This can be found as part of the Identity Inspector in the Utilities section of Interface Builder (see below). The [TypographyKit sample application](https://github.com/rwbutler/TypographyKit/tree/master/Example) provides an example of this.

<br/>

<div align="center">
    <img src="https://github.com/rwbutler/rwbutler.github.io/raw/master/img/dynamic-type-in-ios-with-typographykit.png" alt="Setting a Key Path in Interface Builder’s Identity Inspector">
</div>

<br/>

<div align="center">
_Setting a Key Path in Interface Builder’s Identity Inspector_
</div>

With the key path set appropriately, your UIKit element will respond to changes in UIContentSizeCategory without any further code.

Many app developers work in conjunction with a design team to who are typically keen to support accessibility and enforce visual consistency across an app through consistent use of color and typographical styling however constraining fonts used to system fonts only and the number of text categories to a small number were usually a deal breaker when it came to supporting Dynamic Type.

Apple realised this and thus in iOS 9 four new classes of UIFontTextStyle were introduced with the largeTitle style being introduced later in iOS 11:

- body (iOS7)
- callout (iOS9)
- caption1 (iOS7)
- caption2 (iOS7)
- footnote (iOS7)
- headline (iOS7)
- subheadline (iOS7)
- largeTitle (iOS11)
- title1 (iOS9)
- title2 (iOS9)
- title3 (iOS9)

As it turns out this can still be quite limiting since even if developers are able to limit the number of text styles used in an app to eleven, the classifications above may not be suitable to the developer’s app. Say for example, a developer wishes to apply the same text style to every button in an app consistently. Should this be shoehorned into a caption2 UIFontTextStyle? Is eleven a sufficient number of text styles to cover every typographical style in an app?

TypographyKit solves this issue by allowing the developer to define their own set of UIFontTextStyles. In the TypographyKit.json example above, a style named ‘paragraph’ (as part of the ‘ui-font-text-styles’ section) was defined even though this is not present in the list UIFontTextStyles above. So long as a definition exists for the style in the TypographyKit configuration, a developer may use either existing UIFontTextStyles or create new custom styles by creating an extension on UIFontTextStyle:

```swift
extension UIFontTextStyle {
    static let paragraph = UIFontTextStyle(rawValue: "paragraph")
}
```

It is recommended however that the number of UIFontTextStyles used in an app is kept to a minimum such that the app is as visually consistent as possible since apps which do not enforce consistency tend towards having a chaotic or cluttered UI.

Custom UIFontTextStyles will behave in exactly the same manner as those provided as standard with text scaling according to the settings defined in the TypographyKit configuration.

Another improvement Apple made in iOS 10 was to introduce the UIContentSizeCategoryAdjusting protocol implemented by UILabel, UITextField and UITextView containing the property:

[```var adjustsFontForContentSizeCategory: Bool
```](https://developer.apple.com/documentation/uikit/uicontentsizecategoryadjusting/1771731-adjustsfontforcontentsizecategor)

This property allows the text on implementing UIKit elements to adjust automatically with changes in UIContentSizeCategory and can be set in storyboards by checking the ‘Automatically Adjusts Font’ checkbox. This property does not work if the UIKit element is set to use Attributed rather than Plain text. Another large caveat is that this property does not work where a custom font is used.

Where using TypographyKit these controls will adjust even in iOS 9 including UIButtons and will work with Attributed text (although support for this is currently limited with plans to extend this in future).

The biggest improvement Apple made to the implementation of Dynamic Type was the introduction of the UIFontMetrics class in iOS 11. This allows a developer to obtain a scaled version of a font including custom fonts using the following method:

```swift
func scaledFont(for font: UIFont) -> UIFont
```

By providing a font to be scaled, this method returns the same font face, scaled according to the user’s preferred UIContentSizeCategory using the same transform as would be applied to the system font where requesting UIFont.preferredFont(forTextStyle:). The font sizes for San Francisco at each UIContentSizeCategory step can be found as part of the [Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/ios/visual-design/typography/).

In summary, TypographyKit provides the following benefits to developers:

- Dynamic Type support with custom fonts on iOS 9+ where UIFontMetrics is unavailable.
- Dynamic Type scaling can be clamped to a minimum and / or maximum font size.
- Supporting Dynamic Type can bloat a UIViewController significantly — TypographyKit provides the option to support Dynamic Type entirely through configuration (zero additional code). Convenience methods are provided where implementing programmatically.
- UIContentSizeCategoryAdjusting in iOS 10 does not work with custom fonts forcing the developer to go back to NotificationCenter observation whilst TypographyKit handles this back to iOS 9.
- Enforces visual consistency in your app’s typography and color palette optionally including letter casing as part of typography styles.
- TypographyKit configuration can be hosted remotely providing something akin to CSS for your app.
- Respond to changes in UIContentSizeCategory on iOS 9+ including support for UIButton and NSAttributedString support for which is currently unavailable in UIKit.
- Define custom UIFontTextStyles with your design team to come up with a set of text styles that best suits your app.
- Allows colors to be defined in a single location similarly to Android’s colors.xml and used both programmatically and in Interface Builder using Palette.
- Shim for UIColor(named:) prior to iOS 11.

More information on TypographyKit can be found [over at GitHub](https://github.com/rwbutler/TypographyKit/).

## References

Apple Developer Documentation (Notably [Adding a Custom Font to Your App](https://developer.apple.com/documentation/uikit/text_display_and_fonts/adding_a_custom_font_to_your_app), [didChangeNotification](https://developer.apple.com/documentation/uikit/uicontentsizecategory/1622948-didchangenotification), [preferredContentSizeCategory](https://developer.apple.com/documentation/uikit/uiapplication/1623048-preferredcontentsizecategory) & [UIContentSizeCategory](https://developer.apple.com/documentation/uikit/uicontentsizecategory))

[Apple Developer Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/ios/visual-design/typography/)

[Remotely Configured Colour Palettes in TypographyKit](https://medium.com/@rwbutler/remotely-configured-colour-palettes-in-typographykit-e565c927e2b4)
