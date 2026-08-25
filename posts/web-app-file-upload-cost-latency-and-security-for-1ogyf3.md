# Web App File Upload: Cost, Latency, and Security for Signed-URL Relays

A browser file upload should use a short-lived presigned URL when the application can express its security rules as an upload intent and can accept the file asynchronously; use a proxy backend only when a synchronous decision over the bytes is a real product or regulatory requirement. The direct route usually removes a costly latency and capacity hop from the web app, but it does not remove the backend's responsibility for authorization, acceptance, cleanup, or an intelligible user state.

The important distinction is between issuing permission to transfer and accepting an object into the product. Treating those as one event is how a tidy demo becomes an ambiguous production system.

## Should a beginner use a presigned URL or proxy backend for web app file upload?

A presigned URL is a temporary, scoped capability. In a direct-to-storage design, the browser asks the application for an upload intent, the application authenticates the caller and picks an object key, and storage receives the bytes. The application then learns from trusted object metadata or a completion signal that an object exists, validates it against the intent, and changes its business record to accepted or rejected. The URL is not evidence that the file arrived, and arrival is not evidence that the file is safe to serve.

That separation often feels like extra machinery to a beginner because the proxy version hides it inside one request. The proxy receives the stream, can inspect or transform it before forwarding it, and can return one immediate result. It also makes the application fleet responsible for every byte in both directions, slow-client behavior, buffering, cancellation, retries, and enough headroom to keep ordinary API requests inside their SLO while transfers are active. A relay is a legitimate boundary; it is not a free simplification.

| Decision | Signed browser upload | Application relay | Direct upload with quarantine |
| --- | --- | --- | --- |
| Data path | Browser to object storage | Browser through the backend | Browser to a private landing area |
| Inline byte inspection | Only constraints enforced by the storage request | Available before forwarding | Deferred to a validator |
| Backend capacity | Intent and status traffic | Intent traffic plus all file bytes | Intent traffic plus asynchronous workers |
| Acceptance model | Eventual | Can be immediate | Eventual, explicit pending state |
| Useful when | Ordinary objects fit a policy envelope | Inline transformation or inspection is mandatory | Untrusted content needs post-upload checks |

The catch is product semantics. A presigned URL is not suitable when a user action cannot enter a `pending` state, when a policy forbids a browser-to-storage connection, or when the application must decide from the complete byte stream before anything is stored. Stick with the proxy in those cases, and budget for it as a data-plane service. Conversely, a proxy is a poor default when it merely forwards bytes without making a byte-level decision.

## The production failure is usually an unowned state transition

The incident worth planning for is bounded and ordinary: a client receives authorization, begins an upload, loses its connection, retries, and later a completion notification arrives twice. No single request is necessarily broken. The system still needs to answer whether the object exists, which intent owns it, whether it was validated, and when the abandoned material may be removed. Consider what happens if the first client transfers the bytes successfully but never receives its response, then opens a second browser tab and asks for another intent. If the product marks both intents as complete just because each tab reports success, it can expose duplicate assets. If it marks neither complete until a notification arrives, the user sees an indefinite spinner when the observer is delayed. Neither outcome is solved by changing the URL format. The decision needs durable identifiers, authoritative observations, and a timeout whose owner has permission to resolve or clean up the state.

Make that answer a state machine rather than a collection of optimistic assumptions. Create a durable intent before issuing a capability. Put its unpredictable identifier in the server-generated object key. On completion, read authoritative metadata, compare the expected size and media type with the intent, and make the acceptance transition idempotent. A duplicate signal should find the same accepted state, not produce a second record or a new user-visible asset. A reconciliation job should find old intents with no object and objects with no live intent; lifecycle rules can remove aged material, but they cannot repair an application database.

This is the part teams omit when they estimate the direct path as "one endpoint." It is a distributed operation with multiple observers. The useful SLO is not merely successful intent creation. Track the age of the oldest pending intent, time from object observation to validation, rejected-byte volume, and the count of orphaned objects found by reconciliation. A healthy signing endpoint tells an operator very little if accepted uploads are stalled behind a validator.

Keep the failure injection concrete: cancel a browser halfway through, delay the completion observer, deliver the same event twice, and submit a size that does not match the recorded intent. The expected result in every case is an observable state and a cleanup path.

Short tests. Big payoff.

