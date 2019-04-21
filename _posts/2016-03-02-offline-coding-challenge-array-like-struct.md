---
layout: post
title: Offline coding challenge - Array-like struct
---
I visited Moscow last weekend with my girlfriend. This was my first trip to Russia, and it was a really interesting experience. Especially our visit to the [Museum of Cosmonautics](http://www.kosmo-museum.ru/?locale=en), which I’m a big fan of. Anyway, I decided to spend the time on the plane back to Prague by learning and relearning some basics in computer science -- **data structures and algorithms**. 

While reading about arrays, I remembered one article I read a while back about Swift Array implementation ([this one](http://ankit.im/swift/2016/01/08/exploring-swift-array-implementation/)). I was offline on the plane, and I couldn’t read the article. I had two hours left of the flight and no Internet connection, so I decided to try to implement it by myself.

![Challenge accepted!](/images/2016-03-02/offline-challenge.jpg)

<!-- more -->

**How fun it is to program while offline!** I didn’t remember the last time I used the offline documentation in Xcode. It’s so slow that it’s usually faster to google the class name and see it online. Trying to explore new things without access to StackOverflow is also not ideal *(but you do what you gotta do on a long-ass flight)*.

Despite the circumstances, I was able to come up with code during the flight. I had to use ```NSData``` as an underlying storage, because I wasn’t able to figure out how to do a proper memory management for a Swift struct with dynamic size. I found a solution later when I was back online in this [great response on StackOverflow](http://stackoverflow.com/questions/27715985/secure-memory-for-swift-objects/27721441#27721441).

Doing exercises like this one is fun and it's some kind of relax for me. Even though there is nothing really interesting or special in the final code, I learned some new things while writing it.

The main thing I found out is: **It’s really difficult to use unfamiliar APIs when you’re offline ;-)**

So, here’s what I came up with en route to Prague ([gist](https://gist.github.com/VojtaStavik/d68d729d7c05a343f6ba)):

{% highlight swift %}
import Foundation

struct MyArray<Element> : CustomStringConvertible {
    
    var description: String {
        var mutableDescription = "["
        for i in 0..<count {
            mutableDescription += "\"\(elementAtIndex(i)) "
            if i != count - 1 {
                mutableDescription += ", "
            }
        }
        mutableDescription += "]"
        return mutableDescription
    }
    
    private let dataRepresentation: NSMutableData
    
    init() {
        dataRepresentation = NSMutableData()
    }
    
    init(elements: Element ...) {
        self.init(elements: elements)
    }

    init(elements: [Element]) {
        self.init()
        for element in elements {
            append(element)
        }
    }
    
    private let elementSize = sizeof(Element)
    
    var count: Int { return dataRepresentation.length / elementSize }
    
    subscript(index: Int) -> Element {
        get {
            return elementAtIndex(index)
        }
        set {
            insert(newValue, toIndex: index)
        }
    }
    
    func elementAtIndex(index: Int) -> Element {
        let subdata = dataRepresentation.subdataWithRange(NSRange(location: index * elementSize, length: elementSize))
        return UnsafeMutablePointer<Element>(subdata.bytes).memory
    }
    
    mutating func append(element: Element) {
        var copyElement = element
        dataRepresentation.appendBytes(&copyElement, length: elementSize)
    }
    
    mutating func insert(element: Element, toIndex index: Int) {
        var newElement = element
        dataRepresentation.replaceBytesInRange(NSRange(location: index * elementSize, length: 0), withBytes: NSData(bytes: &newElement, length: elementSize).bytes, length: elementSize)
    }
    
    mutating func removeElementAtIndex(index: Int) {
        CFDataDeleteBytes(dataRepresentation, CFRangeMake(index * elementSize, elementSize))
    }
}
{% endhighlight %}

Here you can see some experiments with MyArray in the Playground:
![Playground screenshot](/images/2016-03-02/offline-challenge-2.png)
