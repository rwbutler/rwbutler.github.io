---
layout: post
title:  "Updating Users to the Latest Version of Your iOS App"
subtitle: "The challenges of transitioning users to the latest app update where the need arises."
---

<div align="center">
    <img src="https://github.com/rwbutler/rwbutler.github.io/raw/master/img/updating-users-to-the-latest-version-of-your-app.png" alt="iPhone home screen">
</div>

__When an emergency arises, it can be beneficial to have a mechanism in place to gently transition users to the latest app update. However, such a mechanism might require a backend solution to let app users know when an update is released which has the potential to be costly both in terms of development overhead and maintenance. Where the choice is between developing such a solution or delivering new functionality to the user, the latter is likely to win out meaning that most apps have little or no means of migrating users should the worst happen.__

When it comes to migrating users to the latest version of an app we are fortunate that iOS does the majority of the legwork for us. The default App Store setting is to automatically download app updates overnight which means that typically the majority of users will receive the latest app release within a few days.

But what if within a few days isn’t quick enough? What if an API we have no control over unexpectedly changes resulting in a loss of all functionality? Or a certificate used for SSL pinning expires unexpectedly? Unlike in web development, isn’t possible to easily roll users back to an earlier version of the software therefore isn’t hard to imagine that there could be times where it might be useful to be able to migrate users to the latest version of an app containing the latest bug fixes etc. App developers will clearly try to avoid such situations but if we fail to anticipate scenarios like these ever occurring then it will be too late to act when an emergency does arise.

Exceptional cases aside, there are probably quite a few occasions where it might be nice to let users to know that a new app version is available — for example, if a new app feature is available in the latest release which has been widely publicized or a fix has been released for that non-critical but annoying issue the user has been experiencing.

# Dropping Support for Older iOS Releases

Apple are highly successful in creating a pull factor which encourages users to update to the latest version of iOS. Additionally, iOS itself makes it clear to users via badging and notifications that a new version of the OS is available. Thus, the majority of iOS users tend to be running either the latest or previous version of the iOS. For developers this means we rarely need to very old versions of the operating system. As far as getting users to update to the latest version of our app is concerned, having a recent version of iOS the majority of devices is obviously a great help - particularly if our app only supports the two most recently released versions of iOS.

However, consider an app supporting an older version of iOS the development team wishes to drop support for. If the team drop supports for the older version of iOS then the app update will no longer be automatically offered to users on that version of the operating system by the App Store until they update. Furthermore if the user’s visits the app’s page in the App Store they will only be able to see the latest update their device is eligible for. It could be worthwhile letting users know that a new version of the app is available to them once they update a later version of iOS so that they don’t make the assumption that development of the app has ended.

# Updates

