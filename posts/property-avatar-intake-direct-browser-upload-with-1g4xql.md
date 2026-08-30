# Property Avatar Intake: Direct Browser Upload with Presigned URLs for Private Buckets

Short answer: for a property-management application accepting private avatars, inspection photos, and lease files, issue a short-lived presigned URL from the Next.js server and send the bytes from the browser to a private object-storage bucket. Keep identity, authorization, and upload state in the application; keep the large byte stream out of application workers. Use a proxy or quarantine stage when every file must be transformed or inspected before it becomes available.

The upload endpoint should be small. It authenticates the user, applies a size and media policy, creates a server-owned object key, records an upload session, and returns a signed capability. The browser reports completion separately. A successful transfer does not make an object safe or attach it to a resident record.

## The incident lesson: large files consume the wrong budget

Consider a move-in inspection day. Staff upload photographs, short videos, lease scans, and a resident avatar over hotel or mobile networks while other staff download private documents. A large video sent through a Next.js route competes with login, session creation, and ordinary API traffic for workers, sockets, bandwidth, and timeout budget. A 413 or a client retry can look like an upload problem even when the real issue is web-tier capacity. The failure is easy to misdiagnose because the first visible symptom is often a red browser progress bar, while the pressure is distributed across connection pools, request timeouts, retry queues, and the worker budget that the rest of the property application needs. I would therefore put upload admission, transfer, verification, and download authorization on separate dashboards before changing the storage vendor or increasing web capacity.

I would model this before choosing a storage system: concurrent uploads, the 95th-percentile file size, retry multiplication, session-creation latency, verification latency, and the maximum time a URL remains usable. The upload transfer and the application control plane need separate SLOs. A green session-creation SLO can coexist with a red verification backlog, so one aggregate request metric is not enough.

Direct browser upload removes the large payload from the application path. It does not remove admission control. Cap active sessions per user or property, bound the requested size, expire sessions, and reconcile objects for which the browser never sent its completion request.

Measure it.

Bytes first.

The invariant is easy to miss: throughput is an authorization concern as well as a network concern. A presigned URL is a bearer capability. It should be short-lived, scoped to one generated key and intended operation, absent from analytics and ordinary logs, and never reused as a permanent download link.

## What should React and Next.js teams measure for private object storage uploads?

Treat the browser flow as a state machine rather than a single button. The useful states are session-created, uploading, upload-failed, completion-pending, verified, and rejected. The server should verify the session owner, expiry, object key, observed size, and content policy before changing an object from temporary to usable.

The key must come from the server. A user-supplied filename can be retained as display metadata, but it should not decide the storage path. Store the owner, property scope, declared media type, byte limit, expiry, and status with the session. Completion must be idempotent because a browser can lose the response after the object has arrived.

For capacity planning, separate the time spent obtaining a URL from the time spent moving bytes and the time spent verifying them. Track active sessions, upload duration by size band, retry count, abandoned sessions, completion lag, quarantine depth, and download-link issuance. Alert on a growing completion backlog even when the URL endpoint remains within its SLO; otherwise the dashboard will report a healthy control plane while residents wait for files that are already in storage.

The browser needs the exact HTTP method and headers covered by the signature. CORS belongs in that browser contract, but it is not an access-control substitute: a permissive origin rule does not make a private bucket public, and a restrictive rule does not establish ownership. For downloads, authorize the current user first and issue a fresh short-lived link. Set `Cache-Control` deliberately so private material is not accidentally retained by a shared cache; the correct directive depends on the response and its trust boundary.

## The smallest safe control-plane path

The following Go example keeps policy and signing behind an interface. It deliberately does not put credentials in browser code, and it leaves provider-specific request construction inside the signing adapter. The session route should persist the returned session before responding, then accept an opaque session ID on completion.

