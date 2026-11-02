# The Complete iOS Data Persistence Guide (2026)

Data persistence on iOS is not a single choice but a **two-dimensional decision**: you must pick the right **format** (structured records vs. files vs. serialized documents) *and* the right **location** (local vs. cloud, and which directory). Getting either dimension wrong causes backup bloat, data loss, or painful refactors later.

This guide covers the persistence tools available in 2026, ties them together with the **Repository pattern**, and finishes with **modern networking integration** into a coherent data layer.

> **Scope note:** Security topics are deliberately excluded. Credential storage, encryption at rest, and transport security hardening are separate concerns and are not covered here.

---

## 1. The Decision Framework: Format × Location

Before choosing an API, ask two questions:

1. **What shape is the data?**
   - Structured records you need to query, filter, or partially update → database (SwiftData, Core Data, SQLite/GRDB).
   - Discrete files (images, PDFs, videos, exports) → `FileManager`.
   - A small, self-contained object or object tree you load and save as a whole → serialization (`Codable`) written to disk with `FileManager`.

2. **What is the data's lifecycle?**
   This determines the directory, and the directory determines backup behavior.

| Directory | Use Case | Backed Up? | Visible to User? |
|---|---|---|---|
| **Documents** | User-created content (edited photos, exported PDFs) | Yes (iCloud backup) | Only if `UIFileSharingEnabled` |
| **Application Support** | App-generated data that must survive | Yes | No |
| **Caches** | Re-downloadable content | **No** (excluded) | No |
| **tmp** | Scratch space | **No** (purged between launches) | No |

A large video file in `Documents` goes straight into iCloud backup and can trigger an App Store rejection for backup bloat; the same file in `Caches` is correct because it can be re-downloaded.

---

## 2. Lightweight Persistence: UserDefaults

`UserDefaults` is a property list stored on disk. It is suitable **only** for non-sensitive, lightweight preferences: theme selection, feature flags, "has seen onboarding," last-selected tab, etc.

Anything beyond a handful of scalar preferences signals that you should move to a proper database. If you ever find yourself storing a collection or an object graph in `UserDefaults`, stop—that is a database problem in disguise.

```swift
UserDefaults.standard.set(true, forKey: "hasCompletedOnboarding")
UserDefaults.standard.set("dark", forKey: "preferredTheme")
```

`UserDefaults` can hold small `Codable` values, but treat that as an escape hatch rather than a pattern. It is covered here only for completeness; real serialized data belongs on disk (§4).

---

## 3. Files and Raw Bytes: FileManager

When the data is a **file**—an image, a PDF, a downloaded video, a JSON export—rather than a record in a database, `FileManager` is the correct tool.

### Write user-created content to Documents (backed up)

```swift
let documents = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0]
let fileURL = documents.appendingPathComponent("MyDocument.pdf")
try data.write(to: fileURL)
```

### Write re-downloadable content to Caches (excluded from backup, purged under pressure)

```swift
let caches = FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask)[0]
let cacheURL = caches.appendingPathComponent("thumbnail.jpg")
try imageData.write(to: cacheURL)
```

If a large file *must* live in `Documents` but should not bloat backups, set `isExcludedFromBackup = true` explicitly on the URL's resource values. Caches are purged under storage pressure and `tmp` is purged between launches, so anything in either must be recreatable.

---

## 4. Archiving and Codable: Serializing Objects to Disk

`FileManager` moves bytes. This section covers **turning your Swift types into those bytes** and back. It is the natural bridge between "I have raw files" and "I have a database," and for many apps it is the entire persistence story.

### 4.1 Serialization vs. Archiving

These two terms are often used interchangeably, but they describe different guarantees:

| | **Serialization** (`Codable`) | **Archiving** (`NSCoding` / `NSKeyedArchiver`) |
|---|---|---|
| What is preserved | Values and the structural hierarchy (nested structs, arrays) | Values **plus object identity and relationships** |
| Shared references | Duplicated — two properties pointing at the same instance become two copies | Preserved — the archive records the reference graph |
| Reference cycles | Impossible (value types cannot cycle) | Supported |
| Format | JSON, property list (XML/binary) | Keyed binary archive (plist-based) |
| Swift-native | Yes | Objective-C era; still supported |
| Typical use | App state, cached API responses, lightweight documents | Legacy code, UIKit state restoration, `NSUserDefaults` internals |

