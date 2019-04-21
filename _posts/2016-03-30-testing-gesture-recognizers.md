---
layout: post
title: Testing Gesture Recognizers
filename: "2016-03-30-testing-gesture-recognizers.md"
---

Testing code that interacts with UIKit is a pain. UIKit is full of hidden states and dependencies, so you soon end up fighting the framework and trying to bend it the least hackiest way possible. Because it’s always easier to blame someone else, the test-unfriendly environment that Apple has built for us could be one of the reasons why iOS developers are still not yet such passionate unit testers.

![I find your lack of unit tests disturbing](/images/2016-03-30/testing-gesture-recognizers.jpg)

I was recently searching for a way of how to easily test the interaction between ```UIViewController``` and ```UIPanGestureRecognizer```. It was easier than I thought! In fact, I ended up very surprised by the readability and brevity of the final code.

<!-- more -->

First, let me show you the app we’re going to test:

![App demo gif](/images/2016-03-30/testing-gesture-recognizers-1.gif)

As you can probably figure out just from the GIF above, the app is built around ```UIPanGestureRecognizer```. This is what the only Storyboard in the project looks like:

![Storyboard screenshot](/images/2016-03-30/testing-gesture-recognizers-2.png)


The green square has a constraint for its width and a constraint for keeping the 1:1 aspect ratio. There are two more constraints for centering the square view with the view controller’s view.

This is the full code of the app:

{% highlight swift %}
import UIKit

class ViewController: UIViewController {

    @IBOutlet weak var square: UIView!
    @IBOutlet weak var widthConstraint: NSLayoutConstraint!
    @IBOutlet weak var centerXconstraint: NSLayoutConstraint!
    @IBOutlet weak var centerYconstraint: NSLayoutConstraint!

    override func viewDidLoad() {
        super.viewDidLoad()
        square.addGestureRecognizer(panGestureRecognizer)
    }

    lazy var panGestureRecognizer: UIPanGestureRecognizer = {
        let recognizer = UIPanGestureRecognizer(target: self, action: #selector(ViewController.handlePan))
        return recognizer
    }()

    func handlePan(sender: UIPanGestureRecognizer) {
        switch sender.state {
        case .Began:
            UIView.animateWithDuration(0.22, animations: {
                self.widthConstraint.constant = 100
                self.view.layoutIfNeeded()
            })

        case .Changed:
            let translation = sender.translationInView(self.view)
            self.centerXconstraint.constant = translation.x
            self.centerYconstraint.constant = translation.y
            self.view.layoutIfNeeded()

        case .Ended, .Cancelled:
            UIView.animateWithDuration(0.22, animations: {
                self.widthConstraint.constant = 50
                self.centerXconstraint.constant = 0
                self.centerYconstraint.constant = 0
                self.view.layoutIfNeeded()
            })

        default:
            break
        }
    }
}
{% endhighlight %}

The view controller has just one method where we handle the actions from ```UIPanGestureRecognizer```:

- When the pan gesture starts, we make the square bigger.
- When the finger position changes, we move the square accordingly.
- When the gesture finishes or is cancelled, we reset all constraints to the original state.

It’s so easy and straightforward, it’s begging to be tested :)

### 3 ... 2 ... 1 ...

One of my favorite patterns I use for testing ```UIButton``` interactions is to call ```sendActionsForControlEvents``` method. It’s very close to real-world usage, and you test not just the event handling but also the correct setup of the button’s target and action.

Unfortunately, we can't do that for gesture recognizers. Target and action are not even a part of ```UIGestureRecognizer```’s public API. We need to fix that. We also need to create a way to simulate user interaction. I haven’t found any better solution than subclass ```UIPanGestureRecognizer``` and add the ```perfomTouch``` function:

{% highlight swift %}
class TestablePanGestureRecognizer: UIPanGestureRecognizer {
    let testTarget: AnyObject?
    let testAction: Selector

