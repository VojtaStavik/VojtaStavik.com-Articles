---
layout: post
title: Customizing UIKit appearance using protocols
---
#### Prologue

I remeber exactly the very moment when I started hating storyboards. It was back in 2013, and I had to solve my first storyboard merge conflict. I didn’t succeed. I convinced myself that storyboards are tools for beginners because “real programmers write code.” We even had it written in our iOS coding guidelines at [STRV](http://www.strv.com): “unless the app is super simple, try to avoid using storyboards.”

<!-- more -->

It all changed with iOS8, the iPhone6 and all these different screen sizes we had to support. I started using storyboards, and within a few weeks, I realized how powerful a tool it is. I fell in love. Of course, you can still write all your constraints for different size classes in code. But let’s face it, it’s not a way how to do an easily maintainable piece of software.


#### Act 1

One of the questions I was facing was how to effectively customize the appearance of UIKit elements. Setting up everything in a storyboard wasn’t an acceptable solution, because I wanted similar elements to share similar attributes. For example, if all my “call to action” buttons are green, I want to set it just once.

I could have used the UIAppearance proxy. But it somehow didn’t feel right. What I wanted was something like [nui](https://github.com/tombenner/nui) but in native Swift. **And then Swift 2.0 introduced protocol extensions. The game changer.**



#### TL;DR

#### Protocol based UIKit customization: ProtocolUI

Let me introduce you **ProtocolUI**. ```ProtocolUI.swift``` ([source code](https://github.com/VojtaStavik/ProtocolUI/blob/master/ProtocolUI.swift)) is a helper file which contains definitions for a dozen protocols. These protocols reflect the very basic *(so far)* UIKit customizable properties. You can use these protocols as a base for your own protocols. By adding extensions to them, you can modify their values and customize views that conform to the protocols.

##### Simple example:
I want to set the ```UIView``` background color to green:

- Pick a base protocol which modifies the ```backgroundColor``` property:

{% highlight swift %}
protocol BackgroundColor { var pBackgroundColor: UIColor { get } }
{% endhighlight %}


- Create you own protocol and its extension, which will return the desired value

{% highlight swift %}
protocol GreenBackgroundColor : BackgroundColor  { }
extension GreenBackgroundColor { 
	var pBackgroundColor : UIColor { return UIColor.greenColor() } 
}
{% endhighlight %}


- Make your custom view to conform to the protocol:

{% highlight swift %}
class MyView : UIView, GreenBackgroundColor { }
{% endhighlight %}


That’s all. When you now use the ```MyView``` class in a storyboard and run the app, the view will have a green background.

![Example 1](/images/2015-07-29/protocol-ui-1.png)


You can apply the very same protocol to other UIKit elements, too:

{% highlight swift %}
class MyButton : UIButton, GreenBackgroundColor { }
class MyTextField : UITextField, GreenBackgroundColor { }
{% endhighlight %}

![Example 2](/images/2015-07-29/protocol-ui-2.png)


##### One more example:
All buttons should have a yellow background and Helvetica Neue font, size 17.0. All *“call to action”* buttons will also have a green border with a width of 2px.

We create protocols for the background color and button font first. Then we create a protocol named ```ButtonAppearance```, which inherits from these two protocols and works as a shared appearance protocol for all buttons.

{% highlight swift %}
protocol YellowBackgroundColor : BackgroundColor  { }
extension YellowBackgroundColor { 
	var pBackgroundColor : UIColor { return UIColor.yellowColor() }
}

protocol ButtonFont : Font { }
extension ButtonFont { 
	var pFont : UIFont { return UIFont(name: "Helvetica Neue", size: 17.0)! }
}

protocol ButtonAppearance : YellowBackgroundColor, ButtonFont { }
{% endhighlight %}


We do the same for the ```CallToActionAppearance``` protocol:

{% highlight swift %}
protocol GreenBorder : BorderColor { }
extension GreenBorder { 
	var pBorderColor : UIColor { return UIColor.greenColor() } 
}

protocol DefaultBorderWidth : BorderWidth { }
extension DefaultBorderWidth { 
	var pBorderWidth : CGFloat { return 2.0 } 
}

protocol CallToActionAppearance : GreenBorder, DefaultBorderWidth { }
{% endhighlight %}


FInally we make our ```UIButton``` subclasses conform to these protocols:

{% highlight swift %}
class RegularButton : UIButton, ButtonAppearance { }
class CallToActionButton : UIButton, ButtonAppearance, CallToActionAppearance { }
{% endhighlight %}

Result:
![Example 3](/images/2015-07-29/protocol-ui-3.png)



And again, you can reuse these protocols for any other ```UIKit``` element:

{% highlight swift %}
class CallToActionTextField : UITextField, CallToActionAppearance { }
{% endhighlight %}

![Example 4](/images/2015-07-29/protocol-ui-4.png)



If you’re not happy with the predefined protocols, or you want more sophisticated customization, you can customize elements with a closure:

{% highlight swift %}
protocol SmartButtonApperance : CustomClosure { }
extension SmartButtonApperance { 
	var pCustomClosure : ProtocolUICustomClosure {    
        return { () -> Void in        
            if let aSelf = self as? UIButton {                
                aSelf.setTitleColor(UIColor.blackColor(), forState: .Normal)
                aSelf.setTitleColor(UIColor.redColor(), forState: .Highlighted)
            }
        }
    }
}

class MySmartButton : UIButton, ButtonAppearance, SmartButtonApperance { }
{% endhighlight %}

![Example 5](/images/2015-07-29/protocol-ui-custom-closure.gif)



#### Epilogue

ProtocolUI is nothing more than just a simple helper providing very basic infrastructure for customizing UI elements via protocol extensions. Unfortunately, I can’t test it in production use until Xcode 7 is officially released. And I won’t lie to you, **I can’t wait!**


[ProtocolUI source code](https://github.com/VojtaStavik/ProtocolUI)