**Rule of thumb:** reach for `Codable` unless you specifically need to preserve *object identity* across a save/load cycle. Most apps do not.

### 4.2 The `Codable` Protocol

`Codable` is a type alias for `Encodable & Decodable`:

```swift
public typealias Codable = Encodable & Decodable
```

Conformance is usually synthesized by the compiler for structs, enums (with or without associated values), and classes whose stored properties are themselves `Codable`:

```swift
struct Trip: Codable, Identifiable {
    var id: UUID
    var name: String
    var destination: String
    var startDate: Date
    var endDate: Date
    var tags: [String]
}
```

That is the entire declaration. No boilerplate, no per-property methods.

### 4.3 Customizing the Mapping

Three customization points cover almost everything:

**Rename keys with `CodingKeys`** — useful for server contracts, `snake_case`, or renaming a property without breaking stored data:

```swift
struct Trip: Codable {
    var id: UUID
    var name: String

    enum CodingKeys: String, CodingKey {
        case id
        case name = "trip_name"
    }
}
```

Note that declaring `CodingKeys` **replaces** the synthesized key set entirely—every property you want encoded must appear in it.

**Custom encoding and decoding** — when the shape on disk differs from the in-memory shape:

```swift
struct Trip: Codable {
    var id: UUID
    var startDate: Date
    var endDate: Date

    init(from decoder: any Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        id = try container.decode(UUID.self, forKey: .id)
        // Store dates as epoch seconds instead of ISO 8601
        let start = try container.decode(Double.self, forKey: .startDate)
        let end = try container.decode(Double.self, forKey: .endDate)
        startDate = Date(timeIntervalSince1970: start)
        endDate = Date(timeIntervalSince1970: end)
    }

    func encode(to encoder: any Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(id, forKey: .id)
        try container.encode(startDate.timeIntervalSince1970, forKey: .startDate)
        try container.encode(endDate.timeIntervalSince1970, forKey: .endDate)
    }
}
```

**`CodingUserInfoKey`** — thread contextual information (a date formatter, an environment flag) through the encoder/decoder without touching the model:

```swift
extension CodingUserInfoKey {
    static let apiVersion = CodingUserInfoKey(rawValue: "apiVersion")!
}

let encoder = JSONEncoder()
encoder.userInfo[.apiVersion] = 2
```

### 4.4 Choosing a Coder

`Codable` is format-agnostic; the coder decides the bytes.

| Coder | Output | Best for |
|---|---|---|
| **`JSONEncoder` / `JSONDecoder`** | UTF-8 JSON | Interop, debuggability, mirroring API payloads, human inspection |
| **`PropertyListEncoder` / `PropertyListDecoder`** (`.xml`) | XML plist | Small config-like documents that a human might edit |
| **`PropertyListEncoder` / `PropertyListDecoder`** (`.binary`) | Binary plist | Compact, fast local storage of Apple-native data |
| Custom coder | Anything | Protobuf, CBOR, YAML via a third-party package |

Configure the coder once and share it—formatter and strategy objects are expensive to create:

```swift
enum Coders {
    static let json: JSONEncoder = {
        let encoder = JSONEncoder()
        encoder.outputFormatting = [.prettyPrinted, .sortedKeys] // omit in release if size matters
        encoder.dateEncodingStrategy = .iso8601
        return encoder
    }()

    static let jsonDecoder: JSONDecoder = {
        let decoder = JSONDecoder()
        decoder.dateDecodingStrategy = .iso8601
        return decoder
    }()

    static let plist: PropertyListEncoder = {
        let encoder = PropertyListEncoder()
        encoder.outputFormat = .binary
        return encoder
    }()
}
```

**Dates are the most common pitfall.** `.iso8601` is readable but truncates sub-second precision and, by default, drops the time zone nuance on decode. If you need exact round-tripping, use `.secondsSince1970` / `.millisecondsSince1970` or `.deferredToDate` (which stores the raw `timeIntervalSinceReferenceDate`). Whichever you choose, the encoder and decoder **must agree**.

### 4.5 Writing Serialized Objects to Disk

Coders return `Data`; `FileManager` (or `Data.write(to:)`) puts it on disk. The two compose directly:

