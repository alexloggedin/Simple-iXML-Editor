# Code Documentation — Field Log iXML Take Editor

This is a **single HTML file** (`ixml-take-editor.html`): all CSS and
JavaScript are inline, there is no build step, and there are no external
JS libraries or npm dependencies. Everything the app needs — RIFF/WAV
chunk parsing, XML parsing, audio decoding, filesystem access — is either
hand-written or a native browser API. This document walks through the
code in the order it appears in the file, then covers how to extend it.

## Why no libraries?

Two deliberate constraints shaped this:

1. **Portability.** Being a single `.html` file with no dependencies means
   it can be opened directly, emailed, or dropped into a project folder —
   no `npm install`, no bundler, no CDN availability risk.
2. **Correctness for a narrow job.** The only binary format-editing this
   tool does is replacing one RIFF chunk (`iXML`) in a WAV file without
   touching the audio payload. That's a small, well-understood operation
   (see "RIFF / iXML chunk-level I/O" below) that's safer to hand-write
   and directly test than to pull in a general-purpose BWF library whose
   edge-case handling can't be inspected as easily.

## File layout (top to bottom in the `<script>` tag)

| Section | Responsibility |
|---|---|
| Icons | Inline SVG strings for the mute/unmute track buttons |
| RIFF / iXML chunk-level I/O | Byte-level reading/writing of the `iXML` chunk |
| iXML (BWFXML) tag helpers | Get/set individual XML tags on a parsed iXML document |
| CSV field catalogue | Data-driven definition of every exportable CSV column |
| State | The two global mutable objects: `state` and `transport` |
| Filtering | Project/Scene header dropdowns |
| Rendering | Building the take list, detail panel, and channel monitor from `state` |
| Save | Writing edited metadata back to disk |
| CSV export | The field-chooser modal and the actual CSV builder |
| Wiring | Top-level event listeners, folder open/rescan |

---

## RIFF / iXML chunk-level I/O

**The core data model:** a WAV file is a RIFF container — a 12-byte
header (`"RIFF"`, 4-byte total size, `"WAVE"`), followed by a flat
sequence of chunks. Each chunk is `4-byte ASCII ID` + `4-byte
little-endian size` + that many bytes of data, padded with one zero byte
if the size is odd (chunks always start on an even offset).

```
Offset 0                                          Offset N
┌──────┬──────────┬──────┬──────┬──────┬──────┬──────┐
│ RIFF │ size(LE) │ WAVE │ fmt  │ data │ iXML │ ...  │
└──────┴──────────┴──────┴──────┴──────┴──────┴──────┘
```

### Key functions

- **`findChunkOffsets(file)`** — walks the chunk list and returns each
  chunk's `{id, offset, dataOffset, size}`. Only ever reads chunk
  *headers* (8 bytes each) via `File.slice()` — never the chunk data
  itself — so this is cheap even on a multi-gigabyte file.

- **`extractIXMLTextLazy(file)`** — finds the `iXML` chunk (if any) and
  reads just its data. Important detail: **many recorders (confirmed on
  Zoom) pre-allocate the `iXML` chunk larger than the actual XML and pad
  the rest with null bytes** — likely so a later metadata edit can happen
  in-place without resizing the file. A strict `DOMParser` treats that
  trailing padding as invalid trailing content and fails the *entire*
  parse (not just a warning) — which is why this function trims the text
  to end at the last `</BWFXML>` closing tag before returning it. If you
  ever see "no iXML found" on a file that clearly has metadata, this is
  the first place to check — inspect whether the chunk's declared `size`
  is larger than the actual XML content.

- **`buildIXMLChunkBlob(xmlText)`** — the inverse: builds the raw bytes of
  a brand new `iXML` chunk (header + XML + padding byte if needed) as a
  `Blob`. Note that saving **does not** preserve a recorder's original
  padding reservation — the chunk is rebuilt sized exactly to the new
  content. This is always correct, just means a file re-opened on the
  original recorder gets a freshly-sized chunk rather than the one it
  wrote.

