---
layout: post
title:  "Swift: Custom Fonts. Less Awful."
date:   2016-09-04 12:00:00
color:  "SEC"
---

#### I'm Holding Out For A UIFont Till The Morning Light

### 2021: Updated for Swift 5!

Neatly managing custom fonts on iOS is _still a pain_, even in 2021. Furthermore, with WCAG catching up with apps in recent years, you need to make sure that your fonts (and the colours you use with them!) are compliant with accessibility standards. This last bit is especially important if you’re working on a large scale application, with a substantial number of customers. Not everyone is perfectly sighted - myself included.

Even if you only have one font in your design system, having to plop `UIFont(name:)` constructors all over your code is… not scalable. You can also probably clone fifty repos and find fifty different solutions to this problem, which makes it hard to know if yours is “right.”

In September of 2016, while I was still learning Swift, I came up with a solution that still works in 2021! Even I'm surprised by this, given Swift's propensity for breaking changes between versions.

---

Actually getting custom fonts to _work_ in an iOS project is a piece of cake. 

- Check the licence for the font, and then copy the files into your project.
- Add the filenames to the “Fonts Provided by Application” key/value pair in your `Info.plist`. 
- Check that they’re added to the target, and that they’re bundled as resources.

Some fonts are already provided with iOS, so you can skip the plist or font register stage if it already comes with the system.

Another approach is to utilize `CoreText`, and register the fonts to your process and avoid touching the `Info.plist` entirely.

{% highlight swift %}
import CoreText

enum Loader {
    static func load() {
        if let fontURLs = Bundle.main.urls(forResourcesWithExtension: "ttf", subdirectory: nil) {
            if #available(iOS 13.0, \*) {
                CTFontManagerRegisterFontURLs(fontURLs as CFArray, .process, true, nil)
            } else {
                CTFontManagerRegisterFontsForURLs(fontURLs as CFArray, .process, nil)
            }
        }
    }
}
{% endhighlight %}

Feel free to name this `enum` to something other than `Loader`. I've used `enum` here so it has no initializer.

You can then call `Loader.load()` where appropriate to have your fonts loaded and accessible. Perhaps somewhere in your AppDelegate?

---

Remember that the system font on iOS has been designed to be accessible on the displays it calls home. It bears repeating that this is not the case for every font out there. Please choose carefully!

Also, it’s almost as if `UIFont` doesn’t want you to use custom fonts, because you’re forced to construct it like this:

{% highlight swift %}
myLabel.font = UIFont(name: "SuperAwesomeFont-SemiboldItalic", size: 16.83")
{% endhighlight %}

Whenever you invoke the above, you’ll need to have the font's PostScript name handy. Humans are dumb, and typos happen. If you make a mistake, iOS will fallback to the system font. An eagle-eyed designer will spot this kind of mistake quicker than blinking, so make sure you double check 😆.

### The Constants File Approach

You’ve got one. I’ve got one. Almost every iOS dev I’ve ever met has used one. It’s a `Constants` file! It’s a useful solution that isn’t over-engineered. For fonts, you’d typically have it setup like so:

{% highlight swift %}
enum Constants { // not struct to avoid an initializer
  enum Fonts {
    static let regular = UIFont(name: “Font-Regular”, size: 14)
  }
}
{% endhighlight %}

…rinse and repeat for variations in weight and size. Invoking a font declared this way usually looks like this:

{% highlight swift %}
myLabel.font = Fonts.regular
{% endhighlight %}

There’s nothing wrong with this! At all! However, it can become messy if you juggle multiple fonts and multiple sizes.

{% highlight swift %}
enum Constants {
  enum Fonts {
    static let regular = UIFont(name: “Font-Regular”, size: 14)
    static let regular16 = UIFont(name: “Font-Slim”, size: 16)
    static let regular20 = UIFont(name: “Font-KindaRegular”, size: 20)
    static let regularSendHelp72 = UIFont(name: “Font-NotRegular”, size: 72)
  }
}
{% endhighlight %}

Imagine this being used for a dozen sizes? How about a dozen weights, and a dozen sizes per weight!

We can make this better. If you’re operating on a small scale, don’t assume that this is a bad approach, because it’s not.
Although it’s still worth looking at other solutions, whichever one that works for you is the best one.

### The Extension Method

Extensions are to Swift what categories are to Objective-C. They do what they say on the tin — they extend classes to add extra/bespoke functionality.

Let’s go ahead and extend `UIFont` with our custom font instead:

{% highlight swift %}
extension UIFont {
  class func regular(at size: CGFloat) -> UIFont {
    return UIFont(name: “Font-Regular”, size: size)
  }
}
{% endhighlight %}

With this approach, we can simply add more functions for each **weight** required, eliminating duplicates but keeping the flexibility of **size**.
We _could_ finish here… or we could push the pedal to the metal and build something totally _bonkers_.

### Swiftiest Swifty Swift!

We’re using `enum` again. 

{% highlight swift %}
import UIKit
import CoreText

enum Fonts: String {
  case regular = "AvenirNext-Regular"
  case medium = "AvenirNext-Medium"
  case mediumItalic = "AvenirNext-MediumItalic"
  case demiBold = "AvenirNext-DemiBold"
  
  func of(_ style: DynamicType) -> UIFont {
    let textStyle = UIFont.TextStyle(rawValue: style.rawValue)
    let metrics = UIFontMetrics(forTextStyle: textStyle)
    guard let size = fontSizes[textStyle], 
    let font = UIFont(name: self.rawValue, size: size) else {
      // If we can't load the size, or the font, CRASH. Something's wrong...
      fatalError()
    }
    
    return metrics.scaledFont(for: font)
  }
  
  static func loadFonts() {
    if let fontURLs = Bundle.main.urls(forResourcesWithExtension: "ttf", subdirectory: nil) {
      if #available(iOS 13.0, *) {
        CTFontManagerRegisterFontURLs(fontURLs as CFArray, .process, true, nil)
      } else {
        CTFontManagerRegisterFontsForURLs(fontURLs as CFArray, .process, nil)
      }
    }
  }
}

extension Fonts {
  enum DynamicType: String {
    case largeTitle = "UICTFontTextStyleLargeTitle"
    case title1 = "UICTFontTextStyleTitle1"
    case title2 = "UICTFontTextStyleTitle2"
    case title3 = "UICTFontTextStyleTitle3"
    case headline = "UICTFontTextStyleHeadline"
    case body = "UICTFontTextStyleBody"
    case callout = "UICTFontTextStyleCallout"
    case subheadline = "UICTFontTextStyleSubheadline"
    case footnote = "UICTFontTextStyleFootnote"
    case caption1 = "UICTFontTextStyleCaption1"
    case caption2 = "UICTFontTextStyleCaption2"
  }

  private var fontSizes: [UIFont.TextStyle: CGFloat] {
    [
      .largeTitle: 34,
      .title1: 28,
      .title2: 22,
      .title3: 20,
      .headline: 17,
      .body: 17,
      .callout: 16,
      .subheadline: 15,
      .footnote: 13,
      .caption1: 12,
      .caption2: 11
    ]
  }
}
{% endhighlight %}

This single `enum` can contain all the fonts you will need to use in your app. It even supports dynamic font sizing! Just add a case for each font you want to use, give it the appropriate PostScript name, and invoke it like so:

{% highlight swift %}
myLabel.font = Fonts.regular.of(.body)
{% endhighlight %}

---

Cool, right?