```go
package upload

import (
	"context"
	"errors"
	"fmt"
	"time"
)

type Policy struct {
	MaxBytes int64
	Expiry   time.Duration
}

type Session struct {
	ID          string
	OwnerID     string
	ObjectKey   string
	ContentType string
	MaxBytes    int64
	ExpiresAt   time.Time
}

type SignUpload func(context.Context, string, string, time.Duration) (string, error)

func CreateSession(
	ctx context.Context,
	ownerID string,
	contentType string,
	size int64,
	policy Policy,
	sign SignUpload,
) (Session, string, error) {
	if ownerID == "" || contentType == "" || size <= 0 || size > policy.MaxBytes {
		return Session{}, "", errors.New("upload does not satisfy policy")
	}
	if policy.Expiry <= 0 || sign == nil {
		return Session{}, "", errors.New("upload signing is not configured")
	}

	now := time.Now().UTC()
	key := fmt.Sprintf("private/%s/%d", ownerID, now.UnixNano())
	session := Session{
		OwnerID:     ownerID,
		ObjectKey:   key,
		ContentType: contentType,
		MaxBytes:    size,
		ExpiresAt:   now.Add(policy.Expiry),
	}

	url, err := sign(ctx, key, contentType, policy.Expiry)
	if err != nil {
		return Session{}, "", err
	}
	return session, url, nil
}
```

The storage write must be followed by a completion call carrying only the opaque session ID. The server, rather than the browser, decides whether the object belongs to the authenticated user and whether it can be used. Client-supplied content type is a policy input, not proof of file contents; use trusted inspection and malware handling when the threat model requires them. OWASP's file-upload guidance supports allowlisted types, size limits, generated filenames, and authorization checks.

The completion handler should be idempotent. On the first call it can check object metadata, record the observed size, enqueue verification, and transition the session to `completion-pending`; on a repeated call it should return the existing state rather than create a second record or issue an unrelated key. Expired or rejected sessions need a cleanup path, because an object left behind is still storage and still data that may require retention handling.

## Browser upload, proxy, or quarantine?

The right choice follows the largest operational risk, not the shortest demo.

| Path | Throughput and failure shape | Security and team cost | Suitable boundary |
| --- | --- | --- | --- |
| Browser to private object storage | Large payload bypasses application workers; retries still need limits | Requires CORS, bearer-capability handling, completion reconciliation, and verification | Large files when clients can reach storage directly |
| Browser to application proxy | Every byte and retry consumes web capacity | Centralizes inspection, but increases worker, timeout, bandwidth, and egress pressure | Small files or mandatory in-path transformation |
| Browser to quarantine storage, then worker promotion | Transfer is direct; availability waits for inspection | Adds a queue and state transitions, but gives scanning a clear boundary | Sensitive files that must be checked before use |

The catch is that direct upload is not suitable when every byte must pass through an application-controlled transform, when clients cannot reach the storage network, or when a required security control only runs inside the application boundary. Stick with a bounded proxy in the first two cases. Choose quarantine when verification must precede availability. Your mileage may vary: network locality, retention rules, and the size distribution of files can change the decision.

Managed object storage reduces the team's responsibility for durability and placement, while making IAM semantics, egress, lifecycle rules, and provider-specific controls part of the long-term decision. A self-hosted S3-compatible service can provide more control over network paths and placement, while adding responsibility for replication, upgrades, capacity, and incident response. I am not sure one default wins for every property portfolio; the decision should follow the failure budget and compliance boundary.

## The preventative operating method

Instrument the upload session, not the signed URL itself. Record a session identifier, tenant or property scope, declared and verified sizes, state transitions, latency buckets, expiry age, retry count, and rejection reason. Keep object keys and URLs out of normal logs. Hashes, if retained, need a clear purpose and retention policy.

Test the boundaries with a real browser: an allowed origin, a disallowed origin, an expired URL, a wrong method, a changed content type, an oversized payload, a lost completion response, and two completion requests arriving together. In Go, make the signing adapter replaceable in tests, and make the state transition conditional on the stored session version so concurrent callbacks cannot promote the same object twice.

Deploy the upload path with a small canary policy first, then watch transfer latency, abandoned sessions, verification lag, and application-worker saturation. Do not call the design finished because a local avatar uploaded once. The meaningful test is a large-file workload with retries while login, download authorization, and cleanup jobs continue to meet their separate SLOs.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control
- https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
