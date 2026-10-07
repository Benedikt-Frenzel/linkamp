# Technology options for Linkamp

Research captured 2026-10-07. This note compares implementation candidates against Linkamp's accepted architecture; it is not an ADR. A candidate becomes a decision only after a disposable prototype passes the stated checks.

## Decision criteria

Linkamp targets Linux and macOS first, a Music Library of at least 100,000 Tracks, a conventional library view plus a distinctive compact player, real-time Visualization, and a codebase one Go learner can maintain. The core must remain independent from UI, audio, decoder, and SQLite implementations.

## Desktop interface

### Fyne

Fyne describes itself as a Go UI toolkit for one desktop/mobile codebase. Its current README requires Go 1.22+, a C compiler, and platform development tools, and provides its own packaging commands. Its `List` pools row objects and its `Table` caches and reuses cell templates, which makes it a credible candidate for a large virtualized Track view rather than requiring 100,000 live widgets.

Primary sources:

- https://github.com/fyne-io/fyne#about
- https://github.com/fyne-io/fyne#prerequisites
- https://github.com/fyne-io/fyne/blob/master/widget/list.go
- https://github.com/fyne-io/fyne/blob/master/widget/table.go

**Fit:** strongest all-Go baseline. It offers conventional widgets for Music Library management while allowing custom canvas objects for the compact Player and Visualization.

**Risks to prototype:** visual styling freedom, keyboard behavior, accessibility on Linux/macOS, smooth high-frequency custom drawing, table interaction, and whether its non-native visual language suits Linkamp.

### Gio

Gio is an immediate-mode Go GUI for macOS, Linux, Windows, mobile platforms, and experimental WebAssembly. Its own README explicitly says its tags are pre-1.0 reference tags without ongoing support and that minor releases may contain breaking changes. Gio has list state and semantic operations in its widgets, but adopting it would require more custom desktop interaction work than a conventional widget toolkit.

Primary sources:

- https://github.com/gioui/gio#gio---httpsgiouiorg
- https://github.com/gioui/gio#tags
- https://github.com/gioui/gio/blob/master/widget/list.go
- https://github.com/gioui/gio/tree/master/io/semantic

**Fit:** strong rendering control and an all-Go implementation, attractive if Linkamp's custom Player becomes more important than conventional library widgets.

**Risk:** API evolution and the amount of table, focus, selection, accessibility, and desktop behavior Linkamp may have to own.

### Wails

Wails wraps a Go backend and an HTML/CSS/JavaScript frontend into a desktop binary, generates TypeScript definitions for bound Go methods, and uses platform web-rendering engines instead of bundling a browser. Wails v2 is documented as stable while v3 is beta. Linux v2 builds require GCC, GTK3, and WebKitGTK; WebKit ABI/package differences vary by distribution. macOS uses WKWebView.

Primary sources:

- https://github.com/wailsapp/wails#introduction
- https://github.com/wailsapp/wails#features
- https://github.com/wailsapp/wails#getting-started
- https://wails.io/docs/gettingstarted/installation
- https://wails.io/docs/guides/linux-distro-support

**Fit:** strongest styling and browser-canvas option, and likely the easiest route to a polished hybrid interface.

**Risks:** two language/tool ecosystems, a Go/JavaScript seam, Linux WebKit packaging, platform differences between web engines, and more framework knowledge for a solo maintainer. The frontend must receive paginated summaries and analyzed Visualization frames, never the entire Music Library or raw PCM.

### UI recommendation

Prototype **Fyne first** and **stable Wails v2 as the challenger**. Keep Gio as a fallback if Fyne's rendering control is insufficient and maintaining a web frontend is undesirable. Do not select Wails v3 while it remains beta.

Both prototypes must implement the same test screen:

1. a 100,000-row generated Track data source with only visible rows rendered;
2. sortable columns, keyboard selection, incremental search, and rapid scrolling;
3. a custom compact Player panel;
4. a fake 60 FPS spectrum fed by bounded snapshots;
5. resizing and high-DPI scaling;
6. Linux and macOS packaging notes;
7. measured startup time, idle memory, scrolling behavior, and developer complexity.

The prototype is disposable and must not import Linkamp's future core packages.

## Audio output

### Oto v3

Oto is a low-level cross-platform playback library. Its official README lists Linux and macOS without CGo. Linux uses a pure-Go PulseAudio package and falls back to dynamically loaded ALSA; macOS links AudioToolbox automatically. One Oto `Context` owns the output configuration and creates players that pull PCM bytes from an `io.Reader`. Oto supports float32 PCM, buffers internally, exposes the buffered byte count, and allows setting its player-buffer size.

Primary sources:

