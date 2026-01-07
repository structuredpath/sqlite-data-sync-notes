# CloudKit Sync in SQLiteData

This repository contains notes on CloudKit sync behavior in [SQLiteData][1], collected while exploring support for customizable conflict resolution as discussed in [https://github.com/pointfreeco/sqlite-data/discussions/272][2].

- [Current CloudKit Sync Behavior][3]: Documentation of the current built-in conflict resolution mechanism based on the “field-wise last edit wins” strategy.
- [A Conceptual Model for Built-in Conflict Resolution][4]: A revised conceptual model of the built-in conflict resolution mechanism, clarifying its semantics and addressing shortcomings in the current behavior.

[1]:	https://github.com/pointfreeco/sqlite-data
[2]:	https://github.com/pointfreeco/sqlite-data/discussions/272
[3]:	CurrentBehavior.md
[4]:	BuiltInConflictResolutionModel.md