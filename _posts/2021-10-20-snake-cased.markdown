---
layout: post
author: Ben
title:  "Swift: camelCase enums in snake_Case"
date:   2021-10-20 00:00:00
color:  "SEC"
---

We've all defined enums in our code, where we've had to conform it to `String` and manually defined the `rawValue` to give it a `SNAKE_CASE` value. But what if we could rely on protocols to do it for us?

### Define The Protocol

Much like `CustomStringConvertible`, we will name our protocol `SnakeCaseConvertible`. The name, being one of the three hard-to-solve problems in computer science, should hopefully indicate that the cases defined within are... convertible to snake case!

{% highlight swift %}
protocol SnakeCaseConvertible {
    associatedtype RawValue

    var rawValue: String { get }
}
{% endhighlight %}

Now this is defined, we'll extend it with a _default_ implementation. This implementation will always be called unless we override it manually.

{% highlight swift %}
extension SnakeCaseConvertible where RawValue == String {
    private func converted() -> String {
        guard let regex = try? NSRegularExpression(pattern: "(\\b[a-z]+|\\G(?!^))((?:[A-Z]|\\d+)[a-z]*)", options: []) else {
            assertionFailure("Invalid NSRegularExpression pattern.")
            return String(describing: self)
        }

        let describingString = String(describing: self)
        let range = NSRange(location: 0, length: describingString.count)
        return regex.stringByReplacingMatches(in: describingString, options: [], range: range, withTemplate: "$1_$2")
    }

    var rawValue: RawValue {
        return converted()
    }
}
{% endhighlight %}

### Finally

Given a bog-standard enum:

{% highlight swift %}
enum MyEnum: String, SnakeCaseConvertible {
  case thatOneTimeAtBandcamp
  case iKnowWhatYouDidLastSummer
}
{% endhighlight %}

You can call `.rawValue` as normal, but this time the result will be in `snake_case`!
