---
layout: post
title:  "Detecting Internet Access on iOS 12+"
subtitle: "iOS 12 introduced the Network framework including NWPathMonitor, a replacement for Reachability."
---

<div align="center">
    <img src="https://github.com/rwbutler/rwbutler.github.io/raw/master/img/detecting-internet-access-on-ios-12.png" alt="Electrical Conductivity">
</div>

_I recently wrote an article on how iOS detects captive portals on connecting to new Wi-Fi networks before presenting a web view allowing the user to sign in or register. This is a scenario that will be familiar to most people who have connected to a public Wi-Fi network in a hotel, bar or coffee shop etc. If you aren’t yet familiar with how this works then [Solving the Captive Portal Problem on iOS](https://rwbutler.github.io/2018-11-16-solving-the-captive-portal-problem-on-ios/) provides a useful background to this article._

Apple’s [Reachability sample code](https://developer.apple.com/library/archive/samplecode/Reachability/Introduction/Intro.html) has been used as the de facto starting point for detecting network access in third party iOS apps for some years. A quick search on [Cocoapods.org](https://cocoapods.org/pods/Reachability) will reveal a long list of libraries which have rewritten this code with a number of considerations such as ARC support or Swift compatibility in mind.

WWDC in June 2018 [introduced the Network framework](https://developer.apple.com/videos/play/wwdc2018/715/) available from iOS 12 onwards which includes the [NWPathMonitor](https://developer.apple.com/documentation/network/nwpathmonitor) class. This class provides us with a means of monitoring changes in network state without having to include a third party library / Apple sample code.

# Usage

In order to make use of the `NWPathMonitor` class simply import the Network framework and then create a `NWPathMonitor` instance:

```swift
let monitor = NWPathMonitor()
```

If you are only interested in state changes in a particular network adapter e.g. Wi-Fi, then it is possible to specify a network adapter to monitor on instantiating the `NWPathMonitor `object using the [`init(requiredInterfaceType:)`](https://developer.apple.com/documentation/network/nwpathmonitor/2998734-init) initializer and providing a [`NWInterface.InterfaceType`](https://developer.apple.com/documentation/network/nwinterface/interfacetype) as the parameter e.g.

```swift
let monitor = NWPathMonitor(requiredInterfaceType: .wifi)
```

You need to be sure to retain the reference to this object somewhere (such as using a strong property) otherwise you may find that the callback you assign stops getting called as ARC releases the `NWPathMonitor` object.

Interface types which can be monitored include:

- `cellular`
- `loopback`
- `other` (for virtual or undetermined network types)
- `wifi`
- `wiredEthernet`

To be notified of state changes you need to assign a callback to the `pathUpdateHandler` property which will be invoked whenever a state change occurs in the network interface e.g. your phone moves from a cellular to a Wi-Fi network. Then whenever a state change occurs, a `NWPath` instance will be returned which can be queried to determine whether or not we are connected as follows:

```swift
monitor.pathUpdateHandler = { path in
    if path.status == .satisfied {
        print("Connected")
    }
}
```

Using the no-argument initializer vs using the initializer which specifies a network adapter affects whether or not the `status` property of the `NWPath` object returned is `satisfied`. For example, should you choose to monitor the cellular network adapter but a state change in the Wi-Fi adapter occurs (e.g your phone connects to a Wi-Fi network) then your callback will not be invoked and the path’s status will remain `unsatisfied` (`NWPathMonitor`’s path can be accessed at any time using the `currentPath` property) because the device is not connected using the specified interface. So if you only wish to know whether a connection is present or not, regardless of whether it’s Wi-Fi or cellular, then it’s best to stick with the no-argument initializer.

As an interesting aside — whilst the `NWPath` object is new to iOS 12 as part of the Network framework, it has in fact been available since iOS 9 as part of the [NetworkExtension.framework](https://developer.apple.com/documentation/networkextension/nwpath) (with some minor differences).

The `NWPath` object returned can be interrogated to find out a lot about the state of the device’s network adapters. One of the more interesting properties is `isExpensive` which returns whether not [the network interface is considered expensive](https://developer.apple.com/documentation/networkextension/nwpath/1406899-isexpensive) to use e.g. a cellular data plan. We can also find out whether the path supports DNS, IPv4 or IPv6. Should we need to find out which interface changed state and triggered our callback we can invoke the `usesInterfaceType` method:

```swift
let isCellular: Bool = path.usesInterfaceType(.cellular)
```

Using `NWPathMonitor` is somewhat similar to using other iOS APIs such as [CLLocationManager](https://developer.apple.com/documentation/corelocation/cllocationmanager) in that we need to call a start method in order to start receiving updates and then call a stop counterpart when we are done with it. The `start` method for `NWPathMonitor` requires that we provide a queue for the object to carry out its work:

```swift
let queue = DispatchQueue.global(qos: .background)
monitor.start(queue: queue)
```

When we are done monitoring changes in state, we simply call `cancel()` on it. Note that at the present time once we have called `cancel` on our `NWPathMonitor` we cannot start it again. Instead, we need to instantiate a new `NWPathMonitor` instance.

Note that `NWPathMonitor`'s `currentPath` property will be `nil` if accessed before start is invoked. In fact, if you print the path returned to your update closure as in:

```swift
monitor.pathUpdateHandler = { path in
    print(path)
}
```

Then something like the following will be printed:

```
Optional(satisfied (Path is satisfied), interface: en0, scoped, ipv4, ipv6, dns)
```

This indicates that the `NWPath`s returned here and by the `currentPath` property are optionals although the API does not indicate them as such (we can infer that the `NWPath` reference returned is an Objective-C pointer bridged to Swift).

# Captive Portals

A captive portal is a web page displayed on connection to a public Wi-Fi hotspot typically used to force login, registration or payment prior to Internet access (or access to other network resources) being granted. In a [previous blog post](https://rwbutler.github.io/2018-11-16-solving-the-captive-portal-problem-on-ios/), I wrote about how from the perspective of an app developer using Reachability it can appear as though Internet access is available where in fact, it is not because a captive portal is being served. This can cause apps to behave incorrectly and even crash — perhaps because the app was expecting some JSON from a RESTful API but in fact received some HTML from a captive portal.

I was intrigued to see whether `NWPathMonitor` improves on `Reachability` in terms of detecting true Internet connectivity. The `NWPath.Status` enum does offer three cases - `satisfied`, `unsatisfied` and `requiresConnection`. Unfortunately whilst the developer documentation doesn’t shed any light on the usage of each of these cases if we look at the older NetworkExtension framework, the original `NWPathStatus` object provides the case [`satisfiable`](https://developer.apple.com/documentation/networkextension/nwpathstatus/satisfiable) which does have some documentation:

_The path is not currently satisfied, but may become satisfied upon a connection attempt. This can be due to a service, such as a VPN or a cellular data connection not being activated._

The `requiresConnection` case would appear to be the analogue of `satisfiable` in the older `NWPathStatus` object.

The good news is that `NWPathMonitor` typically only reports the path as being satisfiable after negotiating the captive portal i.e. after the web view has been presented and the user has logged in etc. Where there is no captive portal to present to the user an action sheet is presented offering the options _Use Without Internet_ or _Use Other Network_. If the user elects to _Use Without Internet_ then the path status returned by `NWPathMonitor` will be `satisfied` even though there is no Internet access.

<br/>

<div align="center">
    <img src="https://github.com/rwbutler/rwbutler.github.io/raw/master/img/connectivity-screenshot.png" alt="Wi-Fi network not connected to the Internet screenshot">
</div>

Through some experimentation with [Charles](https://www.charlesproxy.com) proxy I found that unless _Use Without Internet_ was selected then `NWPathMonitor` correctly did not report a `NWPath`’s `Status` as being satisfied on initial connection to a Wi-Fi network whilst I was blocking the connection. However, if Internet connectivity was restored but then subsequently taken away then this was not detected and the path status did not change from `satisfied`. This could occur for example, should a user only pay for an hour’s worth of Internet access on a train or at a hotel.

# Connectivity

Connectivity is an open-source framework available under MIT license which endeavors to replicate iOS’s inherent means of detecting captive portals. It allows true Internet connectivity to be detected accurately on iOS 8+ using Reachability meaning that we can use this to detect the availability of Internet access where `NWPathMonitor` is not available and on iOS 12, Connectivity can be used to provide a greater level of accuracy over using `NWPathMonitor` alone.

Recently support for `NWPathMonitor` has been added to Connectivity where iOS 12 or above is available. If the `framework` property is set to `network` then the new Network framework will be used in place of the SystemConfiguration framework (Reachability) for observing state changes in network adapters.

```swift
let connectivity = Connectivity()
connectivity.framework = .network
```

Following a state change in a network adapter, Connectivity performs a number of checks to determine whether Internet access is available. A polling option is available which can be used to revalidate whether Internet access is available every so often even where a state change has not occurred — should this be required. This can be achieved by setting `isPollingEnabled = true` and specifying an appropriate value in seconds for the `pollingInterval`.

# Summary

The Network framework introduces some great new classes including `NWPathMonitor` which can be used to observe state changes in a device’s network adapters on iOS 12 and above. It reports a path as `satisfied` after a captive portal has been interacted with by the user but doesn’t detect the subsequent loss of Internet access. [Connectivity](https://github.com/rwbutler/Connectivity) can be used to provide backwards compatibility for apps supporting previous versions of iOS and greater accuracy over using `NWPathMonitor` alone.

## Advantages

- Officially supported by Apple.
- No need to include third party code — simply link Network.framework in iOS 12.
- Reports an `NWPath`’s `Status` as being `satisfied` after having negotiated the captive portal.

## Disadvantages

- Cannot be used prior to iOS 12 which means that if you need to support earlier versions of iOS you’re out of luck.
- Lack of documentation — take a look at [https://developer.apple.com/documentation/network/nwpathmonitor](https://developer.apple.com/documentation/network/nwpathmonitor) for example.
- Doesn’t detect captive portals and other situations where Internet connectivity drops out following an initial successful connection.

<hr/>

_Connectivity can be found open-sourced on [GitHub](https://github.com/rwbutler/Connectivity) under MIT license and is compatible with both [Cocoapods](https://cocoapods.org/pods/Connectivity) and Carthage._

<br/>

<div align="center">
    [<img src="https://github.com/rwbutler/Connectivity/raw/master/docs/images/connectivity-logo.png" alt="Connectivity Logo"/>](https://github.com/rwbutler/Connectivity)
</div>
