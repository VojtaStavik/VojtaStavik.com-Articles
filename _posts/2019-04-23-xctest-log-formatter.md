---
layout: post
title: Simple XCTest Log Formatter in Swift
filename: "2019-04-23-xctest-log-formatter.md"
---

The raw output from `xcodebuild` is very detailed and hard to read. The community go-to tool for fixing this is [`xcpretty`](https://github.com/xcpretty/xcpretty). Unfortunately, it happens to me regularly, that the formatted log `xcpretty` produces doesn't contain the details I consider important. For example, *I'd like to see the raw console output when a test case fails*.

![xcpretty example](/images/xctest-log-formatter/xcpretty-example.png)

I realized recently that I tend to build more and more parts of the tooling I need myself. Especially, when the functionality I usually need is trivial, and writing scripts in Swift is so easy! **So I decided to write my own XCTest log formatter;** because why not?

<!-- more -->

> Don't get me wrong! `xcpretty` is an amazing tool and I still use it to format build logs. I want to replace only the test log formatting part, so it better serves my needs.

### My dream log formatter follows these rules:

1. If everything goes as expected, print only the minimum amount of information.
![xcpretty example](/images/xctest-log-formatter/formatter-passed.png)

2. When something fails, print the raw log from console.
![xcpretty example](/images/xctest-log-formatter/formatter-failed.png)

## Setup

The basic idea of the formatter is simple. **It reads the standard input (`stdin`) and looks for known patterns.** If it matches a certain pattern, it outputs a simplified message describing the matched event to the standard output (`stdout`).

> More about `stdin`, `stdout`, and output redirection: [Input/Output Redirection in the Shell](https://thoughtbot.com/blog/input-output-redirection-in-the-shell).

In addition, it keeps track of the failing tests and the total number of executed tests. It uses this information to print the final summary message.

The formatter itself will be a single `.swift` file which will be executable directly from the terminal. Create a new file named `format_xctest_log.swift` and add the following lines to it:

```swift
#!/usr/bin/env xcrun -sdk macosx swift
print("works!")
```

Go back to the terminal and make the file executable by running:
```sh
$ chmod +x format_xctest_log.swift
```

Test that everything works as expected:
```sh
$ ./format_xctest_log.swift
works!
$
```

## Pattern matching

We will use regular expressions to match the patterns we're interested in.

> If you don't feel comfortable around regular expressions, I'd recommend to check out this awesome talk [Best of Fluent 2012: /Reg(exp){2}lained/: Demystifying Regular Expressions](https://www.youtube.com/watch?v=EkluES9Rvak).

Firstly, we define a helper function which performs the actual regex matching and returns an array of all matched strings.

It's OK to force-unwrap all optionals here. They would be `nil` only for invalid, or very specific, regex patterns. All patterns we are about to use are static strings, so a crash in this function is always a programmer error.

```swift
// MARK: - ------====== Patterns ======------

extension String {
  /// Tries to match the provided regex pattern.
  ///
  /// - Parameter regex: The regex pattern to match.
  /// - Returns: The array of matches. The resulting
  ///     array is empty when no matches were found.
  func getMatches(regex: String) -> [String] {
    let nsrange = NSRange(self.startIndex..<self.endIndex, in: self)
    let regex = try! NSRegularExpression(pattern: regex, options: [])

    var result: [String] = []

    regex.enumerateMatches(in: self, options: [], range: nsrange) { (match, _, _) in
      for i in 0..<match!.numberOfRanges {
        let match = String(self[Range(match!.range(at: i), in: self)!])
        result.append(match)
      }
    }

    return result
  }
}
```

Then we define a function for each line of the log we're interested in. You can find the details about the pattern in the documentation comments of the functions.

```swift
/// The matcher for the Test Suite started line.
///
/// Matches:
///   "Test Suite 'xxx' started at xctest-log-formatter 16:54:46.160"
///
/// - Parameters:
///   - line: The string to be searched.
///   - action: The callback called when the match is successful.
///   - name: The name of the suite 'xxx'.
func suiteStarted(_ line: String, action: (_ name: String) -> Void) {
  let pattern = #"Test Suite '(.*)' started"#
  let matches = line.getMatches(regex: pattern)
  guard matches.isEmpty == false else { return }

  let name = matches[1]
  action(name)
}
```

```swift
/// The matcher for the Test case started line.
///
/// Matches:
///   "Test Case '-[xxx yyy]' started."
///
/// - Parameters:
///   - line: The string to be searched.
///   - action: The callback called when the match is successful.
///   - name: The name of the case 'yyy'.
///
func caseStarted(_ line: String, action: (_ name: String) -> Void) {
  let pattern = #"Test Case '-\[(.*) (.*)\]' started"#
  let matches = line.getMatches(regex: pattern)
  guard matches.isEmpty == false else { return }

  let name = matches[2]
  action(name)
}
```

```swift
/// The matcher for the Test case passed line.
///
/// Matches:
///   "Test Case '-[xxx yyy]' passed (zzz seconds)."
///
/// - Parameters:
///   - line: The string to be searched.
///   - action: The callback called when the match is successful.
///   - suite: The name of the suite 'xxx'.
///   - name: The name of the case 'yyy'.
///   - duration: The duration of the case 'zzz'.
func casePassed(_ line: String, action: (_ suite: String, _ name: String, _ duration: String) -> Void) {
  let pattern = #"(?:Test Case) '-\[(.*) (.*)\]' passed \((\w+.\w+) seconds\)"#
  let matches = line.getMatches(regex: pattern)
  guard matches.isEmpty == false else { return }

  let suite = matches[1]
  let name = matches[2]
  let duration = matches[3]

  action(suite, name, duration)
}
```

```swift
/// The matcher for the Test case failed line.
///
/// Matches:
///   "Test Case '-[xxx yyy]' failed (zzz seconds)."
///
/// - Parameters:
///   - line: The string to be searched.
///   - action: The callback called when the match is successful.
///   - suite: The name of the suite 'xxx'.
///   - name: The name of the case 'yyy'.
///   - duration: The duration of the case 'zzz'.
func caseFailed(_ line: String, action: (_ suite: String, _ name: String, _ duration: String) -> Void) {
  let pattern = #"(?:Test Case) '-\[(.*) (.*)\]' failed \((\w+.\w+) seconds\)"#
  let matches = line.getMatches(regex: pattern)
  guard matches.isEmpty == false else { return }

  let suite = matches[1]
  let name = matches[2]
  let duration = matches[3]

  action(suite, name, duration)
}
```

## Formatting

It should be possible to use Swift's `print()` function directly for writing to the standard output. Unfortunately, **I noticed that the actual output is sometimes significantly delayed.** This causes weird issues when the input stream has finished but the formatted log is not fully printed yet.

Luckily, the fix is easy. We need to define a custom `print` function which writes directly to the standard output:
```swift
// MARK: - ------====== Terminal output formatting ======------

func print(_ string: String) {
  if let data = (string + "\n").data(using: .utf8) {
    FileHandle.standardOutput.write(data)
  }
}
```

I really like the way `xcpretty` uses colors and various font styles to make the final log more readable. The following set of functions will help us to do that, too.
```swift
func bold(_ string: String) -> String { return "\u{001B}[1m\(string)\u{001B}[22m" }
func italic(_ string: String) -> String { return "\u{001B}[3m\(string)\u{001B}[23m" }

func green(_ string: String) -> String { return "\u{001B}[32m\(string)\u{001B}[0m" }
func red(_ string: String) -> String { return "\u{001B}[31m\(string)\u{001B}[0m" }
func darkGrayBackground(_ string: String) -> String { return "\u{001B}[100m\(string)\u{001B}[0m" }

let passed = green("✓")
let failed = bold(red("✗"))
```



## Main loop

Let's start with defining a couple of local variables for tracking the test suite statistics.
```swift
// MARK: - ------====== Main Loop ======------

var totalNumberOfTests = 0
var failingTests: [String] = []

var currentCaseRawLog: [String] = []
```

For reading from the standard input line by line, we can use Swift's function `readLine()`. When `EOF` is reached, the function returns `nil`.

Simple `while` loop should do the work:
```swift
while let line = readLine() {
  // do the filtering here
}
```

## Filtering

The last thing we need to do is to add the filtering code itself to the main loop:
```swift
while let line = readLine() {

   // Save the line to the current raw log
  currentCaseRawLog.append(line)

  suiteStarted(line) { name in
    print("\n  \(bold(name))")
  }

  caseStarted(line) { _ in
    // Clear the raw log
    currentCaseRawLog = [line]
    totalNumberOfTests += 1
  }

  casePassed(line) { _, caseName, durationString in
    let duration = italic("(\(durationString) sec)")
    print("    \(passed) \(caseName) \(duration)")
  }

  caseFailed(line) { suiteName, caseName, durationString in
    failingTests.append("\(suiteName) \(caseName)")

    // print the raw log for the failing test
    let log = "\n" + currentCaseRawLog.joined(separator: "\n")
    print(bold(darkGrayBackground(log)))

    print("    \(failed) \(red("\(caseName) \(italic("(\(durationString) sec)"))"))")
  }
}
```

## Final Message

Let finish with printing the final summary message of the formatter:
```swift
// MARK: - ------====== Summary ======------

print("\n══════════════════════════════════════════════════════════════════════════════")
switch true {
case _ where totalNumberOfTests == 0:
  print("  ❌ No tests executed.")

case _ where failingTests.isEmpty:
  print("  ✅ \(totalNumberOfTests) tests passed.")

default:
  print("  ❌ \(totalNumberOfTests) tests passed, \(failingTests.count) failed:")
  failingTests.forEach { print("\t\(red($0))") }
}
print("══════════════════════════════════════════════════════════════════════════════")
```

## Usage

Because our formatter formats only the test logs, we need to separate the build and test steps. Here are the example commands I'm using for [Alamofire](https://github.com/Alamofire/Alamofire):
```shell
# The build step still uses xcpretty :
$ xcodebuild build-for-testing \
-workspace Alamofire.xcworkspace \
-scheme "Alamofire iOS" \
-sdk iphonesimulator \
ENABLE_TESTABILITY=YES \
| xcpretty

# The test step uses the new formatter:
$ xcodebuild test-without-building \
-workspace Alamofire.xcworkspace \
-scheme "Alamofire iOS" \
-destination "id=FE78C58A-0776-41C0-BE1C-FC7C3A07853A" \
| ./format_xctest_log.swift
```

#### Example: All tests passed

![Custom XCTest Log Formatter example](/images/xctest-log-formatter/xctest-log-formatter-final.gif)


#### Example: 2 tests failed

![Custom XCTest Log Formatter fail example](/images/xctest-log-formatter/xctest-log-formatter-final-fail.gif)


---
## Summary

- Is this formatter more robust and powerful than `xcpretty`? ❌ No.

- Is it the best log formatter **for my needs** at this point? ✅ Yes.

- Did I have fun building it? ✅ Absolutely!


## Links
- [Sample code from this article](https://github.com/VojtaStavik/XCTest-Log-Formatter)

- [xcpretty](https://github.com/xcpretty/xcpretty)
- [Input/Output Redirection in the Shell](https://thoughtbot.com/blog/input-output-redirection-in-the-shell)
- [Best of Fluent 2012: /Reg(exp){2}lained/: Demystifying Regular Expressions](https://www.youtube.com/watch?v=EkluES9Rvak)
- [Regex101](https://regex101.com/)
- [Build your own Command Line with ANSI escape codes](http://www.lihaoyi.com/post/BuildyourownCommandLinewithANSIescapecodes.html)
- [Alamofire - .travis.yml](https://github.com/Alamofire/Alamofire/blob/master/.travis.yml)
