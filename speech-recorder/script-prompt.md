# Writing a Speech Recorder prompt script

This file is written for an AI agent. Point your agent at it (give it the address of this file)
and say what you want to record; it should hand back one JSON file to import into the app.

## What you are making

A prompt script is a JSON file. The Speech Recorder app reads its prompts aloud one at a time;
the person answers each by speaking, and the app records the answers. The agent's whole job is
the file, so work out with the person first:

1. What the recording is for, and who will speak.
2. The questions to ask, in order. Ask a few at a time, and keep each one short enough to be
   understood when heard once, not read.
3. Whether any answer is already known (a project name, a topic). Put those in `fields` and the
   app does not ask them.

## The rules the app enforces

The app refuses a file that breaks any of these and lists every problem, so check each one.

- Strict JSON in UTF-8: no comments, no trailing commas, no repeated keys.
- Top level keys, and only these: `schemaVersion`, `id`, `title`, `prompts`, and optionally
  `description`, `language`, `fields`.
- `schemaVersion` is `1`, or `2` if the file has `fields`. `fields` is refused in version 1.
- `id` (script and every prompt): lower-case letters and digits in groups joined by single
  hyphens, for example `walk-notes`; at most 64 characters. Prompt ids are unique in the file.
  A prompt id never changes once recordings exist, even if the wording does, because recordings
  are matched to prompts by it.
- `title`: 1 to 120 characters, shown in the app's list.
- `description`: up to 500 characters, for the people who read the file. Optional.
- `language`: a language tag such as `en-GB` for the voice that reads the prompts. Optional.
- `prompts`: 1 to 200 items. Each has `id` and `text` (1 to 4000 characters, read aloud), and
  optionally `key` and `hint`. No other keys.
- `key`: the name of the field a tool files the answer under, lower-case letters, digits and
  underscores, starting with a letter, at most 40 characters. Unique in the file. Use it for
  answers that name something, such as a topic or a project.
- `hint`: up to 500 characters, a note for people. Never read aloud.
- `fields` (version 2): an object of known answers. Keys follow the same pattern as `key`; values
  are strings of at most 1000 characters that are not blank; at most 50. A prompt whose `key`
  matches a field is not asked, and at least one prompt must have no matching field. `fields` is
  optional in version 2.
- `language` is 2 or 3 letters, then optional parts joined by hyphens (`en`, `en-GB`), at most 35
  characters.
- No empty or blank titles or texts, and no control characters other than newline and tab. The
  text-direction characters U+202A to U+202E and U+2066 to U+2069 are refused too.
- The file is at most 1 MB.
- When you revise a script, keep its `id` and the prompt ids. The same `id` with new content is
  imported as the next version.

The complete contract is [SPEC.md](SPEC.md) section 4, and the machine-readable form is
[prompt-script.schema.json](prompt-script.schema.json). Validate the file against the schema
before handing it over. Examples that pass: `examples/prompt-script/valid-minimal.json`,
`valid-full.json` and `valid-fields.json`. The files named `invalid-...json` in the same folder
show what is refused. Each is at the same path under this file's folder.

## Writing good prompts

- Put the filing questions first (what is it about, which project), so a recording can be filed
  without listening to it.
- One question per prompt. Write the way a person would say it, with no lists, headings, URLs or
  symbols, which are awkward to hear.
- Spell out anything a speech voice may misread, such as abbreviations.
- Give each prompt an `id` that describes what it asks (`first-point`), not its position
  (`q3`), so reordering does not rename it.

## A complete example

```json
{
  "schemaVersion": 2,
  "id": "earlier-chapters-walk",
  "title": "Earlier chapters walk",
  "description": "Script-level fields answer the filing questions, so only the content is asked.",
  "language": "en-GB",
  "fields": {
    "topic": "Earlier chapters",
    "project": "book"
  },
  "prompts": [
    { "id": "topic", "key": "topic", "text": "What is this recording about?" },
    { "id": "project", "key": "project", "text": "Which project does it belong to?" },
    { "id": "first-point", "text": "Tell me the first point you want to make." }
  ]
}
```

## Handing it over

Give the person the file. They put it on the phone, or somewhere with a public https address,
and use Import in the app's Scripts screen. The same content imported again is recognised and not
saved twice; changed content under the same `id` becomes the next version.
