---
title: Reachability using NWPathMonitor and Combine
date: 2021-02-21 02:00:00 -0100
category: Programming
---

![pinboarding]({{ site.url }}/assets/images/2021-02-21-pinboarding.png)

About a month ago I decided to build a new app to attend a personal need, a desktop client for [Pinboard](https://pinboard.in), a service I've been using for over a decade to store and manage my bookmarks.

As any application which requires internet to fetch and update data, my client needed a way to detect network accessibility. For a long time the strategy adopted by engineers in this situation was to rely on Apple's Reachability class, a sample code the company used to provide for download on their developer portal. Well, either that or one of the many CocoaPods frameworks the community built to replace Apple's drag-and-drop solution. But since iOS 12.0 and macOS 10.14 there's a powerfull first-party alternative.

Introduced a couple of years ago, [`NWPathMonitor`](https://developer.apple.com/documentation/network/nwpathmonitor) is the easiest way to detect network changes (and retrieve connection properties). Its interface is extremely simple to use, providing a callback to get notified of updates.

```swift
let monitor = NWPathMonitor()
let queue = DispatchQueue.global(qos: .background)

monitor.pathUpdateHandler = { path in
    print(path.status) // .unsatisfied, .satisfied, .requiresConnection
}

monitor.start(
    queue: queue
)

// ...

monitor.stop()
```

My application in built in SwiftUI (2.0) and Combine and the solution above wouldn't *feel right* in my View Model. So I started with a simple wrapper, hiding the monitor and exposing a publisher which notifies path changes. 

```swift
let wrapper = NWPathMonitorWrapper()

let cancellable = wrapper
    .pathUpdatePublisher()
    .receive(on: RunLoop.main)
    .sink { path in
        print(path.status)
    }

wrapper.start()
```

But the implementation (below) still requires `NWPathMonitorWrapper.start()` to be called, hurting the beauty of the publisher-subscriber relationship.

```swift
public final class NWPathMonitorWrapper {

    // MARK: - Properties

    private let monitor: NWPathMonitor
    private let queue: DispatchQueue = .global(qos: .background)
    private let pathUpdateSubject = PassthroughSubject<NWPath, Never>()

    // MARK: - Life cycle

    public init(
        monitor: NWPathMonitor = NWPathMonitor()
    ) {
        self.monitor = monitor
        self.monitor.pathUpdateHandler = {
            self.pathUpdateSubject.send($0)
        }
    }

    // MARK: - Public

    public func start() {
        monitor.start(
            queue: queue
        )
    }

    public func stop() {
        monitor.cancel()
    }

    public func pathUpdatePublisher() -> AnyPublisher<NWPath, Never> {
        pathUpdateSubject
            .eraseToAnyPublisher()
    }
}
```

So I looked for a similar example in Apple's frameworks, one which requires a method to be called to trigger the action. Turns out `URLSession`, familiar to every iOS/macOS engineer, is the perfect example since it requires `URLSession.resume()` to be called to fire the network requrest. Recently `URLSession` got a new method, a [publisher](https://developer.apple.com/documentation/foundation/urlsession/3329707-datataskpublisher) which starts the request at the moment there's demand (a subscriber):

```swift
let session = URLSession.shared

let cancellable = session
    .dataTaskPublisher(for: request)
    .receive(on: RunLoop.main)
    .tryMap { result in
        // ...
    }
    .sink(
        receiveCompletion: { _ in },
        receiveValue: { _ in }    
    )
```

And that's the interface I wanted for my monitor. A simple publisher which starts *emitting values* as soon as there's demand.

```swift
let monitor = NWPathMonitor()

let cancellable = monitor
    .pathUpdatePublisher()
    .receive(on: RunLoop.main)
    .sink { path in
        print(path.status)
    } 
```

SwiftUI and Combine are new to me, I spiked and re-implemented some UI components I have up in my sleeve, but nothing too complex. Building this client is helping me to explore and learn new APIs.

The protocols `Publisher`, `Subscriber`, and `Subscription` are examples of these APIs. To implement a custom publisher it's necessary to implement types conforming to two of these protocols.

First, a quick recap:

* [`Publisher`](https://developer.apple.com/documentation/combine/publisher) is the type which emits events over time,
* [`Subscriber`](https://developer.apple.com/documentation/combine/subscriber/) is the type which receives events published by the publisher, and
* [`Subscription`](https://developer.apple.com/documentation/combine/subscription) implements the link between publishers and subscribers.

The first step was to define my publisher interface. Since the monitor requires a queue to run, I decided to pass the queue as argument. In order to simplify its interface, a background queue is passed by default to the implementation.

```swift
extension NWPathMonitor {
  
    public func pathUpdatePublisher(
        on queue: DispatchQueue = .global(qos: .background)
    ) -> NWPathMonitor.PathMonitorPublisher {
      // ...
    }
}
```

The interface provides a hint of the first type to be implemented, `NWPathMonitor.PathMonitorPublisher`. Most of the code is boilerplate, and for this publisher, the `Output` expected is `NWPath`, without a `Failure`.

When the publisher *receives* a subscriber, the link between them is established.

```swift
extension NWPathMonitor {
   
  public struct PathMonitorPublisher: Publisher {

        // MARK: - Nested types
    
        public typealias Output = NWPath
        public typealias Failure = Never
    
        // MARK: - Properties
    
        private let monitor: NWPathMonitor
        private let queue: DispatchQueue
    
        // MARK: - Life cycle
    
        fileprivate init(
            monitor: NWPathMonitor,
            queue: DispatchQueue
        ) {
            self.monitor = monitor
            self.queue = queue
        }
    
        // MARK: - Public
    
        public func receive<S>(
            subscriber: S
        ) where S: Subscriber, S.Failure == Failure, S.Input == Output {
            let subscription = PathMonitorSubscription(
                subscriber: subscriber,
                monitor: monitor,
                queue: queue
            )
    
            subscriber.receive(
                subscription: subscription
            )
        }
    }
}
```

The `Subscription` fulfills the demand from the subscriber. As soon as there's demand, it starts monitoring changes using the monitor's callback, passing changes to the subscriber. And since `Subscription` conforms to [`Cancellable`](https://developer.apple.com/documentation/combine/Cancellable), `cancel()` can be used to stop the monitor.

```swift
extension NWPathMonitor {

    private final class PathMonitorSubscription<
        S: Subscriber
    >: Subscription where S.Input == NWPath {

        // MARK: - Nested types
  
        private let subscriber: S
        private let monitor: NWPathMonitor
        private let queue: DispatchQueue
  
        // MARK: - Life cycle
  
        init(
            subscriber: S,
            monitor: NWPathMonitor,
            queue: DispatchQueue
        ) {
            self.subscriber = subscriber
            self.monitor = monitor
            self.queue = queue
        }
  
        // MARK: - Public
  
        func request(
            _ demand: Subscribers.Demand
        ) {
            guard
                demand == .unlimited,
                monitor.pathUpdateHandler == nil
            else {
                return
            }
  
            monitor.pathUpdateHandler = { path in
                _ = self.subscriber.receive(path)
            }
  
            monitor.start(
                queue: queue
            )
        }
  
        func cancel() {
            monitor.cancel()
        }
    }
}
```

With all in place, the View Model becomes extremely simple and elegant. Below a View Model which publishes a boolean indicating the conectivity status to the View.

```swift
final class ViewModel: ObservableObject {

    // MARK: - Properties

    @Published var isConnected: Bool = false
    private var monitorCancellable: Cancellable?

    // MARK: - Life cycle

    init(
        pathMonitorPublisher: NWPathMonitor.PathMonitorPublisher
    ) {
        monitorCancellable = pathMonitorPublisher
            .receive(on: RunLoop.main)
            .map { $0.status == .satisfied }
            .assign(to: \.isConnected, on: self)
    }
}
```

Overall, I'm happy with the result, and am looking forward to using Combine (and SwiftUI) in production.

Happy hacking.

Resources:

* [Using Combine](https://heckj.github.io/swiftui-notes/)
* [Introducing Combine](https://www.apeth.com/UnderstandingCombine/start.html)

