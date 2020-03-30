GRDBCombine
===========

### A set of extensions for [SQLite], [GRDB.swift], and [Combine]

---

**Latest release**: [version 0.8.1](https://github.com/groue/GRDBCombine/tree/v0.8.1) (March 1, 2020) • [Release Notes]

**Requirements**: iOS 13.0+ / macOS 10.15+ / watchOS 6.0+ &bull; Swift 5.1+ / Xcode 11.0+

**Contact**: Report bugs and ask questions in [Github issues](https://github.com/groue/GRDBCombine/issues).

---

## Usage

To connect to the database, please refer to [GRDB](https://github.com/groue/GRDB.swift), the database library that supports GRDBCombine.

<details>
  <summary><strong>Asynchronously read from the database</strong></summary>
  
This publisher reads a single value and delivers it.

```swift
// AnyPublisher<[Player], Error>
let players = dbQueue.readPublisher { db in
    try Player.fetchAll(db)
}
```

</details>

<details>
  <summary><strong>Asynchronously write in the database</strong></summary>
  
This publisher delivers a single value, after the database has been updated.

```swift
// AnyPublisher<Void, Error>
let write = dbQueue.writePublisher { db in
    try Player(...).insert(db)
}

// AnyPublisher<Int, Error>
let newPlayerCount = dbQueue.writePublisher { db -> Int in
    try Player(...).insert(db)
    return try Player.fetchCount(db)
}
```

</details>

<details>
  <summary><strong>Observe changes in database values</strong></summary>

This publisher delivers fresh values whenever the database changes:

```swift
// A publisher with output [Player] and failure Error
let playersRequest = Player.all()
let cancellable = ValueObservation
    .tracking(value: playersRequest.fetchAll)
    .publisher(in: dbQueue)

// A publisher with output Int? and failure Error
let maxScoreRequest = SQLRequest<Int>(sql: "SELECT MAX(score) FROM player")
let maxScorePublisher = ValueObservation
    .tracking(value: maxScoreRequest.fetchOne)
    .publisher(in: dbQueue)
```

</details>

<details>
  <summary><strong>Observe database transactions</strong></summary>

This publisher delivers database connections whenever a database transaction has impacted an observed region:

```swift
// A publisher with output Database and failure Error
let playersRequest = Player.all()
let playersChangePublisher = DatabaseRegionObservation
    .tracking(playersRequest)
    .publisher(in: dbQueue)

// A publisher with output Database and failure Error
let maxScoreRequest = SQLRequest<Int>(sql: "SELECT MAX(score) FROM player")
let maxScoreChangePublisher = DatabaseRegionObservation
    .tracking(maxScoreRequest)
    .publisher(in: dbQueue)
```

</details>

Documentation
=============

- [Reference]
- [Installation]
- [Demo Application]
- [Asynchronous Database Access]
- [Database Observation]

## Installation

To use GRDBCombine with the [Swift Package Manager], add a dependency to your `Package.swift` file:

```swift
let package = Package(
    dependencies: [
        .package(url: "https://github.com/groue/GRDBCombine.git", ...)
    ]
)
```

To use GRDBCombine with [CocoaPods](http://cocoapods.org/), specify in your `Podfile`:

```ruby
pod 'GRDBCombine'
```


# Asynchronous Database Access

GRDBCombine provide publishers that perform asynchronous database accesses.

- [`readPublisher(receiveOn:value:)`]
- [`writePublisher(receiveOn:updates:)`]
- [`writePublisher(receiveOn:updates:thenRead:)`]


#### `DatabaseReader.readPublisher(receiveOn:value:)`

This methods returns a publisher that completes after database values have been asynchronously fetched.

```swift
// AnyPublisher<[Player], Error>
let players = dbQueue.readPublisher { db in
    try Player.fetchAll(db)
}
```

Any attempt at modifying the database completes subscriptions with an error.

When you use a [database queue] or a [database snapshot], the read has to wait for any eventual concurrent database access performed by this queue or snapshot to complete.

When you use a [database pool], reads are generally non-blocking, unless the maximum number of concurrent reads has been reached. In this case, a read has to wait for another read to complete. That maximum number can be [configured].

This publisher can be subscribed from any thread. A new database access starts on every subscription.

The fetched value is published on the main queue, unless you provide a specific scheduler to the `receiveOn` argument.


#### `DatabaseWriter.writePublisher(receiveOn:updates:)`

This method returns a publisher that completes after database updates have been successfully executed inside a database transaction.

```swift
// AnyPublisher<Void, Error>
let write = dbQueue.writePublisher { db in
    try Player(...).insert(db)
}

// AnyPublisher<Int, Error>
let newPlayerCount = dbQueue.writePublisher { db -> Int in
    try Player(...).insert(db)
    return try Player.fetchCount(db)
}
```

This publisher can be subscribed from any thread. A new database access starts on every subscription.

It completes on the main queue, unless you provide a specific [scheduler] to the `receiveOn` argument.

When you use a [database pool], and your app executes some database updates followed by some slow fetches, you may profit from optimized scheduling with [`writePublisher(receiveOn:updates:thenRead:)`]. See below.


#### `DatabaseWriter.writePublisher(receiveOn:updates:thenRead:)`

This method returns a publisher that completes after database updates have been successfully executed inside a database transaction, and values have been subsequently fetched:

```swift
// AnyPublisher<Int, Error>
let newPlayerCount = dbQueue.writePublisher(
    updates: { db in try Player(...).insert(db) }
    thenRead: { db, _ in try Player.fetchCount(db) })
}
```

It publishes exactly the same values as [`writePublisher(receiveOn:updates:)`]:

```swift
// AnyPublisher<Int, Error>
let newPlayerCount = dbQueue.writePublisher { db -> Int in
    try Player(...).insert(db)
    return try Player.fetchCount(db)
}
```

The difference is that the last fetches are performed in the `thenRead` function. This function accepts two arguments: a readonly database connection, and the result of the `updates` function. This allows you to pass information from a function to the other (it is ignored in the sample code above).

When you use a [database pool], this method applies a scheduling optimization: the `thenRead` function sees the database in the state left by the `updates` function, and yet does not block any concurrent writes. This can reduce database write contention. See [Advanced DatabasePool](https://github.com/groue/GRDB.swift/blob/master/README.md#advanced-databasepool) for more information.

When you use a [database queue], the results are guaranteed to be identical, but no scheduling optimization is applied.

This publisher can be subscribed from any thread. A new database access starts on every subscription.

It completes on the main queue, unless you provide a specific [scheduler] to the `receiveOn` argument.


# Database Observation

Database Observation publishers are based on GRDB's [ValueObservation] and [DatabaseRegionObservation]. Please refer to their documentation for more information. If your application needs change notifications that are not built in GRDBCombine, check the general [Database Changes Observation] chapter.

- [`ValueObservation.publisher(in:)`]
- [`DatabaseRegionObservation.publisher(in:)`]


#### `ValueObservation.publisher(in:)`

GRDB's [ValueObservation] tracks changes in database values. You can turn it into a Combine publisher:

```swift
let observation = ValueObservation.tracking { db in
    try Player.fetchAll(db)
}

// A publisher with output [Player] and failure Error
let publisher = observation.publisher(in: dbQueue)
```

This publisher has the same behavior as ValueObservation:

- It notifies an initial value before the eventual changes.
- It may coalesce subsequent changes into a single notification.
- It may notify consecutive identical values. You can filter out the undesired duplicates with the `removeDuplicates()` Combine operator, but we suggest you have a look at the [removeDuplicates()](https://github.com/groue/GRDB.swift/blob/master/README.md#valueobservationremoveduplicates) GRDB operator also.
- It stops emitting any value after the database connection is closed. But it never completes.
- By default, it notifies the initial value, as well as eventual changes and errors, on the main thread, asynchronously.
    
    This can be configured with the `scheduling(_:)` method. It does not accept a Combine scheduler, but a [GRDB scheduler](https://github.com/groue/GRDB.swift/blob/master/README.md#valueobservation-scheduling).
    
    For example, the `.immediate` scheduler makes sure the initial value is notified immediately when the publisher is subcribed. It can help your application update the user interface without having to wait for any asynchronous notifications:
    
    ```swift
    let publisher = observation
        .publisher(in: dbQueue)
        .scheduling(.immediate)
    ```
    
    Note that the `.immediate` scheduler requires that the publisher is subscribed from the main thread. It raises a fatal error otherwise.

See [ValueObservation Scheduling](https://github.com/groue/GRDB.swift/blob/master/README.md#valueobservation-scheduling) for more information.

:warning: **ValueObservation and Data Consistency**

When you compose ValueObservation publishers together with the [combineLatest](https://developer.apple.com/documentation/combine/publisher/3333677-combinelatest) operator, you lose all guarantees of [data consistency](https://en.wikipedia.org/wiki/Consistency_(database_systems)).

Instead, compose requests or value observations together before building one **single** value publisher.

For example, fetch all requested values in a single observation (this is the technique used in the [Demo Application]):

```swift
// DATA CONSISTENCY GUARANTEED
let hallOfFamePublisher = ValueObservation
    .tracking { db -> HallOfFame in
        let playerCount = try Player.fetchCount(db)
        let bestPlayers = try Player.limit(10).orderedByScore().fetchAll(db)
        return HallOfFame(playerCount:playerCount, bestPlayers:bestPlayers)
    }
    .publisher(in: dbQueue)
```

Or combine observations together:

```swift
// DATA CONSISTENCY GUARANTEED
let playerCountObservation = ValueObservation
    .tracking(value: Player.fetchCount)

let bestPlayersObservation = ValueObservation
    .tracking(value: Player.limit(10).orderedByScore().fetchAll)

let hallOfFameObservation = ValueObservation.combine(
    playerCountObservation,
    bestPlayersObservation)
    .map(HallOfFame.init(playerCount:bestPlayers:))

let hallOfFamePublisher = hallOfFameObservation.publisher(in: dbQueue)
```

See [ValueObservation] for more information.


#### `DatabaseRegionObservation.publisher(in:)`

GRDB's [DatabaseRegionObservation] notifies all transactions that impact a tracked database region. You can turn it into a Combine publisher:

```swift
let request = Player.all()
let observation = DatabaseRegionObservation.tracking(request)

// A publisher with output Database and failure Error
let publisher = observation.publisher(in: dbQueue)
```

This publisher can be created and subscribed from any thread. It delivers database connections in a "protected dispatch queue", serialized with all database updates. It only completes when a database error happens.

```swift
let request = Player.all()
let cancellable = DatabaseRegionObservation
    .tracking(request)
    .publisher(in: dbQueue)
    .sink(
        receiveCompletion: { completion in ... },
        receiveValue: { (db: Database) in
            print("Players have changed.")
        })

try dbQueue.write { db in
    try Player(name: "Arthur").insert(db)
    try Player(name: "Barbara").insert(db)
} 
// Prints "Players have changed."

try dbQueue.write { db in
    try Player.deleteAll(db)
}
// Prints "Players have changed."
```

See [DatabaseRegionObservation] for more information.


[Associations]: https://github.com/groue/GRDB.swift/blob/master/Documentation/AssociationsBasics.md
[Asynchronous Database Access]: #asynchronous-database-access
[@ObservedObject]: https://developer.apple.com/documentation/swiftui/observedobject
[Combine]: https://developer.apple.com/documentation/combine
[Database Changes Observation]: https://github.com/groue/GRDB.swift/blob/master/README.md#database-changes-observation
[Database Observation]: #database-observation
[DatabaseRegionObservation]: https://github.com/groue/GRDB.swift/blob/master/README.md#databaseregionobservation
[Demo Application]: Documentation/Demo/README.md
[GRDB.swift]: https://github.com/groue/GRDB.swift
[Installation]: #installation
[Reference]: https://groue.github.io/GRDBCombine/docs/0.7/index.html
[Release Notes]: CHANGELOG.md
[SQLite]: http://sqlite.org
[Swift Package Manager]: https://swift.org/package-manager/
[ValueObservation]: https://github.com/groue/GRDB.swift/blob/master/README.md#valueobservation
[`DatabaseRegionObservation.publisher(in:)`]: #databaseregionobservationpublisherin
[`ValueObservation.publisher(in:)`]: #valueobservationpublisherin
[`readPublisher(receiveOn:value:)`]: #databasereaderreadpublisherreceiveonvalue
[`writePublisher(receiveOn:updates:)`]: #databasewriterwritepublisherreceiveonupdates
[`writePublisher(receiveOn:updates:thenRead:)`]: #databasewriterwritepublisherreceiveonupdatesthenread
[configured]: https://github.com/groue/GRDB.swift/blob/master/README.md#databasepool-configuration
[database connection]: https://github.com/groue/GRDB.swift/blob/master/README.md#database-connections
[database pool]: https://github.com/groue/GRDB.swift/blob/master/README.md#database-pools
[database queue]: https://github.com/groue/GRDB.swift/blob/master/README.md#database-queues
[database snapshot]: https://github.com/groue/GRDB.swift/blob/master/README.md#database-snapshots
[query interface]: https://github.com/groue/GRDB.swift/blob/master/README.md#requests
[scheduler]: https://developer.apple.com/documentation/combine/scheduler
