# Speech Recorder specification

What the Speech Recorder app produces and what it accepts, for anyone building tools on them. It
is generic: it names no person, project or recording. [AUDIO-FORMAT.md](AUDIO-FORMAT.md) says
exactly how audio is saved and is part of this specification.

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are used as RFC 2119 defines them. A
requirement without a key word is a fact about the app. Section 7 holds suggestions only.

A change to the app that changes anything in sections 1 to 6 changes this document, the JSON
Schemas and the examples in the same change. Tests check the prompt script examples against the
validator and the manifest keys against the code; the manifest examples are not yet checked
against the manifest schema.

## 1. Files and where they are

1.1 A recording is a *session*: a series of audio segment files and one manifest, described in
[AUDIO-FORMAT.md](AUDIO-FORMAT.md).

1.2 On Google Drive the app writes `Speech Recorder/<session>/`, holding the segment files and
`manifest.json`. A consumer MUST treat a session folder without `manifest.json` as incomplete or
still being recorded.

1.3 The app MUST upload `manifest.json` after every segment it lists has reached Drive or been
marked `missing`.

1.3.1 For a session run from a prompt script (`$.promptRun` present) the session folder also
holds `prompt-script.json`, the script exactly as it was when the session started, whose SHA-256
identity (section 5) is `$.promptRun.hash`. The app uploads it before `manifest.json`. It can be
absent, for example when the phone could not keep a copy, and a consumer MUST NOT treat its
absence as an error.

1.4 A consumer MUST ignore a file or a manifest key it does not know.

## 2. The manifest

2.1 `manifest.json` is a JSON object described by
[manifest.schema.json](manifest.schema.json). A consumer SHOULD validate it against that schema.

2.2 The keys the app writes, and nothing else. The last group is written only for a session run
from a prompt script:

| Path                              | Type    | Present                          |
| --------------------------------- | ------- | -------------------------------- |
| `$.id`                            | string  | always                           |
| `$.format`                        | string  | always                           |
| `$.startedAt`                     | integer | always                           |
| `$.lastProgressAt`                | integer | always                           |
| `$.state`                         | string  | always                           |
| `$.segments`                      | array   | always                           |
| `$.segments[].name`               | string  | always                           |
| `$.segments[].durationMillis`     | integer | always                           |
| `$.segments[].bytes`              | integer | unless the segment is missing    |
| `$.segments[].md5`                | string  | unless the segment is missing    |
| `$.segments[].missing`            | boolean | only when `true`                 |
| `$.recordedMillis`                | integer | always                           |
| `$.capture.longestGapMillis`      | integer | once two reads have been timed   |
| `$.capture.overruns`              | integer | once two reads have been timed   |
| `$.capture.lostAtLeastMillis`     | integer | once two reads have been timed   |
| `$.windSpans`                     | array   | when the detector ran            |
| `$.windSpans[].kind`              | string  | only when a span was heard       |
| `$.windSpans[].startMillis`       | integer | only when a span was heard       |
| `$.windSpans[].endMillis`         | integer | only when a span was heard       |
| `$.windSpans[].strength`          | number  | only when a span was heard       |
| `$.segments[].samples`            | integer | prompt run only                  |
| `$.promptRun.scriptId`            | string  | prompt run only                  |
| `$.promptRun.version`             | integer | prompt run only                  |
| `$.promptRun.hash`                | string  | prompt run only                  |
| `$.promptRun.sampleRate`          | integer | prompt run only                  |
| `$.promptRun.fields`              | object  | script has `fields`              |
| `$.promptRun.takes[].promptId`    | string  | prompt run only                  |
| `$.promptRun.takes[].take`        | integer | prompt run only                  |
| `$.promptRun.takes[].promptStart` | integer | prompt run only                  |
| `$.promptRun.takes[].promptEnd`   | integer | once the spoken prompt has ended |
| `$.promptRun.takes[].answerStart` | integer | once the answer has started      |
| `$.promptRun.takes[].answerEnd`   | integer | once the answer has ended        |
| `$.promptRun.takes[].route`       | string  | when the route was known         |

2.2.1 Every `$.promptRun.takes[]` offset counts samples of the session as a whole, at
`$.promptRun.sampleRate`, as the recording thread wrote them. An offset is not a position in wall
time and a consumer MUST NOT convert it using `startedAt`. The `hash` is the identity of the
script version, defined in section 5; `version` is a label. `$.promptRun.fileName` is reserved: the
app does not write it and a consumer MUST NOT rely on it.

2.2.2 An end offset is the offset of the first sample after the span. An offset is the count of
samples already written when the event was noticed, and the count trails the microphone: by up to
one read of 200 milliseconds in steady recording and by up to 4 seconds while one segment closes
and the next opens. A `promptStart` is noted when the phone is asked to speak, so it is earlier
than the sound by the speech engine's start-up time. A consumer MUST allow a tolerance of 5
seconds when it cuts audio at an offset, and SHOULD prefer a margin on the side of the answer's
end.

2.2.3 `$.capture.overruns` counts gaps between two reads of the microphone that were as long as
the microphone's buffer. A non-zero count means the app may have dropped audio at that point
without any error, whatever `durationMillis` says. `$.capture.lostAtLeastMillis` is a lower bound
on audio that certainly was dropped: time beyond what the buffer could hold that no read
delivered, measured against the phone's clock with 0.1% of the elapsed time allowed for the
microphone's clock running slower. Zero there does not rule out a smaller loss. Both are diagnostics and a consumer MUST
NOT rely on them being present.