- https://github.com/ebitengine/oto/tree/main#platforms
- https://github.com/ebitengine/oto/tree/main#linux-freebsd-openbsd
- https://github.com/ebitengine/oto/blob/main/context.go
- https://github.com/ebitengine/oto/blob/main/player.go
- https://github.com/ebitengine/oto/tree/main#advanced-usage

**Fit:** best first output adapter. Linkamp can expose its prepared ring as an `io.Reader` and configure one Oto context for float32 stereo output.

**Architectural correction:** Oto already has an internal buffer, so Linkamp would have two bounded buffering layers. Audible-position estimation must account for both Linkamp's prepared frames and `oto.Player.BufferedSize()`. The prototype must determine whether this remains accurate enough and whether gapless transitions can be driven through one continuous Oto player/reader.

**Limit:** Oto's context configuration is fixed and its simple interface is centered on the default output. Explicit device selection may later require a different adapter.

### malgo / miniaudio

`malgo` is a Go binding to miniaudio. Its README lists CoreAudio on macOS and PulseAudio, ALSA, and JACK on Linux. It requires CGo but does not require linking additional audio libraries on macOS and links only `-ldl` on Linux/BSD. It exposes lower-level device callbacks and device information.

Primary sources:

- https://github.com/gen2brain/malgo#malgo
- https://github.com/gen2brain/malgo#platforms
- https://github.com/gen2brain/malgo/blob/master/device.go
- https://github.com/gen2brain/malgo/blob/master/device_info.go

**Fit:** stronger candidate if Linkamp later needs output-device enumeration, callback-level timing, or behavior Oto cannot provide.

**Risk:** CGo, lower-level callback safety, and a larger platform surface for the maintainer. The Go binding's examples use a separate MP3 decoder; the bundled miniaudio C decoder API is not evidence that malgo exposes a suitable Go decoder interface.

### Audio recommendation

Prototype Oto first. Keep the `Output` seam narrow enough that malgo can replace it. The Oto spike must verify:

- one continuous float32 reader across two synthetic Tracks;
- no inserted silence at the Track boundary;
- pause and restart behavior;
- buffer flush on seek;
- consumed-position estimation;
- underflow reporting;
- clean shutdown;
- Linux PulseAudio and ALSA fallback behavior;
- macOS behavior on a real machine.

Do not add malgo merely to make output-device selection hypothetical; revisit it when Linkamp actually implements device selection or Oto fails a required test.

## Decoding and processing

### Beep v2

Beep provides a small `Streamer` interface, seeking, resampling/composition utilities, MP3 and FLAC adapters, and Oto-based playback. Its format is stereo `float64` sample pairs. Beep's MP3 adapter depends on `github.com/hajimehoshi/go-mp3`; its FLAC adapter depends on `github.com/mewkiz/flac`.

Primary sources:

- https://github.com/gopxl/beep#features
- https://github.com/gopxl/beep/blob/main/interface.go
- https://github.com/gopxl/beep/blob/main/mp3/decode.go
- https://github.com/gopxl/beep/blob/main/flac/decode.go
- https://github.com/gopxl/beep/blob/main/go.mod

**Fit:** useful for an early playback spike because MP3 and FLAC share a seekable stream interface and Beep already has resampling primitives.

**Mismatch:** Linkamp's accepted internal format is interleaved `float32`; adopting Beep directly means either revisiting that internal choice or converting from `[2]float64`. Beep is deliberately stereo, which is acceptable for the initial product but must remain an adapter constraint rather than a Music Library assumption.

### MP3 maintenance risk

The official `go-mp3` README states: **“This project is no longer maintained.”** Oto's README example and Beep's MP3 adapter both use this decoder, so their examples do not remove that maintenance risk.

Primary source:

- https://github.com/hajimehoshi/go-mp3#go-mp3

It may still be adequate for a constrained initial decoder adapter, but Linkamp must test it against a corpus of valid, variable-bitrate, tagged, truncated, and gapless MP3 files. The decoder seam must remain replaceable. A long-term choice may require a maintained native decoder or FFmpeg-based adapter, with the packaging and licensing costs evaluated explicitly.

### FLAC

`mewkiz/flac` exposes FLAC stream, frame, metadata, and seeking APIs. Its project history documents active robustness and seek fixes, including handling corrupt input and fixes around seeking.

Primary sources:

- https://github.com/mewkiz/flac
- https://github.com/mewkiz/flac/blob/master/README.md#changes
- https://pkg.go.dev/github.com/mewkiz/flac

**Fit:** credible first FLAC adapter, subject to fixture and performance tests.

### Decoder recommendation

