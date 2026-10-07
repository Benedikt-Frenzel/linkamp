# Use a modular monolith

Linkamp will be one desktop process containing an application core and a replaceable desktop adapter. Domain behavior is organized by capability, while the UI toolkit, storage, filesystem, media decoding, and audio output remain at explicit seams; this keeps the project understandable and testable as ordinary Go without introducing internal RPC or distributed state before Linkamp has a real need for them.

## Considered Options

A toolkit-centric application would deliver an early interface quickly but couple playback, library, and queue behavior to UI callbacks. A separate player-engine process would support multiple clients and independent playback, but would introduce protocol design, synchronization, supervision, and reconnection complexity that the initial product does not require.

## Consequences

Linkamp will initially ship as one executable and one operating-system process. The application core must not depend on the chosen desktop toolkit, but this does not require an interface around every implementation; seams are introduced only where behavior genuinely varies or tests need a substitute.
