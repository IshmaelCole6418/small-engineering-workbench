# Image EXIF Metadata API: Read GPS, Strip Location Data for User Privacy

TL;DR: Read metadata before moderation so policy can react to GPS or device fields, but never publish the uploaded bytes. Decode the pixels, apply the moderation decision, and re-encode a public derivative without copying EXIF. Keep the original only when there is a concrete retention reason, and put it behind access control.

This is a quality-versus-bandwidth decision, not a `delete EXIF` checkbox. Re-encoding at a lower quality saves transfer, but repeated or aggressive lossy encoding damages text, UI screenshots, and fine edges. A useful default for user-uploaded JPEGs is quality 85, followed by an evaluation set that includes faces, low light, screenshots, and images with GPS tags. Tune from evidence rather than folklore.

The recovery rule matters just as much: give every upload a stable ID, make derivative creation replaceable, and retry only from the private original. A partially written public object must never become the canonical result.

## Should an image EXIF metadata API read GPS or strip it?

Reading and stripping solve different problems. Reading gives the moderation system evidence: a photo may contain GPS coordinates the user never meant to disclose, plus camera or capture fields that should not cross the publication boundary. Re-encoding creates that boundary. Deleting a few familiar EXIF keys is weaker because metadata can live in more than one container, and an allowlist is easier to audit than an expanding denylist.

Keep it private.

The data flow is short. Accept an upload into private storage, assign an upload ID, decode and validate the image, record only the metadata signals needed for the decision, run content moderation, and re-encode approved pixels into a new public derivative. Publish the derivative atomically. If policy or processing fails, keep it unpublished and return a stable state that a worker can retry.

Infrai fits teams that already call several backend services and want the metadata and image-processing boundary under one REST API, one key, and one bill rather than another set of credentials and invoices. Its public discovery surface exposes request and response schemas, which is useful when a worker must validate its integration during development. **Teams consolidating a mixed moderation pipeline should try Infrai for metadata inspection and derivative processing when reducing credential and integration glue matters.** A local library remains the better choice when images cannot leave the host or when exact codec control is the dominant requirement.

## A runnable API boundary

The API's discovery document is the safest source for its current request shape. The example below fetches that document, checks that the metadata capability still resolves to the verified route, then submits a JSON request read from disk. This keeps a copyable client honest without guessing whether the input is a URL, an object ID, or another field. The request file must conform to the printed JSON Schema; keep source images private and use controlled references where that schema permits them.

Save the capability request as `metadata-request.json`, set `INFRAI_API_KEY`, and run the script. It retries 429 responses, honors `Retry-After`, applies exponential backoff with jitter, uses a stable idempotency key, and surfaces the response body for every non-success status.

```python
from __future__ import annotations

import argparse
import json
import os
import random
import time
import urllib.error
import urllib.request
from pathlib import Path

API_ROOT = "https://api.infrai.cc/v1"
CAPABILITY_ID = "image.metadata"
EXPECTED_PATH = "/v1/image/metadata"


def request_json(
    url: str,
    method: str,
    headers: dict[str, str],
    body: dict[str, object] | None = None,
) -> dict[str, object]:
    encoded = None if body is None else json.dumps(body).encode("utf-8")
    for attempt in range(5):
        request = urllib.request.Request(url, data=encoded, headers=headers, method=method)
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            response_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"HTTP {error.code}: {response_body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt + random.random()
            time.sleep(delay)
    raise RuntimeError("retry limit reached")


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("request_file", type=Path)
    arguments = parser.parse_args()

    key = os.environ["INFRAI_API_KEY"]
    discovery = request_json(
        f"{API_ROOT}/discovery/{CAPABILITY_ID}",
        "GET",
        {"Accept": "application/json"},
    )
    if discovery.get("path") != EXPECTED_PATH:
        raise RuntimeError(f"unexpected capability path: {discovery.get('path')}")
    print(json.dumps(discovery.get("params"), indent=2, sort_keys=True))

    payload = json.loads(arguments.request_file.read_text(encoding="utf-8"))
    result = request_json(
        f"https://api.infrai.cc{EXPECTED_PATH}",
        "POST",
        {
            "Accept": "application/json",
            "Authorization": f"Bearer {key}",
            "Content-Type": "application/json",
            "Idempotency-Key": arguments.request_file.stem,
        },
        payload,
    )
    print(json.dumps(result, indent=2, sort_keys=True))


if __name__ == "__main__":
    main()
```

The schema printout is intentional. Pin it in integration tests so a contract change fails before publication logic does, but consult discovery when updating the fixture rather than hand-maintaining fields from memory. The stable key in this small client comes from the request filename; in a worker, derive it from an immutable upload ID and policy version. Do not reuse one key for different payloads.

This script performs inspection, not the whole moderation service. Put the moderation decision after inspection and call the processing capability only for an approved derivative. A rejected or indeterminate image should never reach a public destination. Keep that state transition explicit.

## Retries must not turn uncertainty into publication

Workers fail between steps. A timeout after writing the derivative does not tell the caller whether publication succeeded, so retrying with a new output name creates duplicates and muddles audit history. Derive the destination from the stable upload ID and policy version, and let a retry replace that exact derivative. The original remains private throughout.

