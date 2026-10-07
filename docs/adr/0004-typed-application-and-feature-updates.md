# Use typed application operations and feature-specific updates

Linkamp will use a concrete Application facade to coordinate cross-module workflows through typed Go methods. Modules retain ownership of their state and expose communication suited to their behavior: returned errors for operations that cannot be accepted, immutable snapshots for current state, explicit operation handles for long-running Library Scans, coalesced revisions for committed Music Library changes, and a bounded lossy stream for Visualization data.

## Considered Options

Allowing the desktop adapter to call every module directly would be initially simple but would spread cross-module orchestration across toolkit callbacks and make future control adapters duplicate behavior. A global dispatcher, event bus, and application-state store would provide one uniform update path, but would erase important differences between reliable control state, request-response Catalog queries, scan progress, user notices, and high-frequency disposable Visualization frames.

## Consequences

The Application facade must earn its place by coordinating modules rather than merely forwarding every method. Desktop window and widget state remains in the desktop adapter, and core modules never depend on toolkit scheduling or widget types. Future media-key, MIDI, and remote-control adapters declare small consumer-owned interfaces containing only the typed operations they require. Music Library consumers refresh their queries when a newer revision is published instead of replaying row-level events. Linkamp will not introduce a global event bus or global state store.
