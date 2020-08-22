---
layout: post
title:  "Get More from Codable by Implementing a JSON Key Decoding Strategy"
subtitle: "If you’re not using these, you’re probably writing more code than necessary."
---

<div align="center">
    <img src="https://github.com/rwbutler/rwbutler.github.io/raw/master/img/get-more-from-codable.jpg" alt="Prism">
</div>

_[Codable](https://developer.apple.com/documentation/swift/codable) is a protocol that was introduced in Swift 4 to make the serialization and deserialization of data into and out of Swift structures a breeze. This post assumes some knowledge of how to use Codable so if you’re unfamiliar with the basics it’s worth reading [Encoding and Decoding Custom Types](https://developer.apple.com/documentation/foundation/archives_and_serialization/encoding_and_decoding_custom_types) first as a primer._

Imagine you have the following JSON you wish to deserialize:

<script src="https://gist.github.com/rwbutler/58c0a6769abfbbd0b1228ec182f0e476.js"></script>

Here’s the series of Swift structures you might write to deserialize the data:

<script src="https://gist.github.com/rwbutler/9e8922225600f84ff6d94ce1dbbb3d88.js"></script>

Then using JSONDecoder decode the data as follows:

<script src="https://gist.github.com/rwbutler/9220a6cde820e08d2684eee943cba89d.js"></script>

Very quickly you’ll run into the following error message:

```swift
keyNotFound(CodingKeys(stringValue: “travelsOn”, intValue: nil), Swift.DecodingError.Context(codingPath: [CodingKeys(stringValue: “vehicles”, intValue: nil), _JSONKey(stringValue: “Index 0”, intValue: 0)], debugDescription: “No value associated with key CodingKeys(stringValue: \”travelsOn\”, intValue: nil) (\”travelsOn\”).”, underlyingError: nil))
```

That’s because the `JSONDecoder` is looking for the the key `travelsOn` in your JSON file. Unfortunately, the actual name of the key in the JSON file is `travels-on`.

No problem, we can solve this by implementing `CodingKeys` and custom conformance to the `Codable` protocol as follows:

<script src="https://gist.github.com/rwbutler/cd6638ddb21f062261e39b8f1a45ba16.js"></script>

We’ve implemented our own initializer and encode function in order deserialize / serialize using the JSON keys provided by our CodingKeys enum. But that’s quite a bit of extra code - had our JSON keys been in snake case as follows then it would have made life much easier:

<script src="https://gist.github.com/rwbutler/85c393a155a0272a21cb0d700cec9984.js"></script>

This is because in this instance, we could have specified a JSON key decoding strategy and avoided have to write all of that extra code:

<script src="https://gist.github.com/rwbutler/679d4f89d4d511170198bbeeb6f9b24e.js"></script>

This one line of code specifying a `keyDecodingStrategy` saved us from having to implement an initalizer, encode function and CodingKeys enum.

Unfortunately our original JSON file uses [kebab case](https://en.wikipedia.org/wiki/Letter_case#Special_case_styles) rather than snake case keys and Foundation doesn’t provide a key decoding strategy for kebab case.

However, it does provide the [custom key decoding strategy](https://developer.apple.com/documentation/foundation/jsondecoder/keydecodingstrategy/custom):

```swift
case custom(([CodingKey]) -> CodingKey)
```

This allows us the ability to implement our own custom key decoding strategy for decoding keys in whatever format we wish. All we need to do is provide an implementation of the custom key decoding strategy returning an implementation of the [CodingKey](https://developer.apple.com/documentation/swift/codingkey) protocol (we’ve named our implementation `AnyKey` following the example provided in the [Apple documentation](https://developer.apple.com/documentation/foundation/jsondecoder/keydecodingstrategy/custom)):

<script src="https://gist.github.com/rwbutler/6323a90b2a24c9b196e19de15589ef13.js"></script>

Note that the CodingKey protocol defines two [failable initializers](https://developer.apple.com/swift/blog/?id=17) which must be implemented:

```swift
init?(intValue: Int)
init?(stringValue: String)
```

However we’ve provided an additional non-failable intializer to make our implementation a little more straightforward.

When implementing a custom key decoding strategy, you must provide a closure which accepts an array of type `CodingKey. An array is passed to the closure rather than a single key in order to provide all of the ancestors of the current key providing some context for the current key to be decoded in case this affects the decoding strategy in any way. The current key will always be the last key in the array.

For example, when decoding the key `number-of-wheels` in our original JSON example, the array passed to the closure would look as follows:

```
▿ 3 elements
- 0 : CodingKeys(stringValue: "vehicles", intValue: nil)
▿ 1 : _JSONKey(stringValue: "Index 0", intValue: 0)
- stringValue : "Index 0"
▿ intValue : Optional<Int>
- some : 0
▿ 2 : _JSONKey(stringValue: "number-of-wheels", intValue: nil)
- stringValue : "number-of-wheels"
- intValue : nil
```

The last key is the actual key we are interested in however note that the previous key in the array is a JSON key with an int value of 0. This is because the `number-of-wheels` key belongs to a JSON object at position 0 in the array of vehicles objects.

You might have wondered why we needed to provided an implementation of the initializer [init?(intValue: Int)](https://developer.apple.com/documentation/swift/codingkey/2892625-init) as part of the CodingKey protocol - this is so that we are able to index arrays which aren’t usually indexed by strings.

Using our new key decoding strategy, all we need write to decode our original JSON file is:

<script src="https://gist.github.com/rwbutler/6d5bfad21d4edd62c41dbf470c257f6d.js"></script>

The new key coding strategy can be applied every time we need to use the Codable protocol to deserialize from JSON into a Swift structure from now on making it worth the initial investment of writing a custom key decoding strategy.

However, if you’d rather make use of some pre-written implementations then [LetterCase](https://github.com/rwbutler/LetterCase) provides a number of implementations for converting keys from a number of popular letter cases including kebab case, train case and macro case amongst others:

- convertFromCapitalized
- convertFromDashCase
- convertFromKebabCase
- convertFromLispCase
- convertFromLowerCase
- convertFromLowerCamelCase
- convertFromMacroCase
- convertFromScreamingSnakeCase
- convertFromTrainCase
- convertFromUpperCase
- convertFromUpperCamelCase

[LetterCase](https://github.com/rwbutler/LetterCase) is an open-source framework available under MIT license compatible with [Cocoapods](https://cocoapods.org/pods/LetterCase), Carthage and [Swift Package Manager](https://swiftpackageindex.com/rwbutler/LetterCase).

It also provides an implementation for converting from one letter case to another therefore you if you wanted to decode the original JSON file:

<script src="https://gist.github.com/rwbutler/58c0a6769abfbbd0b1228ec182f0e476.js"></script>

Into variables of macro case e.g.

<script src="https://gist.github.com/rwbutler/5bf0d2b2d9450cd41a4ecdbb5f8e32cd.js"></script>

Then this could be achieved as follows:

<script src="https://gist.github.com/rwbutler/9c0e7b84d6d9d25f5f042a6dd655d8c2.js"></script>

## Summary

Hopefully this article has shown that by spending a little bit of time upfront creating a custom JSON key decoding strategy, a lot of time can be saved further down the road by reusing the same strategies each and every time you need to decode JSON keys of a different letter case.

<hr/>

_LetterCase can be found open-sourced on [GitHub](https://github.com/rwbutler/LetterCase) under MIT license and is compatible with both [Cocoapods](https://cocoapods.org/pods/LetterCase), Carthage and [Swift Package Manager](https://swiftpackageindex.com/rwbutler/LetterCase)._