- **`writeIXMLToFile(fileHandle, file, newXmlText, existingChunk)`** — the
  actual disk write. Rather than reading the whole file into a JS
  `Uint8Array`, mutating it, and writing it all back (which would load
  potentially many GB of audio into memory), this builds the new file as
  a `Blob` composed of *slices* of the original file (the bytes before the
  old chunk, the bytes after it) plus the new chunk's bytes in between.
  Slicing a `File`/`Blob` is a cheap, lazy operation — the browser streams
  the actual bytes straight from the original file to the new one during
  `writable.write()`. Only the 12-byte RIFF header is read into memory
  (to patch the total-size field).

  This has been verified against a real Zoom F4 file: parsed the real
  chunk layout, round-tripped an edit, and diffed the `data` chunk bytes
  before/after to confirm they're byte-identical — only the `iXML` chunk
  (and everything after it, which shifts) changes.

### If you need to support a new chunk type

Follow the same pattern: use `findChunkOffsets` to locate it, read just
its data with `readBytes`, and if you need to write it back, build a
`Blob`-based splice like `writeIXMLToFile` rather than materializing the
whole file in memory.

---

## iXML (BWFXML) tag helpers

The iXML chunk's content is XML with a `<BWFXML>` root element. These
functions work against a parsed `Document` (from `parseIXMLDoc`, which
wraps `DOMParser`), not raw strings:

- `getTag(doc, tagName)` / `setTag(doc, tagName, value)` — read/write a
  top-level tag. Both only ever touch the **first** matching element in
  document order. This matters because e.g. `<NOTE>` can legitimately
  appear twice in real files — once at the top level (the take note) and
  once nested inside `<SPEED><NOTE></NOTE></SPEED>` (which Zoom leaves
  empty) — and `getTag`/`setTag` correctly resolve to the top-level one
  since it comes first in the document.

- `getFamilyUID(doc)` — reads `FILE_SET/FAMILY_UID`, the tag recorders use
  to tie multiple files together as one take. Returns `null` if absent
  (standalone file).

- `getTracks(doc)` — returns **every** `<TRACK>` entry (not just the
  first), each as `{channelIndex, interleaveIndex, name}`, sorted by
  `INTERLEAVE_INDEX`. This sort matters: `INTERLEAVE_INDEX` (1-based) is
  the track's actual position within the *decoded audio buffer's*
  channels, so after sorting, `tracks[0]` corresponds to audio channel 0,
  `tracks[1]` to channel 1, etc. `trackLabel()` (in the transport section)
  relies on this alignment to label playback channels correctly.

---

## Data model: "take" and "file" objects

Understanding these two shapes is the key to understanding the rest of
the app. They're built by `scanAndGroup()` / `loadFileMeta()` and then
read/mutated everywhere else.

**A `file` object** (one per `.wav`/`.bwf` found on disk):
```js
{
  handle,        // FileSystemFileHandle — used to re-read/write this file
  path,          // path relative to the opened folder, e.g. "Session1/SCENE_1A-T004.WAV"
  name,          // just the filename
  file,          // the current File object (re-fetched after every save)
  error,         // set if chunk-parsing threw (e.g. not a valid RIFF file)
  hasIXML,       // false if no iXML chunk was found/parseable — UI flags this
  familyUID,     // FILE_SET/FAMILY_UID, or null
  tracks,        // [{channelIndex, interleaveIndex, name}, ...] sorted by interleaveIndex
  channelMuted,  // [bool, ...] one per audio channel — populated by loadTakeAudio, NOT saved to disk
}
```

**A `take` object** (one per unique `FAMILY_UID`, or one per standalone
file):
```js
{
  key,        // FAMILY_UID, or "standalone:<path>" for files with no family
  familyUID,  // FILE_SET/FAMILY_UID, or null
  project, scene, take, tape, note, circled,  // shared fields — see extractSharedFields()
  files,      // [file, file, ...] — see shape above
}
```

The grouping logic itself lives in `scanAndGroup()`: it walks every file,
computes a group key, and pushes matching files into the same take's
`files` array. **Why this matters for polyphonic recordings**: a Zoom
F4/F8 in polyphonic mode can either (a) write one *interleaved*
multi-channel WAV per take, or (b) split each track into its own *mono*
WAV file, all sharing one `FAMILY_UID`. Both cases produce a take with a
single `familyUID` — the difference is just whether `take.files` has one
entry (with several tracks) or several entries (each with one track).
Everything downstream (playback, the file table, CSV export) already
handles both shapes because it iterates `take.files` and, within each
file, `file.tracks`/decoded channels — it never assumes "one file = one
track" or "one take = one file".

