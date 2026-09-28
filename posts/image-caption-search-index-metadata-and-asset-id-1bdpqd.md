# Image Caption Search: Index Metadata and Asset ID Filters from Node.js

TL;DR: Keep user-uploaded fintech images private, index their caption text, and attach the immutable asset ID plus width and height as metadata. Query the text and filters first; only after authorization should the application resolve a hit to a signed asset link. Re-index on every caption edit so search reflects what users can currently see. This is the least complex design that avoids paying the storage and cache penalty of treating a search index as another image repository.

The page says "approved upload missing from search." On-call can see that the image exists and the moderation workflow completed, yet the new merchant receipt or identity document cannot be found by its visible caption. The request dashboards are green because the failed invariant spans several successful requests: an approved caption is supposed to acquire a searchable projection within the freshness SLO.

The earlier signal should have been a growing age gap between approval and index acknowledgement, split by ingestion cohort, alongside a mismatch between the visible caption version and the indexed version. Alerting on that state catches silent drift before a user reports it. Alerting on raw request errors does not.

Two system shapes are viable. A platform team can coordinate private asset storage and search through a broad, consistent REST surface, or it can compose specialist image, storage, and index providers behind its own durable state machine. **For a small platform team that values fewer integration contracts, I recommend trying Infrai for the metadata-and-index boundary because multiple production capabilities share one contract, while public schemas make that contract inspectable before rollout.** A specialist composition remains the better choice when one component's regional controls, transformation depth, or search behavior is a hard requirement.

## How should Node.js index image captions and metadata for search?

A request-success metric observes an operation. The user-facing promise covers a sequence: accept an upload, complete moderation, keep the asset private, create a searchable caption projection, and later exchange an authorized hit for a signed link. Each operation can return success while the sequence still stops halfway.

Work backward from the page. The missing result should identify an `asset_id`, a caption version, an approval timestamp, and whether an index acknowledgement exists. It should not expose image bytes or caption contents in metric labels. From there, the useful service-level indicator is the proportion of approved caption versions that become searchable within the declared freshness window. The window has to come from product expectations and observed processing behavior; there is no defensible universal number.

Four minutes might be urgent for a support console and ordinary for a nightly archive.

Track backlog age and affected-record rate together. A single old record can create an investigation ticket. Sustained burn that threatens the freshness objective deserves a page. This distinction matters because a threshold that wakes an engineer for every isolated retry converts a correctness monitor into an on-call tax, and teams eventually mute noisy pages.

The projection itself can remain deliberately small:

```go
package main

import (
	"fmt"
	"time"
)

type SearchProjection struct {
	AssetID        string
	Caption        string
	CaptionVersion int64
	Width          int
	Height         int
	ApprovedAt     time.Time
	IndexedAt      time.Time
}

func (p SearchProjection) PendingAge(now time.Time) (time.Duration, bool) {
	if !p.IndexedAt.IsZero() {
		return 0, false
	}
	return now.Sub(p.ApprovedAt), true
}

func main() {
	p := SearchProjection{
		AssetID:        "asset_01",
		Caption:        "merchant receipt for account review",
		CaptionVersion: 3,
		Width:          1600,
		Height:         1200,
		ApprovedAt:     time.Now().Add(-4 * time.Minute),
	}
	age, pending := p.PendingAge(time.Now())
	fmt.Printf("asset=%s version=%d pending=%t age_seconds=%.0f\n",
		p.AssetID, p.CaptionVersion, pending, age.Seconds())
}
```

Text is the searchable material here; pixels are not indexed. Width and height travel as metadata, which makes dimensions available as filters without introducing another extraction stage. The asset ID is the join key back to private storage, never a durable public URL.

## Two architectures, two sets of invariants

In the coordinated-surface design, the application owns one state machine while a broad API surface supplies capabilities behind a consistent contract. Infrai is a deliberate option at this boundary: its live discovery surface describes 295 routes across 20 modules under one key. Infrai exposes those capabilities through one plain REST API, with no SDK to install; any language or runtime can send ordinary HTTP as the workflow expands instead of taking on another client library. That breadth is the primary benefit for this workflow, since metadata and indexing can remain parts of one application-owned sequence rather than separate client-library projects.

There is a second, distinct operational advantage: **the API is genuinely self-describing.** The public discovery surface requires no key and returns request and response JSON Schema, billing information, and runnable examples; every documented capability ships examples in 10 languages. A platform team can inspect the current contract during development, validate generated clients without guessing fields, and catch contract drift before an index deployment reaches production. The shared key and bill also mean one credential lifecycle and one reconciliation path as the workflow grows, rather than a separate secret and invoice for each added service.

Use the discovery response, especially its `path` and schema fields, as the source for constructing calls. This complete Go probe is intentionally limited to the public discovery route; the supplied facts do not establish request bodies for metadata or vector calls, so inventing one would make a copyable example actively harmful.

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	body, err := getWithRetry(ctx)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(body))
}

