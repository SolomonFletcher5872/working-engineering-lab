# Store Generated Images & Thumbnails: Object Storage, Signed URLs, Lifecycle Cleanup

**Short answer:** store generated images and thumbnails in object storage with class-and-owner keys, issue signed URLs from the authorization layer, and use lifecycle cleanup plus a bounded orphan sweep; that combination usually beats optimizing for a tiny per-gigabyte rate because it controls egress, retention, and pager load.

The cheapest design is the one that makes deletion and cache behavior boring. Put an explicit retention class and owner in every object key, issue signed URLs only at the authorization boundary, and make lifecycle rules a backstop rather than your entire cleanup system. That keeps storage, egress, and on-call work visible during capacity planning.

## The incident that changed my key layout

I once reviewed a thumbnail migration after the deploy had already gone green. Original uploads and derivatives were readable in internal tests, but a browser cohort received `403 SignatureDoesNotMatch`. The signer had used a region copied from an old deployment template; the URL looked valid, while the storage service quite correctly rejected its credential scope. The same review found a second, quieter error: the lifecycle rule watched `thumbs/`, while the writer emitted `thumb/`.

One character. Five months of retained derivatives.

The invariant is simple: a bucket can match prefixes and tags, but it cannot infer parentage, ownership, or whether a thumbnail is still referenced. Those meanings must be encoded when the object is created. A later migration over millions of keys is an operational event, not housekeeping.

## What should a Node.js image pipeline decide before the first upload?

Start with a small object contract. A temporary model result, a regenerable thumbnail, and a user-retained original have different retention and access policies. Give each class a stable prefix, put the account or erasure subject next, and make the source asset an explicit path component. For example: `eph/account/asset/variant`, `der/account/asset/variant`, and `dur/account/asset/original`.

Signed URLs belong in the application authorization path. The handler checks access, signs a GET, and returns a redirect; the Node.js process does not stream image bytes. A short, unique URL on every request also defeats a CDN cache, because the query string becomes part of the cache key. Round thumbnail expiry to a shared window when the exposure is acceptable, while keeping original downloads on a shorter, user-specific TTL.

Content disposition is another part of that contract. Thumbnails normally need `inline`; an explicit download should use `attachment` and a safe filename. For user-supplied non-ASCII names, emit the UTF-8 `filename*` form described by MDN rather than trusting a raw `filename=` parameter.

```go
package assets

import (
	"fmt"
	"net/url"
	"path"
	"time"
)

type Class string

const (
	Ephemeral Class = "eph"
	Derived   Class = "der"
	Durable   Class = "dur"
)

func Key(class Class, owner, asset, variant string) string {
	return path.Join(string(class), owner, asset, variant)
}

type Presigner interface {
	PresignGet(key string, expires time.Time, query url.Values) (string, error)
}

func ThumbnailURL(p Presigner, key string, now time.Time) (string, error) {
	// A shared boundary gives identical thumbnails a reusable edge cache key.
	expires := now.Truncate(10 * time.Minute).Add(20 * time.Minute)
	q := url.Values{"response-content-disposition": {"inline"}}
	return p.PresignGet(key, expires, q)
}

func DownloadURL(p Presigner, key, filename string, now time.Time) (string, error) {
	q := url.Values{"response-content-disposition": {
		fmt.Sprintf("attachment; filename=%q", filename),
	}}
	return p.PresignGet(key, now.Add(60*time.Second), q)
}
```

## How do signed URLs, lifecycle cleanup, and object storage fit together?

Treat lifecycle policies as predictable garbage collection. They are good at age-based rules such as deleting `eph/` objects after a short retention period or `der/` objects after their regeneration window. They are not relationship-aware: a newly created derivative can be orphaned when its parent row disappears, and its age will not make it eligible.

Run a bounded reconciliation sweep for that case. List one class prefix, ask the application index which keys are still live, delete only the orphan set, and cap the number of deletions per run. A cap turns a bad index response into a delayed cleanup rather than an availability incident. Emit deleted-object counts, bytes reclaimed, and oldest unprocessed age; a zero count for a week deserves an alert because it often means the writer and rule have drifted apart.

The sweep needs the same engineering discipline as a data migration, even though it is tempting to call it a cron job and move on. Page through storage deterministically, persist a cursor only after a page has been checked, and make deletion idempotent so a retry after an interrupted run cannot widen the blast radius. Separate the discovery phase from the destructive phase: first produce a candidate set with the key, age, class, and index result, then enforce a per-run count and byte budget before issuing deletes. The audit record should include the rule version and the deletion reason, because an operator investigating a deletion report needs to distinguish an age-expiry action from a parent-missing action without reconstructing yesterday's code. Test this against a bucket fixture containing an expired preview, a live thumbnail, an orphan thumbnail, and an object whose owner is undergoing erasure. Then test it again with an empty index result. Don't let a scheduler turn an ambiguous dependency response into a bulk delete. These checks are not glamorous, but they are what keep cleanup from becoming the highest-risk writer in the system.

The erasure path must cover the original, every derivative, and cached copies. GDPR Article 17 makes deletion a data-flow obligation, not merely a database mutation. Owner-first keys let an erasure worker scan a bounded prefix and issue a matching CDN purge. Record completion per class so a retry can resume without guessing what was already removed.

## Buy, build, or transform: which trade-off survives the pager?

| Approach | Operational owner | Main cost driver | Cleanup responsibility | Lock-in pressure |
| --- | --- | --- | --- | --- |
| Managed object storage plus CDN | Provider for durability; your team for policy | Requests, egress, and retained bytes | Lifecycle rules plus reconciliation | Lower when using a standard object API |
| Self-hosted S3-compatible cluster | Your team, including disks and replication | Hardware, redundancy, and on-call time | Entirely yours | Lower API dependence, higher operational burden |
| On-demand transformation service | Split between provider and your team | Transform calls and delivery | Fewer stored derivatives, but provider-specific metadata | Higher URL and transform coupling |

The comparison is not a price leaderboard. Estimate peak PUT and GET rates, derivative count, average bytes, egress locality, recovery objectives, and the engineer-hours needed to operate each option. A nominally cheap bucket is a poor choice if its retention model cannot meet your SLO or its export path is untested.

Public marketing images do not need signed URLs; immutable, content-addressed objects can be cached directly. Conversely, regulated records with mandated multi-year retention should not inherit an automatic age rule. Keep the original when policy requires it and delete only derived material.

The catch is that reconciliation depends on a reliable parent relationship. If a pipeline names outputs only by a model hash, the sweep must join a full listing to an index, which is slower and costs more. Fix the naming contract before object counts grow. I'm not sure there is a clean shortcut once that context has been discarded.

This design also does not suit teams that cannot operate an erasure worker, metrics, and periodic restore tests. In that case, choose a managed workflow with explicit deletion guarantees and accept the provider's boundaries; the right decision is the one your SLO and pager rotation can actually sustain.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
- https://gdpr-info.eu/art-17-gdpr/
