---
layout: post
title:  "TIL: Dictionary Grouping In Swift Is Fast"
date:   2019-12-23 12:00:00
color:  "QUIN"
---

#### Very, very fast.

### Intro

Across our programming careers, it may surprise you but there are times where we need to sort and group collections of data. Sorting itself is a simple concept, but it's the implementation you go with which can make you or break you.

For example, the list of Contacts in your phone. Depending on your circumstances, that list is either small or (sometimes unreasonably) long. This is why there's almost always a way to quickly navigate the list, removing the need to scroll too much. This is achieved with sorting, and grouping. 

You set a few rules, and the app sorts the data accordingly. The implementation needs to be able to handle lots of data, quickly, otherwise opening your Contacts app becomes a chore.

---

Let’s say I queried an API for a list of all movies released in the last twenty years. If I were to display that in an app for iOS, I’d sort it in alphabetical order since we’re talking about (potentially) thousands of movies. The UI would be simple, a `UITableView` with sections A-through-Z, and a rolodex bar on the right hand side.

I’ve seen an implementation similar to this, many times:

{% highlight swift %}
//
// NB: Might not compile...
//

let alphabet = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"

for letter in alphabet {
  var list = [Movies]()

  for movie in movies {
    if let prefix = movie.name.prefix(1), prefix == letter {
      list.append(movie)
    }
  }
  indexedList.append((letter, list))
}
{% endhighlight %}

Loosely, what that’s doing is going through the alphabet and for each matching movie, making a data structure as follows:

{% highlight swift %}
[("A": [Movie, Movie, Movie]), ("B": [Movie, Movie]), ("C".....)]
{% endhighlight %}

The time complexity of this algorithm would probably be _O(n²)_. For each time the outer loop runs N, the inner loop runs M times. In other words, the longer the list of movies becomes, the longer it takes for your implementation to do its job. 

Using this with an iPhone 6, it takes almost a full two seconds to sort a list of 15,000 items. Frustratingly slow. If you do your sort on `viewWillAppear` or `viewDidLoad` it will actually block the transition from happening right away.

### Let's Use Dictionary Grouping

We can somewhat overcome this by using Dictionary grouping in Swift. I took my list of movies, and wrote a predicate:

{% highlight swift %}
let predicate: (Movie) -> String {
  guard let c = String($0.title.prefix(1)) else { fatalError() }
  
  let alphabet = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
  let numeric = "0123456789"
  if alphabet.contains(c) {
    return c
  } else if numeric.contains(c) {
    return "#"
  } else {
    return "?"
  }
}
{% endhighlight %}

I have defined my rules with the predicate. If the title of the movie begins with an alphabet character, group that movie in the appropriate category. If the title of the movie begins with a number, group it under #, and lastly under ? if it doesn’t match any of the above.

Then I created a Dictionary, giving it my predicate, and sort it in ascending order.

{% highlight swift %}
let grouped = Dictionary(grouping: movies, by: predicate).sorted { $0.0 < $1.0 }
{% endhighlight %}

This results in the data structure I desire. The time taken was reduced from two seconds, to a mere 146ms (or 0.146 seconds) on an iPhone 6, sorting the same 15,000 items. I also sampled this on an iPhone X, and was pleasantly surprised by the 25ms (0.025 second) result.

---

### Conclusion

It’s up to you on how you want to partition your data. You can reinvent the wheel, or you can use things that Swift provides to you out of the box. Hopefully this post has given you some motivation to improve your apps, or learn Swift!

In a future iteration of this, I turned it into a service object which I can then Mock for unit testing. I'll post this later.