---

## CSV field catalogue

`CSV_FIELD_DEFS` is the single source of truth for what can appear in the
exported CSV. Each entry is `{id, label, get}`:

```js
{ id: 'tracknames', label: 'Track Names', get: t => /* ... */ }
```

- `id` — stable identifier used in `state.csv.order` / `state.csv.checked`
- `label` — shown in the field-chooser modal and used as the CSV header
- `get(take)` — pulls the value out of a take object

**To add a new exportable column**, add one entry to `CSV_FIELD_DEFS`.
Nothing else needs to change — the modal (`renderCsvModal`) and the
export function (`exportCSV`) both iterate this list generically.

`DEFAULT_CSV_CHECKED` / `DEFAULT_CSV_ORDER` define the default report
shape (`Project | Scene | Take | Track Names | Notes`) — checked fields
first, then every other field appended afterward (unchecked, but visible
in the modal so it's discoverable).

---

## State

Two global objects hold all mutable state, deliberately not hidden inside
closures, so they're inspectable from the browser console while
debugging (`state`, `transport`).

- **`state`** — `dirHandle` (the open folder), `takes` (everything found
  by the last scan), `selectedKey` (which take is shown in the detail
  panel), `filters` (header dropdown selections), `csv` (current export
  column selection/order — persists across exports until changed or reset).

- **`transport`** — see the Audio Transport section below.

---

## Filtering

`state.filters = {project, scene}` — empty string means "no filter on
this field". `takeMatchesFilters(take)` is the single predicate used both
by `getFilteredTakes()` (which drives the take list *and* the CSV export)
and by the "did the selected take just get filtered out?" check in
`onFilterChange()`.

`populateFilterOptions()` rebuilds the `<select>` options from whatever
Project/Scene values actually exist in the current scan (sorted,
deduplicated), and is called once per `rescan()`. If the previously
selected filter value no longer exists in the new list, it resets to "All"
rather than silently keeping a stale/invalid selection.

**To add a new filterable field** (e.g. Tape): add a `<select>` in the
header HTML, extend `state.filters`, extend `takeMatchesFilters`, and add
a case to `populateFilterOptions` and a `change` listener at the bottom of
the script, following the existing Project/Scene pattern.

---

## Audio transport (synced multi-channel playback)

This is the least obvious part of the code, so it's worth explaining the
underlying constraint: **Web Audio `AudioBufferSourceNode`s cannot be
paused or seeked once started** — only stopped, permanently, one time.
Everything here works around that:

- **"Pause"** (`pauseTransport`) means: compute how far we'd gotten
  (`startOffset + elapsed`), remember it, then stop all sources.
- **"Seek"** (the scrub bar's `input` handler in `renderDetail`) means:
  stop everything, update `startOffset` to the new position, and — only
  if we were mid-playback — immediately call `playFrom()` again from
  there.
- **"Play"** (`playFrom(offsetSeconds, take)`) creates a *fresh* set of
  `AudioBufferSourceNode`s, one per file, all told to `.start()` at the
  *same* `AudioContext.currentTime`, so multiple files (or multiple
  channels of one file) stay sample-accurately in sync.

**Multi-channel muting**: when a file's decoded buffer has more than one
channel (the common polyphonic case), `playFrom` inserts a
`ChannelSplitterNode` between the source and the destination, giving each
channel its own `GainNode`. `transport.trackGains[fileIndex][channelIndex]`
holds these, and `renderChannelMonitor`'s click handler flips
`gain.gain.value` between `0` and `1` live, without needing to
stop/restart playback.

**Reading the current playhead position**: `tick()` runs on
`requestAnimationFrame` while playing and computes
`startOffset + (audioContext.currentTime - startCtxTime)` each frame to
update the scrub bar and time readout, then auto-pauses when that
position reaches `transport.duration`.

`loadTakeAudio(take)` decodes every file in the take via
`AudioContext.decodeAudioData` and also (re)initializes each file's
`channelMuted` array to match the *actual* decoded channel count — which
is treated as authoritative even if it disagrees with what the iXML
`TRACK_COUNT` tag claims.

---

## Rendering

Rendering is plain DOM string-templating (`el.innerHTML = ...`) followed
by attaching event listeners to the freshly-created elements — there's no
virtual DOM or diffing. Three render functions matter:

- **`renderTakeList()`** — the left-hand list. Reads from
  `getFilteredTakes()`, not `state.takes` directly, so it always reflects
  the active Project/Scene filters. Shows an empty-state message
  distinguishing "no files scanned at all" from "files scanned but none
  match the current filter".

- **`renderDetail()`** — the metadata form, file table, and audio
  transport for `currentTake()`. Rebuilds its event listeners every call
  (load/play/pause/scrub/save), which is safe since the whole panel's DOM
  is rebuilt each time this runs.

- **`renderChannelMonitor(take)`** — the per-track mute/unmute buttons,
  populated once `loadTakeAudio` has run (so the actual channel count is
  known). Unlike the other two, its click handler updates the clicked
  button **in place** (icon + CSS class) rather than calling itself again,
  so muting one track doesn't cause every other button to flicker/re-render.
  Each button is a real `<button>` (not a checkbox) for a larger,
  easier-to-tap hit target, laid out in a horizontal wrapping row
  (`.channel-monitor-row`) rather than a vertical list, sized for legibility
  at a glance during a multi-track session.

---

## Save flow

`saveTake(take)` reads the current form values, then for **every** file
in the take: re-fetches the file fresh from its handle (so it's safe even
if the file changed since the last scan), re-extracts and re-parses its
current iXML (rather than reusing a cached `Document`, avoiding any stale
in-memory state), applies the six shared-field tags via `setTag`, and
writes the result with `writeIXMLToFile`. Per-file fields (track names)
are deliberately **not** touched by save — see "Known limitations" in the
README if you want to add that.

---

## CSV export

Two pieces:

- **`renderCsvModal()`** — draws the field list from `state.csv.order`
  (which determines both the on-screen row order *and* the export column
  order) and `state.csv.checked` (a `Set` of included field ids). The
  up/down buttons swap adjacent entries in `state.csv.order` and
  re-render; the checkboxes just add/remove from the `Set`. There's no
  separate "preview" — the list *is* the configuration.

- **`exportCSV()`** — filters `state.csv.order` down to checked fields (in
  that order), maps each to its `CSV_FIELD_DEFS` getter, and builds rows
  from `getFilteredTakes()` — so exporting while a Scene filter is active
  exports only that scene.

`csvEscape()` handles standard CSV quoting (wraps in quotes and doubles
internal quotes if a value contains a comma, quote, or newline).

---

## Wiring

The bottom of the script attaches all top-level event listeners exactly
once, at load time (there's no dynamic (re)attachment here — only the
per-take/per-modal-render functions above re-attach listeners on their
own generated DOM). `openFolder()` resets `state.filters` (a new folder's
Project/Scene values are unrelated to the old ones) and always triggers a
full `rescan()`. `rescan()` is also what recomputes the filter dropdown
options and re-enables the Export/Rescan buttons.

---

## Known gotchas for future changes

- **Null-padded iXML chunks.** Covered above — if metadata "goes missing"
  on some new recorder's files, check `extractIXMLTextLazy`'s padding
  trim logic first; other recorders might pad with something other than
  `</BWFXML>` as the closing tag, or use a different root element name.
- **File System Access API is Chromium-only.** `openFolder()` already
  guards for `window.showDirectoryPicker` being undefined, but any new
  feature that touches the filesystem should do the same.
- **`AudioBufferSourceNode` is single-use.** Any new transport feature
  (e.g. loop playback) needs to follow the existing
  stop-everything-then-create-new-sources pattern; there's no "resume".
- **`getTag`/`setTag` only touch the first match.** If you add support
  for an iXML tag that can legitimately appear more than once in a
  non-ambiguous way (unlike `NOTE`, which happens to work out), you'll
  need a more specific accessor.
