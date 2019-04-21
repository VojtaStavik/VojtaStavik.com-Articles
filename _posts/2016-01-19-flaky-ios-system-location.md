---
layout: post
title: Flaky iOS System Location - There and Back again
---
A very strange thing started happening in our office over the last few weeks. Several iPhones during the day randomly traveled to London and back to Prague. At least, that’s what the system location info was saying. 

![London or Prague?](/images/2016-01-19/location-1.jpg)
*London or Prague?*

<!-- more -->

It was super frustrating for our teammate [Patrik](https://twitter.com/vaberer), who’s from Slovakia, and doesn’t know Prague well. He wasn’t able to use navigation on his phone, because the phone was in London. The poor guy slept several nights in the office, because he couldn’t find his way home :)

#### What the f*** is happening?

It was only happening in the corner of the office where our team tables are, and because we’re all engineers, we decided to figure out what was going on. 

#### Wi-Fi? 
Our first thought was that the cause of the problem was our Wi-Fi. If the Wi-Fi location is (somehow) recorded incorrectly and an iPhone uses it for finding a location, what was happening to us would make sense! And it did! When we turned off the Wi-Fi and restarted our devices, they were all in Prague. **Problem solved!** … or was it?

![The End. Or is it?](/images/2016-01-19/location-2.jpg)


#### What do we have in common? 
We were happy with our explanation until it was proven wrong. One device moved to London again, but the Wi-Fi on the device was turned off. We were back to square one. I’m a big fan of House M.D., so after we turned down the idea of a **space-time wormhole** between STRV’s Prague office and Piccadilly Circus, the next question was obvious -- **what do we have in common?**

![You're good!](/images/2016-01-19/location-3.gif)

We realized that the location jumping was not happening to everyone sitting in our office corner.  It was just our team. It wasn’t related to the office location but to the project. **Suddenly, it was all clear:**

### If you turn on a location simulation when you are debugging a project on a real device, it will change the location for the whole system!

<br/>We didn’t know that. So when one of my team members put London as the default value after checking “Allow Location Simulation” several weeks ago, it wasn’t used just for the simulator, but for all the real devices we were using. Because it was a shared scheme, it affected all of us.

![Xcode](/images/2016-01-19/location-4.png)

I recorded a short video showing the process:
<p>
<div class="videoWrapper">
    <iframe width="560" height="349" src="http://www.youtube.com/embed/U9WkiB7jbJE" frameborder="0" allowfullscreen></iframe>
</div>
</p>

Every time we debugged our project on a real device, the system location was changed to London. It jumped back to the real location when we left the office likely because of the significant location change update. **The London location mystery is finally solved, and Patrik manages to make it home every night.**

#### Imagine the possibilities …
I’m not sure if this behavior is a bug or feature. Seems to me more like a feature, but I can’t really imagine a single case where you would need to make the whole system using a simulated location. Can you? Except for maybe using it to prank one of your colleagues!

![The art of trolling](/images/2016-01-19/location-5.jpg)
