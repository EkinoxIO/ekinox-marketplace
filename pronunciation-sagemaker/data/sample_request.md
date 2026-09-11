# Sample request

This is the request that produced [`sample_output.json`](sample_output.json) —
the same request Ekinox uses to validate every build of this container. Send it
against your own endpoint and you get the same assessment back: the same
alignment, the same statuses and faults, the same grapheme spans. The scores
come back close rather than identical — see
[What varies between runs](#what-varies-between-runs) below.

## The three parts

| Part | Value |
|---|---|
| `language` | `fr-fr` |
| `expectedText` | `Les enfants ont mangé une petite tarte aux pommes.` |
| `audio` | [`sample_audio.wav`](sample_audio.wav) — a French speaker reading that sentence |

The audio is WAV, 16 kHz, 16-bit, mono, 5.9 seconds, 190,166 bytes.

It is not a clean read: the response marks `ont` as omitted; `petite`, `tarte`
and `aux` as mispronounced; and the extra word at the end as an `insertion`.
The sample exercises the fault paths rather than returning a perfect score.

Note that `language` is sent as `fr-fr` but comes back as `fr`: the response
always echoes the canonical short code.

## On the wire

The boundary is yours to choose. With `ExampleBoundary`, the body looks exactly
like this — CRLF line endings throughout, as `multipart/form-data` requires:

```http
Content-Type: multipart/form-data; boundary=ExampleBoundary

--ExampleBoundary
Content-Disposition: form-data; name="language"

fr-fr
--ExampleBoundary
Content-Disposition: form-data; name="expectedText"

Les enfants ont mangé une petite tarte aux pommes.
--ExampleBoundary
Content-Disposition: form-data; name="audio"; filename="sample_audio.wav"
Content-Type: audio/wav

<the raw bytes of sample_audio.wav>
--ExampleBoundary--
```

Pick any boundary that does not occur in your audio bytes, and build the body
with a library rather than by hand: what matters is that the boundary in your
`ContentType` matches the one in the body. The notebook in
[`../notebooks/`](../notebooks/) uses `requests_toolbelt.MultipartEncoder`,
which handles both.

The `filename` on the audio part and its `Content-Type: audio/wav` are ignored
by the container, but some HTTP libraries will not emit a file part without
them.

In Batch Transform the boundary is chosen once for the whole job rather than per
request: the same boundary has to be used for every input object and repeated in
the job's `ContentType`. See
[`../docs/api.md`](../docs/api.md#batch-transform).

## Reproducing it

Against a deployed endpoint, with the AWS SDK:

```python
import boto3
from requests_toolbelt.multipart.encoder import MultipartEncoder

with open("sample_audio.wav", "rb") as audio_file:
    encoder = MultipartEncoder(fields={
        "language": "fr-fr",
        "expectedText": "Les enfants ont mangé une petite tarte aux pommes.",
        "audio": ("sample_audio.wav", audio_file, "audio/wav"),
    })
    body, content_type = encoder.to_string(), encoder.content_type

response = boto3.client("sagemaker-runtime").invoke_endpoint(
    EndpointName="<your-endpoint-name>",
    ContentType=content_type,   # includes the generated boundary
    Body=body,
)
print(response["Body"].read().decode("utf-8"))
```

Passing a bare `"multipart/form-data"` without the boundary will fail.

### What varies between runs

The model is not bit-reproducible. The same audio and the same `expectedText`,
sent twice, come back with slightly different `scorePercent` values at every
level. Everything else is stable: `accepted`, every word and phoneme `status`,
`faults`, `referenceWord` and `producedText`, the ids, and every grapheme span
are the same on every run.

So compare structure, not digits. A test that asserts an exact score against
this file will be flaky — assert on `accepted`, on the per-word statuses, or on
a score band wide enough to absorb the drift.
