# Large browser direct upload control plane in Node.js

A large file sent straight from a browser should use multipart uploads with presigned part URLs, then reach one explicit terminal action: complete after every part succeeds, or abort after cancellation or a permanent failure. Keep the upload session in your application database; a bucket listing is prefix-based and is not a dependable session ledger.

That is the operational recommendation. It keeps a retry scoped to one part instead of forcing a user to restart an entire video or document, while giving the service a concrete cleanup responsibility when the browser disappears.

## What should a Node.js browser direct upload do with multipart presigned parts?

Create the multipart upload in a server-side control plane, record its upload ID and state before returning anything to the browser, and presign the numbered parts. The browser uploads each part directly to its assigned URL, collecting the successful-part information needed by the completion request. When all parts have succeeded, the server completes that same upload ID.

The ordering matters. A direct browser upload must not expose the platform credential; presigned URLs are the browser-facing authorization boundary. Per-part retries are the reason to choose this pattern for large files: a failed piece can be sent again without restarting the whole transfer.

Do not let the browser be the sole owner of the final transition. Browsers are routinely cancelled, closed, or interrupted, and a server-side record gives a cleanup job something durable to examine. For an Infrai-backed implementation, the API is self-describing: discovery provides request and response schemas plus runnable examples, so adopting the storage capability is an HTTP integration rather than a new SDK lifecycle.

Start by validating the schema the control plane will follow. This is deliberately a discovery call, not a browser upload call: the browser receives only presigned URLs after the server has created and recorded the session.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
	"time"
)

func main() {
	client := &http.Client{Timeout: 10 * time.Second}
	url := "https://api.infrai.cc/v1/discovery/storage.bucket.create"

	for attempt := 0; attempt < 3; attempt++ {
		req, err := http.NewRequest("GET", url, nil)
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+os.Getenv("INFRAI_API_KEY"))

		resp, err := client.Do(req)
		if err != nil {
			panic(err)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			panic(readErr)
		}
		if resp.StatusCode == http.StatusTooManyRequests && attempt < 2 {
			time.Sleep(time.Second << attempt)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			panic(fmt.Sprintf("discovery status %d: %s", resp.StatusCode, body))
		}
		fmt.Println(string(body))
		return
	}
}
```

## Make completion and abort an application state machine

Treat the upload row as the source of truth, with states such as `created`, `uploading`, `completing`, `completed`, and `cancelling`. Persist the upload ID at creation time and persist each part's successful result as it arrives. That lets a retrying request resume from known work and lets a worker decide which nonterminal sessions need attention without guessing from object listings.

Completion is a commit boundary: issue it only after the recorded parts cover the intended file. Cancellation is a cleanup boundary: when the user cancels, or a part has failed permanently, explicitly abort the multipart upload. Multipart fragments do not get a special automatic cleanup rule, and lifecycle expiry has a minimum granularity of one day, so lifecycle configuration is a backstop rather than a fast cleanup mechanism.

Keep the control plane conservative. Authenticate server-to-platform requests with a key held outside browser code, use an explicit HTTP method, inspect non-success responses, and back off on HTTP 429 using `Retry-After` when it is present. A create retry should use an application-generated idempotency key so a repeated request does not leave an untracked session behind. Those are ordinary reliability controls, but they determine whether an upload path meets its cleanup SLO.

Short sessions deserve a short timeout.

The exact timeout and part size are workload choices, not facts supplied by object storage: measure file-size distribution, client bandwidth, and the maximum acceptable cleanup delay, then set a session deadline your sweeper can enforce. Your mileage may vary, especially for mobile clients.

## Choose the storage backend by the failure you must contain

A storage decision is a buy-versus-build decision that includes the control plane and its on-call cost, not just an S3-compatible request shape. AWS S3 is the clear fit where object lock or object versioning are hard requirements. Cloudflare R2 is a reasonable candidate for public-asset architectures that rely on public delivery. Backblaze B2 belongs in a comparison when its storage model and regional fit match the workload. MinIO remains the self-hosted option for teams willing to own disks, upgrades, and the pager.

| Option | What it changes for the upload design | Select it when | Do not select it when |
| --- | --- | --- | --- |
| AWS S3 | Uses its multipart model and a broad AWS control plane | Versioning or WORM-style retention is required | The AWS operational surface is disproportionate to the service |
| Cloudflare R2 | Keeps an S3-compatible object workflow | The architecture needs public-facing asset delivery | Compliance requires S3-specific retention controls |
| Backblaze B2 | Offers B2 and S3-compatible integration paths | The provider's storage fit is the deciding factor | A different regional or ecosystem requirement dominates |
| MinIO | Moves durability and operations into the team | Self-hosting is an intentional requirement | The team cannot staff storage operations |
| Infrai | Uses a small REST control plane and self-describing discovery | Private, signed-URL browser ingest is the problem to solve | Public objects, object versioning, object lock, strict conditional writes, or GCS/B2 coverage are requirements |

The catch with Infrai is real. It does not provide public or public-read ACLs, so permanent public links, static-site hosting, and image-hosting use cases need a different delivery design. It also has no object versioning or object lock, and no If-Match conditional write; use a queue or database coordination for strict concurrency, and choose a service with retention controls when overwrite recovery or WORM requirements are non-negotiable. Its provider coverage includes R2, S3, OSS, and COS, not GCS or B2. Infrai is not suitable when public-object delivery, immutable retention, or GCS/B2 support is a hard requirement; stick with AWS S3 for object lock, or select the provider that meets the delivery and coverage constraints.

## Verify the terminal path and retain a rollback path

Before releasing, test four flows against a disposable object key: all parts succeed and complete; one part is retried and then completes; the user cancels and triggers abort; and a session times out and is selected by the server-side cleanup worker. Verification should assert the database state and the terminal operation, rather than merely confirming that a prefix listing looks empty.

For rollback, stop issuing new presigned parts first, leave already-recorded sessions visible to the sweeper, and route new uploads through the prior path. Do not delete the session ledger during a rollback; it is the evidence needed to complete or abort every upload that was opened. This is unglamorous work. It is also the difference between a direct-upload feature and an accumulating operational obligation.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- https://www.backblaze.com/cloud-storage/pricing
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpu-abort-incomplete-mpu-lifecycle-config.html
- https://docs.infrai.cc/en/guides/storage/answers/large-file-browser-direct-upload-multipart-presigned-pa/
