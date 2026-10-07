# Use a typed Player with a single state owner

Linkamp's Player will expose ordinary typed Go methods while internally serializing mutations through one owner goroutine. The Player owns the active playback session—including Play Queue coordination, Track transitions, gapless preparation, failure recovery, and audio-pipeline lifecycle—so callers do not have to synchronize queue state with audio state or understand decoder ordering constraints.

## Considered Options

A separate audio Transport would keep queue calculations independent, but would leak gapless look-ahead and transition orchestration into callers. A generic `Dispatch(Command)` interface would be smaller and make additional control adapters straightforward, but would be less discoverable, weaken compile-time guidance, and introduce an exported command framework that Linkamp does not otherwise need.

## Consequences

Desktop controls, system media controls, future MIDI input, and any future remote-control adapter will invoke the same typed application operations. Public Player methods may submit private commands to the owner goroutine, but private commands are an implementation detail rather than part of the external interface. Control-state updates must preserve the latest state, while Visualization analysis is bounded and lossy so it can never delay audio. The Player may have substantial internal modules, but it must not absorb Music Library queries, desktop state, or toolkit behavior.