[Updates](https://github.com/rwbutler/Updates) is an open-source framework which automatically checks to see whether a new version of your app is available. When an update is released, Updates is able to present the new version number and accompanying release notes to the user giving them the choice to update. Users electing to proceed are seamlessly presented the App Store in-app so that updating becomes effortless.

<br/>

<div align="center">
    <img src="https://github.com/rwbutler/Updates/raw/master/docs/images/screenshot.png" alt="Screenshot showing Updates in automatic configuration">
</div>

<br/>

<div align="center">
_Screenshot showing Updates in automatic configuration_
</div>

A framework which is able to handle much of this automatically is beneficial as development teams are usually, and rightly, more focussed on developing new functionality to deliver value to the user than on implementing functionality for updating users which might never need to be used.
Updates makes use of the [iTunes Search API](https://developer.apple.com/library/archive/documentation/AudioVideo/Conceptual/iTuneSearchAPI/index.html) to retrieve the version number of the latest version of your app available from the store. Along with this, the release notes and numeric App Store identifier are fetched for your app which means that when a new version is released, Updates is able to tell your users the version number of the update as well as what’s new.

The framework is able to automatically resolve the user’s device locale ensuring that the app update is for the relevant App Store territory as well as details such as whether or not the user’s device is eligible to receive the update.

Using the numeric App Store identifier, if the user elects to update then Updates can present the App Store using [SKStoreProductViewController](https://developer.apple.com/documentation/storekit/skstoreproductviewcontroller) allowing the user to seamlessly update without ever having to leave the app.

If you would prefer to set this information manually (rather than having Updates retrieve it for you), you may do so by specifying the necessary information as part of a JSON configuration file. Furthermore, having a JSON configuration file allows you to specify whether or not Updates checks automatically or manually — you may then later toggle this setting remotely. It also possible to configure all settings programmatically.

# Check for Updates Automatically

To have Updates automatically check for new versions of your app you may configure the framework using a JSON configuration file. You need to let Updates know where to look for the file by specifying a configuration URL as follows:

```swift
let url = "https://exampledomain.com/updates.json"
Updates.configurationURL = URL(string: url)
```

Alternatively the URL may reference a local file / file in your app bundle using a file URL e.g.

```swift
Updates.configurationURL = Bundle.main.url(forResource: "Updates", withExtension: "json")
```

A simple configuration file might look as follows:

```json
{
    "updates": {
        "check-for": "automatically",
        "notify": "once"
    }
}
```

Note that Updates looks for a top-level key called `updates` which means that it is possible to add to an existing JSON file rather than creating an entirely new one.

The above configuration tells Updates to resolve all of the information needed to determine whether a new version of your app is available automatically with minimal configuration. It also indicates that users should only be notified once about a particular app update to avoid badgering them. Alternative values of this property are `twice`, `thrice`, `never` and `always`.

Having a remote JSON configuration allows for the greatest amount of flexibility once your app has been deployed as this makes it possible to switch from automatic to manual mode remotely and then provide the details of your app’s latest update yourself should you wish to.

You may forego a remote JSON configuration and simply configure Updates programmatically if you want as follows:

```swift
Updates.updatingMode = .automatically
Updates.notifying = .once
```

This is equivalent to the configuration in the above JSON snippet.

# Manually Notify Users of Updates

To manually notify users of updates to your app configure your JSON file as follows:

```json
{
    "updates": {
        "check-for": "manually",
        "notify": "always",
        "app-store-id": "123456",
        "comparing": "major-versions",
        "min-os-required": "12.0.0",
        "version": "2.0.0"
     }
}
```

- `check-for` specifies whether Updates should check for updates automatically or manually.
- The `notifying` parameter allows the developer to specify the number of times the user will be prompted to update.
- The `app-store-id` parameter specifies the numeric identifier for your app in the App Store. This parameter is only required should you wish to use the `UpdatesUI` component to present an `SKStoreProductViewController` allowing the user to update. If developing a custom UI, this parameter may be omitted.
- `comparing` determines the version number increment required for users to be notified about a notify version e.g. `major-versions` indicates that users will only be notified when the app's major version number is incremented. Other possible values here are `minor-versions` and `patch-versions`.
- The `min-os-required` property ensures that if the new version of your app does not support older versions of iOS that were previously supported then users who cannot take advantage of the update are not notified about the new version.
- The `version` property indicates the new app version available from the App Store.

If you chose not to host a remote configuration file, the same configuration may be obtained programmatically:

```swift
Updates.updatingMode = .manually
Updates.notifying = .always
Updates.appStoreId = "123456"
Updates.comparingVersions = .major
Updates.minimumOSVersion = "12.0.0"
Updates.versionString = "2.0.0"
```

# Checking For Updates

Regardless of whether you have configured Updates to check for updates automatically or manually, call `checkForUpdates` in your app to be notified of new app updates as follows:

```swift
Updates.checkForUpdates { result in
    // Implement custom UI or use UpdatesUI component
}
```

The `UpdatesUI` component described in the next section can be used in conjunction with this method call to present the App Store in-app (using either a `SKStoreProductViewController` or `SFSafariViewController` in the event that the former cannot be loaded) allowing users to update seamlessly. Alternatively you may elect to implement your own custom UI in the callback.

The callback returns an `UpdatesResult` enum value indicating whether or not an update is available:

```swift
public enum UpdatesResult {
    case available(Update)
    case none
}
```

In the case that an update is available, an `Update` value is available providing the version number of the update as well as the release notes when using automatic configuration:

```swift
public struct Update {
   public let newVersionString: String
   public let releaseNotes: String?
   public let shouldNotify: Bool
}
```

Note that the value of `notify` property in your JSON configuration is used to determine whether or not `shouldNotify` is `true` or `false`. Where writing custom UI it is up to the developer to respect the value of `shouldNotify`. If using the UpdatesUI component this property will automatically be respected.

# UI Component

The UpdatesUI component is separate from the core Updates framework to allow developers to create a custom UI if needed. For developers who do not require a custom UI, UpdatesUI makes it as simple as possible for users to update. Users will be presented a `UIAlertController` asking whether to Update or Cancel. Should the user elect to update then a `SKStoreProductViewController` will be displayed allowing the update to be initiated in-app.

In order to display the UI simply pass the `UpdatesResult` value returned from the update check to the UI as follows:

```swift
Updates.checkForUpdates { result in
    UpdatesUI.promptToUpdate(result, presentingViewController: self)
}
```

# Summary

It is prudent for developers to plan for the need to migrate users to the latest version of an app whether in the case of an emergency or as part of planned update such as dropping support for an older version of iOS. Updates is a framework designed to help developers with this task by automatically detecting where a new version of an app is available from the App Store and presenting an optional pre-configured UI component to users gently nudging them to update.

Where the choice is between developing new features to deliver value to user or developing functionality to migrate users to the latest app release, the former is likely to win out, making a migration solution with minimal development overhead beneficial.

## Challenges

- Developers cannot rollback app deployments unlike on the web. Instead, they must try to migrate users onto the latest app release in order to fix issues.
- In the event of dropping support for an older version of iOS, users of the older iOS version will not be notified of an app updates by the App Store (until they are eligible to download them through updating iOS). Developers may wish users to be aware that a new version of the app is available once they update.
- It can be cumbersome to host a backend system to let users know that an app update is available and to keep it up to date manually.

## Advantages

- Updates can automatically query whether a new version of an app is available and display a prompt to the user with minimal configuration.
- The UI component provided allows the user to seamlessly update without having to leave the app.
- Can be used to display an update message only to users who are on a device eligible to receive the update, or only to users who need to update iOS before they will be eligible to receive the update.
- Handles challenges such as querying for the correct App Store territory automatically.

<hr/>

_Updates can be found open-sourced on [GitHub](https://github.com/rwbutler/Updates) under MIT license and is compatible with both [Cocoapods](https://cocoapods.org/pods/Updates) and Carthage._

<br/>

<div align="center">
    [<img src="https://github.com/rwbutler/Updates/raw/master/docs/images/updates-logo.png" alt="Updates Logo"/>](https://github.com/rwbutler/Updates)
</div>

