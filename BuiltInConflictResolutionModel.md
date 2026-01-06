# A Conceptual Model for Built-in Conflict Resolution in SQLiteData

This document proposes a revised conceptual model for [SQLiteData’s built-in conflict resolution mechanism][1], which follows a “field-wise last edit wins” strategy. It builds on the observations and shortcomings of the current behavior shared in [this document][2] and restates that behavior in a clearer and more precise form, addressing the identified flaws, with the goal of improving the existing implementation while laying the groundwork for future support of customizable conflict resolution.

## 1. Foundations

The model described below is based on a set of premises that may differ from the current implementation:

- The _last-known server record_ truly reflects state confirmed by or fetched from the server, making it suitable to be used as the _ancestor_ in three-way merge logic.
- All client-side changes to a row share a single modification timestamp in `SyncMetadata.userModificationTime`.
- Server-side state, represented by `CKRecord` instances and used for _last-known and incoming server records_, maintains a modification timestamp for each field.
- Merge logic is executed only when an actual conflict is detected. If there are no client changes, there might be a faster path for upserting from a server version that directly propagates its values. See more on conflict detection in [\#272][3].
- Modification timestamps are used solely for conflict resolution. They are not consulted during regular upserts.

Based on these premises, the conflict resolution model operates as follows:

- The model is based on a three-way merge between the _ancestor_ (last-known server record), _client_ (database row), and _server_ (incoming server record) versions.
- Rather than directly working with `CKRecord` instances, the model uses a `RowVersion` abstraction, a key–value data structure with access to per-field modification timestamps.
- Instead of producing a full merged row representation, the merge logic produces a set of field updates represented as key–value pairs to be applied to the local row, rather than a `CKRecord` plus a set of column names.

[1]:	https://swiftpackageindex.com/pointfreeco/sqlite-data/main/documentation/sqlitedata/cloudkit#Record-conflicts
[2]:	CurrentBehavior.md
[3]:	https://github.com/pointfreeco/sqlite-data/discussions/272#discussioncomment-14975219