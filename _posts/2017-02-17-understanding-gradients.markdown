---
layout: post
author: Ben
title:  "TIL: Understanding CAGradientLayer"
date:   2017-02-17 00:00:00
color:  "QUIN"
---

If there’s one API that irritated me at the beginning of my career as an iOS developer, it was `CAGradientLayer`. As powerful as it is, it’s very verbose and it can be really confronting for a newbie.

Nevertheless, the time came where I had to draw a gradient in an app. No images. I turned to `CAGradientLayer` because I already understood it. Unfortunately it confused my team at the time, who were less familiar with it.

A gradient will be drawn between 0 and 1, on both the X (horizontal) and Y (vertical) axes. 

This is hard to explain, so let’s look at the image below.

<img src="{{site.baseUrl}}/assets/img/plane.png"/>

A `CALayer` gradient uses these points. I sometimes see people printing or whiteboarding this, then drawing on it to assist with planning. As we _all_ know, it sucks to make tiny UI changes and rebuild our Xcode projects due to generally long build times.

So I built a wrapper tool back in 2017 to assist in building `CAGradientLayer` objects.

### The Code

{% highlight swift %}
extension CAGradientLayer {
  enum Point {
    case topLeft
    case centerLeft
    case bottomLeft
    case topCenter
    case center
    case bottomCenter
    case topRight
    case centerRight
    case bottomRight

    var point: CGPoint {
      switch self {
      case .topLeft:
        return CGPoint(x: 0, y: 0)
      case .centerLeft:
        return CGPoint(x: 0, y: 0.5)
      case .bottomLeft:
        return CGPoint(x: 0, y: 1.0)
      case .topCenter:
        return CGPoint(x: 0.5, y: 0)
      case .center:
        return CGPoint(x: 0.5, y: 0.5)
      case .bottomCenter:
        return CGPoint(x: 0.5, y: 1.0)
      case .topRight:
        return CGPoint(x: 1.0, y: 0.0)
      case .centerRight:
        return CGPoint(x: 1.0, y: 0.5)
      case .bottomRight:
        return CGPoint(x: 1.0, y: 1.0)
      }
    }
  }

  convenience init(start: Point, end: Point, colors: [CGColor], type: CAGradientLayerType) {
    self.init()
    self.type = type
    self.startPoint = start.point
    self.endPoint = end.point
    self.colors = colors
    self.locations = (0..<colors.count).map(NSNumber.init)
  }
}
{% endhighlight %}

This wrapper is merely a convenience function around the `CGPoint()` nastiness that we usually have to deal with.

To use it; all we need is:

{% highlight swift %}
let gradient = CAGradientLayer(start: .center, end: .bottomRight, colors: [UIColor.black.cgColor, UIColor.white.cgColor], type: .radial)
gradient.frame = view.bounds
view.layer.addSublayer(gradient)
{% endhighlight %}

Make sure the view you're applying the gradient to has a frame (or bounds) greater than zero, and it will render fine!

---

Easy, no?
