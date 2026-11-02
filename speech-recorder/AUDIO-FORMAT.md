# Speech Recorder audio format

How the Speech Recorder app saves audio. This document is read on its own and is the reference
for anything that consumes the audio. The app creates audio exactly as described here; a change
to the app that changes any fact below changes this document in the same change, and a test
fails if the two disagree.

The key words MUST, SHOULD and MAY are used as RFC 2119 defines them.

## Summary

| Property                     | Value                                                       |
| ---------------------------- | ----------------------------------------------------------- |
| Source                       | Phone or headset microphone, mono                           |
| Sample rate                  | 16 000 Hz                                                   |
| Sample width before encoding | 16-bit signed PCM                                           |
| Channels                     | 1                                                           |
| Segment length               | 60 seconds of audio, each segment its own file              |
| Session name                 | `dictation-YYYY-MM-DD-HHmmss`, the phone's local start time |
| Segment file name            | `<session>-NNN.<ext>`, NNN counting from 001                |

## One recording, many files

A recording (called a *session*) is a series of segment files. Every segment except the last
holds exactly 60 seconds of audio. The last holds whatever remained when recording stopped, so
it is 60 seconds or shorter. Segments are cut by counting samples, not by the clock, so a
segment starts at the sample where the previous one ended and nothing is dropped at a boundary.

Audio time is the count of samples recorded. When the phone's audio route changes (a headset
connects or disconnects) the app reopens the microphone in the middle of a segment, and a moment
of real time can be missing from the audio. A consumer MUST NOT assume that audio time equals
wall-clock time since `startedAt`.

Segments are numbered from 001 in the order recorded. The number has three digits, and four
digits from segment 1000, so past that point the names no longer sort correctly as text. A
consumer MUST order segments by the number, not by sorting the names.

## Container and codec

The app uses Ogg Opus when the phone has an Opus encoder for the requested bit rate, and WAV
otherwise. All segments of one session use the same format, named by the manifest's `format`
field. A consumer MUST accept both:

- `ogg-opus`: `.ogg`, `audio/ogg`
- `wav`: `.wav`, `audio/x-wav`

When recording through a Bluetooth headset the phone uses a call-quality link, so the stored
audio is still 16 kHz but its useful bandwidth can be narrower than from the phone's own
microphone.

### Ogg Opus

- One mono Opus stream per file, encoded from 16 kHz input. The bit rate follows the quality
  setting the user chose:
  - Standard: 24 000 bit/s
  - High (default): 48 000 bit/s
  - Maximum: 96 000 bit/s
- Each file is a complete, independent Ogg stream. A consumer MUST decode each file separately
  and join the decoded audio, and MUST NOT join the encoded files byte by byte.
- Opus decoders normally output 48 kHz. A decoder MAY be asked for 16 kHz output.
- A decoder honours the stream's own start-up (pre-skip) and padding information. A decoded
  segment's length can still differ slightly from the manifest's `durationMillis`; how large the
  difference is has not been measured yet. A consumer SHOULD treat `durationMillis` as the
  duration of the audio recorded.

### WAV

- A 44-byte header followed by raw little-endian 16-bit PCM, mono, 16 000 Hz.
- The app rewrites the header, with the length so far, every 5 seconds and when the segment is
  closed. A segment cut short when the app is killed has the header as of the last rewrite, so
  its declared length can be shorter than the file. A consumer SHOULD read to the end of the
  file and MAY ignore the declared length.

## Where the files are

On the phone, in shared storage: `Recordings/SpeechRecorder/<session>/<session>-NNN.<ext>`.

On Google Drive, in the signed-in user's Drive: a folder `Speech Recorder`, holding one folder
per session named `<session>`, holding the segment files under the same names and a file
`manifest.json`.

A segment is removed from the phone only after Drive reports the same MD5 checksum for it, so
the copy on Drive is byte-identical to what the app wrote.

A session run from a prompt script also has `prompt-script.json` in its folder, the script as it
was when the session started (see [SPEC.md](SPEC.md) 1.3.1).

The manifest is uploaded last, once the session has stopped and every segment it lists has
either reached Drive or been marked `missing`. A consumer MUST treat a session folder without
`manifest.json` as incomplete or still being recorded.

## The manifest

`manifest.json` is a JSON object with these keys:

| Key              | Meaning                                                                                                                                                                                                                |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`             | The session name.                                                                                                                                                                                                      |
| `format`         | `ogg-opus` or `wav`.                                                                                                                                                                                                   |
| `startedAt`      | When recording started, in milliseconds since 1970-01-01 UTC, by the phone's clock.                                                                                                                                    |
| `lastProgressAt` | When the app last wrote the manifest while recording, in the same units.                                                                                                                                               |
| `state`          | `ACTIVE`, `COMPLETE` or `INTERRUPTED`, explained below.                                                                                                                                                                |
| `segments`       | The segments in recorded order, each an object with the keys below.                                                                                                                                                    |
| `recordedMillis` | The sum of the segments' `durationMillis`.                                                                                                                                                                             |
| `promptRun`      | Only for a session run from a prompt script: which script, and where each prompt and answer sits in the audio, and the script's own `fields` when it has any. [SPEC.md](SPEC.md) section 2 lists its keys.             |
| `capture`        | How steadily the app read the microphone, and a lower bound on audio it dropped. Absent from older recordings. [SPEC.md](SPEC.md) section 2.2.3 explains it.                                                           |
| `windSpans`      | Stretches where wind rumble or clipping was heard, with start, end and strength. Empty when none; absent when the detector did not run. [SPEC.md](SPEC.md) 2.2.4. A take's `route` (`headset` or `phone`) is in 2.2.5. |

Each entry of `segments` has:

| Key              | Meaning                                                                                         |
| ---------------- | ----------------------------------------------------------------------------------------------- |
| `name`           | The segment's file name.                                                                        |
| `durationMillis` | The duration of the audio recorded, in milliseconds. 0 means unknown.                           |
| `bytes`          | The size of the file on Drive. Absent when the segment is `missing`.                            |
| `md5`            | The MD5 checksum of the file on Drive, in lower-case hex. Absent when the segment is `missing`. |
| `missing`        | `true` when the segment was no longer on the phone when the upload ran, and so is not on Drive. |
| `samples`        | How many samples the segment holds. Only for a session run from a prompt script.                |

The states are:

- `ACTIVE`: recording, or it was when the app last wrote the manifest.
- `COMPLETE`: stopped by the user, or after a failure the app reported, with every segment closed.
- `INTERRUPTED`: found active with no fresh segment, because the app was killed or hung.

## Interrupted recordings

If the app is killed or hangs, the session's last segment may exist on the phone and Drive but
not in the manifest the app last wrote. The uploaded manifest then lists it with
`durationMillis` 0, and its `state` is `INTERRUPTED`. At most one segment per session is in this
state, the one being recorded.

An Ogg Opus file is complete only once the app has closed it, so such a segment can be
truncated or not decodable at all. A consumer SHOULD try to decode it, and MUST tolerate a
failure by treating the recording as ending at the previous segment. A WAV segment plays up to
its last header rewrite, as described above.

## Joining a recording

1. Order the segments by number.
2. Decode each to 16-bit PCM at one sample rate.
3. Concatenate the decoded audio in order.

The result is the recording. No overlap, gap or crossfade is added between segments. Because
decoded lengths can drift from the recorded ones, a consumer that needs a position in the
recording SHOULD place segment N at (N - 1) x 60 seconds and count inside the segment from
there, rather than adding up decoded lengths.
