---
layout: post
author: Ben
title:  "Swift: Fun With UIDynamicAnimator"
date:   2021-10-18 00:00:00
color:  "SEC"
---

A few years ago, I wrote my first ever custom `UICollectionViewFlowLayout`. It made everything bounce like Messages app does with its chat bubbles on iOS.

If you’ve never noticed the effect before, open a conversation in Messages on your iPhone or iPad, and scroll around a bit. If you pay attention, you’ll see that the items further away from your finger move slower than the items closer to it. In other words, they have inertia.

One weekend in May of 2019, I was cleaning up old files on my Mac and I found the file again. It was terribly written, so I refactored it and wrote a post about it. Tonight, I found it again, and I realised I can improve it even more!

What am I rewriting? This effect for `UICollectionView`: 👇

<img src="{{site.baseUrl}}/assets/img/bounce-old.gif"/>

It’s quite simple, really. It uses everything `UICollectionViewFlowLayout` provides out of the box, but it adds `UIDynamicItem` to create the bounce effect.

>**Note:** You don't need to opt for a grid, you can go for rows too.

The original implementation from 2017 simply added a `UIAttachmentBehavior` to every item in the `UICollectionView.contentSize`. This was fine for relatively short lists, but it began to lag significantly when you approached higher numbers. (I was testing with an iPhone 6S Plus at the time).

### Setup The VC

Make a new Single-view Xcode Project, and add a new file to it. Call this new file `BounceCollectionViewLayout.swift`.

Now, open up `ViewController.swift` and set it up:

{% highlight swift %}
final class ViewController: UIViewController {

  @IBOutlet private weak var collectionView: UICollectionView! {
    didSet {
      collectionView.delegate = self
      collectionView.dataSource = self
    }
  }

  private let fakeDataSource = (0...10000).map { $0 } // Testing with 10,000 items!

  override func viewDidLoad() {
    super.viewDidLoad()
    collectionView.reloadData()
  }

}

extension ViewController: UICollectionViewDelegate {}

extension ViewController: UICollectionViewDataSource {
  func collectionView(_ collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int {
    return fakeDataSource.count
  }

  func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell {
    return collectionView.dequeueReusableCell(withReuseIdentifier: "Cell", for: indexPath)
  }
}
{% endhighlight %}

Add a `UICollectionView` to your Storyboard, making sure to add a prototype cell with an identifier of `"Cell"` for the demo. Set the background color of the prototype cell to a bright color so you can easily see them (such as orange), and don’t forget to connect the collectionView `IBOutlet` to the view controller!

### Setup The Layout

Open up `BounceCollectionViewLayout.swift` and paste in the following boilerplate code:

{% highlight swift %}
import UIKit

final class BounceCollectionViewLayout: UICollectionViewFlowLayout {

  private var dynamicAnimator: UIDynamicAnimator!
  private var visibleIndexPaths: Set<IndexPath> = []
  private var latestDelta: CGFloat = 0

  // MARK: - Initialization
  override init() {
    super.init()
    commonInit()
  }

  required init?(coder aDecoder: NSCoder) {
    super.init(coder: aDecoder)
    commonInit()
  }

  func commonInit() {
    self.dynamicAnimator = UIDynamicAnimator(collectionViewLayout: self)
  }

  deinit {
    dynamicAnimator.removeAllBehaviors()
    visibleIndexPaths.removeAll()
  }

  override func prepare() {
    super.prepare()
    guard let elementsInVisibleRect = super.layoutAttributesForElements(in: visibleRect) else { return }

    // Step 3
  }

  override func layoutAttributesForElements(in rect: CGRect) -> [UICollectionViewLayoutAttributes]? {
    dynamicAnimator.items(in: rect) as? [UICollectionViewLayoutAttributes]
  }

  override func layoutAttributesForItem(at indexPath: IndexPath) -> UICollectionViewLayoutAttributes? {
    dynamicAnimator.layoutAttributesForCell(at: indexPath)
  }

  override func shouldInvalidateLayout(forBoundsChange newBounds: CGRect) -> Bool {
    guard let collectionView = collectionView else { return true }

    // Step 3
    return false
  }

}
{% endhighlight %}

#### Breakpoint

Using `commonInit()`, we can make sure the animator exists if we programatically create the layout, or we set the layout in the Storyboard.

The layout doesn’t do anything “new” yet. But it’s getting there!

### Implement The Layout

Last time I focused on just “getting it done.” I paid little attention to the performance of the layout itself. The app that used this layout only ever displayed 25 items on one screen. It didn’t need to be overly optimised, it just had to look _pretty_.

To make this as efficient as possible, any items not visible on the screen are going to be completely removed. This will be achieved by using a `visibleRect` on the collection view. It will let me know how many items are currently visible, and attach the `UIDynamicBehavior` to only those.

Let's update that layout class:

{% highlight swift %}
final class BounceCollectionViewLayout: UICollectionViewFlowLayout {

  private var dynamicAnimator: UIDynamicAnimator!
  private var visibleIndexPaths: Set<IndexPath> = []
  private var latestDelta: CGFloat = 0

  // MARK: - Initialization
  override init() {
    super.init()
    commonInit()
  }

  required init?(coder aDecoder: NSCoder) {
    super.init(coder: aDecoder)
    commonInit()
  }

  func commonInit() {
    self.dynamicAnimator = UIDynamicAnimator(collectionViewLayout: self)
  }

  // MARK: - Lifecycle
  deinit {
    dynamicAnimator.removeAllBehaviors()
    visibleIndexPaths.removeAll()
  }

  private var visibleRect: CGRect {
    guard let collectionView = collectionView else { return .zero }

    return CGRect(
      x: collectionView.bounds.origin.x, // equivalent to contentOffset.x
      y: collectionView.bounds.origin.y, // equivalent to contentOffset.y
      width: collectionView.frame.size.width,
      height: collectionView.frame.size.height
    )
  }

