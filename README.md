# CloudKit Sync Semantics in SQLiteData

This document captures how CloudKit sync currently behaves in [SQLiteData][1] as of version 1.4, focusing on the built-in “field-wise last edit wins” conflict resolution strategy, employed in both conflict and non-conflict sync scenarios. In particular, it documents how *last-known server records* and *timestamps* are handled today, and where the observed behavior is incorrect and in need of improvement. The goal is to provide a clear reference for the current semantics as a first step towards making the conflict resolution configurable (see [\#272][2]) and to highlight areas that will need to change in order to support a proper three-way merge and custom merge strategies.

## 1 Core Concepts

### 1.1 General Sync Flow

At a high level, SQLiteData’s CloudKit sync operates in two directions, both mediated by an underlying `CKSyncEngine`:

- **Client → Server**: Changes are first written to the local SQLite database and later picked up by the sync engine, translated into `CKRecord`s, and sent to the server. When an upload succeeds, the local database already reflects the desired state, so sync primarily updates metadata.
- **Server → Client**: Changes arriving from the server are applied to the local database using upsert logic, which inserts new rows or updates existing ones. In this direction, incoming server state is reconciled with both the local database state and any pending client changes using the built-in “field-wise last edit wins” strategy. Sync metadata is updated accordingly.

Conflicts arise when client-side and server-side changes affect the same record concurrently. A conflict-on-send scenario is handled explicitly by reacting to a `.serverRecordChanged` error and performing an upsert. A conflict-on-fetch scenario, on the other hand, is handled implicitly by applying the same upsert logic without an explicit record-level conflict detection[^1].

The following sections describe the data structures and timestamps that underpin this flow, before diving into concrete send, fetch, and conflict scenarios.

### 1.2 SyncMetadata Fields

SQLiteData with syncing enabled [manages a `SyncMetadata` table in the metadatabase][3] with a row for each row in the user tables. Data stored in that table is used when interfacing with CloudKit. Among others, the table contains the following columns:

- [`lastKnownServerRecord`][4]: The last known `CKRecord` received from the server serialized via `CKRecord.encodeSystemFields(with:)` and thus containing only the system fields.
- `_lastKnownServerRecordAllFields`: The last known `CKRecord` received from the server serialized via `CKRecord.encode(with:)` and thus containing all fields, including per-field modification timestamps (explained in the next section). As it stands, the last-known server record doesn’t necessarily reflect the server state all the time. There are some nuances that are explained later in the document.
- [`userModificationTime`][5]: A timestamp indicating when the user last modified the record. This value gets updated whenever the row in the user table is updated, be it by a manual edit or due to a change coming from the sync engine.

### 1.3 Per-Field Timestamps

SQLiteData’s [built-in “field-wise last edit wins” strategy][6] employed during conflict resolution relies on keeping track of modification timestamps for individual fields. This allows SQLiteData to reason about which values should win when the same record has been edited concurrently.

These timestamps are stored directly on the `CKRecord`. Each field is accompanied by a corresponding modification timestamp, which SQLiteData reads from and writes to using the `encryptedValues[at: key]` API. In addition, the record carries an overall `userModificationTime` that reflects the maximum of all per-field timestamps. This should not be confused with `CKRecord.modificationDate`, which contains the time that CloudKit persisted the record to the server.[^2]

A simplified snapshot of a record with per-field timestamps looks like this:

```swift
CKRecord(
  recordType: "reminders",
  title: "Buy milk",
  isCompleted: 1,
  sqlitedata_icloud_userModificationTime_title: 60,
  sqlitedata_icloud_userModificationTime_isCompleted: 30,
  sqlitedata_icloud_userModificationTime: 60
)
```

[^1]:	In contrast to conflict-on-send, conflict-on-fetch scenarios are not explicitly signaled by CloudKit. Detecting such conflicts would require SQLiteData to infer them based on the presence of pending local changes for the record.

[^2]:	This value has no equivalent on the client and is therefore unsuitable for conflict resolution. Additionally, the use of `Date` makes it unreliable for precise comparisons.

[1]:	https://github.com/pointfreeco/sqlite-data
[2]:	https://github.com/pointfreeco/sqlite-data/discussions/272
[3]:	https://swiftpackageindex.com/pointfreeco/sqlite-data/main/documentation/sqlitedata/cloudkit#Accessing-CloudKit-metadata
[4]:	https://swiftpackageindex.com/pointfreeco/sqlite-data/main/documentation/sqlitedata/syncmetadata/lastknownserverrecord
[5]:	https://swiftpackageindex.com/pointfreeco/sqlite-data/main/documentation/sqlitedata/syncmetadata/usermodificationtime
[6]:	https://swiftpackageindex.com/pointfreeco/sqlite-data/main/documentation/sqlitedata/cloudkit#Record-conflicts