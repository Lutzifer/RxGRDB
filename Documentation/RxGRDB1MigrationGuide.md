Migrating From RxGRDB 0.x to RxGRDB 1.0
=======================================

**This guide aims at helping you upgrading your applications.**

RxGRDB 1.0 comes with breaking changes. Those changes have the vanilla [GRDB], [GRDBCombine], and [RxGRDB], offer a consistent behavior. This greatly helps choosing or switching your preferred database API. In previous versions, the three companion libraries used to have subtle differences that were just opportunities for bugs.

1. **RxGRDB requirements have been bumped**
    
    - **Swift 5.2+** (was Swift 5.0+)
    - **Xcode 11.4+** (was Xcode 11.0+)
    - **iOS 10.0+** (was iOS 9.0+)
    - **macOS 10.10+** (was macOS 10.9+)
    - tvOS 9.0+ (unchanged)
    - watchOS 2.0+ (unchanged)
    - **GRDB 5.0+** (was GRDB 4.1+)

2. **RxGRDB 1.0 requires GRDB 5**, which comes with changes in the runtime behavior of [ValueObservation], and directly impacts its derived RxGRDB observable. So please check [Migrating From GRDB 4 to GRDB 5] first.

3. **Asynchronous write in the database**
    
    The `rx.writeAndReturn` method has been removed, and renamed `rx.write`:
    
    ```swift
    // BEFORE: RxGRDB 0.x
    // Single<Int>
    let newPlayerCount = dbQueue.rx.writeAndReturn { db -> Int in
        try Player(...).insert(db)
        return try Player.fetchCount(db)
    }
    
    // NEW: RxGRDB 1.0
    // Single<Int>
    let newPlayerCount = dbQueue.rx.write { db -> Int in
        try Player(...).insert(db)
        return try Player.fetchCount(db)
    }
    ```
    
    The `rx.write` method now returns an RxSwift [Single]:
    
    ```swift
    // BEFORE: RxGRDB 0.x
    // Completable
    let write = dbQueue.rx.write { db in
        try Player(...).insert(db)
    }
    
    // NEW: RxGRDB 1.0
    // Single<Void>
    let write = dbQueue.rx.write { db in
        try Player(...).insert(db)
    }
    ```
    
    You can ignore its value and turn it into a [Completable] with the `asCompletable` operator.
    
    ```swift
    // NEW: RxGRDB 1.0
    // Completable
    let write = dbQueue.rx
        .write { db in try Player(...).insert(db) }
        .asCompletable()
    ```

4. **Database Observation**
    
    RxGRDB 1.0 has a single way to start observing database values. First create a [ValueObservation], and then turn it into an RxSwift observable:
    
    ```swift
    let observation = ValueObservation.tracking { db in
        try Player.fetchAll(db)
    }

    // Observable<[Player]>
    let observable = observation.rx.observe(in: dbQueue)
    ```
    
    In previous versions of RxGRDB, ValueObservation observables used to immediately emit an initial value, right on subscription. This is no longer the case: now the initial value is emitted asynchronously. To restore the previous behavior, add the `scheduling: .immediate` parameter:
    
    ```swift
    // Immediate notification of the initial value
    let disposable = observation.rx
        .observe(
            in: dbQueue,
            scheduling: .immediate) // <-
        .subscribe(
            onNext: { players: [Player] in print("fresh players: \(players)") },
            onError: { error in ... })
    // <- here "fresh players" is already printed.
    ```
    
    Note that the `.immediate` scheduler requires that the observable is subscribed from the main thread. It raises a fatal error otherwise.
    
    The `observe(in:startImmediately:observeOn:)` method no longer exists:
    
    - Instead of `startImmediately: false`, use the `skip(1)` RxSwift operator as a replacement.
    - Instead of `observeOn`, use the `scheduling` parameter (see [`ValueObservation.rx.observe(in:scheduling:)`]), or the `observeOn(_:)` RxSwift operator.
    
    Other former ways to observe the database are no longer available:
    
    ```swift
    // BEFORE: RxGRDB 0.x
    let request = Player.all()
    request.rx.observeCount(in: dbQueue) // Observable<Int>
    request.rx.observeFirst(in: dbQueue) // Observable<Player?>
    request.rx.observeAll(in: dbQueue)   // Observable<[Player]>
    request.rx.changes(in: dbQueue)      // Observable<Database>
    
    // NEW: RxGRDB 1.0
    ValueObservation.tracking(request.fetchCount).rx.observe(in: dbQueue) // Observable<Int>
    ValueObservation.tracking(request.fetchOne).rx.observe(in: dbQueue)   // Observable<Player?>
    ValueObservation.tracking(request.fetchAll).rx.observe(in: dbQueue)   // Observable<[Player]>
    DatabaseRegionObservation(tracking: request).rx.observe(in: dbQueue)  // Observable<Database>
    ```

[GRDB]: https://github.com/groue/GRDB.swift
[GRDBCombine]: https://github.com/groue/GRDBCombine
[RxGRDB]: https://github.com/RxSwiftCommunity/RxGRDB
[ValueObservation]: https://github.com/groue/GRDB.swift/blob/master/README.md#valueobservation
[Migrating From GRDB 4 to GRDB 5]: https://github.com/groue/GRDB.swift/blob/master/Documentation/GRDB5MigrationGuide.md
[Single]: https://github.com/ReactiveX/RxSwift/blob/master/Documentation/Traits.md#single
[Completable]: https://github.com/ReactiveX/RxSwift/blob/master/Documentation/Traits.md#completable
[`ValueObservation.rx.observe(in:scheduling:)`]: ../README.md#valueobservationrxobserveinscheduling