```swift
struct TripStore {
    private let fileURL: URL

    init() {
        let support = FileManager.default.urls(
            for: .applicationSupportDirectory,
            in: .userDomainMask
        )[0]
        try? FileManager.default.createDirectory(
            at: support, withIntermediateDirectories: true
        )
        fileURL = support.appendingPathComponent("trips.json")
    }

    func save(_ trips: [Trip]) throws {
        let data = try Coders.json.encode(trips)
        try data.write(to: fileURL, options: [.atomic, .completeFileProtection])
    }

    func load() throws -> [Trip] {
        let data = try Data(contentsOf: fileURL)
        return try Coders.jsonDecoder.decode([Trip].self, from: data)
    }
}
```

Two details worth calling out:

- **`.atomic`** writes to a temporary file and renames it into place. Without it, a crash mid-write leaves a truncated, unparseable file. Use it for every document you care about.
- **Directory choice follows §1.** App-generated state that must survive → `Application Support`. A re-downloadable cached payload → `Caches`. User-facing exports → `Documents`.

**Handle the missing-file case explicitly.** A first launch has no file, and that is not an error:

```swift
func load() throws -> [Trip] {
    guard FileManager.default.fileExists(atPath: fileURL.path) else {
        return []  // First launch — not an error
    }
    let data = try Data(contentsOf: fileURL)
    return try Coders.jsonDecoder.decode([Trip].self, from: data)
}
```

### 4.6 Schema Versioning

A `Codable` document is a schema, whether or not you admit it. The moment you rename a property or add a required field, previously saved files stop decoding. Plan for it from day one by embedding a version:

```swift
struct TripFile: Codable {
    var schemaVersion: Int
    var trips: [Trip]
}
```

On load, branch on the version and migrate in memory before use. Decoding is strict by default, but you can make additions non-breaking by declaring new properties optional (or giving them defaults via a custom `init(from:)` using `decodeIfPresent`). Renames and type changes always require an explicit migration step.

### 4.7 Archiving with `NSCoding` and `NSKeyedArchiver`

Archiving predates `Codable` and remains in the SDK. You will meet it in legacy codebases, in UIKit state restoration, and whenever an API hands you an `NSKeyedArchiver` blob.

A class opts in by conforming to `NSCoding` (modern code should use `NSSecureCoding`, which requires declaring the classes an archive may contain):

```swift
final class LegacyTrip: NSObject, NSSecureCoding {
    static var supportsSecureCoding: Bool { true }

    let id: UUID
    let name: String

    init(id: UUID, name: String) {
        self.id = id
        self.name = name
    }

    func encode(with coder: NSCoder) {
        coder.encode(id as NSUUID, forKey: "id")
        coder.encode(name, forKey: "name")
    }

    required init?(coder: NSCoder) {
        guard let id = coder.decodeObject(of: NSUUID.self, forKey: "id") as UUID?,
              let name = coder.decodeObject(of: NSString.self, forKey: "name") as String?
        else { return nil }
        self.id = id
        self.name = name
    }
}
```

Encoding and decoding go through a keyed archive:

```swift
let data = try NSKeyedArchiver.archivedData(
    withRootObject: trip,
    requiringSecureCoding: true
)

let restored = try NSKeyedUnarchiver.unarchivedObject(
    ofClass: LegacyTrip.self,
    from: data
)
```

**What archiving buys you** is the reference graph: if two properties hold the *same* instance, the archive stores it once and restores both properties pointing at the same object. Cycles survive too. `Codable` cannot express this—value semantics mean shared references collapse into copies, and a cycle simply cannot be constructed from structs.

**Why it is not the default anymore:** the API is Objective-C–shaped, the archive format is opaque and undocumented, and conformance is manual boilerplate per class. For new code, `Codable` covers every case except genuine object-identity graphs, and those are usually better served by a database with explicit relationships (§5, §6).

### 4.8 When to Graduate to a Database

Serializing a whole document works beautifully—until it doesn't. The signals:

- **You load the entire file to change one field.** Every edit rewrites everything.
- **You need to query or filter.** Filtering an in-memory array of 50,000 decoded objects is slow and memory-hungry.
- **The collection grows unbounded.** A single JSON file that reaches tens of megabytes is a performance and corruption risk.
- **You need relationships with integrity guarantees.** Cascading deletes and inverse maintenance belong to a real object graph manager.
- **Multiple screens or contexts write concurrently.** Whole-file rewrites race; last writer wins and silently destroys data.
- **You need partial or lazy loading.** Faulting exists in Core Data/SwiftData; a JSON file has no equivalent.

