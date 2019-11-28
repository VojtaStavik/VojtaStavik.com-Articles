---
layout: post
title: Building a Custom XCTAssert for Multiline Strings
filename: "2019-11-28-custom-multiline-assertion.md"
social-image: "/images/multiline-assertions/multiline-assertion-1.png"
---

`XCTest` provides a bunch of `XCTAssert...` functions to be used for assertions in your tests. Sometimes, however, the functionality you need is different from what the built-in assertions can provide. Sometimes, all you _really_ need is a custom `XCTest` assertion.

![Custom multiline assertion](/images/multiline-assertions/multiline-assertion-1.png)

Custom assertions can make your tests more expressive, readable and maintainable by reducing boilerplate. Expressive and readable tests are crucial to understanding what's happening and why the tests are failing. Having to write less boilerplate code makes people more inclined to writing new tests.

**Let's build a custom `XCTAssertEqual` overload which will work nicely with multiline strings.** You'll see how easy it is to improve the information value of tests and make the developer's life more comfortable.

<!-- more -->

## Before: Without Custom Assertions

Let's say we have a test like this. Notice that we use a built-in assertion function:

![Custom multiline assertion](/images/multiline-assertions/multiline-assertion-2.png)

> I like using this kind of "snapshot" tests when I don't need to test the interaction with the object under tests, but I want to capture its current state as the reference. For example, when testing a concrete API request, I want to validate that all properties are set correctly. I want to express that _this_ is what the API request should be like.

When the test fails, Xcode offers the following UI:

![Custom multiline assertion](/images/multiline-assertions/multiline-assertion-3.png)

Even when we expand the error message, it's still not helpful enough:

![Custom multiline assertion](/images/multiline-assertions/multiline-assertion-4.png)

The error message doesn't clearly show what's wrong. You can imagine that getting this error message can be quite frustrating. **Let's create a custom assertion and improve the developer experience significantly.**

## Custom Multiline String Assertion

1.&nbsp;We start with the function signature. Our goal is to make it as familiar to other developers as possible. The easiest way is to define the new assertion as an overload for `XCTAssertEqual`.

```swift
// 1.
func XCTAssertEqual(
  _ text: String, // the string under test
  multiline reference: String, // the reference string
  file: StaticString = #file, // the file the function is called from
  line: UInt = #line // the line the function is called from
) {

}
```

> If your not familiar with `#file` and `#line` literals, you can find more info in the [documentation](https://docs.swift.org/swift-book/ReferenceManual/Expressions.html#ID390).

2.&nbsp;Because we want to be able to compare the strings line by line, we split them by the new line symbol `\n` into an array of single lines. Notice the `omittingEmptySubsequences` parameter explicitly set to `false`. This means we will get an empty string back also when the line is empty.

3.&nbsp;We iterate over all lines in the strings. The strings can have a different number of lines. To be sure we always check all lines, we use the string with more lines as the upper bound.

```swift
 // 2.
 let textLines = text.split(separator: "\n", omittingEmptySubsequences: false)
 let referenceLines = reference.split(separator: "\n", omittingEmptySubsequences: false)

 // 3.
 for idx in 0..<max(textLines.count, referenceLines.count) {
  // TODO: Compare lines
 }
```

4.&nbsp;Because the strings can have a different number of lines, it's possible we can get out of the valid range with `idx`. Let's define a custom subscript which returns `nil` when the index is invalid, instead of crashing.

```swift
// 4.
private extension Array {
  subscript(safely index: Index) -> Element? {
    if self.indices.contains(index) {
      return self[index]
    } else {
      return nil
    }
  }
}
```

5.&nbsp;Finally, we just call `XCTAssertEqual` with the corresponding pair of lines and let XCTest do its job.

```swift
// 3.
for idx in 0..<max(textLines.count, referenceLines.count) {
  // 5.
  let left = textLines[safely: idx]
  let right = referenceLines[safely: idx]
  
  XCTAssertEqual(left, right, file: file, line: line) 
}
```

## After: With the Custom Assertion

We tell the compiler to use our new overload by providing the `multiline` label for the second parameter:

![Custom multiline assertion](/images/multiline-assertions/multiline-assertion-5.png)

6.&nbsp;However, we still don't see the error on the correct line. Let's fix it. We need to offset the line number by the current line.

```swift
// 3.
for idx in 0..<max(textLines.count, referenceLines.count) {
  // 5.
  let left = textLines[safely: idx]
  let right = referenceLines[safely: idx]

  // 6.
  let line = line + UInt(1 + idx)

  XCTAssertEqual(left, right, file: file, line: line)
}
```

![Custom multiline assertion](/images/multiline-assertions/multiline-assertion-6.png)

7.&nbsp;Let's also get rid of the `Optional("")` noise in the message:

```swift
// 5.
let left = textLines[safely: idx]
let right = referenceLines[safely: idx]

// 6.
let line = line + UInt(1 + idx)

// 7.
if let left = left, let right = right {
  XCTAssertEqual(left, right, file: file, line: line)
} else {
  XCTAssertEqual(left, right, file: file, line: line)
}
```

![Custom multiline assertion](/images/multiline-assertions/multiline-assertion-7.png)

**Much better!** Now it's easy to see, where exactly the problem is.

---

For summary, here's the complete implementation of our custom multiline string assertion:

```swift
// 1.
func XCTAssertEqual(
  _ text: String,
  multiline reference: String,
  file: StaticString = #file,
  line: UInt = #line
) {

  // 2.
  let textLines = text.split(separator: "\n", omittingEmptySubsequences: false)
  let referenceLines = reference.split(separator: "\n", omittingEmptySubsequences: false) 
  
  // 3.
  for idx in 0..<max(textLines.count, referenceLines.count) {
    // 5.
    let left = textLines[safely: idx]
    let right = referenceLines[safely: idx] 
    
    // 6.
    let line = line + UInt(1 + idx) 
    
    // 7.
    if let left = left, let right = right {
      XCTAssertEqual(left, right, file: file, line: line)
    } else {
      XCTAssertEqual(left, right, file: file, line: line)
    }
  }
}

// 4.
private extension Array {
  subscript(safely index: Index) -> Element? {
    if self.indices.contains(index) {
      return self[index]
    } else {
      return nil
    }
  }
}
```

### Ideas
- You can customize the error message and provide even more information. For example something like:
```
🛑 XCTAssertEqual failed: Lines are not equal.
expected: Mrs. Robinson
received: Mrs. Columbo
                 ↑
```

- Custom `XCTAssert` overloads for `Result` (_courtesy: [Boris Bielik](https://twitter.com/h3sperian)_): 
```
XCTAssertEqual(result, success: _success value_)
XCTAssertEqual(result, failure: _error value_)
```