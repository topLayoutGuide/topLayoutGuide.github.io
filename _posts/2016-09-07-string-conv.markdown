---
layout: post
author: Ben
title:  "TIL: CustomStringConvertible Makes Debugging Better"
date:   2016-09-07 12:00:00
color:  "QUIN"
---

I first stumbled across this in 2016 while I was still learning Swift. It's still relevant today, so I'm reposting it here. 

Enjoy!

### Unhelpful Debugger

Suppose we’re developing a directory app and have two simple classes: 

`Person()` and `Department()`:
{% highlight swift %}
class Person {
    let id: UUID
    let givenName: String
    let middleName: String?
    let surname: String
    let preferredName: String?
    var displayName: String {
        guard let preferredName = preferredName else {
            return [givenName, middleName, surname]
                .compactMap { $0 }
                .joined(separator: " ")
        }

        return preferredName
    }

    init(
        givenName: String,
        middleName: String? = nil,
        surname: String,
        preferredName: String? = nil) {
            self.id = UUID()
            self.givenName = givenName
            self.middleName = middleName
            self.surname = surname
            self.preferredName = preferredName
        }
}

class Department {
    let name: String
    let managers: [Person]
    let members: [Person]

    init(name: String, managers: [Person], members: [Person]) {
        self.name = name
        self.managers = managers
        self.members = members
    }
}
{% endhighlight %}

Now we want to instantiate and print a `Department` object like this:

{% highlight swift %}
let dept = Department(
    name: "QA",
    managers: [
        .init(givenName: "Johnny", surname: "Appleseed")
    ],
    members: [
        .init(givenName: "Benjamin", surname: "Deckys", preferredName: "Ben"),
        .init(givenName: "Stefani", middleName: "Joanne Angelina", surname: "Germanotta", preferredName: "Lady Gaga")
    ]
)

print(dept)
{% endhighlight %}

What we see isn't helpful, at all:

{% highlight swift %}
\_\_lldb_expr_21.Department
{% endhighlight %}

We can mitigate this problem somewhat, by changing `class` to `struct`. And of course, since two people can have the same name, let's make our objects `Equatable`.

Printing it now gives us more information but it's still super noisy. Let's clean it up!

### Conforming to CustomStringConvertible

Go ahead, and make our two `struct` types conform to `CustomStringConvertible`:

{% highlight swift %}
extension Person: CustomStringConvertible {
    var description: String {
        displayName
    }
}

extension Department: CustomStringConvertible {
    var description: String {
        "\(name); Managers: \(managers); Members: \(members)"
    }
}
{% endhighlight %}

The `description` property is a protocol-stipulated requirement, and the string we compose will be printed in the console when using `print()`:

{% highlight swift %}
// QA; Managers: [Johnny Appleseed]; Members: [Ben, Lady Gaga]
{% endhighlight %}

### Conclusion

Great! What a huge win for a tiny amount of work! Now we are able to debug our objects more effectively when something goes bad during development. It isn't always smooth sailing!