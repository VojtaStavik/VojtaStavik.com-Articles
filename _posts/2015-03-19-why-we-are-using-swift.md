---
layout: post
title: 4 reasons why we’re using Swift 
---

<h3>(Plus 1 reason why you may not want to)</h3>

I'm an iOS developer at [STRV](http://www.strv.com). Since the day Apple revealed Swift at the 2014 WWDC event, we immediately started playing around with this brand new language for iOS app development. We were more than excited about it! Though the language, and especially its development environment (Xcode), wasn’t ready for production use, we started using it last summer on few of our new projects.

<!-- more -->

It’s been 10 months since Swift was first unveiled, and version 1.2 is now out. By the end of the year, we expect it will be among the [Top 20 most popular programming languages in the world](http://www.engadget.com/2015/01/16/popularity-of-swift-is-on-the-rise-in-a-major-way/). We’ve learned a lot about Swift. All our new projects are now made in Swift (unless a client explicitly requests Objective-C), and we’re now even using it for new classes in our “old” Objective-C projects. Why? Here’s the top 4 reasons why we do it, plus one reason, why you may not want to.
      

### It’s more secure
All apps crash. There’s not much we can do about crashes caused by the operating system itself. But we can do our best (and we sure DO) to prevent crashes in our code. Swift is great with this! It’s much more secure than Objective-C, thanks to its optional values, type safety and a lot of small syntactic improvements. The whole “Optional Values” paradigm forces developers to think about their code in a different and safer way. So we can say, in general, apps written in Swift should crash less than apps in Objective-C. Lots of bugs are caught at the compiler level, while others can’t even be created because of a different language syntax and techniques.

     

### It’s more elegant
Swift was developed over the last several years, while Objective-C has been around since 1983. Even though it’s been modernized and new language features like blocks or modern syntax have been added, Objective-C dates back to a long-ago era when Duran Duran used to have #1 singles. Check out the following example of code doing the same thing in Objective-C and Swift. You don’t have to be an iOS engineer to see the difference.

{% highlight objc %}
// Objective-C

user.userId = @"unknown";
if ([object isKindOfClass:[NSDictionary class]]) {
    if ([[object objectForKey:@"user_id"] isKindOfClass:[NSString class]]) {
        user.userId = [object objectForKey:@"user_id"];
    }
}

{% endhighlight %}

{% highlight swift %}
// Swift:

user.userId = (object as? [NSObject: AnyObject])?["user_id"] as? String ?? "unknown"
{% endhighlight %}

     

### It’s the future
When Apple unveiled Swift, it essentially killed Objective-C. However, knowing Objective-C is still more important today (and will be for some time) than mastering Swift. All of the CocoaTouch API is written primarily in Objective-C, and it’s crucial for a developer to thoroughly understand it. The importance of Objective-C will most likely lessen as Swift becomes the leading language in the Apple environment. But I guess it will be many years before an iOS or Mac developer looks at you and asks: “Objective what?!” Soooo, if you want your code to be up to date for as long as possible, choose Swift. 

     

### It’s more challenging
If you’ve been developing apps for a while, you likely developed a set of code snippets, techniques and practices for basic tasks. These common tasks become a new adventure with Swift. You have to rethink all your practices and adjust them for a new language and its new features. And how cool is that!? Nobody likes doing the same things again and again. Learning Swift is super fun, because it’s a brand new language, and you’re getting in right at the start. It would be a shame to miss out.

     
### … but it’s also more complicated
This is all nice but all that glitters is not gold. There’s also a bunch of serious reasons why Swift isn’t the best choice for production apps. The language is still evolving. Basically, every new version of Xcode means you’ll have to make big or small changes in your old code. Although Xcode has been improved since the Swift launch, it still does not provide the same level of support for Swift that it does for Objective-C. Especially for a bigger projects. So if your time schedule is super tight or your project is really complicated or if you’re just not patient enough to tolerate SourceKitService crashing yet again, then Objective-C is still the best choice for you. And you don’t have be embarrassed about that.