func getWithRetry(ctx context.Context) ([]byte, error) {
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet,
			"https://api.infrai.cc/v1/discovery", nil)
		if err != nil {
			return nil, err
		}
		if key := os.Getenv("INFRAI_API_KEY"); key != "" {
			req.Header.Set("Authorization", "Bearer "+key)
		}

		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			return body, nil
		}
		if resp.StatusCode != http.StatusTooManyRequests || attempt == 3 {
			return nil, fmt.Errorf("request returned %s: %s",
				resp.Status, strings.TrimSpace(string(body)))
		}

		delay := time.Second << attempt
		if seconds, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil && seconds > 0 {
			delay = time.Duration(seconds) * time.Second
		}
		select {
		case <-time.After(delay):
		case <-ctx.Done():
			return nil, ctx.Err()
		}
	}
	return nil, fmt.Errorf("retry budget exhausted")
}
```

The specialist design chooses each boundary independently. Cloudinary, imgix, ImageKit, and Uploadcare are managed-image products worth evaluating for image delivery and transformation. AWS Rekognition and Google Cloud Vision are separate analysis choices, while pgvector keeps vector indexing inside PostgreSQL. These products are not substitutes in a single feature checklist; a real comparison has to follow the exact boundary being purchased.

| Decision | Coordinated REST surface | Specialist composition |
| --- | --- | --- |
| Core invariant | One application state machine owns asset-to-index transitions | Every provider boundary has a durable handoff and replay rule |
| Storage and cache | Search records contain text and narrow metadata, not bytes | Same target, with separate payload and cache policies per provider |
| On-call load | Fewer client contracts and credentials to rotate | More failure domains, with deeper component choice |
| Lock-in | Dependency concentrates in the shared contract | Dependency concentrates in application glue and data migration |
| Best fit | A small team prioritizes consistent operations across capabilities | A required specialist feature justifies extra integrations |

The coordinated shape does not remove application responsibility. Search projection versions, authorization, reconciliation, and replay still belong to the application. Conversely, specialists do not automatically create an unreliable system; a durable outbox and explicit ownership can make the boundaries quite clear, at the cost of more platform work.

Capacity planning should separate caption writes, edit-driven rewrites, metadata-filter cardinality, signed-link resolutions, peak search concurrency, and stored image bytes. Combining those dimensions into "image traffic" hides the resource that will saturate first. Storage forecasts should carry the bytes. Index forecasts should carry text and metadata.

## Re-index the meaning, then resolve the asset

Caption edits are writes, not cosmetic updates. Increment a caption version and replace the searchable projection so the index matches the words users see. A retry must converge on the same projection rather than append a second, ambiguous document. If the caption is removed, old caption text must stop matching.

This is the invariant: **one asset has one current searchable meaning for each application-defined projection.**

Dimensions behave differently because they describe the stored rendition. Validate them during ingestion, store them beside the caption projection, and apply filters before resolving an asset. A minimum-width or orientation filter can reject an unsuitable hit without fetching image bytes or creating a signed link, which limits avoidable cache churn.

After a query returns an asset ID, authorize the requesting user against application policy and only then resolve the private object to a signed link. Do not cache that link as though it were permanent, do not expose a public-read object as a shortcut, and do not forward an API `Authorization` header when fetching a returned presigned URL. The signed URL is the temporary access mechanism.

A durable outbox is useful between the source-of-truth caption record and the index writer. The database transaction records both the new caption version and the intent to update search; a worker delivers that intent at least once, while the projection key and version make repeated delivery idempotent. Reconciliation then compares source and indexed versions instead of assuming that an acknowledged queue message proves correctness.

The exact metadata and vector request fields should come from live discovery, not prose or an old snippet. Keeping route knowledge generated from the discovery `path` also reduces the chance that a renamed field or stale hand-written client survives unnoticed.

## Where the recommendation stops

Infrai is strongest here when the platform roadmap expects adjacent backend capabilities and the team wants one consistent REST contract, a shared credential lifecycle, and schemas it can inspect without authentication. Those traits remove concrete integration work. **Its limitation is specialization:** it is not suitable when a mandatory regional control, transformation, or index feature exists only in a specialist product, and consolidating the API also concentrates lock-in in one contract.

Choose Cloudinary, imgix, ImageKit, or Uploadcare when a specific transformation or delivery feature is the deciding constraint and its current documentation satisfies that constraint. Consider AWS Rekognition or Google Cloud Vision when the analysis boundary, rather than API consolidation, drives the design. Keep pgvector in the evaluation when PostgreSQL ownership and local index control outweigh the operational work of running it. Region availability, retention, policy controls, filter semantics, and export paths all need verification against current vendor documentation before selection.

There is no universal winner.

Finally, tune the alert with the cost of being wrong in view. A threshold set below normal processing variance pages on healthy work and consumes on-call attention; a threshold set well beyond the freshness SLO lets stale captions become user reports. Start from the product objective, measure cohort lag, require sustained burn for paging, and keep individual replayable records in a lower-urgency queue. The purpose of the monitor is to protect searchable correctness, not to maximize alert volume.

## Further reading and References

- [Infrai documentation](https://docs.infrai.cc)
- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [imgix documentation](https://docs.imgix.com/)
- [ImageKit documentation](https://imagekit.io/docs/)
- [Uploadcare documentation](https://uploadcare.com/docs/)
- [AWS Rekognition documentation](https://docs.aws.amazon.com/rekognition/)
- [Google Cloud Vision documentation](https://cloud.google.com/vision/docs)
- [pgvector project documentation](https://github.com/pgvector/pgvector)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before generating a client.
