---
layout: post
title: Namespacing UITableView and Where to Draw the Line
filename: "2017-11-14-namespace-all-the-things.md"
---

Objective-C was the first programming language I learned and used professionally. One of the first things you need understand, when working with ObjC, is the lack of namespaces. The whole Objective-C runtime acts like one namespace. To prevent name collisions, Objective-C uses name prefixes. You simply create your own pseudo-namespace by prefixing all your class' names. That's the reason why we have names like **UI**View, **NS**Object and **MK**MapView.

![Autocompletion](/images/2017-11-14/autocompletion.png)

Swift has module defined namespaces so name prefixes are not needed anymore. *I have to say I miss them. Having your initials as a part of your class' name brings a strangely satisfying feeling of ownership.*

**Let's do a little thought experiment.** ```UITableView``` is one of the cornerstones of iOS development. Introduced in iOS 2.0, it's one of the oldest API we still actively use. What could ```UITableView``` API names look like if it was introduced in 2017 with Swift only support? Let's play around with namespacing.

<!-- more -->

Since ```UIKit``` has it's own namespace, we can simply use ```Table``` as the "base" namespace for table view related types. If you would already have a type named ```Table``` in your module, you'd need to refer to the UIKit version as ```UIKit.Table```. However, in most of the cases a simple ```Table``` would do the work.

Simulating this in the playground is quite easy:

{% highlight swift %}
enum UIKit { // UIKit namespace simulation
  enum Table { // The new "base" namespace for UITableView types
    typealias View = UITableView
    typealias ViewController = UITableViewController
    typealias Cell = UITableViewCell
  }
}

// If there isn't a type named 'Table' in the current module,
// we can refer to 'UIKit.Table' by a simple 'Table'. This
// typealias simulates the desired behavior.
typealias Table = UIKit.Table
{% endhighlight swift %}

This is how you would use the **UITableView 2017 Swift-only edition** in code:

{% highlight swift %}
class MyTableView: Table.View { }

class MyTableViewController: Table.ViewController { }

// You could also refer to the type by its full namespace hierarchy
class MyTableViewCell: UIKit.Table.Cell { }

{% endhighlight swift %}

What about the two most famous protocols in the iOS world ```UITableViewDataSource``` and ```UITableViewDelegate```? Let's give them a proper namespace, too:

{% highlight swift %}
// Namespace simulation
extension UIKit.Table {
  typealias DataSource = UITableViewDataSource
  typealias Delegate = UITableViewDelegate
}

// Usage
extension MyClass: Table.DataSource {
  func tableView(_ tableView: Table.View, cellForRowAt indexPath: IndexPath) -> Table.Cell {
  }
}

extension MyClass: Table.Delegate {
}

{% endhighlight swift %}

Let's not stop here. There's so much more we can namespace!

![Namespace all the things!](/images/2017-11-14/namespace-all-the-things.jpg)

{% highlight swift %}
// Namespace simulation
extension UIKit.Table.Cell {
  typealias Style = UITableViewCellStyle
  typealias SeparatorStyle = UITableViewCellSeparatorStyle
  typealias SelectionStyle = UITableViewCellSelectionStyle
}

// Usage
let cellStyle: Table.Cell.Style = .default
let separatorStyle: Table.Cell.SeparatorStyle = .singleLine
let selectionStyle: Table.Cell.SelectionStyle = .blue

{% endhighlight swift %}

You probably get the idea... The real question here is:

### Is this what we want?

Is it a good idea to namespace all the things? It's quite nice to have the types structured like this. The hierarchy is clearly visible and the global namespace is less *"polluted"*. The shorter names are also less scary to newcomers.

Let's think about drawbacks, too. Reasoning about the types can be more complicated, especially in cases where the are name conflicts. ```Table``` can sometimes mean ```UIKit.Table``` and sometimes not, depending on the context. Autocompletion for nested types currently only works within one namespace "level", so you currently can't do something like this for nested type:

![Autocompletion](/images/2017-11-14/autocompletion-2.png)

### Where to draw the line?

The main inspiration for writing this post comes from the project I'm working on. I realized I tend to pick Objective-C like names for types, even though using Swift's powerful namespacing features would be probably much better.

I don't have a strong opinion on which approach is better, neither where is the line for "you namespace things too much". I'd be glad to hear more opinions on this topic! Feel free to leave a comment below.

See also [gist](https://gist.github.com/VojtaStavik/700f9e925cd52058fac6eec5c2f3804b) with the sample code from this blog post.
