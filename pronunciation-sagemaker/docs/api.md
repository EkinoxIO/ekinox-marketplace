# API reference

The Pronunciation Assessment model package is a SageMaker "bring your own
container" inference image. It exposes three HTTP operations on port 8080.

You never call these endpoints directly — SageMaker does, on your behalf. There
are two ways to reach them:

- [`InvokeEndpoint`](https://docs.aws.amazon.com/sagemaker/latest/APIReference/API_runtime_InvokeEndpoint.html)
  against a real-time endpoint. The request body and `ContentType` you pass are
  forwarded to `POST /invocations` unchanged, and the response body is returned
  to you unchanged.
- [`CreateTransformJob`](https://docs.aws.amazon.com/sagemaker/latest/APIReference/API_CreateTransformJob.html)
  for Batch Transform. Each S3 input object becomes one `POST /invocations`
  body, all of them under the single `ContentType` set on the job, and each
  response is written back to S3 as its own object. See
  [Batch Transform](#batch-transform) below — the shared `ContentType` has a
  consequence you have to plan for.

Either way, this page documents that body on both sides.

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/ping` | Container health check. SageMaker calls this; you do not. |
| `GET` | `/execution-parameters` | Batch Transform settings. SageMaker calls this; you do not. |
| `POST` | `/invocations` | Assess one recording against one expected text. |

There is no API key. Access control is IAM: whoever may call `InvokeEndpoint`
on your endpoint may use the model.

## `POST /invocations`

### Request

`Content-Type: multipart/form-data; boundary=<your boundary>`

Unlike most SageMaker products, this model takes **`multipart/form-data`**, not
JSON or CSV — it has to carry a binary audio file alongside two text fields. You
must generate a boundary, build the body with it, and pass the full
`multipart/form-data; boundary=...` string as `ContentType`. The notebook in
[`../notebooks/`](../notebooks/) does this for you.

Three parts, all required:

| Part | Type | Description |
|---|---|---|
| `language` | text | Language of the recording and the expected text. See below. |
| `audio` | file | The recording to assess, as a WAV file. Must not be empty. |
| `expectedText` | text | What the speaker was supposed to say. Must not be blank. |

Parts may appear in any order. Any part whose name is not one of the three above
is ignored, as is the `Content-Type` of the `audio` part itself.

#### `language` values

| Accepted values | Language | Returned as |
|---|---|---|
| `fr`, `fr-fr`, `fr_fr` | French | `fr` |
| `ar`, `ar-sa`, `ar_sa` | Arabic | `ar` |

Matching ignores case and surrounding whitespace. Any other value is a `400`.
Note that the response always echoes the short form, so sending `fr-fr` returns
`"language": "fr"`.

#### Audio requirements

| | |
|---|---|
| Container | RIFF/WAVE (`.wav`) |
| Encoding | **PCM 16-bit only** — uncompressed, integer |
| Sample rate | Any. Resampled to 16 kHz internally — no benefit to resampling yourself. |
| Channels | Any. Downmixed to mono by averaging channels. |
| Duration | No fixed limit (see the payload cap below) |

Only the encoding is strict. Anything that is not 16-bit integer PCM in a
RIFF/WAVE container — MP3, AAC, Opus, FLAC, 8-bit, 24-bit, or 32-bit float WAV —
is rejected with a `400`. Convert first:

```bash
ffmpeg -i input.m4a -c:a pcm_s16le output.wav
```

In practice your limits are SageMaker's, not the model's, and there are two of
them: `InvokeEndpoint` caps a request at **6 MB** and gives up on a response
after **60 seconds**. The 6 MB works out to roughly three minutes of audio at
16 kHz mono PCM16, or about 35 seconds at 44.1 kHz stereo — but the 60-second
deadline is independent of payload size and can bind first on a long recording.
This model is built for single utterances — a word, a phrase, a sentence — not
for long recordings.

The 6 MB cap applies identically in Batch Transform — that is the
`MaxPayloadInMB` the container reports from
[`/execution-parameters`](#get-execution-parameters), set to match deliberately
so one number covers both modes. The 60-second deadline does not: a transform
job has its own `InvocationsTimeoutInSeconds` instead.

### Response

`200 OK`, `Content-Type: application/json`.

#### Top level

No top-level field is ever null.

| Field | Type | Description |
|---|---|---|
| `language` | string | Canonical language code: `fr` or `ar`. |
| `accepted` | boolean | Whether the utterance passes as an acceptable rendition of `expectedText`. |
| `scorePercent` | number | Overall score. See the note on scoring below. |
| `faults` | string[] | Fault categories detected anywhere in the utterance, e.g. `"substitution"`. |
| `assessmentVersion` | string | Version of the assessment model that produced this result, e.g. `"french-core-v4"`. Record it alongside stored scores — scores are only comparable within one version. |
| `words` | object[] | The alignment, in the order the words occur: one entry per expected word, plus one for each word the speaker inserted. See below. |

#### `words[]`

| Field | Type | Nullable | Description |
|---|---|---|---|
| `id` | string | no | Stable identifier within this response, e.g. `"w0"`, `"w1"` — and `"i9"` for an inserted word, which is not `w`-prefixed. Treat it as opaque rather than parsing the prefix. |
| `referenceWord` | string | **yes** | The expected word. `null` when the speaker produced a word that was not expected (`status: "insertion"`). |
| `producedText` | string | **yes** | What the speaker actually said, as phonemes. `null` when nothing was produced (`status: "omitted"`). |
| `status` | string | no | `correct`, `mispronounced`, `omitted`, or `insertion`. |
| `faults` | string[] | no | Fault categories for this word. |
| `scorePercent` | number | **yes** | Score for this word. `null` only for an `insertion`, which has no reference word to score against. An `omitted` word scores `0` and counts as `0` in the utterance average. |
| `phonemes` | object[] | no | Per-phoneme breakdown. **Empty for an `insertion`**, which has no reference word to break down — guard before averaging over it. |
| `mispronouncedGraphemes` | object[] | no | Character ranges to highlight, relative to this word's `referenceWord`. Empty unless `status` is `mispronounced`, and **not** the union of the phonemes' `graphemes` — see below. |

An `omitted` word still occupies its entry, carrying no produced text, so
omissions never shorten the array. Only an `insertion` changes its length,
adding an entry that has no reference word.

The expected words are the whitespace-separated tokens of `expectedText`, with
attached punctuation kept: the sample's last expected word is `pommes.`, period
included. Split your own text the same way if you need to line the entries up
with it, and remember that grapheme offsets count that punctuation.

#### `words[].phonemes[]`

| Field | Type | Nullable | Description |
|---|---|---|---|
| `referencePhoneme` | string | **yes** | Expected phoneme, in IPA. `null` for an inserted phoneme. |
| `producedPhoneme` | string | **yes** | Phoneme actually produced, in IPA. `null` for an omitted phoneme. |
| `scorePercent` | number | no | Score for this phoneme. |
| `status` | string | no | `correct`, `mispronounced`, `omitted`, or `insertion`. At phoneme level `correct` means *acceptable at this position*, not identical to the reference — see below. |
| `tolerated` | boolean | no | `true` when a **named** tolerance rule forgave this position. The phoneme counts as acceptable and was not produced as written — useful if you want to show a learner the difference without marking it wrong. It keeps `status: "correct"` while scoring low. `false` does **not** mean the speaker matched the reference — see below. |
| `graphemes` | object | **yes** | The letters spelling this phoneme. `null` for an inserted sound, and for spellings the letter-to-sound rules cannot align. A `null` here does not mean the word reports no span — see below. |

#### What a `correct` phoneme means

`status: "correct"` means the phoneme was **acceptable at this position**. It
does not mean `producedPhoneme` equals `referencePhoneme`: the two can differ on
a `correct` phoneme, and the score can be very low when they do.

`tolerated` marks only the subset where a *named* tolerance rule did the
forgiving. It is not the complement of "the speaker matched" — a phoneme can be
`correct`, differ from its reference, score near zero, and still report
`tolerated: false`.

Both cases are in [`../data/sample_output.json`](../data/sample_output.json):

| Word | Reference | Produced | Score | `status` | `tolerated` |
|---|---|---|---|---|---|
| `mangé` | `e` | `ɛ` | 36.2 | `correct` | `true` |
| `petite` | `ə` | `a` | 5.5 | `correct` | `false` |

So do not read "the speaker said the right sound" from `status: "correct"`
alone, and do not colour a phoneme by its score alone. If you want to show a
learner every place their production differed from the reference, compare
`producedPhoneme` against `referencePhoneme` yourself.

#### Grapheme spans

A span is `{ "start": <int>, "end": <int> }` — a half-open character range,
`start` inclusive, `end` exclusive. Every entry of
`words[].mispronouncedGraphemes[]` is one; `words[].phonemes[].graphemes` is
either one or `null`.

Offsets index into **that word's own `referenceWord` string** — not into the
full `expectedText` you submitted. To highlight within the whole sentence, add
the word's own offset in your source text.

Offsets count UTF-16 code units, the same convention as JavaScript string
indices, so `"bonjour".slice(start, end)` in JavaScript or
`referenceWord[start:end]` in Python both give the right substring for text in
the Basic Multilingual Plane — which covers all French and Arabic script.

Per-phoneme `graphemes` spans are independent of one another and **may
overlap**; they are not a partition of the word. In the sample response, `Les`
returns `l` → `[0,1)`, `e` → `[1,3)` and `z` → `[2,3)`, so the character at
index 2 — the `s` carrying the liaison — belongs to two phonemes at once. Treat
each span as its own hint about one sound, and use the word's
`mispronouncedGraphemes` when you need ranges you can paint without
overlapping.

Four behaviours worth coding for:

- `mispronouncedGraphemes` is **empty unless the word's own `status` is
  `mispronounced`**. A `correct` word carries no spans — and neither does an
  `omitted` one, even though nothing in it was said.
- A word's spans are **not** the union of its phonemes' `graphemes`; they mark
  only what went wrong. In the sample response the phonemes of `tarte` cover the
  whole word between them, while the word's `mispronouncedGraphemes` is just
  `{"start": 2, "end": 3}` — the `r` the speaker dropped.
- Nor are a word's spans limited to what its phonemes cover. An inserted sound
  has `graphemes: null` at the phoneme level, yet if it corresponds to a silent
  letter of the reference word, the word still gains a span covering that letter
  — a case this sample happens not to contain. Either way, read the word's
  `mispronouncedGraphemes` for highlighting rather than assembling spans
  yourself.
- A `mispronounced` word can still have an empty `mispronouncedGraphemes`, when
  the fault maps to no letters at all. Fall back to styling the whole word
  rather than assuming there is always something to underline.

### Scoring

`scorePercent` runs from **0 to 100**, higher is better, at every level. It is a
true percentage — render it directly, without rescaling.

| Level | How it is derived |
|---|---|
| Phoneme | How confident the acoustic model was that the expected phoneme was the one produced. `0` when the expected phoneme does not appear among the model's plausible candidates at all. A `tolerated` phoneme scores low by design — the speaker said something else, and a rule forgave it. |
| Word | The average of that word's phoneme scores, and `null` only for an `insertion` — there is no reference word to score it against. An `omitted` word scores `0`. |
| Utterance | The average of the word scores that are not `null`. |

A single badly-missed phoneme drags a short word down sharply. Very low
utterance scores are normal and correct when the speaker said something quite
different from `expectedText` — a completely wrong word scores near zero, not
near 50.

**Do not threshold on `scorePercent` to decide whether a pronunciation passed.**
It is a confidence readout, useful for showing a learner *how close* they were
and for ranking attempts against each other. Whether an attempt is acceptable is
a separate judgement the model has already made for you, accounting for
tolerance rules that a raw score cannot express — read `accepted` for the
utterance and `status` for each word and phoneme.

Scores are also not bit-reproducible. The same recording assessed twice returns
slightly different `scorePercent` values, while the alignment around them —
`accepted`, every `status`, `faults` and every grapheme span — does not change.
Treat a score as a reading with a little noise in it rather than a stable key:
compare statuses when you need to compare two runs, and leave any score-based
assertion room to move.

#### What `accepted` means

`accepted` is `true` when **at least one word aligned to a word of
`expectedText`, and no word was `mispronounced` or `omitted`.**

Note in particular that a word-level `insertion` — the speaker saying something
extra — is reported on its own word entry but never makes the utterance
unacceptable.

`insertion` names three different things: a word `status`, a phoneme `status`,
and a value in `faults`. Only the **word** `status` is read by `accepted`. A
phoneme-level insertion reaches it only indirectly, by turning its own word
`mispronounced` — which does make the utterance unacceptable. The sample
response's last entry is a word-level insertion, and it is not what makes
`accepted` `false`: the omitted `ont` and the three mispronounced words are.

That entry's own `faults` is empty, so `insertion` does not appear in the
top-level `faults` either, even though the speaker said a whole extra word. The
fault vocabulary describes sounds inside an aligned word; an added word is
reported only by its `status`. Detect added words there, not in `faults`.

### Fault categories

`faults` names *what kind* of mistake was made, where `status` only says that
one was. A word's `faults` describes that word; the top-level `faults` is the
distinct union of every word's, so it tells you which kinds of mistake occurred
somewhere in the utterance, not how many or where.

The vocabulary is **language-specific**. French reports general categories:

| Value | Meaning |
|---|---|
| `substitution` | A phoneme was pronounced as a different phoneme. |
| `insertion` | A sound was produced that the reference does not contain. |
| `omission` | A reference phoneme was not produced. |
| `obligatory_liaison` | A liaison that French requires here was not made. |
| `forbidden_liaison` | A liaison was made where French forbids one. |

Arabic reports the specific phonological rule that was broken:

| Value | Meaning |
|---|---|
| `assimilation_al_shams` | The definite article's *lām* was not assimilated to a following sun letter. |
| `close_consonant_confusion` | A consonant was replaced by another that is acoustically close to it. |
| `long_short_vowel_confusion` | Vowel length was wrong — a long vowel shortened, or a short one lengthened. |
| `missing_shadda` | A gemination (*shadda*) was not produced. |
| `extra_shadda` | A gemination was produced where there is none. |
| `ta_marbouta_final` | The final *tāʾ marbūṭa* was realised incorrectly. |
| `tanwin_end` | Word-final *tanwīn* was realised incorrectly. |
| `tashkil_end` | The word-final short vowel (*tashkīl*) was wrong. |
| `tashkil_start_middle` | A short vowel at the start or in the middle of the word was wrong. |
| `wasl_liaison` | A *waṣl* junction between words was not made correctly. |

These are the complete sets for the current model versions. They can grow when a
model version adds a rule, so treat an unrecognised value as an unspecified
fault and show a generic message rather than failing — the response's
`assessmentVersion` tells you which model produced it.

`status` works the other way. `correct`, `mispronounced`, `omitted` and
`insertion` are a closed set, fixed for a given major version of this API and
declared as an enum in [`openapi.yaml`](openapi.yaml), so you may switch on them
exhaustively. Faults are the open vocabulary; statuses are not.

Note that Arabic populates word-level `faults` only for words whose `status` is
`mispronounced`.

### Errors

Errors are returned as `{ "error": "<human-readable message>" }`.

| Status | Cause |
|---|---|
| `400` | The body is `multipart/form-data` but could not be used: the `Content-Type` carries no `boundary`, the body is not parseable against the boundary it does carry, a required part is missing, `audio` is empty or is not decodable PCM16 WAV, `expectedText` is blank, or `language` is not a supported value. |
| `415` | The `Content-Type` is not `multipart/form-data`. There is no JSON or CSV form of the request — the body has to carry binary audio. |
| `503` | The model is not ready to serve. Transient at endpoint start-up; retry with backoff. |
| `500` | Any other failure. |

Note where the line falls between the first two. A `multipart/form-data`
request missing its `boundary` parameter is a `400`, not a `415`: the media type
is the right one, the request is malformed. A `415` means you sent something
that is not multipart at all.

A `400` and a `415` are both final — retrying will not help. The message says
what was wrong: for example `Only RIFF/WAVE files are supported.` or
`Expected PCM16 WAV audio.` See the audio requirements above.

SageMaker maps these onto `InvokeEndpoint` errors: `400` becomes
`ModelError` with `OriginalStatusCode: 400`, and the original JSON body is
available as `OriginalMessage`.

## `GET /ping`

Returns `200 OK` with an empty body once the container is ready to serve.
SageMaker uses it to decide when an endpoint is in service. It reports process
liveness, not model readiness — the first `/invocations` call after start-up may
still return `503` while the model loads.

## `GET /execution-parameters`

`200 OK`, `Content-Type: application/json`.

SageMaker calls this once at the start of a Batch Transform job, to learn the
settings the container expects. You never call it, and you do not need to
configure any of what it reports — this is how sensible values get applied on
your behalf. Real-time endpoints ignore it entirely.

The field names and casing are SageMaker's own, not this product's:

| Field | Value | Meaning |
|---|---|---|
| `MaxConcurrentTransforms` | `3` | Requests SageMaker may have in flight against one instance. |
| `BatchStrategy` | `SINGLE_RECORD` | One record per request. `/invocations` reads a single `multipart/form-data` body and has no notion of several records in one payload, so `MULTI_RECORD` is not offered. |
| `MaxPayloadInMB` | `6` | Largest request body. Matches `InvokeEndpoint`'s real-time ceiling deliberately, so one number covers both modes. |

Values you set on the transform job itself override these. There is no reason
to raise `MaxPayloadInMB` — SageMaker will not deliver more than 6 MB to the
container in any case — and raising `MaxConcurrentTransforms` above `3` puts
more load on an instance than the model is sized for.

## Batch Transform

**Batch Transform is supported.** Real-time is the primary mode — an assessment
is most useful while the speaker is still there — but a transform job is the
right tool for reprocessing a corpus you already hold: re-scoring an archive
after a model update, grading a set of submissions overnight, or evaluating
recordings in bulk without keeping an endpoint warm.

**One utterance per S3 object.** Each input object is one complete
`multipart/form-data` body — exactly the bytes you would pass as the `Body` of
`InvokeEndpoint`, with all three parts in it. There is no format that packs
several utterances into one object.

### The shared boundary

This is the one thing that makes a transform job here different from a typical
one, and the one thing that will fail your job if you get it wrong.

A `multipart/form-data` request carries its boundary in its `Content-Type`, and
the boundary has to match the delimiters inside the body. Batch Transform sets
`ContentType` **once, for the whole job**. So:

> Every input object in a job must be built with the **same** boundary, and that
> same boundary must appear in the job's `ContentType`.

Pick one boundary up front, write every object with it, and pass
`multipart/form-data; boundary=<that boundary>` as the job's `ContentType`. It
must not occur in any of the audio bytes of any object in the job — a long
random ASCII string, generated once and reused, is the simple answer. A library
that generates a fresh boundary per body (including
`requests_toolbelt.MultipartEncoder`, which the notebook uses for real-time) is
the wrong tool here unless you pin its `boundary=` argument to your fixed value.

If the boundary in an object does not match the job's `ContentType`, that object
comes back as a `400` — a per-object failure written to the output, not a
job-level error. A job whose boundary is wrong therefore *runs to completion*
and produces nothing but errors. Check the first output object before waiting on
a large job.

### Job settings

| Setting | Value |
|---|---|
| `ContentType` | `multipart/form-data; boundary=<your fixed boundary>` |
| `SplitType` | `None` — each object is one record, already whole |
| `Accept` | `application/json` |
| `MaxPayloadInMB` | Leave unset. `/execution-parameters` reports `6`. |
| `MaxConcurrentTransforms` | Leave unset. `/execution-parameters` reports `3`. |
| `InvocationsTimeoutInSeconds` | Optional. The 60-second `InvokeEndpoint` deadline does not apply here. |

`BatchStrategy` is reported as `SINGLE_RECORD` and needs no setting; `SplitType:
None` already means one record per object.

Instance types are the same as for real-time: `ml.c5.large` and `ml.m5.large`.

### Output

One output object per input object, `application/json`, named after the input
with `.out` appended. The body is the same `Assessment` document a real-time
call returns — same fields, same `assessmentVersion` — or the same
`{ "error": ... }` shape on failure. Nothing about the response differs between
the two modes.

### When to use real-time instead

If you need the result back while the speaker waits, or you are assessing a
steady stream rather than a fixed corpus, call a real-time endpoint
concurrently. A transform job has minutes of start-up before the first
assessment, which no amount of parallelism inside the job recovers.
