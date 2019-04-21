---
layout: post
title: Making life easier - Swift "Tweaks"
---

What I really like about programing is the idea of reusability. The fact that you can use a chunk of code today for one project and then again the following year for a different project is remarkably satisfying. Learning Swift these days gives us, iOS programers, a great opportunity to re-think our old manners, code snippets and techniques. We can create new ones from scratch and use all we’ve learned over the years to make them even more clever.

Let me show you a bunch of Swift functions and extensions I’m using. Don’t expect any new, rule-breaking ideas, though. These are common and well-known things. Or at least (I think) they should be.

<!-- more -->


- The Swift variation of the good old Objective-C macros for creating UIColor from RGB(A) values.  

{% highlight swift %}
func RGB(red: CGFloat, green: CGFloat, blue: CGFloat) -> UIColor! {
    return RGBA(red, green, blue, 1)
}

func RGBA(red: CGFloat, green: CGFloat, blue: CGFloat, alpha: CGFloat) -> UIColor! {
    return UIColor(red: red/255.0, green: green/255.0, blue: blue/255.0, alpha: alpha)
}
{% endhighlight %}




- This is a simple function created by [Matt Thompson](https://twitter.com/mattt) (I think). It wraps up the unfriendly syntax of the ```dispatch_after``` function into a cool Swifty version.

{% highlight swift %}
func delay(delay:Double, closure:  ()->()) {
    dispatch_after(
        dispatch_time(
            DISPATCH_TIME_NOW,
            Int64(delay * Double(NSEC_PER_SEC))
        ),

        dispatch_get_main_queue(), closure)
}
{% endhighlight %}





- The similar approach as the previous one but for the ```for``` cycle. It allows you to use trailing closure syntax.

*EDIT: Repeat became a keyword in Swift 2.0 so this is no longer a valid code*

{% highlight swift %}
// example:
repeat(100) {    
    println("I will not record vertical oriented videos. \n")
}

func repeat(cycles: Int, closure: () -> ()) {
    for _ in 0..<cycles {
        closure()
    }
}
{% endhighlight %}


- This is a useful function if you use custom fonts and can’t figure out what’s the proper front name.

{% highlight swift %}
func printAllAvailableFonts() {
    let fontFamilyNames = UIFont.familyNames()

    for familyName in fontFamilyNames {        
        println("------------------------------")
        println("Font Family Name = [\(familyName)]")
        let names = UIFont.fontNamesForFamilyName(familyName as! String)
        println("Font Names = [\(names)]")
    }
}
{% endhighlight %}


- I’m using the following function when I need to fill my model classes with random data for testing. 

*Note: This function needs String subscript extension you can find below.* 

{% highlight swift %}
func randomStringWithLength (length : Int) -> String {
    let letters = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
    var randomString = ""
    
    repeat(length) {
        let rand = Int(arc4random_uniform(UInt32(count(letters))))
        randomString.append(letters[rand])
    }

    return randomString
}
{% endhighlight %}



- The subscript extension for String provides you access to a single character of the string via the subscript syntax: ```someString[index]```

{% highlight swift %}
extension String {
    subscript (i: Int) -> Character {
        return self[advance(self.startIndex, i)]
    }
    
    subscript (i: Int) -> String {
        return String(self[i] as Character)
    }
    
    subscript (r: Range<Int>) -> String {
        return substringWithRange(Range(start: advance(startIndex, r.startIndex), end: advance(startIndex, r.endIndex)))
    }
}
{% endhighlight %}




- My personal favorite! Once you get use to this you can’t live without it. It takes an object and returns its normalized version within the minimum and maximum. Great for animations!

{% highlight swift %}
func between<T : Comparable>(minimum: T, maximum: T, value: T) -> T {
    return min( max(minimum, value) , maximum)
}
{% endhighlight %}





- Automatically generated reusable identifiers for UITableViewCell and UICollectionViewCell. You can use these values when you register cell NIBs manually. 
*(By the way, can somebody explain to me why Apple doesn’t allows us to create Interface Builder identifiers programmatically and use it in a similar way we use, for example, @IBInspectable? The current situation is extremely bug friendly.)*

{% highlight swift %}
extension UITableViewCell {
    class func reusableIdentifier() -> String! {
        return NSStringFromClass(self) + "Identifier"
    }
}

extension UICollectionViewCell {
    class func reusableIdentifier() -> String! {
        return NSStringFromClass(self) + "Identifier"
    }
}
{% endhighlight %}






- This is a ```UIImage``` extension which helps you generate a copy of the image in a certain color. This is great for coloring button icons programmatically.

{% highlight swift %}
extension UIImage {	
    func imageWithColor(color: UIColor) -> UIImage {
        UIGraphicsBeginImageContextWithOptions(self.size, false, self.scale)

        let context = UIGraphicsGetCurrentContext()
        CGContextTranslateCTM(context, 0, self.size.height)
        CGContextScaleCTM(context, 1.0, -1.0)
        CGContextSetBlendMode(context, kCGBlendModeNormal)

        let rect = CGRectMake(0, 0, self.size.width, self.size.height)
        CGContextClipToMask(context, rect, self.CGImage)
        color.setFill()
        CGContextFillRect(context, rect)

        let newImage = UIGraphicsGetImageFromCurrentImageContext()
        UIGraphicsEndImageContext()
            
        return newImage;
    }
}
{% endhighlight %}


<p></p>
You can find all the Swift “tweaks” I’m using in my projects on my [GitHub - iOS-SwiftTweaks](https://github.com/VojtaStavik/iOS-SwiftTweaks). I’d love to hear about your tweaks!
