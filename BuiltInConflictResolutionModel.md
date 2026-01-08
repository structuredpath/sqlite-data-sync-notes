# A Conceptual Model for Built-in Conflict Resolution in SQLiteData

This document proposes a revised conceptual model for [SQLiteData’s built-in conflict resolution mechanism][1], which follows a “field-wise last edit wins” strategy. It builds on the observations and shortcomings of the current behavior shared in [this document][2] and restates that behavior in a clearer and more precise form, addressing the identified flaws, with the goal of improving the existing implementation while laying the groundwork for future support of customizable conflict resolution ([\#272][3]).

## 1. Premises

The model described below is based on a set of premises that may differ from the current implementation:

- The _last-known server record_ truly reflects state confirmed by or fetched from the server, making it suitable to be used as the _ancestor_ in three-way merge logic.
- All client-side changes to a row share a single modification timestamp in `SyncMetadata.userModificationTime`.
- Server-side state, represented by `CKRecord` instances and used for _last-known and incoming server records_, maintains a modification timestamp for each field.
- Merge logic is executed only when an actual conflict is detected. If there are no client changes, there would be a faster path for upserting from a server version that directly propagates its values. See more on conflict detection in [\#272][4].
- Modification timestamps are used solely for conflict resolution. They are not consulted during regular upserts.

## 2. Merge Logic

The conflict resolution model is based on a three-way merge logic between the _ancestor_ (last-known server record), _client_ (database row), and _server_ (incoming server record) versions. However, rather than operating directly on `CKRecord` instances, the model introduces a `RowVersion` abstraction: a key–value representation of a row with access to per-field modification timestamps. While not strictly required, modeling this as a value type makes merge behavior easier to reason about, improves testability, and is a natural step toward exposing this abstraction to users for customizable conflict resolution in the future.

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

The following example illustrates a concrete conflict scenario in which the same row is modified independently on the client and the server. It is structured to cover a wide range of field-level conflict cases.

![][image-1]

Conflict resolution is performed on a per-field basis by comparing the ancestor, client, and server versions of a row. It follows the “field-wise last edit wins” strategy. When a field is changed on only one side, that change is taken. When a field is changed on both sides, the modification timestamps determine which value wins. The table below summarizes the outcome for each field in this scenario.

| Field  | Ancestor    | Client            | Server            | Merge Logic & Outcome                                           |
| ------ | ----------- | ----------------- | ----------------- | --------------------------------------------------------------- |
| field1 | “foo” @ t=0 | “foo” @ t=0       | “foo” @ t=0       | No changes → no updates                                         |
| field2 | “foo” @ t=0 | **“bar” @ t=100** | “foo” @ t=0       | Client-only change → keep client (“bar”)                        |
| field3 | “foo” @ t=0 | “foo” @ t=0       | **“baz“ @ t=200** | **Server-only change → update to server (“baz”)**               |
| field4 | “foo” @ t=0 | **“bar” @ t=100** | **“baz“ @ t=200** | **Both changed, server newer → update to server (“baz”)**       |
| field5 | “foo” @ t=0 | **“bar” @ t=100** | **“baz“ @ t=50**  | Both changed, client newer → keep client (“bar”)                |
| field6 | “foo” @ t=0 | **“bar” @ t=100** | **“baz“ @ t=100** | Both changed, equal timestamps → tie-break, keep client (“bar”) |
| field7 | “foo” @ t=0 | **“bar“ @ t=100** | **“bar“ @ t=200** | Both changed, same value → keep client                          |

In the case of equal timestamps (field 6), the outcome depends on a tie-breaking rule. This example assumes the client value is kept. The specific tie-breaker is an implementation choice and may equally favor the server.

To illustrate this in code, the merge can be expressed in terms of a `MergeConflict` value that provides access to all three row versions:

```swift
struct MergeConflict<T: PrimaryKeyedTable> {
  let ancestor: RowVersion<T>
  let client: RowVersion<T>
  let server: RowVersion<T>
}
```

A minimal per-field merge function can then implement the behavior encoded in the table:

```swift
extension MergeConflict {
  func mergedValue<V: Equatable>(
    for keyPath: some KeyPath<T.TableColumns, V>
  ) -> V {
    let ancestorValue = ancestor.row[keyPath: keyPath]
    let clientValue = client.row[keyPath: keyPath]
    let serverValue = server.row[keyPath: keyPath]
    
    // field1 & field7
    if clientValue == serverValue {
      return clientValue
    }

    // field2
    if ancestorValue != clientValue && ancestorValue == serverValue {
      return clientValue
    }
    
    // field3
    if ancestorValue == clientValue && ancestorValue != serverValue {
      return serverValue
    }

    if server.server.modificationTime(for: keyPath) > client.modificationTime(for: keyPath) {
      // field4
      return server.row[keyPath: keyPath]
    } else {
      // field5 & field6
      return client.row[keyPath: keyPath]
    }
  }
}
```

This implementation is self-contained and designed to make the merge behavior predictable. It also provides a clear extension point for introducing a custom per-field merge policy in the future, as sketched towards the end of [this comment￼][5].

The conceptual outcome of the example above is that fields 3 and 4 should adopt the server values and be written into the local database. The current implementation achieves this by upserting directly from the `CKRecord`. Since the model described here no longer operates on raw records, the merge result must be communicated in a different form, and the remaining question is how this outcome should be represented and applied.

One option for the merge logic is to signal only the set of changed fields and extract the corresponding values from the server version when constructing the update statement. While this aligns well with the current implementation, it would fall short when considering future support for customizable conflict resolution. In that context, an API that merges into a full row representation may be preferable. Since the primary key must not be modified, the inner draft type could be used.

Eventually, conflict resolution could be hosted on the table types themselves and generated via macros, allowing individual fields to be annotated with different merge policies (as originally proposed in [\#272][6] under Approach D):

```swift
extension MyTable {
  static func resolve(_ conflict: MergeConflict<Self>) -> Self.Draft { /* … */ }
}
```

Until such customization is introduced, the closest shape to this model would be an API that performs a built-in, hardcoded merge while already reflecting the intended direction:

```swift
extension MergeConflict {
  func resolved() -> T.Draft {
    /* Start with the client version */
    /* Iterate over all writable columns */
      /* Compute merged value for the column */
      /* Set value on draft */
    /* Return final draft */
  }
}
```

Since the goal of this document is to describe the conceptual model, no further implementation details are explored here. Several aspects would require additional investigation, including whether the sketched resolution method is feasible in practice, how to handle values that do not conform to `Equatable`, and how to treat assets.

## Row Sync Lifecycle

While the previous section focused on a single conflict in isolation, this section illustrates the lifecycle of a row across both non-conflict and conflict scenarios.

The table below follows a row with a single field through three phases: initial creation (steps 1–4), a server-side update (steps 5–6), and finally concurrent client-side and server-side edits (steps 7–11). For each step, it shows the state of the _database row_ on the client, the _last-known server record_ stored on the client, and the current _server record_ on the server. Changes are highlighted in bold.

Per the premise above, modification timestamps are omitted here, as they are only relevant for conflict resolution. In step 9, both possible `count` outcomes are shown because either result could win depending on which side last edited the field.

| Step                                                                                                                                                                        | Database Row (Client) | Last-known Server Record (Client) | Server Record (Server) |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------- | --------------------------------- | ---------------------- |
| 1. No row exists yet.                                                                                                                                                       | –                     | –                                 | –                      |
| 2. Client creates row.                                                                                                                                                      | **count=1**           | –                                 | –                      |
| 3. Client sends record to server (initial state). Server sets change tag.                                                                                                   | **count=1**           | –                                 | **count=1 @ a**        |
| 4. Send is confirmed and client saves as last-known server record.                                                                                                          | count=1               | count=1 @ a                       | count=1 @ a            |
| 5. Server updates record independently.                                                                                                                                     | count=1               | count=1 @ a                       | **count=2 @ b**        |
| 6. Client fetches updated record and advances last-known server record.                                                                                                     | count=2               | count=2 @ b                       | count=2 @ b            |
| 7. Client updates row locally.                                                                                                                                              | **count=3**           | count=2 @ b                       | count=2 @ b            |
| 8. Server updates record independently.                                                                                                                                     | **count=3**           | count=2 @ b                       | **count=4 @ c**        |
| 9. Conflict-on-send (client sends stale record) or conflict-on-fetch (client fetches newer record). Client resolves the conflict and advances the last-known server record. | **count=3/4**         | count=4 @ c                       | count=4 @ c            |
| 10. Client sends record to server (merged state). Server sets change tag.                                                                                                   | **count=3/4**         | count=4 @ c                       | **count=3/4 @ d**      |
| 11. Send is confirmed and client saves as last-known server record.                                                                                                         | count=3/4             | count=3/4 @ d                     | count=3/4 @ d          |

While the table shows how the state evolves at each step, it does not convey the path through the history, nor the temporary divergence and later convergence. This is better seen in the branching timeline below, where the lifecycle is visualized. The diagrams use the following legend:

![]()

Initial creation and upload (steps 1–4):

![]()

Server-side update (steps 5–6):

![]()

Concurrent edits and conflict resolution (steps 7–11):

![]()

The lifecycle makes it clear that the client keeps two views of a row: the _local database row_ and the _last-known server record_. As long as edits occur on only one side (steps 1–6), changes propagate in a single direction and the two views converge. When both the client and the server make concurrent changes (steps 7–11), a divergence occurs and conflict resolution is required (step 9) to bring them back together.

Conflict resolution is performed using a three-way merge with access to the ancestor, client, and server versions as described above (step 9a). Effectively, the client version is rebased onto the server version, and the merged state becomes the new client row pending upload (step 9b). Once the server confirms it, a convergent state is restored (steps 10–11).

[1]:	https://swiftpackageindex.com/pointfreeco/sqlite-data/main/documentation/sqlitedata/cloudkit#Record-conflicts
[2]:	CurrentBehavior.md
[3]:	https://github.com/pointfreeco/sqlite-data/discussions/272
[4]:	https://github.com/pointfreeco/sqlite-data/discussions/272#discussioncomment-14975219
[5]:	https://github.com/pointfreeco/sqlite-data/discussions/272#discussioncomment-15170263
[6]:	https://github.com/pointfreeco/sqlite-data/discussions/272#discussioncomment-15170263

[image-1]:	images/ConflictInputs.png
