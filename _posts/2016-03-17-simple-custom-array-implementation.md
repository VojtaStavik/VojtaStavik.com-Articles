---
layout: post
title: Simple Custom Array Implementation -- vol.2
filename: "2016-03-17-simple-custom-array-implementation.md"
---
Two weeks ago, [I wrote about my attempt to implement a very simple Array-like structure](/2016/03/02/offline-coding-challenge-array-like-struct/) during a two hours flight without an Internet connection. I had to use ```NSData``` as the underlying storage, because I had no experience working with ```UnsafeMutablePointer```s in Swift, and I wasn’t able to figure it out without Google.

![I will not accept that](/images/2016-03-17/custom-array-1.gif)

Of course, I wasn’t happy with the result! So I tried to implement it again. This time with an Internet connection and proper manual memory management.

<!-- more -->

### Memory management
```UnsafeMutablePointer```s live in a world outside ARC. Thus, you have to do all the memory management manually. The typical lifecycle of the pointer may look like this:
{% highlight swift %}
// Memory allocation
// -> “1” is number of objects we want to allocate memory for
var pointer = UnsafeMutablePointer<Int>.alloc(1)


// Value initialization
pointer.initialize(155)

// Read the value
let value = pointer.memory  // => 155

// Destroy the value
pointer.destroy()

// Free allocated memory
// => same number of objects as originally allocated
pointer.dealloc(1)

// Release the pointer
pointer = nil

{% endhighlight %}


