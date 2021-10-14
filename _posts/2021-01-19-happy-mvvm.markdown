---
layout: post
title:  "I'm Only Happy When It's MVVM!"
subtitle: "SwiftUI + MVVM + Combine + ReactiveSwift"
date:   2021-01-19 19:53:11
color:  "SEC"
---

#### SwiftUI + MVVM + Combine + ReactiveSwift

### Preface

I think it’s fair to assume that everyone who went through lockdown, ended up rediscovering a passion or two while spending more time at home. I definitely include myself in this, as the monotony of my work-life routine had been turned on its head. I no longer woke up to a barely furnished apartment, dressed for work, and hurried for the train while gulping down my morning coffee. 

My office had moved from a 20 minute train ride, to 20 feet away from my bed. 

After an initial adjustment, and hurried furniture-buying for my apartment, I found my creative mind swimming with motivation for the first time in months. Previously, after commuting to, working at, then commuting home from the office, I was little inclined to do more programming during my downtime.

These past few weeks, I found myself more motivated than ever to learn something new. I decided to further my understanding of the oft-used MVVM app architecture, and learn how to apply it to future apps I make or contribute to using `Combine` and `SwiftUI`.

### Inputs & Outputs

For the last three years, I have been working on a codebase which places a heavy emphasis on I/O as a methodology for creating ViewModel objects and Service types. The approach works perfectly with `ReactiveSwift`, as seen below:

{% highlight swift %}

protocol ViewModel {
  associatedtype Input: Equatable
  associatedtype Output: Equatable
  func apply(_ input: Input)
  var output: Signal<Output, Never> { get } 
}

class SomeViewModel: ViewModel {

  // MARK: - Init
  
  init() {
    // Inject dependencies... etc
    bind()
  }

  // MARK: - Input
  private var viewDidLoadIO = Signal<Void, Never>.pipe()
  
  enum Input: Equatable {
    case viewDidLoad
  }

  func apply(_ input: Input) {
    switch input {
    case .viewDidLoad: viewDidLoadIO.input.send(value: ())
    }  
  }

  // MARK: - Output
  enum Output: Equatable {
    case updateLabel(String)
  }

  private let outputIO = Signal<Output, Never>.pipe()
  var outputSignal: Signal<Output, Never> { outputIO.output }

  // MARK: - Bind Actions
  private func bind() {
    viewDidLoadIO.output
      .map { .updateLabel("Hello, World!") }
      .observeValues(outputIO.input.send)
  }
}
{% endhighlight %}

… but that’s not all. I knew that this approach ought to work with Combine as well, since it is a first-party solution to ReactiveSwift provided by Apple.

In layman's terms, an Input is an event which occurs on a View, such as `didTapLoginButton`, and an Output is the result.

### @State, @ObservedObject, @EnvironmentObject?

I had some trouble wrapping my head around these annotations, but if you understand the principle of an App having a “state,” then you’ll be fine.

With `SwiftUI`, our views are value types, and thus cannot (usually) be mutated. We need to hand over control to the `State`, so that when _it_ updates, the view will show the latest data. Without any of this, your app would be non-functional.

If your “state” comprises of a few simple types such as a `String`, a `Bool`, and an `Int`, then you can go ahead and annotate your properties as `@State` types. It's implied that our state only refers to and controls the current view, so it’s important to mark these properties `private`. Scope matters!

If your state is more complex, such as a ViewModel, and/or may apply to multiple views, you’ll be better off using `@ObservedObject`. This is the same as `@State` in many ways, however when you use these properties in your state, you get to decide whether changes force a refresh or not by observing them (or not!).

Lastly, `@EnvironmentObject` is very similar to `@ObservedObject`. However, these objects are available to all views of your app at any time. If you share a lot of data around screens of your app, these are _incredibly_ useful. The closest comparison to `@EnvironmentObject` that I can give you, is `UIScreen.main`. They are essentially singletons. Use them wisely and sparingly.

### Combining Approaches

After reading about the property wrappers, I decided that my ViewModel itself ought to be an `@ObservableObject`. When properties in the view model change, I want my components to update accordingly. Furthermore, my view model likely won’t only hold simple types forever. I may eventually expand it and make it hold more complex types, or load data from a server.

The first step was to port over my ViewModel protocol, and then find an equivalent for my `.pipe()` objects from `ReactiveSwift`, which conveniently use the I/O principle already.

The closest match to `.pipe()` ended up being `PassthroughSubject`. According to the documentation, these don’t have an initial value or a buffer of the most recently-published element. Great!

More documentation-reading led me to `@Published` properties, which do what they say on the tin. They “publish” values to objects observing them. Thus, my protocol becomes:

{% highlight swift %}
import SwiftUI
import Combine

protocol ViewModel: ObservableObject {
  associatedtype Input: Equatable
  var cancellables: [AnyCancellable] { get }
  func apply(_ input: Input)
}
{% endhighlight %}

Then my ViewModel becomes:

{% highlight swift %}
class SomeViewModel: ViewModel {

  // MARK: - Init
  // Any dependencies (services, properties et al) can be injected here
  
  init() {
    bind()
  }

  var cancellables: [AnyCancellable] = []

  // MARK: - Input
  
  let onAppearSubject = PassthroughSubject<Void, Never>()
  
  enum Input: Equatable {
    case onAppear
  }
  
  func apply(input: Input) {
    switch input {
      case .onAppear: onAppearSubject.send(())
    }
  }
  
  // MARK: - Output
  @Published var text = ""

  // MARK: - Binding View Model
  private func bind() {
    onAppearSubject
      .map { "Hello, World!" }
      .assign(to: \.text, on: self)
      .store(in: &cancellables)
  }
}
{% endhighlight %}

### Conclusion

You just built a ViewModel, using I/O principles from ReactiveSwift, in Combine! Pat yourself on the back...

<img src="{{ site.baseurl }}/assets/img/20210119.png" width="320pt"/>