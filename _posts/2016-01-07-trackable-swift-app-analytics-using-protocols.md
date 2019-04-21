---
layout: post
title: Protocol based Swift app analytics - Trackable
---
I don’t know a single programmer who likes implementing an analytics tracking service into a project. It’s a boring manual job you usually do as the last thing before the app store release. Why is it such a pain?

![Can't someone else do it?](/images/2016-01-07/trackable-1.jpg)

<!-- more -->

**It ruins your perfect project structure.** If you want to track some properties, you have to have access to them at the time of the event tracking. So you end up with a situation where, for example, you would need a reference to a user object in a video player or you have additional delegates you don’t really need, just because of the event tracking.

**Event and property identifiers are plain text.** Mixpanel, Flurry and others all use string identifiers for events and properties. We try to avoid manually typed string identifiers in Swift as much as possible.

**It’s boooring.** It’s not creative work at all. You copy-paste lines of code, change identifiers and test if it’s tracked properly. Repeat those steps again and again for several hours (days) and you get what’s called analytics integration.

Our team faced the same issue when we integrated Mixpanel into our latest project - [Ripple](https://www.ripplenetwork.com). **That’s why we created Trackable - a simple analytics integration helper library.** It’s especially designed for easy and comfortable integration with existing projects.

See [Trackable GitHub project](https://github.com/VojtaStavik/Trackable).

---

### What did we want from Trackable?
- Easy integration to existing classes using extensions and protocols.
- Programmatically generated event and property identifiers.
- Smart tracking of properties by chaining of objects.
 *(if object “A” is set to be a parent of object “B”, event tracked on object “B” will also contain tracked properties from “A”)*

---

## Usage

Integration and usage of the **Trackable** library is very easy and straightforward. All code in this post is from the [example app](https://github.com/VojtaStavik/Trackable/tree/master/TrackableExample), where you can find complete implementation including all features.

### Events and Keys

Define events and keys using enums with ```String``` raw representation. These enums have to conform to ```Event``` or ```Key``` protocols. You can use nesting for better organization. String identifiers are created automatically, and they respect the complete enums structure. Example: ```Events.App.started``` will be translated to ```“<ModuleName>.Events.App.started”``` string.

{% highlight swift %}
enum Events {
    enum User : String, Event {
        case didSelectBeatle
        case didSelectAlbum
        case didRateAlbum
    }
    
    enum App : String, Event {
        case started
        case didBecomeActive
        case didEnterBackground
        case terminated
    }
    
    enum AlbumListVC : String, Event {
        case didAppear
    }
}

enum Keys : String, Key {
    case beatleName
    case albumName
    case userLikesAlbum
    case previousVC
    
    enum App : String, Key {
        case uptime
        case reachabilityStatus
    }
}
{% endhighlight %}

### Tracking

You can track events on any class conforming to the ```TrackableClass``` protocol by calling ```self.track(event: Event)```. You can also call ```self.track(event: Event, trackedProperties: Set<TrackedProperty>)``` if you want to add some specific properties to the tracked event.

```TrackedProperty``` is a struct you can create using a custom infix operator ```~>>``` with ```Key``` and value. Allowed value types are ```String```, ```Double```, ```Int```, ```Bool``` and ```Set<TrackedProperty>```.

{% highlight swift %}
// Example:

import UIKit
import Trackable

class AlbumDetailVC: UIViewController {

    var album : Album!

    @IBOutlet weak var yesButton: UIButton!
    @IBOutlet weak var noButton: UIButton!
    
    @IBAction func didPressButton(sender: UIButton) {
        let userLikesAlbum = (sender === yesButton)
        track(Events.User.didRateAlbum, trackedProperties: [Keys.userLikesAlbum ~>> userLikesAlbum])
    }
}

extension AlbumDetailVC : TrackableClass { }
{% endhighlight %}


### Tracked properties

**Trackable** is designed to allow you to easily track all properties you need. There are three levels, where you can add custom data to tracked events. If you add a property with the same name to the same event multiple times, it will override the previous value with a lower priority level.

---

#### Level 3
-   when calling track()function
-   properties on this level will be added only to the currently tracked event
-   **Typical usage:** When you want to track properties closely connected to the event.

{% highlight swift %}
// Example:
track(Events.User.didRateAlbum, trackedProperties: [Keys.userLikesAlbum ~>> userLikesAlbum])
{% endhighlight %}

---

#### Level 2
-   instance properties added by calling ```setupTrackableChain(trackedProperties:)``` on a ```TrackableClass``` instance.
-   these properties will be added to all events tracked on the object
-   **Typical usage:** When you want to set properties from the outside of the object (the object doesn’t know about them)

{% highlight swift %}
// Example:
import UIKit
import Trackable

class AlbumListTVC: UITableViewController {
    … code … 

    // MARK: - Navigation
    override func prepareForSegue(segue: UIStoryboardSegue, sender: AnyObject?) {
        let destinationVC = segue.destinationViewController as! AlbumDetailVC
        destinationVC.setupTrackableChain([Keys.previousVC ~>> "Album list"]) // all events tracked on destinationVC will have previousVC property included automatically
    }
}

extension AlbumListTVC : TrackableClass { }
{% endhighlight %}

---

#### Level 1
-   computed properties added by custom implementation of the ```TrackableClass``` protocol
-   these properties will be added to all events tracked on the object
-   **Typical usage:** When you want to add some set of properties to all events tracked on the object.

{% highlight swift %}
// Example:
class AlbumListTVC: UITableViewController {
    var albums : [Album]!

    override func tableView(tableView: UITableView, didSelectRowAtIndexPath indexPath: NSIndexPath) {
        track(Events.User.didSelectAlbum) // selectedAlbum property will be added automatically
    }

    var selectedAlbum : Album? {
        if let indexPath = tableView.indexPathForSelectedRow {
            return albums[indexPath.row]
        } else {
            return nil
        }
    }
}

extension AlbumListTVC : TrackableClass {
    var trackedProperties : Set<TrackedProperty> {
        return [Keys.albumName ~>> selectedAlbum?.name ?? "none"]
    }
}
{% endhighlight %}

---

### Chaining
**The real advantage of Trackable comes with chaining.** You can set one object to be a Trackable parent of another object. If class A is a parent of class B, all events tracked on B will also automatically include ```trackedProperties``` from A.

{% highlight swift %}
// Example:
import UIKit
import Trackable

class AlbumListTVC: UITableViewController {

    … some code here … 

    // MARK: - Navigation
    override func prepareForSegue(segue: UIStoryboardSegue, sender: AnyObject?) {
        let destinationVC = segue.destinationViewController as! AlbumDetailVC
        destinationVC.setupTrackableChain([Keys.previousVC ~>> "Album list"], parent: self)
    }
    
    var selectedAlbum : Album? {
        if let indexPath = tableView.indexPathForSelectedRow {
            return albums[indexPath.row]
        } else {
            return nil
        }
    }
}

extension AlbumListTVC : TrackableClass {
    var trackedProperties : Set<TrackedProperty> {
        return [Keys.albumName ~>> selectedAlbum?.name ?? "none"]
    }
}

// All events tracked later on destinationVC will automatically have previousVC and albumName properties,
// without destinationVC even knowing those values exist!
{% endhighlight %}


### Connecting to Mixpanel (or any other service)
In order to perform the actual tracking into an analytics service, you have to provide an implementation for ```Trackable.trackEventToRemoteServiceClosure```.

{% highlight swift %}
// Example:
import Foundation
import Mixpanel
import Trackable

let analytics = Analytics() // singleton (yay!)

class Analytics {
    let mixpanel = Mixpanel.sharedInstanceWithToken("<token>")

    init() {
        Trackable.trackEventToRemoteServiceClosure = trackEventToMixpanel
        setupTrackableChain() // allows self to be part of the trackable chain
    }
    
    func trackEventToMixpanel(eventName: String, trackedProperties: [String: AnyObject]) {        
        mixpanel.track(eventName, properties: trackedProperties)
    }
}

extension Analytics : TrackableClass { }
{% endhighlight %}

Maybe you want to add some properties to all events tracked in your app. **It’s similar to Mixpanel super properties but with dynamic content!** You need to provide a custom implementation of the ```TrackableClass``` protocol:

{% highlight swift %}
extension Analytics : TrackableClass {
    var trackedProperties : Set<TrackedProperty> {
        return [Keys.App.uptime ~>> NSDate().timeIntervalSinceDate(startTime)]
    }
}
{% endhighlight %}
 and set the analytics object as a parent to all objects without a parent by calling ```setupTrackableChain(parent: analytics)``` on them:

{% highlight swift %}
// Example:
import UIKit
import Trackable

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {
    var window: UIWindow?
    
    func application(application: UIApplication, didFinishLaunchingWithOptions launchOptions: [NSObject : AnyObject]?) -> Bool {
        setupTrackableChain(parent: analytics)
        return true
    }

    func applicationDidBecomeActive(application: UIApplication) {
        track(Events.App.didBecomeActive)
    }
    
    func applicationDidEnterBackground(application: UIApplication) {
        track(Events.App.didEnterBackground)
    }
    
    func applicationWillTerminate(application: UIApplication) {
        track(Events.App.terminated)
    }

}

extension AppDelegate : TrackableClass { }
{% endhighlight %}


### See example project
I would recommend you to go through the [example project](https://github.com/VojtaStavik/Trackable/tree/master/TrackableExample) to see some more examples of how you can use **Trackable**. 

![Example app screenshot](/images/2016-01-07/trackable-2.png)

We’re using Trackable successfully for [Ripple](https://www.ripplenetwork.com). With more than 100 events, it took us just 2 days of work to completely switch from the old tracking system to **Trackable**. It started to pay off from the very first moment we did it.

[Trackable GitHub project and source code](https://github.com/VojtaStavik/Trackable)
