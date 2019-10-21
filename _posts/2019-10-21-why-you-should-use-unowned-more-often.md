---
layout: post
title: "Why You Should Be Using `unowned` More Often"
filename: "2019-10-21-why-you-should-use-unowned-more-often.md"
social-image: "/images/unowned/weak-everywhere.jpg"
---

I've seen a lot of interesting codebases in the last two months: new clients of [Industrial Binaries](https://industrial-binaries.co), submissions for my [iOS Code Review](https://www.youtube.com/channel/UCzHnqIT0Ion6rCVPdgk2cHQ?view_as=subscriber) series, and also code challenges of the candidates we're recruiting.

**I was surprised how rarely `unowned` is used in the wild.**


![Weak everywhere](/images/unowned/weak-everywhere.jpg)

I think this is a huge missed opportunity. Not using `unowned` in the right places makes the code harder to maintain and can hide some serious and hard-to-replicate errors.

<!-- more -->

## Validate Your Assumptions

We all know the old programmer's joke: *My code works and I have no idea why*. While the joke is funny, and this situation happens to all of us, it's not something we should accept and be satisfied with! 

**Not knowing why our fix fixes the issue means we don't understand the problem properly.** The bug can still be there and we might have fixed only one of its symptoms. We just made it somebody else's problem - most likely the somebody is us in the future.

> **When programming, it's important to make strong assumptions about the code and - even more important - validate these assumptions.**

That's why we have `assert` and `precondition` in Swift; to make sure our assumptions about how the system works are correct.

Using `weak` and `unowned` correctly, too, requires strong assumptions about the behavior of the system. It forces you to think more about the objects you're working with. And if your assumption is not correct? Great! You didn't understand the behavior correctly and you know more now. It's a game in which you're always the winner.

## The Rule Of Thumb: `weak` vs. `unowned`

The thought process I use when deciding between `weak` and `unowned` is very simple. It consists just from one question:

**Am I (the caller object) the only owner of the other object?**

> - **YES:** Use `unowned`. The lifetime of the object I own shouldn't exceed my own lifetime. Does the code crash? **Awesome!** We've (probably) found a memory leak and we can fix it.

> - **NO:** Use `weak`. We have no control over the other object's lifetime. The callback can be called even when the caller is deallocated.


<!-- more -->

### Examples:

```swift
class Caller {

  var data: Data?

  func getData() {
    let url = URL(string: "industrial-binaries.co")!
    // Caller doesn't own `URLSession.shared`. -> [weak self]
    let task = URLSession.shared.dataTask(with: url) { [weak self] (data, _, _) in
      self?.data = data
    }
    task.resume()
  }
}
```

```swift
class Caller {

  let engine: MathEngine

  init(engine: MathEngine) {
    self.engine = engine
  }

  func computeFactorial() {
    // Caller can't be sure it's the only owner of `engine`. It was created 
    // outside of this class and it can have multiple owners.
    // -> [weak self]
    engine.factorial(123456) { [weak self] (result) in
      self?.saveResult(result)
    }
  }

  func saveResult(_ result: Int) { }
}
```

```swift
class Caller {

  let engine: MathEngine = .init()

  func computeFactorial() {
    // Engine was created inside this class. Caller is
    // its only owner. -> [unowned self]
    engine.factorial(123456) { [unowned self] (result) in
      self.saveResult(result)
    }
  }

  func saveResult(_ result: Int) { }
}
```
## ⚠️Important ⚠️
Do not blindly follow the rule above. It's just a rule of thumb. Always make your own assumptions about the system you're working with and use `weak` and `unowned` accordingly.

> **Don't be afraid of crashing!** Crashing early in the development cycle is better than shipping code with hidden bugs.