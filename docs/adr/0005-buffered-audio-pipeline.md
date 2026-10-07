# Buffer decoded audio ahead of output

Playback will decode, convert, and process audio ahead of the output into a bounded ring buffer. The audio-output path performs only bounded, non-blocking frame transfer, while Visualization consumes a separate bounded and lossy analysis stream that can drop stale work without delaying audio.

The Player's owner goroutine coordinates decoder workers, output lifecycle, seeking, and Track transitions but does not decode audio itself. Seeking and Track replacement increment an internal playback generation, flush prepared audio, cancel obsolete work, and discard any later result belonging to an older generation. Reported playback position follows frames consumed by the output rather than frames merely decoded into the buffer.

## Considered Options

A goroutine and channel for every processing stage would make the conceptual pipeline visible in code, but would introduce unnecessary concurrency seams, buffer transfers, backpressure paths, and difficult flush ordering. Decoding directly inside the audio callback would minimize buffering but expose timing-critical output to file I/O, decoder latency, allocation, and unpredictable error handling.

## Consequences

Playback uses an internal floating-point PCM representation and converts source formats to a stable format negotiated with the output adapter. Future non-destructive normalization sits after source conversion and before both analysis and user volume, allowing Visualization to represent the normalized music independently of listening volume. Gapless playback requires preparing the next Track, honoring available encoder-delay and padding information, and joining converted frames without reopening the output device. Tests use deterministic decoders and output adapters to verify transitions, seeking, stale-generation rejection, underflow, failure recovery, and non-blocking analysis without relying on wall-clock sleeps or a real audio device.
