# Linkamp

Linkamp is a local-first desktop music player for large personal music collections. It combines a modern application foundation with the immediacy, personality, and compact workflows associated with classic desktop music players.

## Language

**Music Library**:
The indexed collection of playable tracks Linkamp has discovered in folders chosen by the listener.
_Avoid_: Database, catalog

**Track**:
A single playable audio recording known to the Music Library. In the initial release, each Track corresponds to one local MP3 or FLAC file.
_Avoid_: Song, media item

**Library Folder**:
A local filesystem folder selected by the listener as a source for the Music Library.
_Avoid_: Watch folder, music path

**Library Scan**:
The process of reconciling the Tracks found in Library Folders with the Music Library.
_Avoid_: Import, crawl

**Unavailable Track**:
A Track retained in the Music Library when its audio file cannot currently be found or read. It may remain referenced by a Saved Playlist or the Play Queue.
_Avoid_: Deleted Track, missing song

**Player**:
The active playback session, including the Play Queue, current Track, playback position, and playback order.
_Avoid_: Engine, transport

**Play Queue**:
The ordered sequence of Tracks scheduled for playback, including the listener's current position and playback order choices.
_Avoid_: Playlist, song list

**Saved Playlist**:
A listener-named, reusable selection of Tracks. Unlike the Play Queue, it does not represent current playback state.
_Avoid_: Play Queue

**Visualization**:
A real-time visual interpretation of the audio currently being played.
_Avoid_: Visualizer, animation
