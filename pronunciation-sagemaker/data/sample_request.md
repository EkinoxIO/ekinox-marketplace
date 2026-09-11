# Sample request

This is the exact request that produces [`sample_output.json`](sample_output.json)
— the same request Ekinox uses to validate every build of this container, so if
you can reproduce it you have reproduced a known-good call.

## The three parts

| Part | Value |
|---|---|
| `language` | `fr-fr` |
| `expectedText` | `Ceci est test final` |
| `audio` | [`sample_audio.wav`](sample_audio.wav) — a French speaker reading that sentence |

The audio is WAV, 44.1 kHz, 16-bit, stereo, 4.5 seconds, 787,646 bytes.

It is not a clean read: the response marks `est` mispronounced and `final`
omitted, so the sample exercises the fault paths rather than returning a
perfect score.

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

Ceci est test final
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
        "expectedText": "Ceci est test final",
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
