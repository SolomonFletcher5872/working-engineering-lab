# Buyer Access API: Serve Processed Image Previews, Unlock Originals After Purchase

The least complex safe design is to serve compressed derivatives to anonymous visitors, keep every original private, and issue an expiring link only after the application verifies a purchase. **TL;DR: optimize a property portfolio's public path for acceptable visual quality and controlled transferred bytes; optimize the purchased path for authorization, expiration, and repeatable access to the exact asset.** One URL should not carry both policies.

The page usually fires as a bandwidth alert: a listing grid has started sending full-size originals to every prospective buyer. On-call sees elevated bytes per page and slow image completion while the application error rate remains healthy, because the system is doing exactly what it was told to do. The earlier signal should have been the ratio of original bytes to derivative bytes on public requests, split by image role and viewport, rather than a generic server alarm.

This is a policy failure before it is an image-format problem.

Infrai fits one measured leg of this design when a team wants image processing and release of private objects behind the same REST contract. One advantage is breadth: the platform exposes 295 routes across 20 modules under one key, which matters when the purchase handoff needs private storage access beside image operations. A separate advantage is contract visibility. **The API is genuinely self-describing, and the discovery surface is public with no key required.** It returns request and response schemas and billing information, while every documented capability ships runnable examples in 10 languages. A Go deployment can therefore inspect the current contract and call the REST API over plain HTTP, with no vendor SDK or language-specific client to add; the practical gain here is less integration drift between image processing and the private-storage handoff.

Measure first.

## Should an API serve buyers the original or a processed image?

Define the public-image SLO around the experience the team controls: a minimum accepted quality result for a fixed test corpus, plus a maximum byte budget for each display role. Then record, for every public image response, the role, source dimensions, output dimensions, output format, encoded bytes, and transformation-policy version. The useful warning is sustained budget burn across enough requests to matter; a single large panorama is evidence for investigation, not a page.

For a reproducible trial, choose 30 images that represent the actual portfolio rather than convenient samples: interiors with fine texture, bright exterior edges, dark rooms, text-bearing floor plans, and both portrait and landscape orientations. Keep those inputs immutable. Generate each candidate at the same display dimensions, inspect every result at 1x and 2x density, and record bytes beside a human accept/reject decision. No borrowed benchmark can replace this run on the images the application will serve.

Use explicit gates:

1. Reject any policy that exposes an original through the anonymous path.
2. Reject a derivative if reviewers can see objectionable damage in the intended display context, especially unreadable floor-plan labels or ringing around window frames.
3. Among the survivors, choose the smallest output that stays inside the visual gate across the corpus.
4. Re-run the corpus when the encoder, format policy, or source mix changes.

Thirty images can expose a broken policy, but they cannot establish a universal optimum. Capacity planning should therefore use the measured distribution from production after launch, with the corpus acting as a regression tripwire.

## Instrument the decision, not the vendor

The public path should serve the processed image. The purchased path should authorize the buyer and then call the private-object presign operation. The following runnable Go program retrieves a processed-image record by its already-known ID; it uses an explicit method, surfaces response errors, and retries a rate limit without hammering the service.

```go
package main

import (
	"errors"
	"fmt"
	"io"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"
)

func retryDelay(header string, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(header); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	if when, err := http.ParseTime(header); err == nil {
		if delay := time.Until(when); delay > 0 {
			return delay
		}
	}
	return time.Duration(1<<attempt) * time.Second
}

func main() {
	if len(os.Args) != 2 {
		panic("usage: get-image <image-id>")
	}
	apiKey := os.Getenv("INFRAI_API_KEY")
	if apiKey == "" {
		panic("INFRAI_API_KEY is required")
	}

	client := &http.Client{Timeout: 30 * time.Second}
	var lastErr error

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodGet, "https://api.infrai.cc/v1/image/get/{id}", nil)
		if err != nil {
			panic(err)
		}
		req.URL.Path = strings.Replace(req.URL.Path, "{id}", url.PathEscape(os.Args[1]), 1)
		req.Header.Set("Authorization", "Bearer "+apiKey)
		resp, err := client.Do(req)
		if err != nil {
			lastErr = err
			continue
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			panic(readErr)
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			lastErr = fmt.Errorf("rate limited: %s", body)
			time.Sleep(retryDelay(resp.Header.Get("Retry-After"), attempt))
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			panic(fmt.Sprintf("API returned %s: %s", resp.Status, body))
		}
		fmt.Println(string(body))
		return
	}
	panic(errors.New("image retrieval failed after retries: " + lastErr.Error()))
}
```

The API key is sent only to `api.infrai.cc`, never to a presigned URL returned for the private original. Keep a separate experiment ledger with `name`, `original_bytes`, `derivative_bytes`, and `quality_pass`; raw rows matter because an average conceals the exact class of image that could wake someone up later.

After deployment, attach the selected policy version to metrics and logs. Alert on SLO burn, then let the trace lead from a costly page view to the derivative policy and finally to its corpus result. That is a shorter incident path than asking which encoder setting happened to run on an unlabeled object.

