# Pronunciation Assessment — AWS Marketplace (SageMaker)

Assess how closely a recording matches the text a speaker was supposed to say,
in **French** and **Arabic**. You get an overall score, a per-word breakdown,
per-phoneme detail in IPA, and character ranges you can use to highlight
mispronunciations directly in your UI.

The model runs entirely inside your own AWS account. It makes no network calls
and reaches no external service — the model and everything it needs are baked
into the container image.

## What's here

| | |
|---|---|
| [`docs/api.md`](docs/api.md) | Field-by-field request and response reference |
| [`docs/openapi.yaml`](docs/openapi.yaml) | The same contract as an OpenAPI 3.0 spec |
| [`notebooks/assess-pronunciation-realtime.ipynb`](notebooks/assess-pronunciation-realtime.ipynb) | Runnable end-to-end example: deploy, invoke, clean up |
| [`notebooks/assess-pronunciation-gradio.ipynb`](notebooks/assess-pronunciation-gradio.ipynb) | Record yourself in the browser and read the assessment in a small web interface |
| [`data/`](data/) | A real sample request and the response it produces |

## Quick start

1. Subscribe to the product on
   [AWS Marketplace](https://aws.amazon.com/marketplace/pp/prodview-uzfcfzqcmds7i)
   and accept the EULA.
2. Open [`notebooks/assess-pronunciation-realtime.ipynb`](notebooks/assess-pronunciation-realtime.ipynb)
   in SageMaker Studio, a notebook instance, or local Jupyter, and run it top to
   bottom. It deploys an endpoint, sends
   [`data/sample_audio.wav`](data/sample_audio.wav), prints the assessment, and
   deletes the endpoint again.
3. To hear how it does on *your* voice, leave the endpoint up — stop before that
   notebook's clean-up section — and run
   [`notebooks/assess-pronunciation-gradio.ipynb`](notebooks/assess-pronunciation-gradio.ipynb).
   It starts a small web interface where you record yourself and see the words
   scored and the mispronounced letters underlined. It reaches your endpoint and
   nothing else: the app is served through your own notebook's HTTPS proxy, so
   the audio never leaves your AWS account. Delete the endpoint when you are
   done.

The notebook already holds the **Model Package ARNs** of the current version for
every supported region, and picks the one matching the session's region — so
there is usually nothing to edit. If your subscription shows a different ARN,
paste it into `MODEL_PACKAGE_ARN_BY_REGION` in section 2. To find it on the
listing: click **Configure**, pick any of the three options — nothing is
launched at that point — and read the ARN at the bottom right.

## How you call it

This model takes **`multipart/form-data`**, not JSON or CSV, because it carries
a binary audio file alongside two text fields. Three parts:

| Part | Example |
|---|---|
| `language` | `fr-fr` |
| `expectedText` | `Ceci est test final` |
| `audio` | a 16-bit PCM WAV file, any sample rate, any channel count |

You must pass a `ContentType` containing the boundary of the body you built —
`multipart/form-data; boundary=...`, not a bare `multipart/form-data`. The
notebook shows the pattern.

```python
import boto3
from requests_toolbelt.multipart.encoder import MultipartEncoder

with open("audio.wav", "rb") as audio_file:
    encoder = MultipartEncoder(fields={
        "language": "fr-fr",
        "expectedText": "Ceci est test final",
        "audio": ("audio.wav", audio_file, "audio/wav"),
    })
    body, content_type = encoder.to_string(), encoder.content_type

response = boto3.client("sagemaker-runtime").invoke_endpoint(
    EndpointName="<your-endpoint-name>",
    ContentType=content_type,   # carries the generated boundary
    Body=body,
)
print(response["Body"].read().decode("utf-8"))
```

You get back JSON: `accepted`, an overall score, `faults`, and a `words` array
with per-phoneme detail. See [`docs/api.md`](docs/api.md) for every field, and
[`data/sample_output.json`](data/sample_output.json) for a complete real
response.

### Reading the score

`scorePercent` is a genuine 0–100 percentage at every level — utterance, word,
and phoneme — and higher is better.

**Do not threshold on it to decide whether an attempt passed.** It is a
confidence readout, good for showing a learner how close they were and for
ranking attempts. Whether an attempt is acceptable has already been decided for
you, accounting for tolerance rules a raw number cannot express: read `accepted`
for the utterance and `status` for each word. `accepted` is true when at least
one word aligned to the expected text and none was mispronounced or omitted — an
extra inserted word never rejects the phrase.

Scores near zero are normal and correct when the speaker said something quite
different from `expectedText`: a completely wrong word scores near 0, not
near 50.

## Supported languages

| Send any of | Language | Response echoes |
|---|---|---|
| `fr`, `fr-fr`, `fr_fr` | French | `fr` |
| `ar`, `ar-sa`, `ar_sa` | Arabic | `ar` |

Matching ignores case and surrounding whitespace, and the response always
echoes the short form — send `fr-fr` and `language` comes back as `fr`.

## Deployment

**Real-time endpoints** are the primary mode — an assessment is most useful
while the speaker is still there. The notebook deploys one.

**Batch Transform is supported too**, for reprocessing a corpus you already
hold. One utterance per S3 object, `SplitType: None`. The one catch: SageMaker
sets `ContentType` once for a whole job and the multipart boundary lives inside
it, so every input object of a job has to be built with the *same* boundary, and
that boundary has to appear in the job's `ContentType`. See
[`docs/api.md`](docs/api.md#batch-transform) for the mechanics.

When you need results while a speaker waits, or you are assessing a steady
stream rather than a fixed corpus, call a real-time endpoint concurrently rather
than running a job.

| | |
|---|---|
| Inference modes | Real-time endpoints and Batch Transform |
| Regions | 16, listed below |
| Instance types | `ml.c5.large`, `ml.m5.large` |
| Network | None required. The notebook creates the model with network isolation on. |
| Accelerator | None. CPU only. |

| Area | Regions |
|---|---|
| US | `us-east-1`, `us-east-2`, `us-west-1`, `us-west-2` |
| Europe | `eu-west-1`, `eu-west-2`, `eu-west-3`, `eu-central-1`, `eu-north-1` |
| Asia Pacific | `ap-south-1`, `ap-northeast-1`, `ap-northeast-2`, `ap-southeast-1`, `ap-southeast-2` |
| Other | `ca-central-1`, `sa-east-1` |

## Versioning

Every response carries `assessmentVersion`, identifying the model that produced
it. Scores are only comparable within one version — store it alongside any score
you keep, so you can tell later whether a change came from the speaker or from a
model update.

## Support

Questions about the product, or about a result that looks wrong: contact Ekinox
through the support channel on the AWS Marketplace listing.

Problems with the sample code or documentation in this repository: open an issue
on [github.com/EkinoxIO/ekinox-marketplace](https://github.com/EkinoxIO/ekinox-marketplace/issues).

## License

The sample code and documentation here are [Apache-2.0](../LICENSE). The product
itself is licensed through its AWS Marketplace listing, under the EULA shown
there.
