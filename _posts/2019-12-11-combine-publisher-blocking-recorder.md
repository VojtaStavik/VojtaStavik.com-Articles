---
layout: post
title: Testing Publishers Synchronously with a Blocking Recorder
filename: "2019-12-11-combine-publisher-blocking-recorder.md"
social-image: "/images/publisher-recorder/publisher-recorder-1.png"
---

![Combine publisher blocking recorder](/images/publisher-recorder/publisher-recorder-1.png)

When releasing `Combine`, Apple didn't show us what their vision for testing the codebase with `Combine` was. _Surely they must have thought about it._

Fortunately, the authors of `Combine` didn't try to reinvent the wheel. The concepts, they built the framework around, are quite similar to the concepts in other reactive frameworks used on iOS. It means we can reuse the knowledge and best practices that evolved in the existing iOS-reactive-framework communities.

Let's take `RxSwift` as _(probably)_ the most used reactive framework of the pre-Combine era. It has a convenient [set of tools](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/UnitTests.md) for use in tests. We'll use it as inspiration for today.

**We're going to build a simple Combine test helper allowing us to record and synchronously assert values produced by a publisher.**

<!-- more -->

## Recorder

Let's start with defining `Recorder`. It's a concrete `Subscriber` implementation that stores all incoming values and the completion event to its `records` array.

```swift
extension Recorder {
  /// Possible types of records.
  public enum Record {
    case value(Input)
    case completion(Subscribers.Completion<Failure>)
  }
}

/// A subscriber which records all values from the attached publisher.
public class Recorder<Input, Failure: Error> {
  public var records: [Record] = []
}

extension Recorder: Subscriber {
  public func receive(subscription: Subscription) {
    // The number of values we request from the publisher is unlimited.
    subscription.request(.unlimited)
  }

  public func receive(_ input: Input) -> Subscribers.Demand {
    // Because the `receive` function can be called on any thread, we need
    // to always dispatch on the main thread.
    DispatchQueue.main.async { 
      self.records.append(.value(input)) 
    }
    return .unlimited
  }

  public func receive(completion: Subscribers.Completion<Failure>) {
    DispatchQueue.main.async { 
      self.records.append(.completion(completion)) 
    }
  }
}
```

The `Recorder` will be initialized with the number of values we want it to record:

```swift
  public init(numberOfRecords: Int) {
    assert(numberOfRecords > 0, "numberOfRecords must be greater than zero.")
    self.numberOfRecords = numberOfRecords
  }

  private let numberOfRecords: Int
```

Producers can (and usually do) produce their values asynchronously. Testing asynchronous code in `XCTest` can easily become complex and hard to read and maintain.

We'd like to tell `Recorder` to start recording and wait until the given number of records is recorded. If the attached producer doesn't provide the number of values the recorder expects within the given time period, the recorder should fail.

```swift
  // 1. 
  // We use XCTestExpectation and XCTWaiter to pause 
  // the execution of the program. We don't need to provide 
  // a custom message to the expectation, because we're not 
  // going to surface it to the user.
  private let expectation = XCTestExpectation()
  private let waiter = XCTWaiter()

  /// Pauses the execution of the current test and waits,
  /// until the required number of values is recorded.
  ///
  /// - Parameter timeout: The amount of time within which
  /// the required number of records should be recorded.
  public func waitForAllValues(
    timeout: TimeInterval = 1,
    file: StaticString = #file,
    line: UInt = #line
  ) {

    // 2. 
    // If we already have the number of records we need, we can
    // return and continue without blocking the execution.
    guard records.count < numberOfRecords else { return }

    // 3. 
    // We want to wait until the expectation is fulfilled. This
    // is the very line that actually blocks the further execution
    // of the program.
    let result = waiter.wait(for: [expectation], timeout: timeout)
    
    // 4. 
    // We need to check the result of the waiter. If the expectation
    // wasn't successfully fulfilled, we need to fail the test.
    // Otherwise, we can finally return from this function and continue.
    if result != .completed {
      func valueFormatter(_ count: Int) -> String {
        "\(count) value" + (count == 1 ? "" : "s")
      }

      XCTFail("""
        Waiting for \(valueFormatter(numberOfRecords)) timed out.
        Received only \(valueFormatter(records.count)).
        """,
        file: file,
        line: line
      )
    }
  }
```

Finally, we have to add the code which fulfills the expectation when `Recorder` records the number of values it needs:

```swift
  public var records: [Record] = [] {
    didSet {
      if numberOfRecords == records.count {
        expectation.fulfill()
      }
    }
  }
```

## Publisher Extension