Below those thresholds, `Codable` + `FileManager` is not a compromise—it is the correct, simplest tool. Above them, move to SwiftData or Core Data.

**One useful middle ground:** SwiftData and Core Data can store `Codable` value types *as attributes* (SwiftData directly, Core Data via a value transformer). This is ideal for small nested structures—a `Trip.preferences` struct, an `Address`, a set of tags—that belong to a record but do not need their own table. It is not a substitute for modeling genuine entities: anything you need to query independently should be its own entity.

---

## 5. Structured Local Data: SwiftData (The Default for New Apps)

SwiftData is Apple's modern, declarative persistence framework built on Core Data. For new projects targeting iOS 17+, it is the recommended default. It uses macros to eliminate boilerplate and integrates natively with SwiftUI.

### Model definition with `@Model`

```swift
import SwiftData

@Model
final class Trip {
    #Index<Trip>([\.name], [\.startDate])
    #Unique<Trip>([\.name, \.startDate])

    @Attribute(.preserveValueOnDeletion) var name: String
    var destination: String
    var startDate: Date
    var endDate: Date

    @Relationship(deleteRule: .cascade, inverse: \BucketListItem.trip)
    var bucketList: [BucketListItem] = []

    init(name: String, destination: String, startDate: Date, endDate: Date) {
        self.name = name
        self.destination = destination
        self.startDate = startDate
        self.endDate = endDate
    }
}
```

### Container setup

```swift
@main
struct TravelApp: App {
    var body: some Scene {
        WindowGroup { ContentView() }
            .modelContainer(for: [Trip.self, BucketListItem.self])
    }
}
```

### Querying with `@Query`

```swift
struct TripListView: View {
    @Query(
        filter: #Predicate<Trip> { $0.startDate > .now },
        sort: [SortDescriptor(\Trip.startDate, order: .forward)]
    )
    private var upcomingTrips: [Trip]

    @Environment(\.modelContext) private var context

    var body: some View {
        List(upcomingTrips) { trip in Text(trip.name) }
    }
}
```

`#Predicate` is the type-safe replacement for `NSPredicate`. SwiftData auto-saves changes to the `ModelContext` (explicit `context.save()` is available but rarely required).

### SwiftData with iCloud

SwiftData syncs automatically to a **private** CloudKit database when the app has iCloud entitlements:

```swift
let config = ModelConfiguration(cloudKitDatabase: .automatic)
let container = try ModelContainer(for: Trip.self, configurations: config)
```

For a **public** CloudKit database, Core Data remains necessary.

### Migrating from Core Data

There are three adoption paths:

1. **Full conversion** — move all entities to SwiftData.
2. **Incremental migration** — convert feature by feature.
3. **Coexistence** — run both stacks pointed at the **same store URL** so screens can migrate one at a time. When coexisting, namespace Core Data classes (`CDTrip`) to avoid collisions and enable persistent history tracking (see §6.8).

---

## 6. Core Data: Concepts Overview

Core Data is no longer the default choice for new apps, but it remains essential for existing apps, for apps that need a **public CloudKit database**, and for schema evolution that benefits from **staged migrations**. Understanding its concepts also makes SwiftData far less mysterious, since SwiftData is a Swift-native façade over the same engine.

### 6.1 What Core Data Actually Is

Core Data is an **object graph management and persistence framework**—not a database. You model entities and relationships; Core Data tracks changes to instances of those entities and writes them to a **persistent store**, which is usually SQLite but can also be binary or in-memory.

The distinction matters: Core Data manages object identity, relationships, change tracking, and faulting for you. The SQLite file underneath is an implementation detail.

### 6.2 The Core Data Stack

Five pieces, layered:

| Component | Role |
|---|---|
| **NSManagedObjectModel** | Describes entities, attributes, relationships (`.xcdatamodeld`, compiled to `.momd`) |
| **NSPersistentStoreCoordinator** | Mediates between the model and the store; owns the store instances |
| **NSPersistentStore** | The physical storage (SQLite, binary, in-memory) |
| **NSManagedObjectContext** | A scratchpad of managed objects; tracks inserts, updates, deletes |
| **NSPersistentContainer** | Convenience wrapper (iOS 10+) that builds the whole stack from a model name |

`NSPersistentContainer` is the standard setup for new Core Data code:

