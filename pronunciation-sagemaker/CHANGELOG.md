# Changelog

## 1.0.2

First published version of this documentation. The AWS Marketplace listing is
live and subscribable, and the notebook carries the Model Package ARNs for all
16 regions — the same ones your subscription shows.

**Batch Transform is supported**, alongside real-time endpoints. One utterance
per S3 object, `SplitType: None`. Because SageMaker sets `ContentType` once for
a whole job and the multipart boundary lives inside it, every input object of a
job has to be built with the same boundary — see
[`docs/api.md`](docs/api.md#batch-transform).

The assessment model is `french-core-v4`, the value returned as
`assessmentVersion` in every response.
[`data/sample_output.json`](data/sample_output.json) is generated from a real run
of the container against [`data/sample_audio.wav`](data/sample_audio.wav), never
hand-written, and is re-captured whenever the model pack or the sample recording
changes. The sample is a French speaker reading *"Les enfants ont mangé une
petite tarte aux pommes."* — deliberately an imperfect read, so it exercises the
fault paths: `ont` is omitted, `petite`, `tarte` and `aux` are mispronounced, and
the speaker adds a word the reference does not contain, which comes back as a
word entry with `status: "insertion"`, a null `referenceWord` and the id `i9`.

**Scores are not bit-reproducible.** The same audio and the same `expectedText`,
sent twice, come back with slightly different `scorePercent` values at every
level, while everything around them — `accepted`, every `status`, `faults`, the
ids and every grapheme span — is identical run to run. Compare structure, not
digits, and leave any score-based assertion room to move. See
[`data/sample_request.md`](data/sample_request.md#what-varies-between-runs).

[`docs/openapi.yaml`](docs/openapi.yaml) is generated from the same description
the container is tested against, so it describes what the product does rather
than what it was once documented to do. Worth reading closely on three points.
On insertions: a word-level insertion never makes an utterance unacceptable
while a phoneme-level one can, an inserted word's id is `i`-prefixed rather than
`w`-prefixed, `referenceWord` keeps attached punctuation, and `faults` and
`phonemes` are empty on an insertion entry. On the `words` array: an omitted word
keeps its entry with `producedText: null`, so only insertions change the array's
length. And on `status: "correct"` at phoneme level: it means *acceptable at this
position*, not identical to the reference — a phoneme can be `correct`, differ
from its `referencePhoneme`, score near zero and still report
`tolerated: false`.
