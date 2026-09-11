# Changelog

## 1.0.0

First public release. The AWS Marketplace listing is live and subscribable.

**Batch Transform is now supported**, alongside real-time endpoints. One
utterance per S3 object, `SplitType: None`. Because SageMaker sets `ContentType`
once for a whole job and the multipart boundary lives inside it, every input
object of a job has to be built with the same boundary — see
[`docs/api.md`](docs/api.md#batch-transform).

The Model Package ARNs changed with this release. The notebook carries the 1.0.0
ARNs for all 16 regions; if you copied a pre-release ARN, replace it with the one
your subscription now shows.

Documents assessment model `french-core-v4` — the value returned as
`assessmentVersion` in every response, and the version
[`data/sample_output.json`](data/sample_output.json) was captured from. That
sample is generated from a real run of the container against
[`data/sample_audio.wav`](data/sample_audio.wav), never hand-written. Re-capture
it whenever the model pack changes.
