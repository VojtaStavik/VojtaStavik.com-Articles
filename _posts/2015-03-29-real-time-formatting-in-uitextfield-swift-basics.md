---
layout: post
title: Real-time text formatting in UITextField
---

Real-time text formatting within ```UITextField``` has became a common feature in iOS apps. Even though there are lots of ready-to-use solutions available over the Internet, sometimes you don’t need a complex and super powerful library. Imagine this situation: We’re using a SSN to sign into the app. It should work like this: 
<br/>
![Example video](/images/2015-03-29/vstextfield-example1.gif)


Let’s build it from scratch. It won’t take more than 10 minutes!

<!-- more -->
*Note: I’m using Xcode 6.3 and Swift 1.2.*

{% highlight swift %}
class SSNTextField: UITextField {
    required init(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
        registerForNotifications()
    }
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        registerForNotifications()
    }
    
    private func registerForNotifications() {        
        NSNotificationCenter.defaultCenter().addObserver(self, selector: "textDidChange", name: "UITextFieldTextDidChangeNotification", object: self)
    }

    deinit {
        NSNotificationCenter.defaultCenter().removeObserver(self)
    }
}
{% endhighlight %}


We start with subclassing ```UITextField```. Our new class will be called ```SSNTextField```. We have to implement both required initializers, ```init(coder aDecoder: NSCoder)``` and ```init(frame: CGRect)```. Because we want real-time formatting, we need a way to find out if the text changed. To receive a notification about it, we add an observer for ```UITextFieldTextDidChangeNotification```. We want to be notified just about text changes related to the object itself, not about other ```UITextField```s in the app. That’s why we use ```object: self``` instead of the usual ```object: nil```. Don’t forget to remove ```self``` as an observer in ```deinit``` function. 


{% highlight swift %}
var formattingPattern = "***-**-****"

var replacementChar: Character = "*"
{% endhighlight %}


We prepare the formatting pattern for the SSN. It should look like this: ```“***-**-****”```. Then we choose ```“*”``` to be a replacement character. So every occurrence of ```“*”``` will then be replaced with a number. All other characters will remain unchanged.



Before we start with a formatting function, we need one more helping function. This function takes the input string and removes all characters except numbers.

{% highlight swift %}
func makeOnlyDigitsString(string: String) -> String {
    return join("", string.componentsSeparatedByCharactersInSet(NSCharacterSet.decimalDigitCharacterSet().invertedSet))
}
{% endhighlight %}


And finally the formatting function. This is where all the magic happens:

{% highlight swift %}
func textDidChange() {
    if count(text) > 0 && count(formattingPattern) > 0 {
        let tempString = makeOnlyDigitsString(text)
        
        var finalText = ""
        var stop = false
        
        var formatterIndex = formattingPattern.startIndex
        var tempIndex = tempString.startIndex
        
        while !stop {
            let formattingPatternRange = Range(start: formatterIndex, end: advance(formatterIndex, 1) )
                        
            if formattingPattern.substringWithRange(formattingPatternRange) != String(replacementChar) {
                finalText = finalText.stringByAppendingString(formattingPattern.substringWithRange(formattingPatternRange))
            } else if count(tempString) > 0 {
                let pureStringRange = Range(start: tempIndex, end: advance(tempIndex, 1))
                finalText = finalText.stringByAppendingString(tempString.substringWithRange(pureStringRange))
                tempIndex++
            }
            
            formatterIndex++
            
            if formatterIndex >= formattingPattern.endIndex || tempIndex >= tempString.endIndex {
                stop = true
            }
        }
        
        text = finalText
    }
}
{% endhighlight %}


Looks complicated but it's actually very simple:

 ```UITextFieldTextDidChangeNotification``` starts the function after the text changes.


First, we filter all digits from the current text in ```text``` (we could also use more Objective-C-like syntax: ```self.text```) and save them to ```tempString```. Now we’re going to apply new formatting to them. We create two indexes. One is named ```formatterIndex``` and is used for enumerating through the ```formattingPattern``` string. The other is ```tempIndex```, and we use it for enumerating through ```tempString```.

The logic is super simple. We take characters from ```formattingPattern``` one by one. If the character is not equal to ```replacementCharacter```, we append it to the ```finalText``` string. If it is equal, we append a digit from ```tempString``` instead and shift ```tempIndex``` to the next digit in ```tempString```. In both cases, we increase ```formatterIndex``` to move to the next character in ```formattingPattern```.

Finally, we check whether ```formatterIndex``` or ```tempIndex``` reached the end of their strings. If so, we’re done here. Let’s save ```finalText``` to ```text``` and get yourself a coffee.

#### We're done!

I think this was more fun than learning how to use some powerful formatting library for such a simple task. You can find a slightly advanced version of this class on my [GitHub](https://github.com/VojtaStavik/VSTextField).