The same discipline applies to security. The signing endpoint must authorize the caller before it grants a capability, generate the destination key rather than accepting a user-supplied path, keep the capability lifetime bounded by the transfer expectation, and avoid recording it in logs or analytics. CORS decides which browser origins can send a request; it does not establish tenant authorization. Declared content type is a request attribute, not proof of file content. For untrusted material, a private quarantine area plus asynchronous scanning and validation makes the trust boundary explicit, provided the user experience can accurately say that the upload is pending.

## How should a web app set cost and latency budgets for file upload?

Start with the byte rate, not the diagram. Let `R` be peak accepted bytes per second, then add a retry factor derived from client behavior and headroom for deploys and recovery. A proxy needs to sustain that traffic on ingress and egress while retaining its API latency objective. It must also bound concurrent streams: a small number of slow clients can exhaust file descriptors, memory, or worker slots before aggregate bandwidth looks alarming.

Direct uploads move the bulk data path away from the application, but they introduce different work: issuing intents, observing completions, running validators, retaining quarantine data, and reconciling state. Cost follows those responsibilities. Compare measured transferred bytes, request counts, retained bytes, worker time, and operational coverage under the same workload rather than asserting a percentage saving from a diagram. Provider billing dimensions and contract terms vary, so a current internal calculator is more useful than a generic claim.

The following handler owns the control-plane boundary. It does not assume a particular storage implementation: the signer can bind the constraints supported by the chosen storage service, while the application owns authorization, object naming, and the durable intent. The resulting response says `pending` because issuing a URL is not acceptance.

```go
package upload

import (
	"context"
	"encoding/json"
	"net/http"
	"time"
)

type Signer interface {
	SignPut(ctx context.Context, key, mediaType string, size int64, expires time.Time) (string, error)
}

type IntentStore interface {
	Create(ctx context.Context, userID, uploadID, key, mediaType string, size int64) error
}

type request struct {
	UploadID  string `json:"upload_id"`
	MediaType string `json:"media_type"`
	Size      int64  `json:"size"`
}

func IntentHandler(signer Signer, intents IntentStore, userID func(*http.Request) string) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		var in request
		if err := json.NewDecoder(http.MaxBytesReader(w, r.Body, 4096)).Decode(&in); err != nil {
			http.Error(w, "invalid request", http.StatusBadRequest)
			return
		}
		if in.UploadID == "" || in.MediaType == "" || in.Size <= 0 {
			http.Error(w, "invalid upload intent", http.StatusBadRequest)
			return
		}
		uid := userID(r)
		if uid == "" {
			http.Error(w, "unauthorized", http.StatusUnauthorized)
			return
		}

		key := "quarantine/" + uid + "/" + in.UploadID
		if err := intents.Create(r.Context(), uid, in.UploadID, key, in.MediaType, in.Size); err != nil {
			http.Error(w, "intent unavailable", http.StatusConflict)
			return
		}
		url, err := signer.SignPut(r.Context(), key, in.MediaType, in.Size, time.Now().Add(10*time.Minute))
		if err != nil {
			http.Error(w, "cannot issue upload", http.StatusServiceUnavailable)
			return
		}
		w.Header().Set("Content-Type", "application/json")
		_ = json.NewEncoder(w).Encode(map[string]string{"upload_url": url, "state": "pending"})
	}
}
```

Ten minutes is only an example. Set expiration, maximum size, concurrency, and validation capacity from the observed file-size distribution and an explicit acceptance SLO. Load testing should use large objects and interrupted clients, not only successful small transfers. The capacity review is complete when the chosen path has a bounded queue, a cleanup owner, and a clear degradation behavior for the user.

## Keep the decision reversible

The architecture can change without rewriting the product contract if both paths preserve the same intent identifiers, acceptance states, and audit record. Put storage-specific signing and metadata reads behind a small interface, preserve idempotency at the application boundary, and test the state transitions independently of the browser UI. This also makes a future buy-versus-build review less emotional: the team can compare on-call load, policy requirements, and measured data-plane demand instead of defending an earlier implementation.

There is no universal winner. Choose the direct route when the storage policy envelope and eventual acceptance state meet the requirement. Choose the relay when synchronous byte-level control is indispensable. Don't call the upload complete until the system can explain where the object is, who owns it, and what will happen next.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- https://cloud.google.com/storage/docs