Use Beep v2 only as a **prototype accelerator**, not as an architectural dependency yet. Build the accepted Linkamp decoder contract around behavior—format, frame reads, duration, seek, close, and errors—and compare:

1. thin adapters over Beep's MP3 and FLAC decoders;
2. direct adapters over `go-mp3` and `mewkiz/flac`;
3. a maintained native/FFmpeg option if the MP3 corpus exposes correctness or gapless limitations.

The corpus must include at minimum:

- constant- and variable-bitrate MP3;
- MP3 files with ID3v2 metadata and embedded artwork;
- MP3 gapless metadata where available;
- 16-bit and 24-bit FLAC;
- mono and stereo FLAC;
- files with different sample rates;
- truncated and malformed inputs;
- seek checks near start, middle, and end.

## SQLite

### modernc.org/sqlite

The project describes itself as a pure-Go SQLite driver with no CGo. Its source includes generated Go translations of SQLite and supports `database/sql`.

Primary sources:

- https://gitlab.com/cznic/sqlite
- https://pkg.go.dev/modernc.org/sqlite
- https://github.com/modernc-org/sqlite#pure-go-sqlite-no-cgo

**Fit:** strongest portability baseline, especially for reproducible builds and cross-compilation.

**Risks:** large generated implementation and dependency graph, compile cost, and performance characteristics that must be measured with Linkamp's workload rather than assumed.

### mattn/go-sqlite3

`go-sqlite3` implements `database/sql` and uses CGo, requiring `CGO_ENABLED=1` and a C compiler. Its documented feature tags include FTS5.

Primary sources:

- https://github.com/mattn/go-sqlite3#description
- https://github.com/mattn/go-sqlite3#installation
- https://github.com/mattn/go-sqlite3#features

**Fit:** mature alternative when CGo is already acceptable and if measurements or SQLite feature requirements favor it.

**Risk:** cross-compilation and platform build setup become more involved.

### Search

SQLite FTS5 provides full-text search virtual tables and `MATCH` queries. Linkamp should not commit to it until a prototype compares its ranking and prefix behavior with the desired Track/artist/album search, but ordinary substring scans should not be assumed adequate for every 100,000-Track query.

Primary source:

- https://www.sqlite.org/fts5.html

### SQLite recommendation

Prototype `modernc.org/sqlite` first through `database/sql`, with SQL owned by the private Music Library implementation. The benchmark should generate 100,000 Tracks and measure:

- initial batched insertion;
- incremental update of 1,000 Tracks;
- title/artist/album search;
- paginated sorted browsing;
- reads while a scan transaction commits bounded batches;
- marking unseen Tracks unavailable;
- migration and reopen time;
- database size and process memory.

Use WAL mode, foreign-key enforcement, and a busy timeout as explicit connection policy, then verify behavior rather than relying on defaults. Run the same benchmark with `mattn/go-sqlite3` only if modernc performance, correctness, binary size, or required features are unsatisfactory.

## Provisional stack for experiments

This is the recommended order of investigation, not the final production stack:

| Concern | First candidate | Challenger / fallback |
| --- | --- | --- |
| Desktop UI | Fyne | Wails v2; Gio if custom rendering dominates |
| Audio output | Oto v3 | malgo/miniaudio |
| Decoder facade | Linkamp-owned contract | — |
| MP3 experiment | Beep v2 / go-mp3 adapter | maintained native or FFmpeg adapter |
| FLAC experiment | Beep v2 or mewkiz/flac adapter | alternative only if corpus fails |
| SQLite | modernc.org/sqlite | mattn/go-sqlite3 |
| Full-text search | SQLite FTS5 experiment | indexed normalized SQL queries |

## Architecture consequences

No researched library requires changing the accepted modular-monolith design. The following seams remain justified:

- desktop adapter around typed Application operations;
- audio-output adapter consumed by Player;
- decoder registry internal to Playback;
- private SQLite implementation shared by Catalog and Scanner.

The main finding that affects ADR-0005 is that Oto performs its own internal buffering. A prototype must establish whether Linkamp needs a separate ring buffer, a thinner bounded reader in front of Oto, or a lower-level output adapter. ADR-0005 specifies the behavior—decode ahead and keep output non-blocking—not a mandatory duplicate buffering implementation.

## Recommended next experiments

1. **Audio walking spike:** synthetic PCM through Oto, then one MP3 and one FLAC, followed by a continuous two-Track transition.
2. **Database benchmark:** generated 100,000-Track library with realistic search and scan writes using modernc SQLite.
3. **Fyne UI spike:** virtualized Track table and fake Visualization.
4. **Wails UI spike:** the same screen and measurements, without reusing production frontend code.
5. Decide each dependency only from experiment results and record accepted choices as individual ADRs.