```swift
final class PersistenceController {
    let container: NSPersistentContainer

    init(inMemory: Bool = false) {
        container = NSPersistentContainer(name: "TravelModel")

        if inMemory {
            container.persistentStoreDescriptions.first?.url =
                URL(fileURLWithPath: "/dev/null")
        }

        container.loadPersistentStores { _, error in
            if let error { fatalError("Failed to load store: \(error)") }
        }

        container.viewContext.automaticallyMergesChangesFromParent = true
        container.viewContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
    }
}
```

### 6.3 Managed Objects and the Model

An **entity** is the model-level definition; a **managed object** (`NSManagedObject`) is an instance of it living in a context.

- **Attributes** map to scalar properties.
- **Relationships** connect entities: **to-one** and **to-many**. Every relationship should have an **inverse**, which keeps the object graph consistent when one side changes.
- **Delete rules** (Nullify, Cascade, Deny) determine what happens to related objects when the owning object is deleted.

Subclasses use `@NSManaged` to bind properties to model attributes:

```swift
@objc(Trip)
public class Trip: NSManagedObject {
    @NSManaged public var name: String
    @NSManaged public var startDate: Date
    @NSManaged public var endDate: Date
    @NSManaged public var bucketList: Set<BucketListItem>
}
```

### 6.4 Faults (Lazy Loading)

A **fault** is a placeholder object whose property values have not been loaded yet. Core Data hands you a fault instead of the full row, and "fires" it (loads the data) the first time you access a property.

This is why fetching 10,000 objects is cheap *until* you start touching properties on all of them. It is also why `fetchBatchSize` matters: it controls how many rows are pulled from SQLite per batch as faults fire.

### 6.5 The Object Lifecycle

1. **Insert** — `context.insert(object)` (or `NSEntityDescription.insertNewObject`).
2. **Mutate** — set properties; the context records the change.
3. **Save** — `try context.save()` pushes pending changes to the store.
4. **Delete** — `context.delete(object)`, then save.

Until `save()` succeeds, changes exist only in the context. Discarding them is as simple as `context.rollback()` or throwing the context away.

### 6.6 Fetching and Filtering

```swift
let request = NSFetchRequest<Trip>(entityName: "Trip")
request.predicate = NSPredicate(format: "startDate > %@", Date.now as NSDate)
request.sortDescriptors = [NSSortDescriptor(keyPath: \Trip.startDate, ascending: true)]
request.fetchLimit = 50
request.fetchBatchSize = 20

let trips = try context.fetch(request)
```

- **`NSPredicate`** filters at the store level (not in memory).
- **`fetchLimit` / `fetchOffset`** implement pagination.
- **`fetchBatchSize`** controls fault-firing granularity.
- **`includesPropertyValues = false`** returns faults only—useful when you just need object IDs.

In SwiftUI, `@FetchRequest` mirrors SwiftData's `@Query`:

```swift
@FetchRequest(
    sortDescriptors: [NSSortDescriptor(keyPath: \Trip.startDate, ascending: true)],
    predicate: NSPredicate(format: "startDate > %@", Date.now as NSDate)
)
private var trips: FetchedResults<Trip>
```

In UIKit, `NSFetchedResultsController` performs the same role, driving table view updates via change notifications.

### 6.7 Concurrency: Contexts and Queues

Contexts are **not thread-safe**. Each context has a queue; you must use it on that queue.

- **`viewContext`** — main queue, bound to the UI.
- **`newBackgroundContext()`** — private queue, for import and sync work.

```swift
let backgroundContext = container.newBackgroundContext()
backgroundContext.perform {
    // Fetch, insert, mutate, save — all on the context's queue
    try? backgroundContext.save()
}
```

**Parent–child contexts** allow nested change sets: a child context holds unsaved work and only pushes to its parent on save. `automaticallyMergesChangesFromParent = true` on `viewContext` makes background saves appear in the UI without manual merging.

**Merge policies** resolve conflicts when the same object changes in two contexts: `NSMergeByPropertyObjectTrumpMergePolicy` (in-memory wins) and `NSMergeByPropertyStoreTrumpMergePolicy` (store wins) are the common choices.

### 6.8 Persistent History Tracking

For apps with multiple contexts, app extensions, or background sync, enable:

```swift
description.setOption(true as NSNumber, forKey: NSPersistentHistoryTrackingKey)
```

History tracking records every transaction so a context can replay changes made elsewhere (including in an extension). It is also the mechanism that makes Core Data ↔ SwiftData coexistence reliable.