  /// A "disused" behaviour exists within the dynamicAnimator, but not the visible rect's layoutAttributes array.
  private func removeDisusedBehaviours(from layoutAttributes: [UICollectionViewLayoutAttributes]) {
    let indexPaths = layoutAttributes.map { $0.indexPath }

    dynamicAnimator.behaviors
      .compactMap { $0 as? UIAttachmentBehavior }
      .filter {
        guard let layoutAttributes = $0.items.first as? UICollectionViewLayoutAttributes else { return false }
        return !indexPaths.contains(layoutAttributes.indexPath)
      }
      .forEach { object in
        dynamicAnimator.removeBehavior(object)
        visibleIndexPaths.remove((object.items.first as! UICollectionViewLayoutAttributes).indexPath)
    }
  }

  /// A "new" behaviour is contained within the layoutAttributes array, but not in the visibleIndexPaths.
  private func addNewBehaviours(for layoutAttributes: [UICollectionViewLayoutAttributes]) {
    guard let collectionView = collectionView else { return }

    let touchLocation = collectionView.panGestureRecognizer.location(in: collectionView)

    layoutAttributes
      .filter {
        !visibleIndexPaths.contains($0.indexPath)
      }
      .forEach { item in
        let center = item.center
        let behaviour = UIAttachmentBehavior(item: item, attachedToAnchor: center)
        behaviour.length = 0
        behaviour.damping = 0.5
        behaviour.frequency = 0.8
        behaviour.frictionTorque = 0.0

        if !touchLocation.equalTo(.zero) {
          let distanceFromTouchY: CGFloat = abs(touchLocation.y - behaviour.anchorPoint.y)
          let distanceFromTouchX: CGFloat = abs(touchLocation.x - behaviour.anchorPoint.x)
          let scrollResistance = (distanceFromTouchY + distanceFromTouchX) / 1500
          guard let item = behaviour.items.first as? UICollectionViewLayoutAttributes else { return }

          var center = item.center
          center.y += latestDelta < 0 ? max(latestDelta, latestDelta * scrollResistance) : min(latestDelta, latestDelta * scrollResistance)
          item.center = center
        }

        dynamicAnimator.addBehavior(behaviour)
        visibleIndexPaths.insert(item.indexPath)
    }
  }

  override func prepare() {
    super.prepare()
    guard let elementsInVisibleRect = super.layoutAttributesForElements(in: visibleRect) else { return }

    removeDisusedBehaviours(from: elementsInVisibleRect)
    addNewBehaviours(for: elementsInVisibleRect)
  }

  override func layoutAttributesForElements(in rect: CGRect) -> [UICollectionViewLayoutAttributes]? {
    dynamicAnimator.items(in: rect) as? [UICollectionViewLayoutAttributes]
  }

  override func layoutAttributesForItem(at indexPath: IndexPath) -> UICollectionViewLayoutAttributes? {
    dynamicAnimator.layoutAttributesForCell(at: indexPath)
  }

  override func shouldInvalidateLayout(forBoundsChange newBounds: CGRect) -> Bool {
    guard let collectionView = collectionView else { return true }

    // Step 4
    return false
  }
}
{% endhighlight %}

When items fall outside of the `visibleRect`, which is defined as the current bounds of the collection view, they will be removed completely. Anything new will be added, and the current scroll delta applied so the effect still works.

Finally, implement the `shouldInvalidateLayout()` function:

{% highlight swift %}
override func shouldInvalidateLayout(forBoundsChange newBounds: CGRect) -> Bool {
  guard let collectionView = collectionView else { return true }

  let delta = newBounds.origin.y - collectionView.bounds.origin.y
  let touchLocation = collectionView.panGestureRecognizer.location(in: collectionView)
  latestDelta = delta

  dynamicAnimator.behaviors
    .compactMap { $0 as? UIAttachmentBehavior }
    .forEach {
      guard let item = $0.items.first else { return }

      let distanceFromTouchY: CGFloat = abs(touchLocation.y - $0.anchorPoint.y)
      let distanceFromTouchX: CGFloat = abs(touchLocation.x - $0.anchorPoint.x)
      let scrollResistance = (distanceFromTouchX + distanceFromTouchY) / 1500
      var center = item.center

      center.y += delta > 0 ? min(delta, delta * scrollResistance) : max(delta, delta * scrollResistance)
      item.center = center

      dynamicAnimator.updateItem(usingCurrentState: item)
  }

  return false
}
{% endhighlight %}

### Finally

Set the layout of the `UICollectionView` to ours in the Storyboard, then run the app and scroll around! You may notice a weird clipping effect… this is because we are deleting the items deemed as they fall out of the `visibleRect`. It's not pretty, and it’s especially noticeable at the bottom.

To overcome this, add some overflow to the `visibleRect`:

{% highlight swift %}
private var visibleRect: CGRect {
  guard let collectionView = collectionView else { return .zero }

  return CGRect(
    x: collectionView.bounds.origin.x, // equivalent to contentOffset.x
    y: collectionView.bounds.origin.y, // equivalent to contentOffset.y
    width: collectionView.frame.size.width,
    height: collectionView.frame.size.height
  ).insetBy(dx: 0, dy: -200)
}
{% endhighlight %}

Now we have slightly more than a screen’s worth of cells on-screen with attached behaviours at any given time, but since it helps make the effect smoother I think it's an acceptable trade-off.

<img src="{{site.baseUrl}}/assets/img/smooth.gif"/>

The source code is [AVAILABLE ON GITHUB](https://github.com/topLayoutGuide/BounceCollectionViewLayout). All feedback is welcome!