2.2.4 `$.windSpans[]` marks audio where the app heard wind rumble (`wind`) or clipping
(`clipping`). Times count recorded audio from the start of the session, so they follow section
2.4, not the wall clock. `strength` runs from 0 to 1. A span is a hint that the speech there may be
spoilt; the audio is unchanged. The key is written, as an empty list when nothing was heard,
whenever the detector ran, so an empty list means the detector found nothing and an absent key
means there is no detector result (an older app, or a recording that ended before its first
segment closed). A span absent from the list does not prove the audio is clean.

2.2.5 `$.promptRun.takes[].route` is `headset` when the audio of the take was routed through a
Bluetooth headset when its prompt opened, and `phone` when it was the phone's own microphone and
loudspeaker. It is fixed at that moment; if the headset drops mid-take, the next take records the
new route. A consumer MUST NOT rely on the key being present.

2.3 `$.state` is `ACTIVE`, `COMPLETE` or `INTERRUPTED`. A consumer MUST NOT treat `ACTIVE` as
final.

2.4 `durationMillis` of 0 means unknown. A consumer SHOULD treat `durationMillis` as the duration
of the audio recorded, and MUST NOT assume that audio time equals wall-clock time since
`startedAt`.

## 3. Stitching a recording

3.1 A consumer MUST order segments by their number, not by sorting names as text.

3.2 A consumer MUST decode each Ogg Opus segment separately and MUST NOT join the encoded files
byte by byte.

3.3 To place a moment in the recording, a consumer SHOULD place segment N at (N - 1) x 60 seconds
and count inside the segment, rather than adding up decoded lengths. The steps are in
[AUDIO-FORMAT.md](AUDIO-FORMAT.md), "Joining a recording".

3.4 A consumer MUST tolerate the last segment of an `INTERRUPTED` session being truncated or not
decodable, by treating the recording as ending at the previous segment.

## 4. Prompt scripts

A prompt script is a JSON file the app imports, saves and reads aloud one prompt at a time. This
section is the contract for anyone writing one. For a ready-made prompt that has an AI agent write
one, see [script-prompt.md](script-prompt.md).

4.1 A prompt script MUST be a JSON document that is valid against
[prompt-script.schema.json](prompt-script.schema.json), written in UTF-8. A byte-order mark is
tolerated.

4.2 A producer MUST give the script a stable `id` and MUST give every prompt an `id` that is
unique in the file and does not change when the text is reworded, because recordings are matched
to prompts by it.

4.3 A producer SHOULD give a prompt a `key` when its answer names a field of the recording (for
example a topic); a `key` MUST be unique in the file.

4.4 The importer MUST refuse, naming every problem with its path: a document that is not strict
JSON (a repeated key, a trailing comma, a comment), a schema violation, a repeated prompt `id`, a
repeated `key`, and a `schemaVersion` higher than the importer knows. It MUST also refuse a title
or prompt text that is empty once spaces are trimmed, and any field holding a control character
(other than a newline or tab) or a text-direction embedding, override or isolate character
(U+202A to U+202E, U+2066 to U+2069).

4.5 Limits: at most 200 prompts; at most 4000 characters of text per prompt; 64 characters per
`id`; 1 MB per file, however it is imported.

4.6 `schemaVersion` is 1 or 2. A change that an older importer cannot read raises it. Version 2
adds the optional `fields` object (4.7); a version 1 script is read exactly as before.

4.7 `fields` (schemaVersion 2 only) holds answers the script already knows: an object whose keys
match `^[a-z][a-z0-9_]*$` (at most 40 characters) and whose values are strings, not empty once spaces are trimmed, of at most
1000 characters; at most 50 fields. A prompt whose `key` names a field is not asked: it has no take
and is not spoken, and the app writes the values to `$.promptRun.fields`. A consumer fills the front
matter field `key` from `$.promptRun.fields.<key>`, and otherwise from the take whose `promptId`
belongs to the prompt with that `key` in the `prompt-script.json` copy. The importer MUST refuse
`fields` in a version 1 script, and a script in which every prompt is answered by `fields`.

## 5. Identity and versions of prompt scripts

5.1 The identity of a saved script is the SHA-256 hash of its file after a byte-order mark is
removed and every line ending is turned into LF. An editor's re-save is therefore not a new
version.

5.2 A script imports as version 1. A file with the same `id` and different content is the next
version, counted per `id`; the count survives deleting a version.

5.3 A file whose content matches any saved version is refused, naming that version.

5.4 A reinstall of the app starts again at version 1, so a consumer MUST treat the hash, not the
version label, as the identity.

## 6. Importing from an address

6.1 Only https addresses are accepted. An address carrying a user name or password is refused.

6.2 A GitHub page address (`github.com/<owner>/<repo>/blob/<ref>/<path>` or `.../raw/...`) is
rewritten to the raw file at `raw.githubusercontent.com`. A gist page is rewritten to its raw
file; a gist with several files gives its first unless the address names the file. Any other
address is used as it is.

6.3 The size limit is counted on bytes after decompression. The content type is not checked.

## 7. Suggestions

7.1 A consumer that matches words to prompts SHOULD use prompt `id`, not position or text.

7.2 A producer of scripts SHOULD keep the first prompts to the questions that file a recording
(what it is about, which project), so a recording can be filed without listening.

7.3 A speech-to-text pipeline SHOULD transcribe each decoded segment, or the joined recording,
and keep the segment number with the text, so a word can be traced back to its audio.

7.4 A pipeline SHOULD read the manifest of the session once its `state` is `COMPLETE` or
`INTERRUPTED`, and SHOULD retry later when it is `ACTIVE` or the manifest is absent.
