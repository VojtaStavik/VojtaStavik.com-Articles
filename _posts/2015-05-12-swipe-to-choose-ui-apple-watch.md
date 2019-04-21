---
layout: post
title: Swipe to choose (aka Tinder)
---

I joined the [WatchKit hackaton](https://www.facebook.com/media/set/?set=a.10153172562140804.1073741833.116155005803&type=1) we organized in our office last weekend, and it was awesome! The project I worked on was a WatchKit app called “FaceMatch”. Our company is growing really fast and sometimes it’s difficult to remember all the new faces and names. The FaceMatch app shows you a random photo and name, and you have to decide whether the name matches the photo. This is a cool and fun way to learn new colleagues’ names!

I wanted to use the well-known **Tinder-like “swipe to choose”** UI. At the beginning, it seemed impossible to implement it because of the very limited WatchKit UI. **We don’t even have gestures!** However, after several iterations, this is what I came up with:

![Swipe to choose (aka Tinder)](/images/2015-05-12/swipe-to-choose-1.gif)

<!-- more -->

It’s pretty clear from the video that I used a page-based navigation to **“fake” swipe gestures**. The storyboard for the app looks like this:

![Swipe to choose - Storyboard](/images/2015-05-12/swipe-to-choose-2.png)

We have three ```WKInterfaceController```s connected with page-based segues. The left and right ones have full screen YES and NO buttons. I choose buttons, because, unlike labels, I could set their background color directly. They work just as static assets, so it could be practically anything you want (image, label …).

Let’s start with the NO ```WKInterfaceController```. It’s pretty straight-forward:

{% highlight swift %}
class NOIC: WKInterfaceController {
    override func willActivate() {
        super.willActivate()
        NSNotificationCenter.defaultCenter().postNotificationName(ShowMiddleControllerNotificationName, object: nil)
    }
}
{% endhighlight %}

I noticed that the controller’s  ```willActivate()``` function is called up when the page controller finishes its scroll animation to that particular controller. That’s exactly what I needed. So, when the scroll is finished and the app should proceed to the next image, I just send a notification about it.

The notification is handled in the middle ```WKIntefaceController```.

{% highlight swift %}
let ShowMiddleControllerNotificationName = "ShowMiddleControllerNotificationName"

class InterfaceController: WKInterfaceController {

    @IBOutlet weak var imageInterface: WKInterfaceImage!
    @IBOutlet weak var labelInterface: WKInterfaceLabel!
    
    override func awakeWithContext(context: AnyObject?) {
        super.awakeWithContext(context)
        
        NSNotificationCenter.defaultCenter().addObserverForName(ShowMiddleControllerNotificationName, object: nil, queue: NSOperationQueue.mainQueue()) { ( _ ) -> Void in
            dispatch_async(dispatch_get_main_queue(), { () -> Void in
                if let beatle = game.next() {
                    self.labelInterface.setText(beatle.name)
                    self.imageInterface.setImage(beatle.image)
                }
            })
            
            self.becomeCurrentPage()
        }
    }
}
{% endhighlight %}

When the middle controller receives the notification, it updates the image and label with the next random "Beatle" and calls ```self.becomeCurrentPage()```. This forces the page controller to scroll back to the middle controller.


### But ...

… there’s one obvious problem. The first interface controller in the page navigation is the YES controller. Since there’s no way (so far) of how to set what should be the initial controller, the app always starts by showing the YES button before scrolling to the middle controller. It’s a bit confusing for users, so I tried to fix it.

{% highlight swift %}
class YESIC: WKInterfaceController {
    override func awakeWithContext(context: AnyObject?) {
        super.awakeWithContext(context)
        // We want to hide YES button when the controller is loaded for the first time
        button.setAlpha(0)
    }

    @IBOutlet weak var button: WKInterfaceButton!
    
    override func willActivate() {
        super.willActivate()
        
        NSNotificationCenter.defaultCenter().postNotificationName(ShowMiddleControllerNotificationName, object: nil)
        button.setAlpha(1.0)
    }
}
{% endhighlight %}


In the ```awakeWithContext()``` function, I set the button’s ```alpha``` value to ```0```, so the button is not visible until the loading of the controller is finished, and ```willActivate()``` is called. Obviously, the ```willActivate()``` function is also the place where ```ShowMiddleControllerNotification``` is posted, which causes the page controller to scroll to the middle controller. It seems that the button’s ```alpha``` value change takes more time than the page change. Thanks to this, the user doesn’t see the YES button when the app starts.




### We're done!

I was a bit surprised how easy it was to implement this thing. WatchKit is a very limited framework these days, but it still has some powerful features. I think we will see a lot of interesting and surprising usages of WatchKit UI elements in the next months. Consider how much you can achieve with group nesting! Also, a lot of WatchKit elements (groups included) can have animated images for backgrounds, which is a quite powerful thing, too. 

[Source code of the project](https://github.com/VojtaStavik/SwipeToChooseWatchKit)
