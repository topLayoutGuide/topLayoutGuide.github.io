---
layout: post
author: Ben
title:  "Swift: Marquee Label"
date:   2021-10-28 00:00:00
color:  "SEC"
---

There come times when building our apps, where we need a label to either _wrap_ its text, or become a marquee in order to stop truncation of text. The latter effect is quite rare, but can be found in apps such as PayPay (Japan) while using their P2P payment system.

A few years ago, I needed this effect. So I built a `UIScrollLabel` - _10 points for an inventive title_ - and put it on Github. I looked back upon it today, and realised that I thoroughly overcomplicated my solution.

### Overcomplication

My original solution had a simple structure. It is a `UIScrollView` containing two `UILabel` with padding between them. It had some other configurable properties, and it utilised a [CADisplayLink](https://developer.apple.com/documentation/quartzcore/cadisplaylink) refreshing at the cadence of the display to scroll.

The drawback of `CADisplayLink` is simple - it refreshes at a minimum of 60 times _per second_, or whatever refresh rate the display uses. It was overkill.

The main reason I used `CADisplayLink` was to achieve the same sort of effects I did with my `UIAnimatedLabel`. I wanted the `.easeIn`, and `.easeOut` (et al) animation curves available to me.

### Today

I had a task at work which required me to fish out this code from my repository, and reimplement it. Revisiting the old code, as a more ~jaded~ wizened iOS engineer, I realised I could drastically trim it down. 

I could keep the same structure - a `UIScrollView` comprising of two `UILabel` objects. I could also keep some customisation properties available too. 

But the biggest cutback would be removing `CADisplayLink` and using `UIView.animate()` to get the job done.

### Setup

Let's go ahead and make a new file in a sample project called `MarqueeLabel`. Inside it, put the following:

{% highlight swift %}
private var labels: [UILabel] = []

var text: String = "" {
    didSet {
        configure()
    }
}

var textColor: UIColor = .white {
    didSet {
        labels.forEach {
            $0.textColor = textColor
        }
    }
}

var font: UIFont = UIFont.boldSystemFont(ofSize: 14) {
    didSet {
        labels.forEach {
            $0.font = font
        }
    }
}
{% endhighlight %}

Remember that we're faking our marquee effect by using a `UIScrollView` for the container. We want our child labels to be `private`, and only accessible to us. So, we need to define some common `UILabel` properties on our `MarqueeLabel`, which `UIScrollView` types do not normally have, such as `font`, `textColor`, and `text`.

Let's define some more properties:

{% highlight swift %}
var scrollDelay: TimeInterval = 1 // seconds

var duration: TimeInterval = 6 // seconds

var spaceBetweenLabels: CGFloat = 50 // points
{% endhighlight %}

These properties are pretty self-explanatory, right? The first controls how long the animation ought to wait before repeating; the second controls how long we should animate for; and the third one controls the space between the labels.

### Implementation

Now that all the properties have been defined, let's actually get it working. To do that, first we need to do some setup. The illusion will be spoiled if the user can see the scroll bar moving during the animation, or if they can interact with the scroll view as it's already animating.

{% highlight swift %}
required init?(coder aDecoder: NSCoder) {
    super.init(coder: aDecoder)
    commonInit()
}

override init(frame: CGRect) {
    super.init(frame: frame)
    commonInit()
}

private func commonInit() {
    showsVerticalScrollIndicator = false
    showsHorizontalScrollIndicator = false
    isScrollEnabled = false
    isUserInteractionEnabled = false
    backgroundColor = .clear
    clipsToBounds = true

    addLabels()
    configure()
}
{% endhighlight %}

Next, we need to implement the `addLabels()` and `configure()` functions, which are obviously missing:

{% highlight swift %}
func stop() {
    contentOffset = .zero
    layer.removeAllAnimations()
}

private func configure() {
    // Stop all existing operation
    stop()

    // Relayout the labels
    layoutLabels()

    // Ensure the contentSize is accurate
    configureContentSize()
}

private func addLabels() {
    for _ in 0..<2 {
        let label: UILabel = UILabel()
        label.textColor = textColor
        label.numberOfLines = 1
        addSubview(label)
        labels.append(label)
    }
}

private func layoutLabels() {
    var offset: CGFloat = 0

    labels.forEach {
        $0.text = text
        $0.font = font
        $0.sizeToFit()
        $0.frame = CGRect(x: offset, y: 0, width: $0.frame.width, height: bounds.height)
        $0.center = CGPoint(x: $0.center.x, y: round(center.y - frame.minY))

        offset += round($0.bounds.width) + spaceBetweenLabels
    }
}

private func configureContentSize() {
    guard let mainLabel = labels.first else { return }

    var size: CGSize = .zero
    size.width = mainLabel.bounds.width + bounds.width + spaceBetweenLabels
    size.height = bounds.height
    contentSize = size
}
{% endhighlight %}

#### Breakpoint

What do these functions _do_ though? Let's break them down:

- `stop()`: Removes any active animations, and resets the position of the "label" to 0.
- `configure()`: The above, in addition to laying out the labels and making sure the scrollable size of the `UIScrollView` is correct - the `contentSize` property.
- `addLabels()`: Loops from 0 to 1, and adds a label to our scroll view for each iteration.
- `layoutLabels()`: For each label, position it at the offset (starting at 0) on the _x_-axis, give it the text, font, and center positions. Then increment the offset for the next iteration. Our second label's _x_-position should be `0 + width_of_label_one + spacing`.
- `configureContentSize()`: Ensures that the scrollable area is enough.

### Animate!

To animate our label, we need to define just _two_ more functions:

{% highlight swift %}
private func start() {
    guard let mainLabel = labels.first, mainLabel.bounds.width > bounds.width else { return }

    let labelWidth = mainLabel.bounds.width
    let delay = scrollDelay

    contentOffset = .zero

    UIView.animate(
        withDuration: duration,
        delay: scrollDelay + 0.5,
        options: ([.curveLinear]),
        animations: { [weak self] in
            guard let self = self else { return }
            self.contentOffset.x = labelWidth + self.spaceBetweenLabels
        },
        completion: { isCompleted in
            if isCompleted {
                DispatchQueue.main.asyncAfter(deadline: .now() + delay) { [weak self] in
                    self?.start() // recursive, must weakify self
                }
            }
        }
    )
}

func scrollIfNeeded() {
    guard let mainLabel = labels.first else { return }

    if mainLabel.bounds.width > bounds.width {
        labels.forEach { $0.isHidden = false }
        start()
    } else {
        labels.forEach { $0.isHidden = $0 != mainLabel }
        contentSize = bounds.size
        mainLabel.frame = bounds
        mainLabel.textAlignment = .left
    }
}
{% endhighlight %}

#### Breakpoint

- `scrollIfNeeded()`: Checks if there's at least one label exceeding the container size, and scrolls if so.
- `start()`: Animates! On completion, we recursively (and weakly!) call `start()` again to continuously trigger the animation after our delay.

### Finally

This implementation is _far leaner_ than my earlier iteration. It is worth noting that I initially switched to `CADisplayLink` to avoid the pre-iOS 5 problem of _the entire_ app having touches blocked if an animation was taking place, and that's roughly when this code was written!

If we want different animation curves, we can expose a property onto our `MarqueeLabel` and pass that into our animation block! Whichever is supported by `UIView.animate` will work.

<img src="{{site.baseUrl}}/assets/img/marquee.gif"/>

Nice right? 🙌

[GITHUB](https://github.com/topLayoutGuide/UIScrollLabel)