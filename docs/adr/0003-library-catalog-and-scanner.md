# Separate Music Library reads from scanning

The Music Library will provide Catalog and Scanner facets backed by a shared private SQLite store. Catalog provides paginated, domain-specific reads, while Scanner owns incremental filesystem reconciliation, metadata extraction, bounded transaction batches, cancellation, scan problems, and Unavailable Track handling; neither facet exposes generic database CRUD operations.

Track identity is durable. Rebuilding derived metadata and search indexes does not delete identity records, and Saved Playlists and the restored Play Queue refer to opaque Track IDs rather than database row details or file paths. A normal scan updates Tracks in place, missing files become Unavailable Tracks, and future moved-file detection may reconnect a Track only when the match is sufficiently confident; Linkamp will leave an ambiguous Track unavailable rather than guess incorrectly.

## Considered Options

A database-shaped Library with generic insert, update, delete, and filter operations would leak scan invariants, transaction behavior, and persistence concepts into callers. A single Library object owning every query and scan operation would have a small external interface but risk concentrating unrelated read, indexing, progress, and filesystem implementation concerns in one oversized module.

## Consequences

Catalog and Scanner initially live in the same `internal/library` package and share one SQLite implementation; they are facets of one in-process capability, not separate services. Extracted metadata, normalized search values, and caches are rebuildable, while Track identity, Saved Playlists, queue restoration, and preferences are durable user data. Schema migrations are required from the first database version. A destructive Music Library reset is distinct from rebuilding derived indexes and must warn when durable references could become unresolved.
