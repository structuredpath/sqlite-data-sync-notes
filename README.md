# CloudKit Sync Semantics in SQLiteData 1.4

This document captures how CloudKit sync currently behaves in SQLiteData, focusing on the built-in “last edit wins” conflict resolution strategy, employed in both conflict and non-conflict sync scenarios. In particular, it documents how *last-known server records* and *timestamps* are handled today, and where the observed behavior can be surprising or incorrect. The goal is to provide a clear reference for the current semantics as a first step towards making the conflict resolution configurable (see [\#272][1]) and to highlight areas that will need to change in order to support a proper three-way merge and custom merge strategies.

## 1 Core Concepts

### 1.1 SyncMetadata Fields

SQLiteData with syncing enabled [manages a `SyncMetadata` table in the metadatabase][2] with a row for each row in the user tables. Data stored in that table is used when interfacing with CloudKit. Among others, the table contains the following columns:
- [`lastKnownServerRecord`][3]: The last known `CKRecord` received from the server serialized via `CKRecord.encodeSystemFields(with:)` and thus containing only the system fields.
- `_lastKnownServerRecordAllFields`: The last known `CKRecord` received from the server serialized via `CKRecord.encode(with:)` and thus containing all fields, including per-field modification timestamps (explained in the next section). As it stands, the last-known server record doesn’t necessarily reflect the server state all the time. There are some nuances that are explained later in the document.
- [`userModificationTime`][4]: A timestamp indicating when the user last modified the record. This value gets updated whenever the row in the user table is updated, regardless of the source of the change.

[1]:	https://github.com/pointfreeco/sqlite-data/discussions/272
[2]:	https://swiftpackageindex.com/pointfreeco/sqlite-data/main/documentation/sqlitedata/cloudkit#Accessing-CloudKit-metadata
[3]:	https://swiftpackageindex.com/pointfreeco/sqlite-data/main/documentation/sqlitedata/syncmetadata/lastknownserverrecord
[4]:	https://swiftpackageindex.com/pointfreeco/sqlite-data/main/documentation/sqlitedata/syncmetadata/usermodificationtime