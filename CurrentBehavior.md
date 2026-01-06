# Current CloudKit Sync Behavior in SQLiteData

This document captures how CloudKit sync currently behaves in [SQLiteData][1] as of version 1.4, focusing on the implementation of the [built-in “field-wise last edit wins” conflict resolution strategy][2]. In particular, it documents how *last-known server records* and *timestamps* are handled today. The goal is to provide a clear reference for the current semantics as a first step towards making the conflict resolution customizable per the discussion in [\#272][3] and to highlight areas that will need to change in order to support a proper three-way merge and custom merge strategies.

## 1. Core Concepts

### 1.1. General Sync Flow

At a high level, SQLiteData’s CloudKit sync operates in two directions, both mediated by an underlying `CKSyncEngine`:

- **Client → Server**: Changes are first written to the local SQLite database and later picked up by the sync engine, translated into `CKRecord`s, and sent to the server. When an upload succeeds, the local database already reflects the desired state, so sync primarily updates metadata.
- **Server → Client**: Changes arriving from the server are applied to the local database using upsert logic, which inserts new rows or updates existing ones. In this direction, incoming server state is reconciled with both the local database state and any pending client changes using the built-in “field-wise last edit wins” strategy. Sync metadata is updated accordingly.

Conflicts arise when client-side and server-side changes affect the same record concurrently. A conflict-on-send scenario is handled explicitly by reacting to a `.serverRecordChanged` error and performing an upsert. A conflict-on-fetch scenario, on the other hand, is handled implicitly by applying the same upsert logic without an explicit record-level conflict detection.[^1]

The following sections describe the data structures and timestamps that underpin this flow, before diving into concrete send, fetch, and conflict scenarios.

### 1.2. SyncMetadata Fields

SQLiteData with syncing enabled [manages a `SyncMetadata` table in the metadatabase][4] with a row for each row in the user tables. Data stored in that table is used when interfacing with CloudKit. Among others, the table contains the following columns:

- [`lastKnownServerRecord`][5]: The last known `CKRecord` received from the server serialized via `CKRecord.encodeSystemFields(with:)` and thus containing only the system fields.
- `_lastKnownServerRecordAllFields`: The last known `CKRecord` received from the server serialized via `CKRecord.encode(with:)` and thus containing all fields, including per-field modification timestamps (explained in the next section). As it stands, the last-known server record doesn’t necessarily reflect the server state all the time. There are some nuances that are explained later in the document.
- [`userModificationTime`][6]: A timestamp indicating when the user last modified the record. This value gets updated whenever the row in the user table is updated, be it by a manual edit or due to a change coming from the sync engine.

### 1.3. Per-Field Timestamps

SQLiteData’s built-in “field-wise last edit wins” strategy employed during conflict resolution relies on keeping track of modification timestamps for individual fields. This allows SQLiteData to reason about which values should win when the same record has been edited concurrently.

These timestamps are stored directly on the `CKRecord`. Each field is accompanied by a corresponding modification timestamp, which SQLiteData reads from and writes to using the `encryptedValues[at: key]` API. In addition, the record carries an overall `userModificationTime` that reflects the maximum of all per-field timestamps. This should not be confused with `CKRecord.modificationDate`, which contains the time that CloudKit persisted the record to the server and has no equivalent on the client.

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

## 2. Sync Scenarios

This section walks through concrete sync scenarios to illustrate how SQLiteData behaves in practice and where assumptions start to break down.

### 2.1. Sending Client Change

This scenario describes a local change to a row that is sent to the server without any concurrent server-side edits.

1. The record is fully in sync. The local database row and the last-known server record reflect the same server state.
2. The user modifies the record locally.
	- The change is written to the local database.
	- `SyncMetadata.userModificationTime` is updated via a trigger.
	- In a trigger, the modification is recorded with the sync engine as pending for upload.
3. The sync engine picks up the pending change in `SyncEngine.nextRecordZoneChangeBatch(…)` and prepares the record to send.
	- The `CKRecord` is created from `SyncMetadata._lastKnownServerRecordAllFields`. If no last-known server record exists, a fresh `CKRecord` is created instead.
	- The local row is then applied to this record using `CKRecord.update(with:userModificationTime:)`, stamping each modified field with the current `userModificationTime` while leaving unchanged fields with their existing timestamps. Even if individual fields were changed at different times, only the latest timestamp is used, so all modified fields end up sharing a single modification time despite timestamps being tracked per field in records.
4. Even before the upload is confirmed, SQLiteData attempts to persist the constructed `CKRecord` instance as the last-known server record by calling `refreshLastKnownServerRecord(…)`. This update only succeeds if no previous last-known server record exists, i.e. when uploading a record for the first time. For previously synced records, the stored last-known server record and the constructed upload record share the same `modificationDate`, causing the refresh to be skipped.

> [!WARNING]
> While this behavior does not cause immediate issues in this scenario, it is conceptually incorrect to treat a pending upload record as a record known to the server. To support conflict resolution via a proper three-way merge, the last-known server record must reflect server-acknowledged state, not client intent.
> 
> This issue becomes visible in tests, where the mock server does not populate `modificationDate` when confirming a sent record. As a result, the last-known server record gets updated with an in-flight record, which then breaks when a conflict is encountered. But really, a much bigger problem is that the test setup diverges from production behavior.

5. The record is sent to the server and the result is reported in `SyncEngine.handleSentRecordZoneChanges(…)`.
	 - Since there are no concurrent server-side changes in this scenario, the upload succeeds without conflict.
	- The confirmed server record is stored as the new last-known server record, now reflecting state that has been acknowledged by the server.
	- `SyncMetadata.userModificationTime` is updated from the server-confirmed record, but effectively retains the same value that was used when constructing the upload.
	- No changes are applied to the local database row, as it already reflects the desired state.

### 2.2. Fetching Server Change

This scenario describes a server-side change that is received and applied locally when there are no pending client-side modifications for the record.

1. The record is fully in sync. The local database row and the last-known server record reflect the same server state.
2. The record is modified on the server from another device and now contains updated field values, updated per-field modification timestamps, and an updated overall user modification timestamp.
3. The updated record is delivered to the client and processed in `SyncEngine.handleFetchedRecordZoneChanges(…)` as a modification, which is routed to `SyncEngine.upsertFromServerRecord(…)` to apply the upsert logic.
	- A corresponding `SyncMetadata` row is ensured to exist, with a populated last-known server record.
	- The server record’s overall `userModificationTime` is overwritten with the locally stored value from `SyncMetadata`, effectively discarding the incoming server-provided timestamp.
	- The server record is reconciled with both the last-known server record and the current row fetched from the database in `CKRecord.update(with:row:columnNames:parentForeignKey:)`. This method restores field values from the last-known server record when they (a) differ and (b) their timestamp is newer or equal to those in the incoming server record. It also narrows down the set of columns to be written by excluding columns whose values were restored (`didSet` flag) as well as columns with pending local edits (`isRowValueModified` flag). In this scenario, however, there are no local edits and the last-known server record does not contain any newer values, so no columns are excluded.
	- Next, the selected columns are updated on the local database row using values from the server record.
	- After this update, a trigger fires and sets `SyncMetadata.userModificationTime` to the current time.
	- The (potentially mutated) server record is then persisted as the new last-known server record.
	- Finally, `SyncMetadata.userModificationTime` is updated again, this time by copying the `userModificationTime` from the server record.

### 2.3. Conflict on Send

This scenario describes a local change to a row that is sent to the server while a concurrent server-side edit exists, resulting in a `.serverRecordChanged` error.

1. The record is fully in sync. The local database row and the last-known server record reflect the same server state.
2. The record is modified on the server from another device and now contains updated field values, updated per-field modification timestamps, and an updated overall user modification timestamp.
3. The user modifies the record locally, potentially editing different fields than the server change.
	- The change is written to the local database.
	- `SyncMetadata.userModificationTime` is updated via a trigger.
	- In a trigger, the modification is recorded with the sync engine as pending for upload.
4. The sync engine picks up the pending change in `SyncEngine.nextRecordZoneChangeBatch(…)` and prepares the record to send.
	- The `CKRecord` is created from `SyncMetadata._lastKnownServerRecordAllFields`.
	- The local row is then applied to this record using `CKRecord.update(with:userModificationTime:)`, stamping each modified field with the current `userModificationTime` while leaving unchanged fields with their existing timestamps.
	- `refreshLastKnownServerRecord(…)` is then invoked with the constructed record. In production, this call is skipped because the existing last-known server record has the same `modificationDate`. In tests, however, the check succeeds, causing the client-constructed upload record to be incorrectly persisted as the last-known server record (see 2.1).
5. The result is reported in `SyncEngine.handleSentRecordZoneChanges(…)` where the send fails with a `.serverRecordChanged` error that includes the latest server record. This error is handled by routing the server record into `SyncEngine.upsertFromServerRecord(…)`, which applies the same upsert logic as in section 2.2, but this time, it resolves the conflict between the server state and the pending client change.
	- The server record’s overall `userModificationTime` is overwritten with the client’s value.
	- The server record is reconciled with both the last-known server record and the current row fetched from the database in `CKRecord.update(with:row:columnNames:parentForeignKey:)`.
	- Inside this method, for each field, two flags are computed:
		- `didSet`: Attempts to restore the last-known server record’s value and timestamp to the incoming server record. Returns `true` only if (a) the values differ and (b) the last-known server record’s timestamp for this field is great than or equal to the server’s timestamp. In production with a clean last-known server record, this typically returns `false` because the last-known server record has older timestamps, so the incoming server’s value is kept.
		- `isRowValueModified`: Compares the current database row value to the last-known server record’s value, ignoring timestamps. Returns `true` if they differ, indicating pending client edit exists for this field.
	- If either `didSet` or `isRowValueModified` is `true`, the field is excluded from the set of columns to be written into the database.
	- The marked columns are updated for the database row with values from the (potentially mutated) server record.
	- After this update, a trigger fires and sets `SyncMetadata.userModificationTime` to the current time.
	- The (potentially mutated) server record is saved as the new last-known server record and the `SyncMetadata.userModificationTime` is overwritten with value from the server record.

> [!WARNING]
> In tests, the last-known server record can be polluted with client-constructed state before uploads are confirmed (see section 2.1), causing `didSet` equal to `true` to occur more frequently during conflict resolution.
> 
> It was observed that `didSet` being equal to `true` can occur in production when concurrent edits happen at the exact same timestamp. Because the comparison uses `<=` (less than or equal), equal timestamps cause the last-known server record’s value to win. This leads to non-deterministic outcomes depending on which device’s change was processed first. 

> [!WARNING]
> It’s conceptually inappropriate to save a mutated server record as the last-known one acknowledged by the server. It’s optimistically counting on the server to accept on the next send, but another conflict might be encountered with no access to a clean last-known server record.
> 
> Unlike the noop last-known server record refresh on send described in section 2.1, this premature refresh happens in both production and test environments.

6. After the conflict is resolved, the failed record is re-queued for upload.
7. The sync engine picks up the re-queued record for upload in `SyncEngine.nextRecordZoneChangeBatch(…)` using the same logic as step 4. It constructs a `CKRecord` from the updated last-known server record and applies values for fields with pending edits to it.
8. The sync engine reports the result in `SyncEngine.handleSentRecordZoneChanges(…)`. If the save succeeds, the sync engine updates the metadata by storing the reported record as the last-known server record.

### 2.4. Conflict on Fetch

TODO

[^1]:	In contrast to conflict-on-send, conflict-on-fetch scenarios are not explicitly signaled by CloudKit. Detecting such conflicts would require SQLiteData to infer them based on the presence of pending local changes for the record.

[1]:	https://github.com/pointfreeco/sqlite-data
[2]:	https://swiftpackageindex.com/pointfreeco/sqlite-data/main/documentation/sqlitedata/cloudkit#Record-conflicts
[3]:	https://github.com/pointfreeco/sqlite-data/discussions/272
[4]:	https://swiftpackageindex.com/pointfreeco/sqlite-data/main/documentation/sqlitedata/cloudkit#Accessing-CloudKit-metadata
[5]:	https://swiftpackageindex.com/pointfreeco/sqlite-data/main/documentation/sqlitedata/syncmetadata/lastknownserverrecord
[6]:	https://swiftpackageindex.com/pointfreeco/sqlite-data/main/documentation/sqlitedata/syncmetadata/usermodificationtime