# CloudKit Sync in SQLiteData

This repository contains notes on CloudKit sync behavior in [SQLiteData][1], collected while exploring support for customizable conflict resolution as discussed in [https://github.com/pointfreeco/sqlite-data/discussions/272][2].

- [Current CloudKit Sync Behavior][3]: Documentation of the current built-in “field-wise last edit wins” conflict resolution strategy.

[1]:	https://github.com/pointfreeco/sqlite-data
[2]:	https://github.com/pointfreeco/sqlite-data/discussions/272
[3]:	CurrentBehavior.md