    var testState: UIGestureRecognizerState?
    var testLocation: CGPoint?
    var testTranslation: CGPoint?

    override init(target: AnyObject?, action: Selector) {
        testTarget = target
        testAction = action
        super.init(target: target, action: action)
    }

    func perfomTouch(location: CGPoint?, translation: CGPoint?, state: UIGestureRecognizerState) {
        testLocation = location
        testTranslation = translation
        testState = state
        testTarget?.performSelector(testAction, onThread: NSThread.currentThread(), withObject: self, waitUntilDone: true)
    }

    override func locationInView(view: UIView?) -> CGPoint {
        if let testLocation = testLocation {
            return testLocation
        }
        return super.locationInView(view)
    }

    override func translationInView(view: UIView?) -> CGPoint {
        if let testTranslation = testTranslation {
            return testTranslation
        }
        return super.translationInView(view)
    }

    override var state: UIGestureRecognizerState {
        if let testState = testState {
            return testState
        }
        return super.state
    }
}
{% endhighlight %}

The goal wasn’t to create a mock recognizer. I wanted to “extend” the pan gesture recognizer and give it a better API for use within unit tests. This subclass exposes target and action properties so we can call them programmatically. It also allows you to inject values the recognizer should return for ```locationInView```, ```translationInView``` and ```state```.

We can now use this testable recognizer as a drop-in replacement in the ```ViewController```, and everything will still work perfectly:

{% highlight swift %}
lazy var panGestureRecognizer: UIPanGestureRecognizer = {
    let recognizer = TestablePanGestureRecognizer(target: self, action: #selector(ViewController.handlePan))
    return recognizer
}()
{% endhighlight %}

You can prepare “testable” subclasses of all gesture recognizers you need and share them among your projects. It’s a code you write just once, and you don’t need to change it.


### Test!

Finally, writing tests with this ```UIPanGestureRecognizer``` subclass is then a simple task. Notice that we use almost the same amount of code for the implementation and for its tests:

{% highlight swift %}
import Quick
import Nimble

class ViewControllerTests: QuickSpec {
    override func spec() {
        describe("View Controller") {
            var vc: ViewController!

            beforeEach {
                vc = UIStoryboard(name: "Main", bundle: NSBundle(forClass: ViewController.self))
                    .instantiateInitialViewController() as! ViewController
                _ = vc.view // load view
            }

            describe("pan gesture recognizer") {
                var recognizer: TestablePanGestureRecognizer!

                beforeEach({
                    recognizer = vc.panGestureRecognizer as! TestablePanGestureRecognizer
                })

                it("should be added to the square") {
                    expect(vc.square.gestureRecognizers?.contains(recognizer)).to(beTrue())
                }

                it("should change the square width to 100 when begins") {
                    recognizer.perfomTouch(nil, translation: nil, state: .Began)
                    expect(vc.widthConstraint.constant).toEventually(equal(100))
                }

                it("should change the square position by gesture translation when gesture changes") {
                    recognizer.perfomTouch(nil, translation: CGPoint(x: -50, y: 50), state: .Changed)
                    expect(vc.centerXconstraint.constant).toEventually(equal(-50))
                    expect(vc.centerYconstraint.constant).toEventually(equal(50))
                }

                it("should reset the square position when gesture ends") {
                    recognizer.perfomTouch(nil, translation: nil, state: .Ended)
                    expect(vc.centerXconstraint.constant).toEventually(equal(0))
                    expect(vc.centerYconstraint.constant).toEventually(equal(0))
                    expect(vc.widthConstraint.constant).toEventually(equal(50))
                }
            }
        }
    }
}
{% endhighlight %}

![Happy](/images/2016-03-30/testing-gesture-recognizers-3.gif)

Source code of the project: [https://github.com/VojtaStavik/TestingGestureRecognizers](https://github.com/VojtaStavik/TestingGestureRecognizers)
