---
layout: post
title: "Mocking Network Calls Using URLProtocol"
filename: "2019-09-10-mocking-network-calls-using-urlprotocol.md"
---
For many apps, networking code is one of the most important parts of the whole codebase. Yet it's also a part which is quite difficult to test properly.

Some people just give up and rely on thorough manual testing. This approach doesn't really scale well. A medium-sized app can have hundreds of various API requests. Having to test all of them manually makes any future changes and refactoring tedious and dangerous.

![URLProtocol Xcode screenshot](/images/url-protocol/url-protocol.png)

Another popular solution you can see in the wild is making real network calls, especially in integration and UI tests. *Not great, not terrible.* ☢️ Real network calls make tests simple but [slower by the order of several magnitudes](https://twitter.com/srigi/status/917998817051541504?s=20). It also makes them much less reliable. The success of the tests depends on your current networking conditions. *Ask yourself if it's OK that your tests fail when the train you're working from enters a tunnel.*

**Can we somehow keep the simplicity of real network calls while making it more reliable and network-conditions independent?**

<!-- more -->

### Custom `URLProtocol` Subclass to Help

`URLProtocol` subclasses is what iOS uses to figure out the loading behavior for specific URLs and schemes. By default, Apple provides these protocols: `_NSURLHTTPProtocol`, `_NSURLDataProtocol`, `_NSURLFTPProtocol`, `_NSURLFileProtocol`, `NSAboutURLProtocol`.

By providing a custom protocol to the iOS URL loading system, we can intercept the networking communication and make sure *certain* URL requests are given a special treatment.

```swift
class MockResponseURLProtocol: URLProtocol {
  override class func canonicalRequest(for request: URLRequest) -> URLRequest {
    // Overriding this function is required by the superclass.
    return request
  }
}

// We need to register the custom URLProtocol so the URL
// loading system knows about it.
URLProtocol.registerClass(MockResponseURLProtocol.self)
```

To find out which `URLProtocol` to use, the system asks all registered `URLProtocol` classes whether they `canInit(with request: URLRequest)`. We don't want to intercept all network communication, so we're going to identify the requests we want to intercept by their URL and the simulated response:

```swift
class MockResponseURLProtocol: URLProtocol {

  /// The key-pairs of URLs this URLProtocol intercepts with
  /// their simulated response.
  static var mockResponses: [URL: Result<Data, Error>] = [:]

  override class func canInit(with request: URLRequest) -> Bool {
    guard let url = request.url else { return false }
    return mockResponses.keys.contains(url)
  }

  { ... }
}
```

### Simulate Receiving Data

The next thing we need to do is to provide the implementation for the `startLoading()` method. We check for the mock response we're suppose to provide and then call the appropriate methods on `self.client`. The client's type is `URLProtocolClient` and it's the way `URLProtocol` communicates with the URL loading system.

```swift
class MockResponseURLProtocol: URLProtocol {

  { ... }

  override func startLoading() {
    guard let response = MockResponseURLProtocol.mockResponses[self.request.url!] else {
      fatalError("""
        No mock response for \(request.url!). This should never happen. Check
        the implementation of `canInit(with request: URLRequest) -> Bool`.
        """)
    }

    // Simulate the response on a background thread.
    DispatchQueue.global(qos: .default).async {
      switch response {
      case let .success(data):
        // Simulate received data.
        self.client?.urlProtocol(self, didLoad: data)

        // Finish loading (required).
        self.client?.urlProtocolDidFinishLoading(self)

      case let .failure(error):
        // Simulate error.
        self.client?.urlProtocol(self, didFailWithError: error)
      }
    }
  }

  override func stopLoading() {
    // Required by the superclass.
  }
}
```

### Setting Up the Mock Response

Here's the complete Playground page showcasing all code needed to mock a network response:

```swift
import Foundation

/// The URLProtocol subclass allowing to intercept the network communication
/// and provide custom mock responses for the given URLs.
class MockResponseURLProtocol: URLProtocol {

  /// The key-pairs of URLs this URLProtocol intercepts with their simulated response.
  static var mockResponses: [URL: Result<Data, Error>] = [:]

  override class func canInit(with request: URLRequest) -> Bool {
    guard let url = request.url else { return false }
    return mockResponses.keys.contains(url)
  }

  override class func canonicalRequest(for request: URLRequest) -> URLRequest {
    // Overriding this function is required by the superclass.
    return request
  }

  override func startLoading() {
    guard let response = MockResponseURLProtocol.mockResponses[self.request.url!] else {
      fatalError("""
        No mock response for \(request.url!). This should never happen. Check
        the implementation of `canInit(with request: URLRequest) -> Bool`.
        """)
    }

    // Simulate response on a background thread.
    DispatchQueue.global(qos: .default).async {
      switch response {
      case let .success(data):
        // Simulate received data.
        self.client?.urlProtocol(self, didLoad: data)

        // Finish loading (required).
        self.client?.urlProtocolDidFinishLoading(self)

      case let .failure(error):
        // Simulate error.
        self.client?.urlProtocol(self, didFailWithError: error)
      }
    }
  }

  override func stopLoading() {
    // Required by the superclass.
  }
}

// We need to register the custom URLProtocol so the URL loading system knows about it.
URLProtocol.registerClass(MockResponseURLProtocol.self)

// The URL we want to load
let url = URL(string: "industrial-binaries.co")!

// Define the mock response
MockResponseURLProtocol.mockResponses[url] = .success(Data(repeating: 0, count: 2000))

// Load data
let data = try! Data(contentsOf: url)

// Check the data is correct
assert(data.count == 2000) // All good 👍
```

### Downloading Progress Simulation

Let's try something more complex. By adding just a couple of lines of code we can easily simulate progressive downloading:

```swift
class MockResponseURLProtocol: URLProtocol {

  { ... }

  override func startLoading() {
    guard let response = MockResponseURLProtocol.mockResponses[self.request.url!] else {
      fatalError("""
        No mock response for \(request.url!). This should never happen. Check
        the implementation of `canInit(with request: URLRequest) -> Bool`.
        """)
    }

    // Simulate response on a background thread.
    DispatchQueue.global(qos: .default).async {
      switch response {
      case let .success(data):

        // Step 1: Simulate receiving an URLResponse. We need to do this
        // to let the client know the expected length of the data.
        let response = URLResponse(
          url: self.request.url!,
          mimeType: nil,
          expectedContentLength: data.count,
          textEncodingName: nil
        )

        self.client?.urlProtocol(self, didReceive: response, cacheStoragePolicy: .notAllowed)

        // Step 2: Split data into chunks
        let chunkSize = 10
        let chunks = stride(from: 0, to: data.count, by: chunkSize).map {
          data[$0 ..< min($0 + chunkSize, data.count)]
        }

        // Step 3: Simulate received data chunk by chunk.
        for chunk in chunks {
          self.client?.urlProtocol(self, didLoad: chunk)
        }

        // Step 4: Finish loading (required).
        self.client?.urlProtocolDidFinishLoading(self)

      case let .failure(error):
        // Simulate error.
        self.client?.urlProtocol(self, didFailWithError: error)
      }
    }
  }

  { ... }

}
```

We're going to use `URLSession`to test the progress simulation:

```swift
let task = URLSession.shared.dataTask(with: url) { _, _, _ in
  print("Done!")
}

let observation = task.progress
  .observe(\.fractionCompleted) { progress, _ in
    print(progress.fractionCompleted, terminator: " ")
  }
```

Looks good! 👍

![URLProtocol Progress Xcode screenshot](/images/url-protocol/url-protocol-2.png)

### Ideas
- You can introduce an artificial delay and send the data chunks to the client gradually. It's a great way to simulate bad network conditions.
- The mocked responses don't have to identified only by their `URL`. You can also check `URLRequest`'s body, HTTP method and other parameters. The response can even be dynamic based on the request's content!
- You can prepare mock responses for all main API points your app uses. It's a great way to make sure the app has stable conditions for UI testing.
