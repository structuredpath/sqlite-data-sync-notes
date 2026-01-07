# A Conceptual Model for Built-in Conflict Resolution in SQLiteData

This document proposes a revised conceptual model for [SQLiteData’s built-in conflict resolution mechanism][1], which follows a “field-wise last edit wins” strategy. It builds on the observations and shortcomings of the current behavior shared in [this document][2] and restates that behavior in a clearer and more precise form, addressing the identified flaws, with the goal of improving the existing implementation while laying the groundwork for future support of customizable conflict resolution ([\#272][3]).

## 1. Premises

The model described below is based on a set of premises that may differ from the current implementation:

- The _last-known server record_ truly reflects state confirmed by or fetched from the server, making it suitable to be used as the _ancestor_ in three-way merge logic.
- All client-side changes to a row share a single modification timestamp in `SyncMetadata.userModificationTime`.
- Server-side state, represented by `CKRecord` instances and used for _last-known and incoming server records_, maintains a modification timestamp for each field.
- Merge logic is executed only when an actual conflict is detected. If there are no client changes, there might be a faster path for upserting from a server version that directly propagates its values. See more on conflict detection in [\#272][4].
- Modification timestamps are used solely for conflict resolution. They are not consulted during regular upserts.

## 2. Core Mechanics

The conflict resolution model is based on a three-way merge between the _ancestor_ (last-known server record), _client_ (database row), and _server_ (incoming server record) versions. However, rather than operating directly on `CKRecord` instances, the model introduces a `RowVersion` abstraction: a key–value representation of a row with access to per-field modification timestamps.

In its minimal form, a `RowVersion` can be expressed as follows:

```swift
struct RowVersion<T: PrimaryKeyedTable> {
  let row: T
  func modificationDate(for column: PartialKeyPath<T.TableColumns>) -> Int64 { /* … */ }
}
```

A `RowVersion` may be constructed from different sources depending on its role in the merge. The _client_ version is derived from the local database row together with its associated timestamp metadata. The _ancestor_ and _server_ versions are derived from `CKRecord` instances, decoded into a row representation while preserving the per-field modification timestamps.

```swift
extension RowVersion {

  /// Creates a client-side row version, assigning the given user modification timestamp to fields 
  /// changed relative to the ancestor.
  init(
    clientRow row: T,
    userModificationTime: Int64,
    ancestorVersion: RowVersion<T>
  ) { /* … */ }

  /// Creates a row version from a server record, preserving per-field modification timestamps.
  init(from record: CKRecord) throws { /* … */ }

}
```

The following example shows a concrete conflict scenario and how different combinations of client and server changes are resolved. It is structured to cover a wide range of field-level conflict cases.

![][image-1]

[1]:	https://swiftpackageindex.com/pointfreeco/sqlite-data/main/documentation/sqlitedata/cloudkit#Record-conflicts
[2]:	CurrentBehavior.md
[3]:	https://github.com/pointfreeco/sqlite-data/discussions/272
[4]:	https://github.com/pointfreeco/sqlite-data/discussions/272#discussioncomment-14975219

[image-1]:	images/ConflictInputs.png