---
layout: post
title:  "Solving the Captive Portal Problem on iOS"
subtitle: "Making Internet connectivity detection more robust by detecting Wi-Fi networks without Internet access."
---

<div align="center">
    <img src="https://github.com/rwbutler/rwbutler.github.io/raw/master/img/solving-the-captive-portal-problem-on-ios.png" alt="Globe at Night">
</div>


In iOS development, the de facto means of detecting Internet connectivity has been to make use of Apple’s [Reachability sample code](https://developer.apple.com/library/archive/samplecode/Reachability/Introduction/Intro.html). However Reachability cannot actually detect whether connectivity is present, _only that an interface is available that might allow a connection_.

Consider the case of an app user making use of a public Wi-Fi hotspot which requires the user to register or agree to terms of service via a [captive portal](https://en.wikipedia.org/wiki/Captive_portal) prior to Internet connectivity being established e.g. at your local Starbucks branch. The device will appear to have connected to a Wi-Fi network but any request for data will fail until the user has agreed to the Wi-Fi hotspot’s terms of service or registered as a new user — depending on the requirements of the hotspot. Reachability under these circumstances will return a response indicating that Wi-Fi is available even though true Internet connectivity is in fact unavailable.

This can result in confusion as your app will behave as though it is online whilst connected to such a hotspot since Reachability checks will indicate the presence of a Wi-Fi connection. Meanwhile, attempts made by your app to retrieve data from the Internet will fail. This in turn may translate to poor reviews on the App Store.

# How iOS Solves the Captive Portal Issue

So how do we go about ensuring that our app has true Internet connectivity? As it turns out iOS already has a solution to this problem.

iOS adopts a protocol called Wireless Internet Service Provider roaming ([WISPr 2.0](https://wballiance.com/glossary/)) published by the [Wireless Broadband Alliance](https://wballiance.com). This protocol defines the Smart Client to Access Gateway interface describing how to authenticate users accessing public IEEE 802.11 (Wi-Fi) networks using the [Universal Access Method](https://en.wikipedia.org/wiki/Universal_access_method) in which a captive portal presents a login page to the user.

The user must then register or provide login credentials via a web browser in order to be granted access to the network using [RADIUS](https://www.cisco.com/c/en/us/support/docs/security-vpn/remote-authentication-dial-user-service-radius/12433-32.html) or another protocol providing centralized Authentication, Authorization, and Accounting ([AAA](https://en.wikipedia.org/wiki/AAA_(computer_security))).

In order to detect a that it has connected to a Wi-Fi network with a captive portal, iOS contacts a number of endpoints hosted by Apple — an example being [https://www.apple.com/library/test/success.html](https://www.apple.com/library/test/success.html). Each endpoint hosts a small HTML page of the form:

```html
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2//EN">
<HTML>
<HEAD>
	<TITLE>Success</TITLE>
</HEAD>
<BODY>
	Success
</BODY>
</HTML>
```

If on downloading this small HTML page iOS finds that it contains the word `Success` as above then it knows that Internet connectivity is available. However, if a login page is presented by a captive portal then the word `Success` will not be present and iOS will realize that the network connection has been hijacked by a captive portal and will present a browser window allowing the user to login or register.

Apple hosts a number of these pages such that should one of these pages go down, a number of fallbacks can be checked to determine whether connectivity is present or whether our connection is blocked by the presence of a captive portal. Unfortunately iOS exposes no framework to developers which allows us to make use of the operating system’s awareness of captive portals.

Connectivity is an open-source framework available under MIT license which wraps Reachability and endeavors to replicate iOS’s means of detecting captive portals. When Reachability detects Wi-Fi or WWAN connectivity, Connectivity contacts a number of endpoints to determine whether true Internet connectivity is present or whether a captive portal is intercepting the connections. This approach can also be used to determine whether an iOS device is connected to a Wi-Fi router with no Internet access.

Connectivity provides an interface as close to Reachability as possible so that it is familiar to developers used to working with Reachability. This includes providing the methods `startNotifier()` and `stopNotifier()` to begin checking for changes in Internet connectivity. Once the notifier has been started, you may query for the current connectivity status synchronously using the status property (similar to Reachability’s `currentReachabilityStatus`) or asynchronously by registering as an observer with the default `NotificationCenter` for the notification `kNetworkConnectivityChangedNotification` (in Swift this is accessed via `Notification.Name.ConnectivityDidChange`) — similar to Reachability’s notification `kNetworkReachabilityChangedNotification`.

By default, Connectivity contacts a number of endpoints already used by iOS but it recommended that these are supplemented by endpoints hosted by the developer by appending to the `connectivityURLs` property. Further customization is possible through setting the `successThreshold` property which determines the percentage of endpoints contacted which must result in a successful check in order to conclude that connectivity is present. The default value specifies that 75% of URLs contacted must result in a successful connectivity check.

# Usage

To get started using Connectivity, simply instantiate an instance and assign a closure to be invoked when Connectivity detects that you are connected to the Internet, when disconnected, or in both cases as follows:

```swift
let connectivity: Connectivity = Connectivity()
let connectivityChanged: (Connectivity) -> Void = { [weak self] connectivity in
     self?.updateConnectionStatus(connectivity.status)
}
connectivity.whenConnected = connectivityChanged
connectivity.whenDisconnected = connectivityChanged
func updateConnectionStatus(_ status: Connectivity.ConnectivityStatus) {
    switch status {
	    case .connectedViaWiFi:
	    case .connectedViaWiFiWithoutInternet:
	    case .connectedViaWWAN:
	    case .connectedViaWWANWithoutInternet:
	    case .notConnected:
    }
        
}
```

Then to start listening for changes in Connectivity call:

```swift
connectivity.startNotifier()
```

Remember to call `connectivity.stopNotifier()` when you are done.

# One-Off Checks

Sometimes you only want to check the connectivity state as a one-off. To do so, instantiate a Connectivity object then check the status property as follows:

```swift
let connectivity = Connectivity()
switch connectivity.status {
	case .connectedViaWiFi:
	
	case .connectedViaWiFiWithoutInternet:
	
	case .connectedViaWWAN:
	
	case .connectedViaWWANWithoutInternet:
	
	case .notConnected:
	
}
```

Alternatively, you may check the following properties of the `Connectivity` object directly if you are only interested in certain types of connections:

```swift
var isConnectedViaWWAN: Bool
var isConnectedViaWiFi: Bool
    
var isConnectedViaWWANWithoutInternet: Bool
var isConnectedViaWiFiWithoutInternet: Bool
```

# Connectivity URLs

It is possible to set the URLs which will be contacted to check connectivity via the `connectivityURLs` property of the Connectivity object before starting connectivity checks with `startNotifier()`.

```swift
connectivity.connectivityURLs = [URL(string: “https://www.apple.com/library/test/success.html")!]
```

# Notifications

If you prefer using notifications to observe changes in connectivity, you may add an observer on the default NotificationCenter:

[`NotificationCenter.default.addObserver(_:selector:name:object:)`](https://developer.apple.com/documentation/foundation/notificationcenter/1415360-addobserver)

Listening for `Notification.Name.ConnectivityDidChange`, the object property of received notifications will contain the `Connectivity` object which you can use to query connectivity status.

# Polling

In certain cases you may need to be kept constantly apprised of changes in connectivity state and therefore may wish to enable polling. Where enabled, Connectivity will not wait on changes in Reachability state but will poll the connectivity URLs every 10 seconds (this value is configurable). `ConnectivityDidChange` notifications will be emitted and `whenConnected` / `whenDisconnected` closures will be invoked only where changes in connectivity state occur.

To enable polling:

```swift
connectivity.isPollingEnabled = true
connectivity.startNotifier()
```

# SSL

As of Connectivity 1.1.0, using HTTPS for connectivity URLs is the default setting. If your app doesn’t make use of [App Transport Security](https://developer.apple.com/security/) and you wish to make use of HTTP URLs as well as HTTPS ones then either set `isHTTPSOnly` to `false` or set `shouldUseHTTPS` to `false` when instantiating the Connectivity object as follows*:

```swift
let connectivity = Connectivity(shouldUseHTTPS: false)
```

*Note that the property will not be set if you have not set the `NSAllowsArbitraryLoads` flag in your app's Info.plist first.

# Threshold

To set the number of successful connections required in order to be deemed successfully connected, set the `successThreshold` property. The value is specified as a percentage indicating the percentage of successful connections i.e. if four connectivity URLs are set in the `connectivityURLs` property and a threshold of 75% is specified then three out of the four checks must succeed in order for our app to be deemed connected:

```swift
connectivity.successThreshold = Connectivity.Percentage(75.0)
```

# Summary

Connectivity emulates iOS’s method of detecting captive portals and exposes this functionality to the developer with a familiar interface. Being able to reliably detect more nuanced situations such as where an iOS device is connected to a Wi-Fi router without Internet access allows app developers to provide users with better information and to develop more robust online functionality.

<hr/>

_Connectivity can be found open-sourced on [GitHub](https://github.com/rwbutler/Connectivity) under MIT license and is compatible with both [Cocoapods](https://cocoapods.org/pods/Connectivity) and Carthage._

<div align="center">
    <img width="150" height="150" src="https://github.com/rwbutler/Connectivity/raw/master/docs/images/connectivity-logo.png" alt="Connectivity Logo">
</div>

_If you found this article interesting, a subsequent article [Detecting Internet Access on iOS 12+](https://rwbutler.github.io/2018-12-26-detecting-internet-access-on-ios-12) explains how the Network framework introduced in iOS 12 can be used in place of Reachability._