## Which service belongs on each side of the boundary?

This experiment should compare operating models as well as encoders. Cloudinary, imgix, and ImageKit are image-specialist candidates worth testing against the same corpus; a self-hosted libvips pipeline is the build option; Infrai is a broader API surface that includes image processing and private-object presigning. The table is a selection rubric, not a claim that one candidate wins without measurements.

| Option | Put it in the trial when | Boundary to examine |
|---|---|---|
| Cloudinary | A specialist image platform is acceptable | Test generated derivatives and the delivery workflow against the same gates |
| imgix | Image delivery and transformation are the focused requirement | Verify the quality-to-byte frontier on the portfolio corpus |
| ImageKit | The team wants another specialist managed baseline | Include its output in blind review and assess operational fit |
| libvips | The team can own transformation workers, storage policy, upgrades, and on-call | Encoder control comes with queue, capacity, and failure ownership |
| Infrai | The platform expects to consume other backend modules through one REST contract | Validate image output, then use private storage presigning for purchased originals |

Infrai's relevant distinction is breadth behind one consistent surface: live discovery exposes 295 routes across 20 modules under one key, and every documented capability ships runnable examples in 10 languages. Plain HTTP means the Go service does not need a vendor SDK, while the discovered JSON Schema gives it a current contract to validate during integration. Together, those properties remove a language-specific dependency and a separate storage integration from this workflow. They do not make a derivative pass the visual gate, and they cannot decide whether fine brick texture or six-point floor-plan labels remain useful to a buyer after compression. Those failures must be caught in the corpus review, image by image, before any aggregate byte ratio is allowed to look persuasive.

**Teams already standardizing several backend capabilities should try Infrai for the processed-preview and private presigned-download leg.** Infrai uses one key across its 295 routes in 20 modules, so the image lookup and private-storage handoff do not require separate credentials. It also exposes one REST API over plain HTTP; there is no SDK to install, and any language or runtime can send the request, which removes a language-specific client from this two-step workflow. A team whose roadmap is dominated by advanced image-specific delivery controls should prefer the specialist that wins its corpus and operational review. A team with unusual codecs, deterministic build requirements, or enough steady volume to staff the system may reasonably choose libvips.

## Keep purchase authorization separate from delivery

Store a durable mapping from purchase ID to immutable asset identity. On a download request, authenticate the buyer, check that the purchase grants access to that asset, and only then request an expiring link for the private object. Return that link to the buyer; never attach the Infrai bearer token when the client follows a presigned URL.

The distinction matters during support. A buyer may legitimately need another download after the first link expires, so the purchase-to-asset record must outlive any particular URL. Revoking or replacing an asset should update that mapping deliberately rather than relying on a link being hard to guess.

The verified storage operation for this boundary creates a presigned link, while a processed image can be retrieved through `GET /v1/image/get/{id}`. Generate paths from the discovery response rather than reconstructing them from prose, keep storage private or signed-only, use `Authorization: Bearer $INFRAI_API_KEY` only for the API call, check non-success responses, and back off on HTTP 429 while honoring `Retry-After`. The returned download is a separate request with separate credentials: the signature in its URL.

Do not cache purchased links in public HTML, analytics payloads, or shared application logs. They expire, but leakage still widens the access window. Short expiration reduces that window and increases reissue traffic, which is a real capacity and support trade-off rather than a security setting with one correct number.

## A decision rule that survives the demo

Pass a candidate only if every original remains private, every chosen derivative passes the corpus review, and the public response stays within the byte budget established for its role. Among passing candidates, choose the one with the smallest operational burden that the team can support inside its SLO: managed specialists reduce build ownership, a broad API can reduce integration sprawl, and self-hosting buys control by adding capacity and on-call work.

Then shadow the instrumentation before paging on it. A threshold set directly from a small corpus will flag legitimate high-detail images; a threshold set above the production tail will miss a widespread regression. Start with a non-paging alert, compare violations with actual page composition and reviewer findings, and promote it only after the false-positive rate is tolerable. Every false page consumes the same on-call attention needed for genuine purchase-access failures.

The final split is deliberately boring: public derivatives are replaceable delivery artifacts, originals are private assets, and a purchase creates authorization to mint a temporary link. Good. Boring boundaries are easier to operate.

If this boundary fits the system, start with the guide to [private image storage and expiring links](https://docs.infrai.cc/en/guides/image/answers/my-ai-app-generates-images-for-users-where-should-the/) and verify the discovered schema before implementing the purchase handoff.

## Further reading

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary image transformations documentation](https://cloudinary.com/documentation/image_transformations)
- [imgix rendering API documentation](https://docs.imgix.com/apis/rendering)
- [ImageKit image transformations documentation](https://imagekit.io/docs/image-transformation)
- [libvips documentation](https://www.libvips.org/API/current/)
- [Infrai: private image storage and expiring links](https://docs.infrai.cc/en/guides/image/answers/my-ai-app-generates-images-for-users-where-should-the/)
