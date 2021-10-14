---
layout: post
title:  "Swift: Getting The Height Of A UIImage"
date:   2019-04-21 12:00:00
color:  "SEC"
---

#### Only A Genius Could Love A Woman Like She

### What, What You Say?

Have you ever wanted to get the size of a `UIImage` contained within a `UIImageView`? It’s quite easy really, just use the following line of code:

{% highlight swift %}
let size = imageView.image?.size ?? .zero // optional for brevity
{% endhighlight %}

However, if you happen to’ve set a `contentMode` such as `.scaleAspectFill` on your `UIImageView`, the size returned upon calling the getter will not be the view’s `bounds` (the size of the image view on screen) — it will be the size of the actual image inside it. 

This can be anywhere between `.zero` and a few trillion gigapixels in size, or however big the image was which revealed the M87 Black Hole.

>Sidebar: Could an iOS device in this day and age render such an image? I still wonder...

### Scenario

- You download an image from a URL, and set it on a `UIImageView` with content mode `.scaleAspectFill`.
- This is embedded inside a `UICollectionViewCell` of size `CGSize(width: 50, height: 50)`. 

It would be ideal if we could find the relative size of the image, after the “scale” has been applied, as opposed to returning a potential image size equal to that of the planet Saturn...

{% highlight swift %}
import UIKit

extension UITraitCollection {
	var validDisplayScale: CGFloat {
		guard displayScale > 0.0 else { return UIScreen.main.scale }
		return displayScale
	}
}

extension UIImageView {
	func size(for contentMode: UIImageView.ContentMode) -> CGSize {
		switch contentMode {
		case .scaleAspectFill:
			return aspectFillSize
		case .scaleAspectFit:
			return aspectFitSize
		case .scaleToFill, .redraw:
			return self.bounds.size
		case .center,
		     .top, 
		     .left, 
		     .bottom, 
		     .right, 
		     .topLeft, 
		     .topRight, 
		     .bottomLeft, 
		     .bottomRight:
			return self.image?.size ?? .zero
		}
	}

	/// The size of the image, when the parent imageView has a contentMode of .scaleAspectFit
	/// Querying the image.size directly returns the non-scaled size. This helper property is needed for accurate results.
	private var aspectFitSize: CGSize {
		guard let image = image else { return .zero }

		var aspectFitSize = bounds.size
		let relativeWidth: CGFloat = aspectFillSize.width / image.size.width
		let relativeHeight: CGFloat = aspectFillSize.height / image.size.height

		if relativeHeight < relativeWidth {
			aspectFitSize.width = ceil(relativeHeight * image.size.width) * traitCollection.validDisplayScale
		} else if relativeWidth < relativeHeight {
			aspectFitSize.height = ceil(relativeWidth * image.size.height) * traitCollection.validDisplayScale
		}

		return aspectFitSize
	}

	/// The size of the image, when the parent imageView has a contentMode of .scaleAspectFill
	/// Querying the image.size returns the non-scaled, vastly too large size. This helper property is needed for accurate results.
	private var aspectFillSize: CGSize {
		guard let image = image else { return .zero }

		var aspectFillSize = bounds.size
		let relativeWidth: CGFloat = aspectFillSize.width / image.size.width
		let relativeHeight: CGFloat = aspectFillSize.height / image.size.height

		if relativeHeight > relativeWidth {
			aspectFillSize.width = ceil(relativeHeight * image.size.width) * traitCollection.validDisplayScale
		} else if relativeWidth > relativeHeight {
			aspectFillSize.height = ceil(relativeWidth * image.size.height) * traitCollection.validDisplayScale
		}

		return aspectFillSize
	}
}
{% endhighlight %}

_Much better!_ I divided the functions to find the scaled size between each other to make the Math clear for demonstration. But you could probably adjust it to wind up with one computed variable which covers all four conditions. Remember to K.I.S.S. - Keep It Super Simple!

### Rundown

- Given a `UICollectionViewCell` of size `CGSize(width: 100, height: 250)`
- It has a `UIImageView` of size `CGSize(width: 100, height: 100)` and takes an image downloaded from a URL, and the content mode is `.scaleAspectFill`.
- Once the image has been set, we want to calculate the relative, scaled down height of the image based on the width. The math for this is standard aspect ratio calculation.
- Given a `bounds.width` of 100, we divide 100 by the accessible size of the image itself, which, in my example is 750x750px. `100 / 750 == 0.1333333333`. Since my image view frame is square, the same applies for the height calculation.
- The width and height are now computable by multiplying the `relativeWidth` by the `image.size.height` and the `relativeHeight` by the `image.size.width`.
- New width value: `0.1333333333 * 750 == 99.999999975` or `100`, as you expect. The height is also the same in this case.
- _Disclaimer: The values will differ if the image holds an aspect ratio other than 1:1!_
- I don’t calculate the scaled size for other content modes, because when you use those, the image is set to its actual size, and positioned left/right/top/bottom etcetera.
- I use `ceil` because I want whole numbers.
- I also use `traitCollections` well, because you should be anyway but if it’s not, it falls back to `UIScreen` scale.

### Conclusion

You may wonder why I used an extension as opposed to a separate utility class. Everything is relative! You can use whichever approach you choose, as long as it works. Feel free to switch it up!