To make our life easier, we can create a convenient extension of `Publisher` which creates a new recorder and subscribe it to the given publisher:

```swift
extension Publisher {
  /// Creates a new recorder subscribed to this publisher.
  public func record(numberOfRecords: Int) -> Recorder<Output, Failure> {
    let recorder = Recorder<Output, Failure>(numberOfRecords: numberOfRecords)
    subscribe(recorder)
    return recorder
  }
}
```

## Using in Tests

First, we need to make the `Recorder.Record` type `Equatable` when possible. Otherwise, we can't use `XCTAssertEqual` on it.

```swift
extension Recorder.Record: Equatable 
  where Input: Equatable, Failure: Equatable 
{ }
```

Let's give it a shot and use `Recorder` in tests. You'll be amazed by how simple the tests are!

```swift
  func testPassthroughSubject() {
    // 1.
    // We create the publisher we want to test.
    let publisher = PassthroughSubject<Int, Never>()

    // 2.
    // We create a new recorder.
    let recorder = publisher.record(numberOfRecords: 2)

    // 3.
    // Let's send the values to the publisher asynchronously
    let queue = DispatchQueue.global(qos: .default)
    queue.asyncAfter(deadline: .now() + 0.1) { publisher.send(1) }
    queue.asyncAfter(deadline: .now() + 0.2) { publisher.send(2) }

    // 4.
    // On this line, the execution of the program pauses.
    recorder.waitForAllValues()

    // 5.
    // Finally, let's assert the recorded values.
    XCTAssertEqual(recorder.records, [.value(1), .value(2)])
  }
```

What if we send just one value?

![Combine publisher blocking recorder](/images/publisher-recorder/publisher-recorder-2.png)

The recorder times out. Great.

Let's try to send a wrong value:

![Combine publisher blocking recorder](/images/publisher-recorder/publisher-recorder-3.png)

**Ugh!** The error message is quite ugly. Let's improve that. We provide a custom `CustomStringConvertible` implementation which will forward the description call to values associated with the cases:

```swift
extension Recorder.Record: CustomStringConvertible {
  public var description: String {
    switch self {
    case let .value(inputValue):
      return "\(inputValue)"
    case let .completion(completionValue):
      return "\(completionValue)"
    }
  }
}
```

![Combine publisher blocking recorder](/images/publisher-recorder/publisher-recorder-4.png)

That looks much better!

## Custom Assertions

Writing `.value` every time we want to assert an incoming value can become tedious. We can specify a custom `XCTAssert` function for the situations where we don't need to assert on the completion event:

```swift
public func XCTAssertRecordedValues<Input: Equatable, Failure: Error>(
  _ recorder: Recorder<Input, Failure>,
  _ expectedValues: [Input],
  file: StaticString = #file,
  line: UInt = #line
) {
  // Get only the values, ignore other records
  let values = recorder.records.compactMap { record -> Input? in
    if case let .value(inputValue) = record {
      return inputValue
    } else {
      return nil
    }
  }
  XCTAssertEqual(values, expectedValues, file: file, line: line)
}
```
This simplifies the tests even more and allows us the write the final assertion like this:
```swift
XCTAssertRecordedValues(recorder, [1, 2])
```

> You can find more info about custom `XCTAssert` functions in my article about [Building a Custom XCTAssert for Multiline Strings](/2019/11/28/custom-multiline-assertion/).

### Ideas
- What if we move the `recorder.waitForAllValues()` into the `XCTAssertRecordedValues` function? It would mean we wouldn't have to write it manually but it could also feel like "to much magic":
```swift
public func XCTAssertRecordedValues<Input: Equatable, Failure: Error>(
  _ recorder: Recorder<Input, Failure>,
  _ expectedValues: [Input],
  file: StaticString = #file,
  line: UInt = #line
) {
  recorder.waitForAllValues(file: file, line: line)
  // ... //
}

  func testPassthroughSubject() {
    let publisher = PassthroughSubject<Int, Never>()
    let recorder = publisher.record(numberOfRecords: 2)

    let queue = DispatchQueue.global(qos: .default)
    queue.asyncAfter(deadline: .now() + 0.1) { publisher.send(1) }
    queue.asyncAfter(deadline: .now() + 0.2) { publisher.send(2) }

    XCTAssertRecordedValues(recorder, [1, 2])
  }
```
- We could go with even more "magic" and pause the execution when we try to access the `records` value 🤯. But I guess no one wants so much magic in their APIs.

> The source project for this blog post can be found on [https://github.com/industrialbinaries/CombineTestExtensions](https://github.com/industrialbinaries/CombineTestExtensions).