### 6.9 Batch Operations

For bulk work, bypass the context—it is dramatically faster because it skips object materialization and validation.

```swift
let deleteRequest = NSBatchDeleteRequest(
    fetchRequest: NSFetchRequest<NSFetchRequestResult>(entityName: "Trip")
)
deleteRequest.resultType = .resultTypeObjectIDs

let result = try context.execute(deleteRequest) as? NSBatchDeleteResult
if let objectIDs = result?.result as? [NSManagedObjectID] {
    NSManagedObjectContext.mergeChanges(
        fromRemoteContextSave: [NSDeletedObjectsKey: objectIDs],
        into: [context]
    )
}
```

The same pattern applies to `NSBatchInsertRequest` and `NSBatchUpdateRequest`. **Critical:** batch operations execute directly against the store, so in-memory contexts do not see them until you merge the returned object IDs.

### 6.10 Migration

- **Lightweight migration** — inferred automatically when changes are simple (adding/removing entities, attributes, or optional relationships). `NSPersistentContainer` enables it by default via `shouldMigrateStoreAutomatically` and `shouldInferMappingModelAutomatically`.
- **Mapping models** — required when the change cannot be inferred (splitting an entity, transforming attribute types with custom logic).
- **Staged migrations** (`NSStagedMigrationManager`, iOS 17+) — describe the schema as an ordered sequence of versions, each either an `NSLightweightMigrationStage` or a `NSCustomMigrationStage`. This is the modern approach for multi-version evolution and replaces the older single-hop mapping model model.

### 6.11 Newer Model Features (iOS 17+)

- **Composite attributes** (`NSCompositeAttributeDescription`) — group several attributes into a single structured attribute, useful for value-type-like groupings.
- **Staged migrations** — described above.
- **Ordered relationships** — model ordering explicitly instead of relying on a sort attribute.

### 6.12 Core Data ↔ SwiftData Concept Map

| Core Data | SwiftData |
|---|---|
| `.xcdatamodeld` model | `@Model` classes |
| `NSManagedObject` + `@NSManaged` | `@Model` class properties |
| `NSPersistentContainer` | `ModelContainer` |
| `NSManagedObjectContext` | `ModelContext` |
| `NSFetchRequest` + `NSPredicate` | `FetchDescriptor` + `#Predicate` |
| `@FetchRequest` | `@Query` |
| `NSSortDescriptor` | `SortDescriptor` |
| Persistent history tracking | No direct equivalent |

---

## 7. Lower-Level SQLite Access: GRDB and SQLiteData

When you need direct SQL control, custom indexing, or performance tuning beyond what SwiftData exposes, Swift-friendly SQLite wrappers like **GRDB** and **SQLiteData** are legitimate choices. They sit below SwiftData/Core Data in abstraction level and suit apps with complex query requirements or existing SQLite schemas.

For most apps, SwiftData provides sufficient performance with dramatically better ergonomics; reach for GRDB/SQLiteData only when you have a concrete requirement that SwiftData cannot meet.

---

## 8. The Repository Pattern: Unifying the Data Layer

With multiple persistence tools in play, you need an architectural boundary so that views and business logic do not become coupled to a specific storage technology. The **Repository pattern** provides that boundary.

### The core rule

**Repository protocols live in the Domain layer**—they describe *what* data operations exist, without specifying *how* they are performed. **Concrete implementations live in the Data layer**—they own SwiftData, Core Data, `Codable` files, networking, and sync I/O.

```
┌─────────────────────────────────────────────┐
│  Feature (View + ViewModel)                 │
│  No SwiftData / Core Data / file I/O /      │
│  networking                                 │
├─────────────────────────────────────────────┤
│  Domain                                     │
│  Models + Repository protocols              │
├─────────────────────────────────────────────┤
│  Data                                       │
│  @Model / NSManagedObject entities,         │
│  Codable document stores, repository impls, │
│  network clients, sync                     │
└─────────────────────────────────────────────┘
```

This inversion means the Domain never depends on the Data layer, and you can swap SwiftData for Core Data, a JSON file store, a network API, or a mock without touching business logic.

### Repository protocol (Domain)

```swift
// Domain/Repositories/TripRepository.swift
// Pure Swift — no framework imports
protocol TripRepository: Sendable {
    func fetchAll() async throws -> [Trip]
    func fetch(id: UUID) async throws -> Trip?
    func save(_ trip: Trip) async throws
    func delete(id: UUID) async throws
}
```

