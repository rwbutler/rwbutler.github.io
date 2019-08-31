---
layout: post
title:  "Accessible UICollectionViews With Dynamic Type and Self-Sizing Cells"
subtitle: "Make your apps accessible by ensuring that UICollectionViews resize with changes in user text size preference."
---

<div align="center">
    <img src="https://github.com/rwbutler/rwbutler.github.io/raw/master/img/accessible-uicollectionviews-with-dynamic-type-and-self-sizing-cells.png" alt="Accessible Car Parking">
</div>

_Dynamic Type is a feature that was introduced in iOS 7 allowing users to change the default font size used across iOS. It is intended predominantly to support visually impaired users but in practice, there are many iOS users who simply prefer a smaller/larger reading size for a variety of reasons._

Over successive releases of iOS, Apple has made numerous improvements to assist developers with the implementation of accessibility features. Each year at [WWDC](https://developer.apple.com/wwdc19/), Apple has held sessions on accessibility improvements and pushed developers to develop more inclusive experiences through the implementation of assistive technologies. For example, apps which have provided great experiences for users of VoiceOver have often been featured as part of the [Apple Design Awards](https://developer.apple.com/design/awards/). Of all the assistive technologies provided by iOS, Dynamic Type is without doubt one of the most important, allowing users to increase or decrease the text size on their device via a slider in the settings.

Despite the push from Apple to develop inclusive and accessible experiences, and despite Dynamic Type having been introduced way back in iOS 7, many well-known apps still lack support for it. There are numerous reasons for this:

- Lack of support for custom fonts. Apps developed by larger organizations often use a custom typeface supporting the visual identity of the organization. Until `UIFontMetrics` was introduced in iOS 11, scaling a custom font with changes in `UIContentSizeCategory` was a non-trivial amount of work. Apps developed by larger organizations frequently need to support older versions of iOS meaning that `UIFontMetrics` may be unavailable to them.
- Listening for text size changes limits customization. Observing changes in text size using `UILabel`, `UITextField`, and `UITextView` is much easier from iOS 10 onwards since the introduction of the `UIContentSizeCategoryAdjusting` protocol. The protocol, which is implemented on these UIKit elements declares a `Bool` property `adjustsFontForContentSizeCategory` which when set true allows these UIKit elements to automatically adjust their text size inline with the user’s text size preference in the Settings app. The downsides, however, are that this protocol is only implemented on these three UIKit elements, it provides no support for attributed text and in order to make use of it you must select from one of the eleven pre-defined UIFontTextStyles which means that you are forced to use the system font (currently San Francisco) and the predetermined font sizes associated with these styles. Although there is usually a large amount of goodwill towards implementing Dynamic Type, the associated constraint of only being able to use pre-determined font faces and sizes is often the factor resulting in it never being implemented at all.
- The greatest challenge to implementing Dynamic Type on iOS is ensuring that as the text size of a UIKit element increases, so does the content size of its parent views and that sibling views are laid out around the resized views in a way that doesn’t make the design feel broken and unusable. Auto Layout provides the means of achieving this but careful thought is required when choosing constraints in order to ensure that the interface still looks good at the larger content size. It’s easy to end up in a situation where Auto Layout needs to break a constraint in order to satisfy the layout, leading to one or more elements ending up at the wrong size or even in the wrong place altogether.

We can use self-sizing cells (introduced in iOS 8) when displaying textual content in a UITableView or a UICollectionView to ensure that the content fits correctly without being squashed or truncated. This also comes in handy when the cell text is dynamic or when localizing an app to accommodate different languages which could take up varying amounts of space to convey the same sentiment.

## Self-Sizing UITableViewCells

When it comes to UITableViews, implementing self-sizing cells is relatively straightforward — in fact, from iOS 11 onwards so long as you set up your constraints in a such a way as Auto Layout can determine the height of cell then that’s all you need to do. Usually, this means ensuring your views have a constrained width when implementing flexible cell heights either by setting a width constraint explicitly or setting leading and trailing edge constraints — since if cell subviews can have infinite width then there’s no way to calculate the height of the cell. Conversely, when implementing flexible cell widths you will want to ensure that it is possible to concretely determine the cell’s height using constraints. The hardest part of implementing self-sizing cells is determining the constraints to be added in an order that the height (or width) of a cell may be calculated without ambiguity.

If your cell contains any UILabel elements, you’ll want to set the `Lines` property in Interface Builder to `0` which indicates that the label may use as many lines as needed to fit the textual content. You will also need to set the `Line Break` property to `Word Wrap` so that rather than truncating content, the content wraps to the next line. When configuring a `UILabel` programmatically, the same thing can be achieved as follows:

```swift
label.lineBreakMode = .byWordWrapping
label.numberOfLines = 0
```

If you need to support versions of iOS prior to 11, then you will need to set the `estimatedRowHeight` property to a value in order to enable cell self-sizing. It is common to set the value of this property to `44.0` as this the default height of a UITableViewCell. The Apple Human Interface Guidelines also states that you should attempt to [maintain a minimum tappable area of 44pt x 44pt for all controls](https://developer.apple.com/design/human-interface-guidelines/ios/visual-design/adaptivity-and-layout/#general-layout-considerations). Most tutorials will recommend setting the `estimatedRowHeight` property and `rowHeight` property to `UITableView.automaticDimension` which indicates that the UITableView should attempt to determine the heights of cells based on their constraints.

```swift
tableView.estimatedRowHeight = 44.0
tableView.rowHeight = .automaticDimension
```

In practice, it should only be necessary to set the `estimatedRowHeight` (prior to iOS 11) since the default value of the `rowHeight` property is already `automaticDimension`. It’s also worth noting the following from the Apple developer documentation for estimatedRowHeight:

_The default value is [`automaticDimension`](https://developer.apple.com/documentation/uikit/uitableview/1614961-automaticdimension), which means that the table view selects an estimated height to use on your behalf. Setting the value to 0 disables estimated heights, which causes the table view to request the actual height for each cell. If your table uses self-sizing cells, the value of this property must not be `0`._

Should you only need certain cells to self-size then you may implement [`tableView(_:heightForRowAt:)`](https://developer.apple.com/documentation/uikit/uitableviewdelegate/1614998-tableview) in `UITableViewDelegate`, returning `automaticDimension` only for the cells you wish to self-size. You may also provide an implementation for [`tableView(_:estimatedHeightForRowAt:)`](https://developer.apple.com/documentation/uikit/uitableviewdelegate/1614926-tableview) in order to return a different estimated row height for each cell.

## Self-Sizing UICollectionView Cells

Implementing self-sizing UICollectionViewCells is a little more involved but follows much the same pattern. We need to configure the constraints of our cell in such a way is that it is possible for the UICollectionView layout to determine the height of the cell on our behalf. The provided `UICollectionViewLayout` is [`UICollectionViewFlowLayout`](https://developer.apple.com/documentation/uikit/uicollectionviewflowlayout) which lays out cells in a grid and has an `itemSize` property and `estimatedItemSize` property. We can set the `itemSize` property to a CGSize value (the default value is `(50.0, 50.0)`) however this will result in all cells having the same dimensions. Another means of providing the item size to the layout is to implement the optional delegate method [`collectionView(_:layout:sizeForItemAt:)`](https://developer.apple.com/documentation/uikit/uicollectionviewdelegateflowlayout/1617708-collectionview).

Within your implementation of this method, you can then calculate the CGSize of the cell for the provided IndexPath on a per-cell basis meaning that cells can have distinct sizes. This involves calculating the width and height of your UIKit subviews (labels, images, etc.) within the cell. Calculating the height of text in a UILabel can be achieved using the NSString method [`boundingRect(with:options:attributes:context:)`](https://developer.apple.com/documentation/foundation/nsstring/1524729-boundingrect).

The heights of images may be obtained using the `intrinsicContentSize` property. If your views are laid out either horizontally or vertically in a stack then UIStackView can be used to make life much easier here. By adding your views as arranged subviews of a UIStackView you can then access the `intrinsicContentSize` property of the stack view in order to determine its height.

Should your UICollectionViewCell contain any UILabel elements remember to set the `numberOfLines` property and `lineBreakMode` as with table view cells to ensure that text wraps and the labels are able to grow in height with taller textual content.

Using the aforementioned methods, we can support Dynamic Type by allowing our cells to have individual sizes and to change the height based on their content however the cells are not yet self-sizing. To enable self-sizing cells we need to provide a non-zero estimate for the cell size by setting the `estimatedItemSize` property on UICollectionViewFlowLayout.

_The default value of this property is [`CGSizeZero`](https://developer.apple.com/documentation/coregraphics/cgsizezero). Setting it to any other value causes the collection view to query each cell for its actual size using the cell’s [`preferredLayoutAttributesFitting(:)`](https://developer.apple.com/documentation/uikit/uicollectionreusableview/1620132-preferredlayoutattributesfitting) method._

Prior to iOS 10, it was necessary to set the `itemSize` property to `UICollectionViewFlowLayout.automaticSize` and the `estimatedItemSize` to an estimate that as accurately as possible determined what the size of the cell would be.

```swift
if let layout = collectionView.collectionViewLayout as? UICollectionViewFlowLayout {
    layout.estimatedItemSize = CGSize(50.0, 100.0)
    layout.itemSize = UICollectionViewFlowLayout.automaticSize
}
```

From iOS 10 onwards, we only need to set the `estimatedItemSize` property to `UICollectionViewFlowLayout.automaticSize`:

```swift
if let layout = collectionView.collectionViewLayout as? UICollectionViewFlowLayout {
    layout.estimatedItemSize = UICollectionViewFlowLayout.automaticSize
}
```

You may alternatively wish to create an outlet for your UICollectionViewLayout if defining your collection view in a nib or storyboard and then utilize a `didSet` observer to set the `estimatedItemSize`.

Once the layout is aware that it should attempt to automatically size cells based on their constraints, the next thing we need to do is disable the auto-resizing mask on the cell’s content view.

Prior to the introduction of Auto Layout, iOS used a system known as _springs and struts_ for determining how a UIView’s frame would update in relation to changes in the frame of its superview. The springs represented how a UIView’s width and height would either stretch or compress whilst the struts represented a UIView’s insets from its superview and neighboring views. Essentially springs when present, enabled a subview to grow in size as its superview increased in size whilst struts allowed a consistent margin from sibling and parent views to be defined.

An [auto-resizing mask](https://developer.apple.com/documentation/uikit/uiview/1622559-autoresizingmask) is an integer bitmask representing a view’s springs and struts. When Auto Layout was introduced, in order to retain a measure of backward compatibility with the old springs and struts system, by default a UIView’s auto-resizing mask was translated into constraints. Unfortunately, this automatic translation of a view’s springs and struts into constraints does not always give the desired result and therefore a means of disabling the automatic translation exists in the form of a property named `translatesAutoresizingMaskIntoConstraints`. If set to `false`, then a view’s auto-resizing mask will no longer be translated into Auto Layout constraints.

According to the documentation for [`translatesAutoresizingMaskIntoConstraints`](https://developer.apple.com/documentation/uikit/uiview/1622572-translatesautoresizingmaskintoco):

_Note that the autoresizing mask constraints fully specify the view’s size and position; therefore, you cannot add additional constraints to modify this size or position without introducing conflicts. If you want to use Auto Layout to dynamically calculate the size and position of your view, you must set this property to `false`, and then provide a non ambiguous, nonconflicting set of constraints for the view._

All collection view cells have a subview called the content view to which your labels, image views, etc. are added. Unfortunately, the content view’s auto-resizing mask is translated into constraints by default and when this occurs the content view’s dimensions may no longer resize due to the constraints which are added as part of the translation which concretely specifies the view’s size. Therefore, in order to allow our cells to self-size we need to disable translation of the auto-resizing mask and add our own constraints in order to specify the size and position of the cell’s content view within the cell as follows:

```swift
override func awakeFromNib() {
super.awakeFromNib()
contentView.translatesAutoresizingMaskIntoConstraints = false
NSLayoutConstraint.activate([
contentView.leftAnchor.constraint(equalTo: leftAnchor),
contentView.rightAnchor.constraint(equalTo: rightAnchor),
contentView.topAnchor.constraint(equalTo: topAnchor),
contentView.bottomAnchor.constraint(equalTo: bottomAnchor)
])
}
```

The above will disable constraint translation and add new constraints to pin each edge of the content view to the respective edges of the parent cell when a cell is inflated from a storyboard or nib. If you are instantiating a cell programmatically instead then you may wish to do this within the cell’s initializer.

So long as you have provided sufficient constraints within your cell’s content view and on its subviews to allow the AutoLayout system to determine the height of your cell (or width if you are resizing horizontally) then your cells should now self-size.

If the layout isn’t quite as you’d like to be, it is possible for a UICollectionViewCell subclass to modify its layout attributes by providing an implementation of the method `preferredLayoutAttributesFitting(_:)`.

_The default implementation of this method adjusts the size values to accommodate changes made by a self-sizing cell. Subclasses can override this method and use it to adjust other layout attributes too. If you override this method and want the cell size adjustments, call `super` first and make your own modifications to the returned attributes._

Therefore, by providing an implementation as part of a UICollectionViewCell subclass, you may adjust the layout attributes created by your collection view layout before they are used by the UICollectionView to layout cells as follows:

```swift
override func preferredLayoutAttributesFitting(_ layoutAttributes: UICollectionViewLayoutAttributes) -> UICollectionViewLayoutAttributes {
    setNeedsLayout()
    layoutIfNeeded()
    let size = contentView.systemLayoutSizeFitting(layoutAttributes.size)
    var newFrame = layoutAttributes.frame
    // Make any additional adjustments to the cell's frame
    newFrame.size = size
    layoutAttributes.frame = newFrame
    return layoutAttributes
}
```

In the above example, we indicate that the cell needs to be laid out by calling `setNeedsLayout()` then invoke `layoutIfNeeded()` in order to position the content view’s subviews according to the constraints defined upon them.

With the cell correctly laid out, we call `systemLayoutSizeFitting(_:)` in order to determine the optimum size for the cell, which is as close as possible to the size we pass to the method as a parameter (in this example, the size we pass is the size returned by the layout — UICollectionViewFlowLayout) whilst still satisfying all of the constraints on the cell’s subviews. It’s worth noting that it’s also possible to get the smallest size that will satisfy the cell’s constraints or the largest size using either [`layoutFittingCompressedSize`](https://developer.apple.com/documentation/uikit/uiview/1622568-layoutfittingcompressedsize) or [`layoutFittingExpandedSize`](https://developer.apple.com/documentation/uikit/uiview/1622532-layoutfittingexpandedsize) respectively e.g.

```swift
let size = contentView.systemLayoutSizeFitting(UIView.layoutFittingCompressedSize)
```

It should only be necessary to adjust a cell’s layout attributes if the constraints defined within the cell do not provide enough information to allow AutoLayout to calculate the height/width of the cell unambiguously. Occasionally, it may useful to provide an implementation of `preferredLayoutAttributesFitting(_:)` for those supporting older versions of iOS which may suffer from layout quirks.

<br/>

<div align="center">
    <img width="50%" height="50%" src="https://github.com/rwbutler/FlexibleRowHeightGridLayout/raw/master/docs/images/uicollectionviewflowlayout.png" alt="Self-sizing cells in UICollectionViewFlowLayout resulting in cells of different sizes within the same row"/>
</div>

_Self-sizing cells in UICollectionViewFlowLayout resulting in cells of different sizes within the same row._

<br/>

Cells should now be self-sizing according to the size of their content. In the screenshot above, I have added a background color to each cell’s content view so that it possible to see that the height of each cell is adjusting according to the amount of text contained within.

With a background color or with a border on each cell however you may notice that things look a bit odd. Although each cell is self-sizing according to its content, the cells aren’t the same size as their neighboring cells in the same row which looks jarring.

In order to fix this, we can use an open-source UICollectionViewLayout named [FlexibleRowHeightGridLayout](https://github.com/rwbutler/FlexibleRowHeightGridLayout). Its purpose is largely to behave like a flow layout except that each row is sized to fit the tallest cell within that row so that you end up with true grid which fits its content rather than the jarring presentation shown above. With FlexibleRowHeightGridLayout, we can obtain a much nicer looking grid as shown in the screenshot below:

<br/>

<div align="center">
    <img width="50%" height="50%" src="https://github.com/rwbutler/FlexibleRowHeightGridLayout/raw/master/docs/images/flexiblerowheightgridlayout.png" alt="Cells in FlexibleRowHeightGridLayout whereby all rows fit their content"/>
</div>

_Cells in FlexibleRowHeightGridLayout whereby all rows fit their content._

<br/>

This layout was also created specifically to support Dynamic Type, therefore as the user adjusts their text size preference in the Settings app, this layout will automatically re-layout its content in order to fit the new text size preference, unlike UICollectionViewFlowLayout.

## FlexibleRowHeightGridLayout

Just as with UICollectionViewFlowLayout, FlexibleRowHeightGridLayout is designed to layout cells in a grid and also just like UICollectionViewFlowLayout it supports section headers and footers. Making use of it is relatively straightforward and involves two steps:

- Instantiate the layout and assign it to the UICollectionView’s `collectionViewLayout` property.
- Implement the layout’s delegate FlexibleRowHeightGridLayout.Delegate.

If you are creating your UICollectionView programmatically then you can instantiate the layout at the same time as follows:

```swift
let layout = FlexibleRowHeightGridLayout()
layout.delegate = self
let collectionView = UICollectionView(frame: .zero, collectionViewLayout: layout)
```

Otherwise, if your UICollectionView is instantiated from a storyboard or XIB then simply assign the layout to the UICollectionView’s `collectionViewLayout` property:

```swift
let layout = FlexibleRowHeightGridLayout()
layout.delegate = self
collectionView.collectionViewLayout = layout
```

If the code above is added to your UIViewController’s `viewDidLoad` implementation then that should be all you need. However, if you assign the layout to the UICollectionView after the data has been loaded then you may need to invoke `invalidateLayout` on the UICollectionView to ensure that the layout is queried by the UICollectionView for the new layout attributes. If the underlying data has also changed then you may need to call `reloadData`.

The second step involves implementing the layout’s delegate - FlexibleRowHeightGridLayout.Delegate. If you aren’t interested in having headers or footer in your UICollectionView then there are only two methods you need to implement:

- `func collectionView(_ collectionView: UICollectionView, layout: FlexibleRowHeightGridLayout, heightForItemAt indexPath: IndexPath) -> CGFloat`
- `func numberOfColumns(for size: CGSize) -> Int`

In order to layout your UICollectionViewCells correctly there are two pieces of information the layout requires. Firstly, it needs to know how tall the content in each of your cells is — using this information, the layout is then able to determine the height of each row. Secondly, the layout needs to know how many columns your grid should have — allowing the layout to determine the width available to each cell. You may want your layout to have a different number of columns when the device orientation is landscape as opposed to portrait, therefore, this delegate method will be invoked whenever a device orientation change occurs.

In order to help you determine the height of your content, FlexibleRowHeightGridLayout provides a couple of useful methods such as `textHeight(_ text: String, font: UIFont)` to help you calculate the height required to display a String rendered using the specified font. If your cell were to contain a label pinned to each of the edges of the cell’s content view then the height of the cell’s content can easily be calculated as follows:

```swift
func collectionView(_ collectionView: UICollectionView, layout: FlexibleRowHeightGridLayout, heightForItemAt indexPath: IndexPath) -> CGFloat {
    let text = dataSource.item(at: indexPath.item)
    let font = UIFont.preferredFont(forTextStyle: .body)
    return layout.textHeight(text, font: font)
}
```

These helper methods are particularly useful for developers who are already using [TypographyKit](https://github.com/rwbutler/typographykit). If you are unfamiliar with it, TypographyKit is a framework which also supports Dynamic Type by automatically updating the text size on UIKit elements (UILabel, UIButton etc) when the users changes the text size preference on their device as well as by allowing developers to define all the text and color styles needed by their app in a JSON configuration file which can be updated remotely. For more information, see my previous post [Dynamic Type on iOS with TypographyKit](https://medium.com/@rwbutler/dynamic-type-in-ios-with-typographykit-9ed0ac5dbf64). For those familiar with TypographyKit already, calculating the height of a cell containing a single label can be achieved as follows:

```swift
func collectionView(_ collectionView: UICollectionView, layout: FlexibleRowHeightGridLayout, heightForItemAt indexPath: IndexPath) -> CGFloat {
let text = dataSource.item(at: indexPath.item)
let font = Typography(for: .cellText).font()
return layout.textHeight(text, font: font)
}
```

Or if your cell is defined in a nib, then it is possible to inflate a cell in order to calculate the cell height, although this is a more expensive operation as it involves reading from a file:

```swift
let text = dataSource.item(at: indexPath.item)
guard let nib = Bundle.main.loadNibNamed("CustomCell", owner: CustomCell.self, options: nil), let cell = nib?[0] as? CustomCell else {
    
    return
}
// Ensure that your content has been set
cell.label.text = text
// Assuming your custom cell has a content view
cell.contentView.setNeedsLayout()
cell.contentView.layoutIfNeeded()
let size = cell.contentView.systemLayoutSizeFitting(UIView.layoutFittingCompressedSize)
return size.height
```

If you also wish to include headers and/or footers in your `UICollectionView` then there are two more delegate methods which need to be implemented as needed:

- `@objc optional func collectionView(_ collectionView: UICollectionView, layout: FlexibleRowHeightGridLayout, referenceHeightForHeaderInSection section: Int) -> CGFloat`
- `@objc optional func collectionView(_ collectionView: UICollectionView, layout: FlexibleRowHeightGridLayout, referenceHeightForFooterInSection section: Int) -> CGFloat`

These methods query your layout for the height of the header/footer for each section of your `UICollectionView`. Should you return a height of `0` from either of these methods then no header / footer will be added.

A full demo can be found as part of the example app in the project’s [GitHub repository](https://github.com/rwbutler/flexiblerowheightgridlayout).

## Summary

Using self-sizing cells helps us to accommodate changes in content size, in particular changes in text size where supporting Dynamic Type. This helps us to develop apps which are more accessible and inclusive to all. Whilst implementing self-sizing cells with UITableView tends to be a little easier on the whole, the process is largely the same for both UITableViews and UICollectionViews - particularly on more recent versions of iOS. With either one, implementing cell self-sizing requires some skill when choosing constraints to unambiguously layout cell content such that Auto Layout is able to determine the height of the cell.

The default UICollectionViewLayout provided for UICollectionView is UICollectionViewFlowLayout which lays out content in a grid with optional section headers and footers however this can lead to rows where content is misaligned as each cell may take on a different height.

[FlexibleRowHeightGridLayout](https://github.com/rwbutler/FlexibleRowHeightGridLayout) is an open-source UICollectionViewLayout designed both to layout content in a grid with headers and footers like UICollectionViewFlowLayout as well as to support Dynamic Type. Row heights are flexible with this layout i.e. each row may have different height whereby the height of the row is determined by the tallest cell in the row so that the row height will always fit the content within the row . The layout is also designed to automatically re-layout with changes in text size on the device (UIContentSizeCategory) or device orientation changes.

[TypographyKit](https://github.com/rwbutler/typographykit) supports Dynamic Type by automatically resizing UIKit elements with text size changes as well as allowing developers to define text styles in a configuration file which may be updated remotely. FlexibleRowHeightGridLayout is particularly useful for developers who are already using TypographyKit by providing helper methods to assist developers in the calculation of text heights for a particular typography style.

For more guidance, take a look at the example app provided with FlexibleRowHeightGridLayout in the [GitHub repository](https://github.com/rwbutler/FlexibleRowHeightGridLayout#exampleapp).

<hr/>

_[FlexibleRowHeightGridLayout](https://github.com/rwbutler/flexiblerowheightgridlayout) and [TypographyKit](https://github.com/rwbutler/TypographyKit) can both be found open-sourced on GitHub under MIT license and are compatible with both [Cocoapods](https://cocoapods.org/pods/FlexibleRowHeightGridLayout) and Carthage._

<br/>

<div align="center">
    <img src="https://github.com/rwbutler/TypographyKit/raw/master/docs/images/typography-kit-logo.png" alt="TypographyKit Logo"/>
</div>
