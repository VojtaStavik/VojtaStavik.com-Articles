---
layout: post
title: How to open the parent app in the foreground from the WatchKit app
---

This post could be very short. You simply cannot open the parent app from the WatchKit app in the foreground. That’s all. Thank you for reading my blog.

…
But wait. Seriously. What if we really need to open the parent app in the foreground? For example, when we need a user to login using their Facebook account to be able to finish an order. Facebook SDK cannot run in the watch extension, so there’s really no way how to login a user directly from the watch app.

Well, what we can do is to help users with opening our app. What we are looking for is a technology called **Handoff**.

<!-- more -->

The idea behind Handoff is simple. You start your work on one device and seamlessly continue working on another device. This is almost what we need. Our user starts the work on the watch and at some point we need him to switch to the parent app, perform login and finish the order. 

Though we can’t force open the parent app, iOS8 offers a shortcut. You’ve probably noticed it before. It’s the Handoff icon in the bottom left corner of the lock screen.

![Handoff icon](/images/2015-04-30/handoff-1.png)


This icon appears when these conditions are met: 

- There is some device with the same iCloud account present; 
- The device is reachable via Bluetooth LE 4.0;
- There’s also some app on the other device which broadcasts current user activity and (finally)
- One of your apps installed on your phone is signed up for handling this kind of activity. 

**Pretty specific, huh?**

In the pre-watch times, this scenario didn’t happen often. To be honest, I had never seen another app besides Safari or Mail appeared on the lock screen. But *the times they are a changin’.* The Apple Watch meets all the conditions required for Handoff by default. It almost seems that the watch was the main reason why Handoff was created.

So, let’s play with Handoff a bit. **This is our goal:**
<p>
<div class="videoWrapper">
    <iframe width="560" height="349" src="http://www.youtube.com/embed/EOiDrgN_pV4" frameborder="0" allowfullscreen></iframe>
</div>
</p>

Imagine you want to order a shirt. You can choose the color and the size of your new shirt in the watch app. Then you open the parent app and continue with the order there. I know it doesn’t make a sense, but let’s pretend it’s the most convenient way to order new shirts. The point is, the app opens on the same screen and in the same selected state as it was in the watch app.

Please download the [source code of the project](https://github.com/VojtaStavik/WatchKitHandoffTest) from my GitHub. I won’t put all the necessary code here. Just the most important parts.

*Note: You will need actual Apple Watch to test Handoff. iOS simulator doesn't allow it (so far).*


### Part 0 - Preparation

Each activity we want to handle has to have a unique string identifier. Common practice is to use a reverse domain notation. These identifiers are shared between the watch app extension and the regular app. I like creating a ```struct``` where these values are stored, so you don’t have to copy/paste them later in the code. I put this ```struct``` in a new file. Don’t forget to include this file in both the regular app and the watch extension targets.

We need 2 activities - “choose color” and “choose size”.

{% highlight swift %}
struct ActivityKeys {
    static let ChooseColor = "com.vojtastavik.handofftest.ChooseColor"
    static let ChooseSize = "com.vojtastavik.handofftest.ChooseSize"
}
{% endhighlight %}


### Part 1 - Watch App

 ![Watch app screen 1](/images/2015-04-30/handoff-2.png) 


Updating user activity from the watch app is very straight forward. When a user presses one of the buttons, we simply call:

{% highlight swift %}
WKInterfaceController.updateUserActivity(type: String, userInfo: [NSObject : AnyObject]?, webpageURL: NSURL?)
{% endhighlight %}


The real code then looks like:
{% highlight swift %}
WKInterfaceController.updateUserActivity(ActivityKeys.ChooseColor, userInfo: ["current": 0], webpageURL: nil)
{% endhighlight %}


Notice the name of activity and the user info we’re sending. The name should be ```.ChooseColor``` or ```.ChooseSize```, according to the current interface controller. In the ```userInfo``` dictionary, we’re passing the current selection index. We will use this info later in the iPhone app for the initial value setting. 

Believe it or not, that’s all we have to do in the WatchKit app.





### Part 2 - iPhone App

This is what our app looks like. We have one tab bar controller and two detail controllers: ```ChooseColorVC``` and ```ChooseSizeVC```. Each detail view controller has a segmented control.

 ![iPhone app storyboard](/images/2015-04-30/handoff-3.png) 


To let iOS know that our app can handle certain kinds of user activity, we have to add these keys to the ```info.plist``` file. This is the place where you have to copy/paste the value. If you change the name of activities or if you add new ones, don’t forget to update the names here, too!

 ![info plist file](/images/2015-04-30/handoff-4.png) 

Then we have to add one method to our app delegate class. This method handles incoming request for continuing user activity.

{% highlight swift %}
@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {
    var window: UIWindow?

    func application(application: UIApplication, didFinishLaunchingWithOptions launchOptions: [NSObject: AnyObject]?) -> Bool {
        // Override point for customization after application launch.
        return true
    }

    func application(application: UIApplication, continueUserActivity userActivity: NSUserActivity, restorationHandler: ([AnyObject]!) -> Void) -> Bool {
        window?.rootViewController?.restoreUserActivityState(userActivity)
        return true
    }
}
{% endhighlight %}

We handle the request by forwarding it to our root view controller. *The best way to solve problems, isn’t it?*

Our root controller is of course our ```UITabBarController```. Let’s handle the request here:

{% highlight swift %}
class MainTabBarController: UITabBarController {
    override func restoreUserActivityState(activity: NSUserActivity) {
        switch activity.activityType {
        case ActivityKeys.ChooseColor:
            if let colorVC = viewControllers?[0] as? ChooseColorVC {
                colorVC.restoreUserActivityState(activity)
                selectedViewController = colorVC
            }

        case ActivityKeys.ChooseSize:
            if let sizeVC = viewControllers?[1] as? ChooseSizeVC {
                sizeVC.restoreUserActivityState(activity)
                selectedViewController = sizeVC
            }
            
        default:
            println("Unknown activity")
        }
    }
}
{% endhighlight %}


We check the ```NSUserActivity``` type, and if it’s something we know, we open the certain view controller. Notice that we pass the ```NSUserActivity``` object to that view controller, too.

We use it there for setting up the initial value of segmented controls:

{% highlight swift %}
class ChooseColorVC: UIViewController {

    var initialValue = 0
    
    @IBOutlet weak var segmentedControl: UISegmentedControl!
    
    override func viewDidLoad() {        
        super.viewDidLoad()
        segmentedControl.selectedSegmentIndex = initialValue
    }
    
    override func restoreUserActivityState(activity: NSUserActivity) {
        if let currentSelection = activity.userInfo?["current"] as? Int {            
            initialValue = currentSelection
            segmentedControl?.selectedSegmentIndex = currentSelection
        }
    }
}
{% endhighlight %}

As you can see, we have to set the segmented control value in two different places. In addition to setting it as an initial value in ```viewDidLoad()``` , we also have to set it directly to the segmented control. It’s because we want to be sure that the value is updated even when the app was already running in the background. We use ```segmentedControl?``` (yes, with a question mark), because if the app wasn’t running in the background, ```segmentedControl``` is not initialized at the time ```restoreUserActivityState``` method is called.

### We're done!

Handoff used to be one of the things Apple revealed but had no real use for. After you spend a few days with the Apple Watch, you will realize how important and useful this technology will become.

[Source code of the project](https://github.com/VojtaStavik/WatchKitHandoffTest)
