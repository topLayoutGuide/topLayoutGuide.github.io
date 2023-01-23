---
layout: post
author: Ben
title:  "Swift: Animating UILabel Text Change"
date:   2021-10-15 00:00:00
color:  "SEC"
---

Have you ever needed a `UILabel` to keep track of a numerical value? My best guess is that you have, but if you’ve ever wanted to animate between values, you probably found your choices limited.

I’ve wanted to animate `UILabel` content updates many times. Whenever I’m writing an app, I have a tendency to focus on the tiny details. I truly believe that microinteractions make the final product _better_, even if the hours I spend curating the perfect animation go unnoticed.

Anyway. Years and years ago, I found a Github repo which did exactly what I wanted. At the time, it was perfect! I just dropped it in and off I went! My score label started at 0 and ticked up or down, until it reached the intended value.

In 2016, I needed the behavior again. I searched Github, however none of the versions I found had been updated for the latest version of Swift at the time. So I rewrote it in Swift 3!

In 2021, I'm rewriting it again using Swift 5.

### Why Do I Do This To Myself?

A fair question. I don't like porting things from Objective-C to Swift, without harnessing the utilities that Swift provides. In 2016 it was a fun challenge for a coding camp weekend. In 2021, it's just some fun on a Friday night 🙌

My `UIAnimatedLabel` will use a `CADisplayLink`, which will refresh the label at the native cadence of the display. In combination with some fancy math, this gives us a nice animation "curve" during the count. The default option here is `.easeInOut` which will start off slow, speeds up, then slow down before finishing. 

I can't wait to see how this works on a ProMotion display... 🤩

### Let's Get Started

Open a new project in Xcode, and then make a new file called `UIAnimatedLabel`. In this new file, create a `class` called `UIAnimatedLabel` and have it subclass `UILabel`.

Inside our `class` let's define two `enum`:

{% highlight swift %}
enum Method {
  case linear
}

enum Formats {
  case zeroDecimals, oneDecimal

  var format: String {
    switch self {
    case .zeroDecimals: return "%.0f"
    case .oneDecimal: return "%.1f"
    }
  }
}
{% endhighlight %}

#### Breakdown

These `enum` are defining how we want our "count" effect to work, and how many decimal points the values in the label should have. If you're keeping score (i.e. in a game), then you typically want `.one` or `.zero`. I'll leave it to you to add more formats.

Now, let's define some `private` variables. These are ones that only _we_ want to use. We want to control what people can and cannot use in our label subclass.

{% highlight swift %}
private var currentValue: Float {
  if progress >= duration { return to }
  return from + (update(t: Float(progress / duration)) * (to - from))
}

private var from: Float = 0
private var to: Float = 0
private var progress: TimeInterval = 0
private var lastUpdatedTime: TimeInterval = 0
private var duration: TimeInterval = 0
private var displayLink: CADisplayLink?
private var completion: (() -> Void)?
{% endhighlight %}

#### Breakdown

- `currentValue` will be the current value displayed on our label
- `from` will be the number it starts from (i.e. 456)
- `to` will be the number it will end at (i.e. 0)
- `progress` is the current progress of our animation
- `lastUpdatedTime` is the time we last updated the label. Useful for finetuning the progress value. The value is a Unix timestamp.
- `duration` will hold our animation duration value.
- `displayLink` is our `CADisplayLink` which will tick (update) at 60Hz (or 120Hz) per second.
- `completion` is an optional completion handler we can call upon animation complete.

Now add some ordinary variables, to allow users to set the animation counting method and the format to use for the number displayed:

{% highlight swift %}
var format = Formats.zeroDecimals
var method = Method.easeInOut
{% endhighlight %}

### The Functions

Our label will have two main functions. One will take a `from` and `to` value, then count between them. The other will count `to` a value, from whatever the current value of the label happens to be. Both of them will also have a `duration` value, which we can supply to control how long the animation runs for.

Let's go ahead and define those now:

{% highlight swift %}
func count(from: Float, to: Float, duration: TimeInterval = 10.0, _ completion: (() -> Void)? = nil) {
  self.completion = completion
  self.from = from
  self.to = to
  self.progress = 0.0
  self.duration = duration
  self.lastUpdatedTime = Date.timeIntervalSinceReferenceDate
}

func countFromCurrent(to: Float, duration: TimeInterval = 10.0, _ completion: (() -> Void)? = nil) {
  count(from: currentValue, to: to, duration: duration, completion)
}

func stopAnimating() {
  progress = duration
}
{% endhighlight %}

Our functions do nothing at the moment. In fact the app doesn't even compile! So let's add more implementation.

Let's define some more functions:

{% highlight swift %}
private func addDisplayLink() {
  displayLink = CADisplayLink(target: self, selector: #selector(self.updateValue(timer:)))
  displayLink?.add(to: .main, forMode: .default)
  displayLink?.add(to: .main, forMode: .tracking)
}

private func resetTimer() {
  displayLink?.invalidate()
  displayLink = nil
}

@objc private func updateValue(timer: Timer) {
  let now: TimeInterval = Date.timeIntervalSinceReferenceDate
  progress += now - lastUpdatedTime
  lastUpdatedTime = now
}
{% endhighlight %}

#### Breakdown

- Our `addDisplayLink` function sets up and registers our `CADisplayLink`
- We need to add it to the `default` run loop, and the `tracking` run loop, so that the animation can complete while we're still interacting with our app.
- The `resetTimer` function invalidates the link (stops it), and removes the instance.
- Our `updateValue` function updates the progress of our animation.

If we ran our app now, still nothing would work visually. But we're getting closer. Let's extend our `count` func. Add the following after all our `self.` statements.

{% highlight swift %}
resetTimer()

if duration == 0.0 {
  text = String(format: decimalPoints.format, to)
  completion?()
  return
}

addDisplayLink()
{% endhighlight %}

What does this do? Just in case our function is called more than once during the current animation, we reset the timer. Then we check if the duration is 0 seconds (no animation), set the final value on our label, call the completion handler, and call it a day.

Right at the end, we setup our display link once again to start off our animation.

---

Let's turn our attention to the `updateValue` function. Add this to the end of the function:

{% highlight swift %}
if progress >= duration {
  resetTimer()
  progress = duration
}

text = String(format: decimalPoints.format, currentValue)

if progress == duration { completion?() }
{% endhighlight %}

Now that we've done this, the last thing left is to define our `update(t:)` function, which is missing so far.

### Update

Let's define the following functions in our source file:
{% highlight swift %}
private func update(t: Float) -> Float {
  t
}
{% endhighlight %}

Now that we have everything defined, if we ran the app now and told the label to count from and to a value, it would do so in a linear fashion. For example, counting from 0 to 100 per tick of the display over the duration provided.

We _could_ extend the `Method` enum with more animaton curves... such as `easeIn`, `easeOut`, and `easeInOut`. But there's a lot more math involved!

### Finally

<img src="{{site.baseUrl}}/assets/img/countupdown.gif"/>

Doesn't it look _beautiful_?

You can view the final (more complete) project on [GITHUB](https://github.com/topLayoutGuide/UIAnimatedLabel)

Thanks for reading! Let me know if you have any feedback! 🎉