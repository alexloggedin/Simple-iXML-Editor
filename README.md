# Field Log — iXML Take Editor

A single-file, no-install browser tool for viewing, editing, and previewing
the **iXML** metadata that field recorders embed inside WAV files, and for
exporting a CSV "sound report" from a folder of takes.

Built against a **Zoom F4** recorder, but the iXML schema (`PROJECT`,
`SCENE`, `TAKE`, `NOTE`, `CIRCLED`, `TRACK_LIST`, `FILE_SET`/`FAMILY_UID`)
is shared by most BWF-writing field recorders (Zoom, Sound Devices,
Zaxcom, etc.), so it should work unmodified on files from those too.

## What it does

- **Open a folder** of recordings (with any number of subfolders) and see
  every take listed, grouped correctly:
  - A polyphonic take split across several mono files, or a single
    interleaved multi-channel file (e.g. one WAV containing `BOOM` / `MIC
    A` / `MIC B`), are both recognized as **one take**, using the
    recorder's `FAMILY_UID` tag.
  - Files with no `FAMILY_UID` are shown as their own standalone take.
- **Edit metadata** — Project, Tape, Scene, Take, Note, Circled — for a
  take, and save it back into every file that belongs to that take at
  once. This is a true in-place edit of the file's `iXML` chunk; the audio
  data itself is never touched or re-encoded.
- **Play back and scrub** a take's audio, with every file/channel in the
  take kept in sync — so a 3-channel interleaved file, or a take split
  across 3 mono files, scrubs and plays as a single unit.
- **Mute/monitor individual tracks** during playback (e.g. solo the boom,
  drop a wireless track) without touching the saved metadata.
- **Filter** the take list by Project or Scene using the dropdowns in the
  header.
- **Export a CSV sound report**, with a field-chooser dialog that lets you
  pick exactly which columns to include and in what order. The export
  respects the current Project/Scene filter, so you can export just one
  scene's takes.

## Requirements

- A **Chromium-based browser** — Chrome or Edge. The tool relies on the
  [File System Access API](https://developer.mozilla.org/en-US/docs/Web/API/File_System_Access_API)
  to read and write files directly on your local disk, which isn't
  supported in Firefox or Safari.
- No installation, no build step, no server. Just open the `.html` file.

## Using it

1. Visit [https://alexloggedin.github.io/Simple-iXML-Editor/](https://alexloggedin.github.io/Simple-iXML-Editor/) in Chrome or Edge. (Note: You can also just download the index.html to use locally)
2. Click **Open Folder** and pick the top-level folder containing your
   session's recordings (subfolders are scanned automatically). You'll be
   asked to grant read/write access — this is required so metadata edits
   can be saved back to the files.
3. Click a take in the left-hand list to see its metadata and files.
4. Edit any of the Project / Tape / Scene / Take / Note / Circled fields
   and click **Save iXML to N files** to write the changes back to every
   file in that take.
5. Click **Load Audio** to decode the take's audio, then **Play** and use
   the scrub bar to move through it. Use the **Channel monitor** buttons
   below the transport to mute/unmute individual tracks for monitoring —
   this does not change anything on disk.
6. Use the **Project** / **Scene** dropdowns in the header to narrow the
   take list to a specific project or scene.
7. Click **Export CSV Sound Report** to choose which columns to include
   (and their order) before downloading `sound_report.csv`. The default
   columns are `Project | Scene | Take | Track Names | Notes`.
8. If you edit files with another program while this tool is open, click
   **Rescan** to reload everything from disk.

## Known limitations (possible next steps)

- Track names (`TRACK_LIST`) and the `SPEED`/timecode block are read and
  displayed but not currently editable — only the take-level fields
  (Project/Tape/Scene/Take/Note/Circled) can be edited and saved.
- No validation UI for malformed/corrupted iXML beyond falling back to "no
  metadata found, will create on save."
- Audio decoding happens fully client-side via the Web Audio API, so very
  long (multi-hour) files will take a moment to load for playback. This
  doesn't affect metadata reading/writing, which never loads audio data
  into memory.
- There is currently no bulk file/folder renaming feature.

## For developers

See [`CODE_DOCS.md`](./CODE_DOCS.md) for a full walkthrough of the code's
structure — the RIFF/iXML byte-level parsing, the take-grouping model, the
audio transport, and the rendering/state layer — written so someone
unfamiliar with the codebase can make changes confidently.