### SwiftData implementation (Data)

```swift
// Data/Repositories/SwiftDataTripRepository.swift
import SwiftData

final class SwiftDataTripRepository: TripRepository {
    private let modelContext: ModelContext

    init(modelContext: ModelContext) {
        self.modelContext = modelContext
    }

    func fetchAll() async throws -> [Trip] {
        try modelContext.fetch(FetchDescriptor<TripEntity>())
            .map { $0.toDomain() }
    }

    func save(_ trip: Trip) async throws {
        let entity = TripEntity(from: trip)
        modelContext.insert(entity)
        try modelContext.save()
    }
}
```

### Codable implementation (Data)

The same protocol is satisfied by a file store—no database required:

```swift
// Data/Repositories/FileTripRepository.swift
actor FileTripRepository: TripRepository {
    private let store: TripStore  // from §4.5
    private var cache: [Trip]?

    func fetchAll() async throws -> [Trip] {
        if let cache { return cache }
        let trips = try store.load()
        cache = trips
        return trips
    }

    func save(_ trip: Trip) async throws {
        var trips = try await fetchAll()
        if let index = trips.firstIndex(where: { $0.id == trip.id }) {
            trips[index] = trip
        } else {
            trips.append(trip)
        }
        try store.save(trips)
        cache = trips
    }
}
```

Switching from a JSON file to SwiftData means changing **one initializer call**, not every view. That is the payoff.

The Core Data equivalent follows the same shape, substituting `NSManagedObjectContext` and `NSFetchRequest`, and wrapping access in `context.perform { }` when using a background context.

### Dependency injection

Repositories are injected as protocol dependencies—never instantiated inside views or view models:

```swift
@Observable
final class TripListViewModel {
    private let repository: TripRepository
    var trips: [Trip] = []

    init(repository: TripRepository) {
        self.repository = repository
    }

    func load() async {
        trips = (try? await repository.fetchAll()) ?? []
    }
}
```

---

## 9. Networking Integration: URLSession, async/await, and Offline-First Sync

Networking and persistence are not separate concerns—they are two sides of the same data layer. The modern pattern is **offline-first**: the repository reads from the local store, and a background sync service merges remote data into that store.

### A reusable API client with async/await

`URLSession` combined with Swift's `async/await` provides a clean, readable networking API. Completion-handler variants should not be used in new code.

```swift
enum HTTPMethod: String {
    case get = "GET"
    case post = "POST"
    case put = "PUT"
    case delete = "DELETE"
}

actor APIClient {
    private let baseURL: URL
    private let session: URLSession

    init(baseURL: URL, session: URLSession = .shared) {
        self.baseURL = baseURL
        self.session = session
    }

    func request<T: Decodable>(
        path: String,
        method: HTTPMethod = .get,
        body: (any Encodable)? = nil
    ) async throws -> T {
        guard let url = URL(string: path, relativeTo: baseURL) else {
            throw NetworkError.invalidURL
        }

        var request = URLRequest(url: url)
        request.httpMethod = method.rawValue
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")

        if let body {
            request.httpBody = try JSONEncoder().encode(body)
        }

        let (data, response) = try await session.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse else {
            throw NetworkError.invalidResponse
        }
        guard (200...299).contains(httpResponse.statusCode) else {
            throw NetworkError.httpError(statusCode: httpResponse.statusCode, data: data)
        }

        return try JSONDecoder().decode(T.self, from: data)
    }
}
```

**Critical detail:** `URLSession` does **not** throw for 4xx/5xx responses. You must validate `HTTPURLResponse.statusCode` explicitly before decoding.

### DTOs are not domain models

The `Decodable` types crossing the network boundary are **DTOs**: they mirror the wire format, including optionality and naming quirks you do not want leaking into the domain. Mapping DTO → domain model happens in the Data layer, in exactly the same place the SwiftData entity → domain mapping happens. This keeps `Codable` conformance (a serialization concern) out of the Domain layer.

### Offline-first architecture

The repository is the **source of truth for all reads**. View models load from the repository (which reads the local store) and **never wait for a network response** before showing data. A background sync service fetches remote data and merges it into the local store; the view model re-fetches afterward.

