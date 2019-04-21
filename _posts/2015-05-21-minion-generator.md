---
layout: post
title: MinionGenerator &#35;Banana
---

Since the day I read this [article](http://natashatherobot.com/watchkit-create-table/) and saw [Natasha](https://twitter.com/NatashaTheRobot) was using Minions for placeholder data, I started to do it, too. It’s hilarious to see these funny creatures on the places where users, clients or even bank officers are supposed to be.

![Kevin Loan Officer](/images/2015-05-21/minions-1.png)

But how you get a Minion when you need him? Oh yeah, you need a **Minion generator**!

<!-- more -->

```MinionGenerator``` is a ```struct``` with a few static functions:
{% highlight swift %}

// returns one Minion
static func randomMinion() -> Minion 

// returns array of random Minions
static func minions(count: Int) -> [Minion]

{% endhighlight %}


```Minion```, then, is a ```class``` looking like this:
{% highlight swift %}

class Minion: Equatable {
   let name: String
   let profilePicture: UIImage
   let profilePictureURL: NSURL
}

{% endhighlight %}


### Example

Usage is very simple:
{% highlight swift %}

let minion = MinionGenerator.randomMinion()
let arrayOfMinions = MinionGenerator.minions(15)

{% endhighlight %}


```UITableViewController``` with Minions:

![Minions example](/images/2015-05-21/minions-example.gif)




### Other assets

If you don’t need Minions, but some other random Minion-related data, you can use the following functions (their names are self-explanatory):

{% highlight swift %}

static func randomSentence() -> String
static func randomParagraph(sentences: Int) -> String     

static func randomPicture() -> UIImage 
static func randomPictures(count: Int) -> [UIImage]

static func randomPrictureURL() -> NSURL 
static func randomPictureURLs(count: Int) -> [NSURL] 

{% endhighlight %}





### “Gelato!”

It’s obvious that once you start using MinionGenerator, your life will never be the same. Can’t wait? You can find the project’s GitHub repository [here](https://github.com/VojtaStavik/MinionGenerator). (*EDITED*) The preferable way how to install MinionGenerator is by using [Carthage](https://github.com/Carthage/Carthage). Add this line to your ```Cartfile```

    github "VojtaStavik/MinionGenerator"


 <strike>Simply add this repository as a Git submodule, add ```MinionGenerator.swift``` and ```Minions.xcassets``` files to the Xcode project **(without copying it)**, and you’re good to go!</strike>
 
 

*Note: This project should be used only for testing purposes. The code isn’t bulletproof at all, and it’s meant to crash if something goes to an undefined state (aka: exclamation marks everywhere ;-))*

[Source code of the project](https://github.com/VojtaStavik/MinionGenerator)