[When I tried this the first time](http://vojtastavik.com/2016/03/02/offline-coding-challenge-array-like-struct/), I used Struct as an object type (as does the Swift standard library Array). The problem with Struct is that you can’t use ```deinit``` or ```dealloc``` on it, so you won’t know when to deallocate the memory. I was looking for a solution:

*[http://stackoverflow.com/questions/35986292/how-to-dealloc-unsafemutablepointer-referenced-from-swift-struct](http://stackoverflow.com/questions/35986292/how-to-dealloc-unsafemutablepointer-referenced-from-swift-struct)*

![StackOverflow question](/images/2016-03-17/custom-array-2.png)

[@olebegemann](https://twitter.com/olebegemann) advised me to use this interesting workaround:

![StackOverflow answer](/images/2016-03-17/custom-array-3.png)

Unfortunately, we can’t do this with generic objects :/

![Xcode error](/images/2016-03-17/custom-array-4.png)

![Dr.Who sad](/images/2016-03-17/custom-array-5.gif)

This is more an aesthetic than technical issue, because we can still declare class B outside the struct A and then use it inside A without any compiler errors:

{% highlight swift %}
class B<S> { }
struct A<T> {
    let b: B<T>
}
{% endhighlight %}

Our ```PointerBox``` class would then look something like this:

{% highlight swift %}
class PointerBox<Element> {
    var pointer: UnsafeMutablePointer<Element>
    let count: Int
    init(pointer: UnsafeMutablePointer<Element>, count: Int) {
        self.pointer = pointer
        self.count = count
    }
    deinit {
        pointer.destroy(count)
        pointer.dealloc(count)
        pointer = nil
    }
}
{% endhighlight %}

We can use the ```deinit``` method on the class and properly release the memory allocated by the pointer.

---

### The Beauty of Protocol Extensions

There is a general agreement within the Swift community that protocols and protocol extensions are great and cool. But one of the times when you’ll realize their true power and awesomeness is when you start adopting some standard library protocols for your custom objects. **You get so much functionality for free!**

This is the base implementation of the ```MyArray``` struct:

{% highlight swift %}
struct MyArray<Element> {
    private var buffer: PointerBox<Element>
    private(set) var count = 0
    init() {
        buffer = PointerBox(pointer: UnsafeMutablePointer<Element>.alloc(count), count: count)
    }
    func elementAtIndex(index: Int) -> Element {
        precondition(index < count)
        return buffer.pointer[index]
    }
}
{% endhighlight %}

It doesn’t do much yet. In fact, you can’t even initialize it with values. Let’s change this by adopting the ```ArrayLiteralConvertible``` protocol.

{% highlight swift %}
extension MyArray: ArrayLiteralConvertible {
    init(arrayLiteral elements: Element...) {
        count = elements.count
        buffer = PointerBox(pointer: UnsafeMutablePointer<Element>.alloc(count), count: count)
        buffer.pointer.initializeFrom(elements)
    }
}
{% endhighlight %}

Now we can do this:
{% highlight swift %}
var a: MyArray = ["a", "b", "c", "d"]
a.elementAtIndex(2)
{% endhighlight %}

![Emma bored](/images/2016-03-17/custom-array-6.gif)

Our array is pretty boring and useless so far. We can change it by adopting the ```CollectionType``` protocol. This will extend ```MyArray``` by all the fancy functions we know from other ```CollectionType```s like ```Array```, ```Set``` and ```Dictionary```. The adoption is pretty straightforward:

{% highlight swift %}
extension MyArray: CollectionType {
    typealias Generator = IndexingGenerator<MyArray>
    func generate() -> Generator {
        return Generator(self)
    }

    typealias Index = Int

    var startIndex: Index { return 0 }
    var endIndex: Index { return count }

    subscript(position: Index) -> Element {
        get { return elementAtIndex(position) }
    }
}
{% endhighlight %}

Look what we can do now!

{% highlight swift %}
var a: MyArray = ["a", "b", "c", "d"]
let b = a.map{ $0 }
let c = a.reverse()
for d in a {
    print(d)
}
{% endhighlight %}

[And much, much more …](https://developer.apple.com/library/tvos/documentation/Swift/Reference/Swift_CollectionType_Protocol/index.html)


---

### RangeReplaceableCollectionType -- that “one” protocol

Our array is pretty powerful now, but it’s immutable. In order to make it mutable, we should adopt the ```RangeReplaceableCollectionType``` protocol. This protocol comes with functions like ```append```, ```insert```, ```removeAll```, etc.

Believe it or not, to adopt it, you have to add implementation for just one function -- ```replaceRange:withElements:```:

{% highlight swift %}
extension MyArray: RangeReplaceableCollectionType {
    mutating func replaceRange<C : CollectionType where C.Generator.Element == Generator.Element>(subRange: Range<MyArray.Index>, with newElements: C) {
        let rangeLength = subRange.startIndex.distanceTo(subRange.endIndex)
        let countDelta = newElements.count.toIntMax() - rangeLength
        let newNumberOfElements = count + Int(countDelta)

        // Create new buffer
        let newBuffer = UnsafeMutablePointer<Element>.alloc(newNumberOfElements)

        let newElementsStartIndex = Int(subRange.startIndex.toIntMax())
        let newElementsEndIndex = Int(newElementsStartIndex + newElements.count.toIntMax())

        // We use generators to iterate through both original and new elementes
        var newElementsIndicesGenerator = newElements.indices.generate()
        var originalIndicesGenerator = self.indices.filter{ subRange.contains($0) == false }.generate()

        for i in 0..<newNumberOfElements {
            if newElementsStartIndex <= i && i < newElementsEndIndex {
                let e = newElements[newElementsIndicesGenerator.next()!]
                newBuffer.advancedBy(i).initialize(e)
            } else {
                let e = self[originalIndicesGenerator.next()!]
                newBuffer.advancedBy(i).initialize(e)
            }
        }

        count = 0
        self.buffer = PointerBox(pointer: newBuffer, count: newNumberOfElements)
        self.count = newNumberOfElements
    }
}
{% endhighlight %}

**Ugh! OK.**

This one looks a bit more complicated, but it’s in fact very simple:

- It first counts the number of elements in the array after the replacement.
- Then it creates a new ```UnsafeMutablePointer``` and generators for both the original array and new elements.
- The next step is to copy values from these collections to the new buffer.
- Finally, it replaces the old buffer with the new one. Thanks to the ```PointerBox``` class we created at the beginning, the old pointer’s allocated memory is automatically released.

Adopting ```RangeReplaceableCollectionType``` allows us to do things like this:

![Playground screenshot w/o CustomStringConvertible](/images/2016-03-17/custom-array-7.png)

#### I can't see it!

Oh, sure! We should adopt CustomStringConvertible, too:

{% highlight swift %}
extension MyArray: CustomStringConvertible {
    var description: String {
        return "[" + self.map{"\"\($0)\""}.joinWithSeparator(", ") + "]"
    }
}
{% endhighlight %}

Now it’s better:

![Playground screenshot w/ CustomStringConvertible](/images/2016-03-17/custom-array-8.png)

---

You can find the gist with the ```MyArray``` source code [here](https://gist.github.com/VojtaStavik/b3073c9b5f186016e639).

**Special thanks to [@airspeedvelocity](https://twitter.com/airspeedswift) and [@olebegemann](https://twitter.com/olebegemann) for sharing their knowledge about unsafe pointers and protocols in Swift.**
