---
layout: post
title: Organize your errors
filename: "2017-04-23-organize-your-errors-swift.md"
---

There are two types or programmers: Those who handle errors properly, and those you don't want to work with.

![Windows BSOD](/images/2017-04-23/bsod.png)

It's happening over and over again. Something has stopped working and I need to fix it. After tens of minutes of debugging and diving deeper into the code base, I finally find it. Someone (including several-years-ago myself), ignored an error and left it unhandled.

<!-- more -->

The reason, why is this situation so annoying is simple. The error information, usually including the reason why the error is happening, was right there! Someone just had decided to ignore it.

### Proper error handling: the old (Objective-C) times

![The old times](/images/2017-04-23/ibm-702.png)

Objective-C APIs have quite obscure way, how to signalize errors. Take ```NSData``` and its ```writeToFile``` function as an example:
{% highlight swift %}
- (BOOL)writeToFile:(NSString *)path
            options:(NSDataWritingOptions)writeOptionsMask
              error:(NSError **)errorPtr;
{% endhighlight swift %}

The function returns a ```BOOL``` value indicating whether the operation was successful. When the value is ```NO```, you can then find more info about the error inside the object referenced by ```errorPtr```. Notice that ```erorrPtr``` is a pointer to a pointer to a ```NSError``` object.

{% highlight swift %}
NSData* data = [NSData new]; // some data
NSString* path = @"path"; // some path

// ⚠️ DO NOT DO THIS ⚠️ 👎
// This code completely ignores the error and it will, sooner or later,
// bite you back (or ever worse: bite your colleagues).
[data writeToFile:path options:0 error:nil];

// YOU SHOULDN'T EVEN DO THIS 👎
// This is definitely better than the previous. However, you need
// to check the returning value, not just the error pointer.
// Some APIs could return non-nil error even for non-error states.
NSError* error1;
[data writeToFile:path options:0 error:&error1];
if (error1) {
    // Do something with the error. If nothing, at least log it
    // to the console.
}

// THE PROPER WAY 👍
NSError* error2;
if ([data writeToFile:path options:0 error:&error2]) {
    // Happy path
} else {
    // Here we're sure something went wrong
    if (error2) {
        // We even have more info about it
    }
}
{% endhighlight swift %}

### Proper error handling: Swift

Even though Swift 2 made error handling easier and more straightforward, there are still many ways how not to do it properly.

{% highlight swift %}
let data = Data() // Some data
let url = URL(string: "url")! // Some URL

// ⚠️ DO NOT DO THIS ⚠️ 👎
// Using try! will crash your app
// when an error occurs. Like implicitly
// unwrapped optionals - use try!
// only if you really know what you
// are doing (i.e. in your tests).
try! data.write(to: url)


// ⚠️ DO NOT DO THIS ⚠️ 👎
// This particular usage of try?
// is also not good. When the write
// function fails, the error is
// silently consumed and no one
// will ever know about it.
try? data.write(to: url)


// YOU CAN DO THIS 👍
// If the return value of the throwing
// function is what matters, it might be
// a good idea to use try?. You won't
// have any further information about
// the error, though.
guard let data = try? Data(contentsOf: url) else {
   // Something went wrong, but we don't why
   return
}
// Happy path: Everything is OK and we can use data
print(data.description)


// BASIC ERROR HANDLING 👍
do {
   try data.write(to: url)
} catch {
   // You can access variable 'error' of
   // the type 'Error' within this scope.
}
{% endhighlight swift %}

![You can't get runtime error](/images/2017-04-23/cant-get-runtime-error.jpg)

### Custom error types in Swift

This is where the fun begins. You can define your own error types and make your functions throwing them:

{% highlight swift %}
enum Error: Swift.Error {
    case network(NetworkError)
    enum NetworkError {
        case notReachable
        case unknown
    }

    case device(DeviceError)
    enum DeviceError {
        case notEnoughSpace
        case unsupportedSystemVersion
    }
}
{% endhighlight swift %}

You can then handle specific errors a different way than the others:

{% highlight swift %}
// COMPLETE ERROR HANDLING 👍
// (WITH CUSTOM ERRORS)

do {
    try doSomething()

} catch let Error.network(error) {
    // The returned error is a network error.
    print(type(of: error)) // prints `Error.NetworkError`

    // ... do something with the error (log it, etc.)

} catch let Error.device(error) {
    // The returned error is a device error.
    print(type(of: error)) // prints `Error.DeviceError`

    // ... do something different with the error (show alert etc.)

} catch {
    // Some other error without specific handling.
}

{% endhighlight swift %}
