---
layout: post
title: Playing with properties - 3 facts I didn't know
---
I experienced a very interesting bug in our code last week. Our unit tests from time to time froze. It only happened when we ran some of our test groups separately. When we ran the full test set, everything was OK. We have a lot of untestable legacy code in the project -- singletons, missing dependency injections, weird code coupling.

![Help me StackOverflow ... ](/images/2016-02-02/playing-with-properties.jpg)

<!-- more -->

I found the bug quickly. Several singletons had circle references to each other. Everything worked if they were initialized in the “correct” order. If the order changed, the app ended up in the loop and froze. I found some interesting facts I didn't know (or realize) about properties and their initializing while I was fixing the bug.

### 1. Sometimes you get “lazy” behavior even when you don’t want it

We had some singletons defined like this:

{% highlight swift %}
import Foundation
let loner = Loner()

class Loner {
    init() {
        print("Loner init")
    }
    func sayHi() {
        print("Hi!")
    }
}
{% endhighlight %}

When I wrote this code, I thought the ```loner``` variable would be initialized at the moment the app started. I was expecting the similar behavior you get with the instance ```let``` property. It’s in fact initialized the moment you use it for the first time. And that’s how we describe lazy initialization, right?

It means this code
{% highlight swift %}
class ViewController: UIViewController {
    override func viewDidAppear(animated: Bool) {
        super.viewDidAppear(animated)
        print("View did appear")
        loner.sayHi()
    }
}
{% endhighlight %}
will result in this console output:
{% highlight html %}
App did finish launching
View did appear
Loner init
Hi!
{% endhighlight %}


### 2. Non-lazy properties are initialized before init is called

This is quite obvious when you think about it. However, sometimes you need to say it out loud to realize it. Instance properties with assigned values are initialized before a proper initializer is called.

When we have this code
{% highlight swift %}
struct George {
    init() {
        print("Creating George")
    }
}

struct Beatles {
    let george = George()
    init() {
        print("Creating The Beatles")
    }
}
{% endhighlight %}
the console output is:
{% highlight swift %}
Creating George
Creating The Beatles
{% endhighlight %}
*(aka you first have to create George to create The Beatles.)*

This becomes important when you initialize complicated classes with singleton references and other state-related hell. 
{% highlight swift %}
let beatles = Beatles() // We use singleton because The Beatles are unique

struct George {
    var numberOfGreatSongs = 3 {
        didSet {
            beatles.printNumberOfGreatSongs()
        }
    }
    init() {
        print("Creating George")
    }
}

struct Beatles {
    let george: George = {
        var george = George()
        george.numberOfGreatSongs = 8
        return george
    }()
    
    init() {
        print("Creating The Beatles")
    }
    
    func printNumberOfGreatSongs() {
        print(george.numberOfGreatSongs)
    }
}
{% endhighlight %}
The console output is:
{% highlight html %}
Creating George
{% endhighlight %}
The code above ends up in the loop, and ```Beatles```’ initializer will never be called. This is an obvious example, and **nobody will write code like that.** However, it’s not unrealistic on a bigger project with a bigger team and a complicated project structure. *At least, this is what happened to us.*


### 3. You can reinitialize a lazy property

Look at this code:
{% highlight swift %}
struct George {
    init() {
        print("Creating George")
    }
}

struct Beatles {
    lazy var george = George()
    init() {
        print("Creating Beatles")
    }
}

var beatles = Beatles()
print("The Beatles without George?")
let george = beatles.george
{% endhighlight %}
Console output:
{% highlight html %}
Creating Beatles
The Beatles without George?
Creating George
{% endhighlight %}
```george``` is a lazy property, and it’s initialized at the moment we access it for the first time. Everything works as expected. 

Let’s do some experiments:
{% highlight swift %}
struct George {
    init() {
        print("Creating George")
    }
}

struct Beatles {
    lazy var george: George? = George()
    init() {
        print("Creating Beatles")
    }
}

var beatles = Beatles()
print("The Beatles without George?")

let george = beatles.george
beatles.george = nil
let newGeorge = beatles.george
{% endhighlight %}
Console:
{% highlight html %}
Creating Beatles
The Beatles without George?
Creating George
nil
{% endhighlight %}
The ```george``` property type is the optional ```George?``` type in this example. It means we can assign ```nil``` to its value. Once we do that, the value really is ```nil```. Again, this is nothing we wouldn’t expect.

Look at the third example:
{% highlight swift %}
struct George {
    init() {
        print("Creating George")
    }
}

struct Beatles {
    lazy var george: George! = George()
    init() {
        print("Creating Beatles")
    }
}

var beatles = Beatles()
print("The Beatles without George?")

let george = beatles.george
beatles.george = nil
let newGeorge = beatles.george
print(newGeorge)
{% endhighlight %}

Console:
{% highlight html %}
Creating Beatles
The Beatles without George?
Creating George
Creating George
George()
{% endhighlight %}
The ```george``` property type in this example is an implicitly unwrapped optional ```George!```. As you can see, even after we set its value to ```nil```, it reinitialized again! 

This is a very interesting thing to know. **If a property is lazy and its type is an implicitly unwrapped optional, you can reinitialize it by assigning ```nil``` to it.** I use this trick when I need to completely reset my Core Data stack after user logs out.
