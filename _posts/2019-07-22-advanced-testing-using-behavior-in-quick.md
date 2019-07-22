---
layout: post
title: Advanced testing using `Behavior` in Quick
filename: "2019-07-22-advanced-testing-using-behavior-in-quick.md"
---

`Behavior<Context>` is a hidden gem in the Quick testing framework. It was [introduced silently in 2017](https://github.com/Quick/Quick/pull/701) and it’s not even mentioned in the documentation! [Here’s my PR which fixes that](https://github.com/Quick/Quick/pull/905). **The more I know about it the more I think _this_ should be the way we write specs in Quick.**

So, what’s `Behavior`? You can think of it as of a better and safer way of defining shared expectation. But that would be very shortsighted. There’s much more!

<!-- more -->

### The Example Code to Test

The problem with many articles about testing is that the examples are too simple and distant from the real-world problems. I tried to come up with an example complex enough. Bear with me and please read the following code carefully. The functionality is explained in the documentation comments:

```swift
class Counter {
  /// Adds `a` to the total count.
  /// - Returns: The value of the counter after the update.
  func add(_ a: Int) -> Int {
    count += a
    return count
  }

  private var count: Int = 0
}

class ComplexCounter: Counter {
  /// Creates a new instance of ComplexCounter.
  /// - Parameter numberOfBuckets: The number of additional "buckets" counters.
  init(numberOfBuckets: Int) { 
    for _ in 0..<numberOfBuckets {
      counters.append(Counter())
    }
  }

  /// Adds `a` to the total count of the master counter. Also adds `a` to every bucket counter.
  /// - Returns: The value of the master counter after the update.
  override func add(_ a: Int) -> Int {
    counters.forEach { _ = $0.add(a) }
    return super.add(a)
  }

  /// Adds `a` to the bucket counter specified by `bucketIndex`.
  /// - Returns: The value of the bucket counter after the update.
  func add(_ a: Int, bucketIndex: Int) -> Int {
    assert(counters.indices.contains(bucketIndex))
    return counters[bucketIndex].add(a)
  }

  private var counters: [Counter] = []
}
```

**Why is this example not trivial to test?** Let’s think about it.
- The `Counter` object has just one function and its behavior is easy to test. A small complication there is the hidden state but that shouldn't stop us.
- The `ComplexCounter`’s behavior is much more complex (pun not intended). It has the master counter and a configurable number of “bucket” counters. The master counter behaves similarly to the simple `Counter`. Additionally, it also silently increments all the bucket counters with the same number. 
- The bucket counters behave like standalone counters and don’t interact with each other.
- It means we need test the intended interaction between the counters while making sure the master counter and all bucket counters still _behave like_  simple counters.


### The “Counter” Behavior

`Behavior` is a simple class defined in `Quick`. It has just two elements: `name` and `spec`.

```swift
/**
 A `Behavior` encapsulates a set of examples that can be re-used in several 
 locations using the `itBehavesLike` function with a context instance of 
 the generic type.
 */
open class Behavior<Context> {

    public static var name: String { get }

    /**
         Override this method in your behavior to define a set of reusable examples.
    
         This behaves just like an example group defines using `describe` or 
         `context`--it may contain any number of `beforeEach` and `afterEach` closures, 
         as well as any number of examples (defined using `it`).
    
         - parameter aContext: A closure that, when evaluated, returns a `Context` instance 
         that provide the information on the subject.
        */
    open class func spec(_ aContext: @escaping () -> Context)
}
```

For some reason, the `name` property is defined as `static` and it’s not possible to override it in subclasses. You will see later it can be quite annoying. [I opened a PR to fix this](https://github.com/Quick/Quick/pull/906).

`Behavior` is generic over the `Context` type. That's the type of the object the Behavior tests.

I don’t find the name `Context` fortunate but it has historical reasons. `Behavior` was introduces as a better alternative for creating shared examples. _And sure they use the name `context` for their input parameter._

**Let’s define what the Counter behavior looks like:**

```swift
class BehavesLikeCounter: Behavior<(Int) -> Int> {
  override class func spec(_ aContext: @escaping () -> (Int) -> Int) {

    var add: ((Int) -> Int)!

    beforeEach {
      add = aContext()
    }

    it("should have the initial value 0") {
      expect(add(0)) == 0
    }

    it("should add numbers correctly") {
      expect(add(5)) == 5
      expect(add(5)) == 10
      expect(add(5)) == 15
    }
  }
}
```

There are two important things to notice in the code above:

  - The generic type of the behavior is pinned to `(Int) -> Int` which matches the signature of `Counter`’s `add(_ a: Int) -> Int` function. We don’t want to restrict ourself to test just `Counter` classes and subclasses. Any function with the `(Int) -> (Int)` signature can be tested for this specific `Counter`-like behavior.

  - The content of `spec` method looks exactly the same as any other “normal” `spec` in `QuickSpec` subclasses. The **only** difference is that we don’t create the object we’re testing directly. We ask the context to create it for us.


### Using Behaviors in Specs

To use `Behavior` in your tests suite, you simply use it as a parameter for `itBehavesLike`:
```swift
itBehavesLike(BehavesLikeCounter.self) { … }
```

I like to use a custom overload of `it` which takes `Behavior` as a parameter ([PR with this change](https://github.com/Quick/Quick/pull/907)). It allows me to be more flexible in how I name Behaviors. You’ll see an example of this later in the post.

```swift
public func it<C>(
  _ behavior: Quick.Behavior<C>.Type, 
  file: FileString = #file, 
  line: UInt = #line, 
  context: @escaping () -> C
) {
  itBehavesLike(behavior, file: file, line: line, context: context)
}
```

Let’s use the `BehavesLikeCounter` behavior to test `Counter`.
```swift
class CounterSpec: QuickSpec {
  override func spec() {
    it(BehavesLikeCounter.self) { Counter().add }
  }
}
```

The resulting console output looks like this:
```
Test Case '-[BehaviorExamplesTests.CounterSpec BehavesLikeCounter__should_add_numbers_correctly]' started.
Test Case '-[BehaviorExamplesTests.CounterSpec BehavesLikeCounter__should_add_numbers_correctly]' passed (0.001 seconds).
Test Case '-[BehaviorExamplesTests.CounterSpec BehavesLikeCounter__should_have_the_initial_value_0]' started.
Test Case '-[BehaviorExamplesTests.CounterSpec BehavesLikeCounter__should_have_the_initial_value_0]' passed (0.000 seconds).
```
> You can see that the "BehavesLikeCounter" part od the test name is a little bit odd. That's why I opened the [PR to make the name configurable](https://github.com/Quick/Quick/pull/906), too.

**Looks pretty neat, doesn’t it?** Wait until you see the complete spec for `ComplexCounter`:

```swift
class CommplexCounterSpec: QuickSpec {

  override func spec() {

    var complexCounter: ComplexCounter!

    beforeEach {
      complexCounter = ComplexCounter(numberOfBuckets: 3)
    }

    it(BehavesLikeCounter.self) { complexCounter.add }

    describe("every bucket") {
      it(BehavesLikeCounter.self) { { number in complexCounter.add(number, bucketIndex: 0) } }
      it(BehavesLikeCounter.self) { { number in complexCounter.add(number, bucketIndex: 1) } }
      it(BehavesLikeCounter.self) { { number in complexCounter.add(number, bucketIndex: 2) } }
    }

    it("should count buckets individually") {
      expect(complexCounter.add(0, bucketIndex: 0)) == 0
      expect(complexCounter.add(2, bucketIndex: 1)) == 2
      expect(complexCounter.add(3, bucketIndex: 2)) == 3

      expect(complexCounter.add(0, bucketIndex: 0)) == 0
      expect(complexCounter.add(2, bucketIndex: 1)) == 4
      expect(complexCounter.add(3, bucketIndex: 2)) == 6
    }

    describe("changing the master counter") {
      it("should also update all bucket counters") {
        expect(complexCounter.add(0, bucketIndex: 0)) == 0
        expect(complexCounter.add(2, bucketIndex: 1)) == 2
        expect(complexCounter.add(3, bucketIndex: 2)) == 3

        expect(complexCounter.add(100)) == 100

        expect(complexCounter.add(0, bucketIndex: 0)) == 100
        expect(complexCounter.add(2, bucketIndex: 1)) == 104
        expect(complexCounter.add(3, bucketIndex: 2)) == 106
      }
    }
  }
}
```

```
Test Case '-[BehaviorExamplesTests.CommplexCounterSpec BehavesLikeCounter__should_add_numbers_correctly]' started.
Test Case '-[BehaviorExamplesTests.CommplexCounterSpec BehavesLikeCounter__should_add_numbers_correctly]' passed (0.001 seconds).
Test Case '-[BehaviorExamplesTests.CommplexCounterSpec BehavesLikeCounter__should_have_the_initial_value_0]' started.
Test Case '-[BehaviorExamplesTests.CommplexCounterSpec BehavesLikeCounter__should_have_the_initial_value_0]' passed (0.000 seconds).
Test Case '-[BehaviorExamplesTests.CommplexCounterSpec changing_the_master_counter__should_also_update_all_bucket_counters]' started.
Test Case '-[BehaviorExamplesTests.CommplexCounterSpec changing_the_master_counter__should_also_update_all_bucket_counters]' passed (0.000 seconds).
Test Case '-[BehaviorExamplesTests.CommplexCounterSpec every_bucket__BehavesLikeCounter__should_add_numbers_correctly]' started.
Test Case '-[BehaviorExamplesTests.CommplexCounterSpec every_bucket__BehavesLikeCounter__should_add_numbers_correctly]' passed (0.000 seconds).
Test Case '-[BehaviorExamplesTests.CommplexCounterSpec every_bucket__BehavesLikeCounter__should_add_numbers_correctly_2]' started.
Test Case '-[BehaviorExamplesTests.CommplexCounterSpec every_bucket__BehavesLikeCounter__should_add_numbers_correctly_2]' passed (0.009 seconds).
Test Case '-[BehaviorExamplesTests.CommplexCounterSpec every_bucket__BehavesLikeCounter__should_add_numbers_correctly_3]' started.
Test Case '-[BehaviorExamplesTests.CommplexCounterSpec every_bucket__BehavesLikeCounter__should_add_numbers_correctly_3]' passed (0.000 seconds).
Test Case '-[BehaviorExamplesTests.CommplexCounterSpec every_bucket__BehavesLikeCounter__should_have_the_initial_value_0]' started.
Test Case '-[BehaviorExamplesTests.CommplexCounterSpec every_bucket__BehavesLikeCounter__should_have_the_initial_value_0]' passed (0.000 seconds).
Test Case '-[BehaviorExamplesTests.CommplexCounterSpec every_bucket__BehavesLikeCounter__should_have_the_initial_value_0_2]' started.
Test Case '-[BehaviorExamplesTests.CommplexCounterSpec every_bucket__BehavesLikeCounter__should_have_the_initial_value_0_2]' passed (0.000 seconds).
Test Case '-[BehaviorExamplesTests.CommplexCounterSpec every_bucket__BehavesLikeCounter__should_have_the_initial_value_0_3]' started.
Test Case '-[BehaviorExamplesTests.CommplexCounterSpec every_bucket__BehavesLikeCounter__should_have_the_initial_value_0_3]' passed (0.000 seconds).
Test Case '-[BehaviorExamplesTests.CommplexCounterSpec should_count_buckets_individually]' started.
Test Case '-[BehaviorExamplesTests.CommplexCounterSpec should_count_buckets_individually]' passed (0.000 seconds).
```

Look how we use the same behavior first for the "master" counter and then for each bucket counter. 

The fact that we abstracted the Counter behavior allows us to easily re-use the specification any time we need.

### The "ShouldNotHaveMemoryLeaks" behavior

We don’t need to pin `Behavior`’s generic type to a concrete type. We can keep it generic. Take a look at this example:

```swift
class ShouldNotHaveMemoryLeaks<T>: Behavior<T> where T: AnyObject {
  override class func spec(_ aContext: @escaping () -> T) {
    it("should be released from memory") {
      weak var weakObject: T?
      autoreleasepool {
        let object = aContext()
        weakObject = object
      }
      expect(weakObject).to(beNil())
    }
  }
}
```

We can now easily test both our Counter types for memory leaks:
```swift
class CounterSpec: QuickSpec {

  override func spec() {
    it(BehavesLikeCounter.self) { Counter().add }

    //////////////// 👇👇👇 ////////////////
    it(ShouldNotHaveMemoryLeaks.self) { Counter() }
  }
}
```

```swift
class CommplexCounterSpec: QuickSpec {

  override func spec() {

    var complexCounter: ComplexCounter!

    beforeEach {
      complexCounter = ComplexCounter(numberOfBuckets: 3)
    }

    it(BehavesLikeCounter.self) { complexCounter.add }

    describe("every bucket") { … }

    it("should count buckets individually") { … }

    describe("changing the master counter") { … }

    //////////////// 👇👇👇 ////////////////
    it(ShouldNotHaveMemoryLeaks.self) { ComplexCounter(numberOfBuckets: 5) }
  }
}
```