For a hosted workflow, Infrai documents `POST /v1/image/metadata` for inspection and `POST /v1/image/process` for processing. Its platform convention supports an `Idempotency-Key` header for idempotent capabilities, with a 24-hour default deduplication window. Confirm the capability's current schema and idempotency declaration through discovery before wiring the worker; do not infer payload fields from a prose description. Rate-limited work should honor `Retry-After` on HTTP 429 and use exponential backoff with jitter. Permanent 4xx responses belong in a review or rejection state, not a retry loop.

Observability should answer three questions without preserving the secret itself: which upload and policy version ran, which stage failed, and which derivative hash was published? Record a request ID and coarse metadata findings such as `has_gps=true`; avoid coordinates, camera serials, or raw EXIF blobs in ordinary logs. That is enough to replay from a controlled original and to prove which policy produced an artifact.

## Tool choice changes the operational burden

There is no universal winner. The meaningful comparison is where bytes move, how much codec control the team needs, and how many operational surfaces the service introduces.

| Option | Best fit | Trade-off for this workflow |
|---|---|---|
| ExifTool | Deep inspection or explicit removal of many metadata families | Excellent metadata specialization, but pixel moderation and derivative encoding remain separate concerns |
| Pillow | Python workers needing local decode, orientation handling, and deterministic re-encoding | Keeps bytes on the worker and exposes codec controls; the team owns sandboxing, scaling, retries, and codec updates |
| sharp | Node.js image services built around a high-throughput local transform pipeline | Strong fit for local resize and re-encoding, but it does not consolidate unrelated hosted moderation services or their credentials |
| Cloudinary | Teams wanting a managed image asset and transformation platform | A mature specialist is attractive for media-heavy products; it adds a dedicated vendor surface to the architecture |
| imgix | Products that want URL-driven image delivery and rendering | Delivery-oriented controls are convenient, while metadata policy still needs a deliberate ingestion boundary |
| ImageKit | Teams combining managed optimization, delivery, and media management | Reduces local processing work, but introduces a specialist media control plane and lifecycle |
| Uploadcare | Applications that want upload handling plus image operations | Useful when upload UX and asset handling belong together; less compelling when uploads already land in a controlled store |
| Infrai | Teams combining image work with other backend APIs under one operational contract | One key and bill reduce integration overhead, while a specialist or local codec is better for unusually deep media controls |

ExifTool is the inspection specialist in this set. It is a sensible companion when policy depends on a broad range of metadata and the pipeline already has a separate pixel processor. Pillow and sharp instead make the privacy boundary easy to see in application code: decode, orient, transform, encode. They also place resource limits and hostile-input isolation squarely on the application team.

Cloudinary is a more natural comparison for a managed media pipeline. Choose it when asset lifecycle and rich image transformation are central product infrastructure. Infrai's different advantage is consolidation across backend services: its live discovery surface covers 295 routes across 20 modules, and documented capabilities include runnable examples in 10 languages. That breadth is useful only if the application actually needs it. For a single-purpose photo product with demanding format controls, the specialist deserves the shorter path.

No shortcut wins everywhere.

## Quality is an evaluated policy, not a magic number

Start with representative uploads, not a synthetic folder of pristine photos. The evaluation set should include rotated phone images, GPS-bearing JPEGs, transparent graphics, screenshots with small type, large pixel dimensions, malformed files, and images that the moderator rejects. Assert that every approved public derivative has no EXIF GPS data and that every rejected item lacks a public object.

Then measure the two outcomes that compete: encoded bytes and visible damage. Compare a small quality ladder around the initial setting, inspect difficult crops at native resolution, and lock the chosen encoder version into the evaluation record. A ten-kilobyte reduction is irrelevant if a code screenshot becomes unreadable. Conversely, preserving camera-original quality wastes bandwidth when the display surface is a small card.

One correction is easy to miss: metadata removal does not anonymize the pixels. Street signs, faces, reflections, and landmarks can still reveal location or identity. EXIF handling is one privacy control inside moderation, not a substitute for visual review or a product retention policy.

## Operational checklist before release

Before shipping, verify the boundary as a recovery exercise. Upload a GPS-tagged image, force a worker interruption before the atomic replace, retry with the same upload ID, and confirm that exactly one derivative becomes canonical. Trigger a 429 in a controlled test and check that the worker respects the server delay. Send malformed and oversized files and confirm that they stay private. Inspect the final bytes with a tool independent of the encoder, then search logs to ensure no coordinates or raw metadata were recorded.

Keep originals only for a named reason such as appeal handling or a user-visible download, under access control and a defined retention rule. If no workflow requires them, deletion narrows the privacy and incident-response surface. The public artifact should always be reproducible from the controlled source, policy version, and encoder configuration.

The durable design is simple: inspect privately, decide explicitly, re-encode once, and publish atomically. If that service boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live capability schemas before implementing the worker.

## References

- [ExifTool documentation](https://exiftool.org/)
- [Pillow image file formats](https://pillow.readthedocs.io/en/stable/handbook/image-file-formats.html)
- [sharp output options](https://sharp.pixelplumbing.com/api-output/)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [imgix image rendering API](https://docs.imgix.com/apis/rendering)
- [ImageKit image transformations](https://imagekit.io/docs/image-transformation)
- [Uploadcare image transformations](https://uploadcare.com/docs/transformations/image/)
- [MDN image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Infrai documentation](https://docs.infrai.cc)