```swift
@Observable
final class TripListViewModel {
    private let repository: TripRepository
    private let syncService: TripSyncServiceProtocol
    var trips: [Trip] = []
    var syncError: String?

    func loadTrips() async {
        trips = (try? await repository.fetchAll()) ?? []
    }

    func syncInBackground() async {
        do {
            try await syncService.syncTrips()
            await loadTrips() // Refresh from local store
        } catch {
            syncError = error.localizedDescription
            // Non-fatal — cached data remains visible
        }
    }
}
```

This eliminates blank-screen loading states: the app launches instantly with cached data, works offline, and shows fresh data as soon as sync completes.

### The sync service

Sync services should be isolated actors that fetch from the API and upsert into the local store. The network layer itself should be a **pure HTTP client returning `Sendable` DTOs**, with no persistence awareness.

```swift
@ModelActor
actor TripSyncService: TripSyncServiceProtocol {
    func syncTrips() async throws {
        let remoteTrips: [TripDTO] = try await apiClient.request(path: "/trips")
        for dto in remoteTrips {
            let entity = TripEntity(from: dto)
            modelContext.insert(entity)
        }
        try modelContext.save()
    }
}
```

With Core Data, the same service uses `container.newBackgroundContext()` and `context.perform { }` instead of `@ModelActor`.

### Upsert strategy

Naive `insert` on every sync creates duplicates. Choose one:

- **Stable remote identifiers** — map the server's `id` to a unique constraint (`#Unique` in SwiftData, a unique constraint on the entity in Core Data) and use an upsert merge policy.
- **Fetch-then-merge** — fetch existing entities by remote ID, update them, insert only the genuinely new ones.
- **Full replace** — delete all local rows and re-insert. Acceptable only for small datasets where offline edits are not preserved.

### Stale-while-revalidate reads

For lists and feeds, the repository can expose an `AsyncStream` that first yields cached data immediately, then yields fresh data after a background refresh. This gives users instant content while keeping data current.

### Handling background execution

Sync triggered while the app is backgrounded needs `BGTaskScheduler` (or `URLSession` background transfers for large downloads). The persistence layer is unchanged—only the trigger differs. Keep sync idempotent so a re-run after interruption is harmless.

---

## 10. Final Cheat Sheet

| Scenario | Recommended Tool |
|---|---|
| **New app, iOS 17+** | **SwiftData** (default for structured local data) |
| **Existing Core Data app** | Keep Core Data; migrate incrementally or coexist via shared store URL |
| **Public CloudKit database** | **Core Data** (SwiftData supports private databases only) |
| **Complex multi-version schema evolution** | **Core Data** staged migrations |
| **Small self-contained document, loaded whole** | **`Codable`** + `FileManager` (JSON or binary plist) |
| **Object graph with shared references/cycles to persist** | **`NSCoding` / `NSKeyedArchiver`** (legacy; prefer a database) |
| **Small nested value type inside a record** | `Codable` struct stored as a SwiftData/Core Data attribute |
| **Simple preferences / flags** | **UserDefaults** |
| **User-created files** | **FileManager** → Documents (backed up) |
| **Re-downloadable content** | **FileManager** → Caches (purged, excluded from backup) |
| **Scratch space** | **FileManager** → tmp (purged between launches) |
| **Direct SQL control / complex indexing** | **GRDB** or **SQLiteData** |
| **Architecture boundary** | **Repository pattern**: protocol in Domain, implementation in Data |
| **Networking** | **URLSession + async/await**, with explicit status-code validation |
| **Freshness strategy** | **Offline-first**: repository reads locally; background sync merges remote data |

---

The 2026 persistence story is not "pick one tool and commit forever." It is **layer by lifecycle, abstract by protocol, and sync in the background**.

`Codable` + `FileManager` is the right answer for small whole-document state—and the right *first* answer, because it is trivial to implement and trivial to swap out. `NSCoding`/`NSKeyedArchiver` remains only for object-identity graphs and legacy interop. SwiftData handles the structured local store for new apps. Core Data remains the right answer for legacy code, public CloudKit, and complex migrations—and its concepts (stack, contexts, faults, fetch requests, merge policies, batch operations, staged migration) still explain what SwiftData does underneath. `UserDefaults` handles lightweight preferences. The Repository pattern—protocol in Domain, implementation in Data—keeps all of them swappable, testable, and hidden from your views. Networking is not a separate track but the **write path into the local store**, with offline-first reads ensuring the app is useful before the network ever responds.
