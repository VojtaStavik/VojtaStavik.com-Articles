---
layout: post
title: Another world of Notifications - Darwin Notifications
---
As an iOS developer, you are probably familiar with the concept of notifications and the way it’s represented by ```NSNotification``` and ```NSNotificationCenter``` in CocoaTouch. These objects are part of the Foundation framework. Because Foundation is built on top of the CoreFoundation framework, it’s not surprising that there is a CF “version” of ```NSNotificationCenter``` called ```CFNotificationCenter```. It can be super fun to use it, because it offers you some cool additional functionality.

![Darvin approved!](/images/2016-02-17/darwin.jpg)

<!-- more -->

All ```NSNotification```s you send are distributed just within an app’s current process. It means you won’t receive notifications send by other apps, and your notifications are not sent to other apps. This is true for ```NSNotificationCenter```. However, there are two (three on Mac) types of ```CFNotificationCenter```s. A “local” notification center distributes notifications the same way ```NSNotificationCenter``` does, yet the “Darwin” notification center distributes notifications across the whole system!
![WOW](/images/2016-02-17/wow.gif)

This means you can send and receive notifications between different apps, and/or an app and its extensions. See for example [MMWormhole](https://github.com/mutualmobile/MMWormhole), which is using Darwin notifications for real time communication between the iPhone and Watch app.

```CFNotificationCenter``` is a C-based API, so using it in Swift is a bit adventurous. If you like challenges (or you are [Chris Eidhof](https://www.youtube.com/watch?v=-ag-f9N8SJE)), you can do it manually. However, for simple experimenting I would suggest you use [CCHDarwinNotificationCenter](https://github.com/choefele/CCHDarwinNotificationCenter), which wraps Darwin notifications to an object based NS API.

There is a bunch of [known private system notifications](http://iphonedevwiki.net/index.php/SpringBoard.app/Notifications) distributed by the Darwin notification center, which you can listen to and make some cool stuff. See the video below and [example code on GitHub](https://github.com/VojtaStavik/DarwinNotifications-Example). Of course, this is probably not AppStore safe.

<p>
<div class="videoWrapper">
    <iframe width="560" height="349" src="http://www.youtube.com/embed/99YFpnnqgc4" frameborder="0" allowfullscreen></iframe>
</div>
</p>
