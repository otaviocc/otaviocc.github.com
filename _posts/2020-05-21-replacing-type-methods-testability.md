---
title:  How to replace type methods in Swift to improve testability
date:   2020-05-21 12:00:00 -0100
layout:   default
description: Is it possible to test code which interact with class/static methods?
---

Like them or not, [type methods](https://docs.swift.org/swift-book/LanguageGuide/Methods.html), aka *class methods* or *static methods*, are heavily used in Swift and part of our daily lives as developers. 

From analytics trackers to requesting system permissions, we’ve all had to face type methods from external libraries in which we had no control over. Testing code that interacts with them might seem hard without using [method swizzling](http://nshipster.com/method-swizzling/), but fortunately, this doesn't always have to be the case.

Let's suppose there's a class called `APIWrapper`, which implements a type method that requests search results from an endpoint.

```swift
public class APIWrapper {
    public static func search(
        query: String,
        completion: @escaping (Result<[String], Error>) -> Void
    )
}
```

The method takes a query, `String`, and a completion block, `(Result<[String], Error>) -> Void`, which is triggered once the request finishes. Its internal implementation doesn't really matter since it could be from an external framework or generated by a [code generator](https://github.com/swagger-api/swagger-codegen) from the API specification.

Consuming the method is straightforward:

```swift
APIWrapper.search(query: "fancy restaurant") { result in
    // do something with 'result'
}
```

But testing the part of the application that interacts with it isn't that obvious. Especially when we want to:

1. avoid making real network requests
2. stress all the possibilities the interface allows. For instance:
   - Is the search method called with the correct query string?
   - What happens when the completion block returns an empty array?
   - What happens when the completion block returns an array of results?
   - What happens when the completion block is called with an error?

### **Basic Data Source implementation**

The snippet below shows `SearchResultsTableViewDataSource`, a `UITableViewDataSource` subclass which uses the API wrapper to populate the table view when the network request completes. Testing it triggers a real network request and doesn't provide what's needed to test the query passed to the wrapper and the different behaviors when the result returns.

```swift
final class SearchResultsTableViewDataSource: NSObject, UITableViewDataSource {
    private let tableView: UITableView
    private var results: [String] = []

    init(
        tableView: UITableView
    ) {
        self.tableView = tableView
    }

    func fetchSearchResuls(query: String) {
        APIWrapper.search(
            query: query
        ) { [weak self] result in
            switch result {
            case .success(let results): self?.results = results
            case .failure: self?.results = []
            }
            self?.tableView.reloadData()
        }
    }

    // ... UITableViewDataSource methods ...
}
```

If `APIWrapper` had an instance method instead of a type method, two easy and obvious solutions would emerge: *protocols* and *subclasses*. Using either/or it would be possible to replace the wrapper with a [test double](https://martinfowler.com/bliki/TestDouble.html) that captures the query parameter and completion block for verifying expectations.

### **Dependency injection to the rescue**

In the example above, the API wrapper interface can't be changed and has to stay as a type method, therefore the easiest way to overcome this is via [dependency injection](https://en.wikipedia.org/wiki/Dependency_injection). In this case, *method injection* via object initialization

The code below stores a method which matches the qualified symbol name of the wrapper method, using it to perform the search. In order to simplify the `SearchResultsTableViewDataSource` interface, the wrapper's search method is assigned using a [default parameter value](https://docs.swift.org/swift-book/LanguageGuide/Functions.html#ID169).

```swift
final class SearchResultsTableViewDataSource: NSObject, UITableViewDataSource  {
    typealias SearcherCompletion = (Result<[String], Error>) -> Void
    typealias Searcher = (String, @escaping SearcherCompletion) -> Void

    private let tableView: UITableView
    private var results: [String] = []
    private var searcher: Searcher

    init(
        tableView: UITableView,
        searcher: @escaping Searcher = APIWrapper.search(query:completion:)
    ) {
        self.tableView = tableView
        self.searcher = searcher
    }

    func fetchSearchResuls(query: String) {
        searcher(query) { [weak self] result in
            switch result {
            case .success(let results): self?.results = results
            case .failure: self?.results = []
            }
            self?.tableView.reloadData()
        }
    }

    // ... UITableViewDataSource methods ...
}
```

### **Testing with Dependency Injection**

Finally, testing becomes trivial. The injected method is used to capture the query term and the completion block passed to the API wrapper. Once captured, these can be used to verify expectations.

```swift
final class SearchResultsTableViewDataSourceTests: XCTestCase {
    var tableView: UITableView!
    var dataSource: SearchResultsTableViewDataSource?
    var lastSearchQuery: String?
    var lastSearchCompletion: SearchResultsTableViewDataSource.SearcherCompletion?

    override func setUp() {
        super.setUp()

        tableView = UITableView()

        dataSource = SearchResultsTableViewDataSource(
            tableView: tableView
        ) { query, completion in
            self.lastSearchQuery = query
            self.lastSearchCompletion = completion
        }
    }

    func testFetchSearchResultsQuery() {
        // When the method is called
        dataSource?.fetchSearchResuls(query: "Mocked Search Query")

        XCTAssertEqual(
            lastSearchQuery,
            "Mocked Search Query",
            "It calls the API wrapper with the correct search query"
        )
    }

    func testFetchSearchResultsWithoutResults() {
        // When the method is called
        dataSource?.fetchSearchResuls(query: "Mocked Search Query")

        // And the API wrapper completion block is called with an empty array
        lastSearchCompletion?(.success([]))

        XCTAssertEqual(
            dataSource?.tableView(tableView, numberOfRowsInSection: 0),
            0,
            "It empties the table view"
        )
    }

    func testFetchSearchResultsWithResults() {
        // When the method is called
        dataSource?.fetchSearchResuls(query: "Mocked Search Query")

        // And the API wrapper completion block is called with results
        lastSearchCompletion?(.success(["result 1", "result 2"]))

        XCTAssertEqual(
            dataSource?.tableView(tableView, numberOfRowsInSection: 0),
            2,
            "It adds the correct number of results to the table view"
        )

        XCTAssertEqual(
            dataSource?.tableView(
                tableView,
                cellForRowAt: IndexPath(row: 0, section: 0)
            ).textLabel?.text,
            "result 1",
            "It configures the first cell"
        )

        XCTAssertEqual(
            dataSource?.tableView(
                tableView,
                cellForRowAt: IndexPath(row: 1, section: 0)
            ).textLabel?.text,
            "result 2",
            "It configures the second cell"
        )
    }

    func testFetchSearchResultsWithError() {
        // When the method is called
        dataSource?.fetchSearchResuls(query: "Mocked Search Query")

        // And the API wrapper completion block is called with an error
        enum MockError: Error { case someGenericError }
        lastSearchCompletion?(.failure(MockError.someGenericError))

        XCTAssertEqual(
            dataSource?.tableView(tableView, numberOfRowsInSection: 0),
            0,
            "It empties the table view"
        )
    }
}
```

More importantly, not only does this method work with type methods but also with *free functions*, such as `SecItemAdd(_:_:)` used to [add items to the Keychain](https://developer.apple.com/documentation/security/1401659-secitemadd).

Happy testing.
