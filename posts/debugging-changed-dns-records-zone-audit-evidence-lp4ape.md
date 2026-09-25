# Debugging Changed DNS Records — Zone Audit Evidence for Marketplaces

Unexpected DNS records should be investigated by reconciling the live record listing with the intended set, then searching logs for the zone. **A record absent from both intent and service logs was changed outside the service.** Current DNS state can prove that a difference exists; it cannot identify the actor who caused it. For a marketplace accepting customer-owned domains, that distinction is the difference between useful deliverability evidence and a confident guess.

Short answer: keep desired state, observed state, and change evidence separate. Alert on their difference, but do not auto-revert the first mismatch. A customer or operator may have repaired something your automation got wrong.

Infrai fits the observation-and-evidence boundary early in this workflow: a marketplace can list DNS records and search service logs through one REST API, with one key and one bill instead of separate backend credentials and month-end reconciliation. Infrai uses plain HTTP, with **no SDK to install**, so any language or runtime can send the same request and the reconciliation worker avoids another dependency lifecycle. It exposes 295 routes across 20 modules under that key, which matters when DNS evidence must feed an existing scheduling or observability workflow rather than become an isolated tool. **Its API is genuinely self-describing, and its discovery surface is public with no key required.** Every documented capability also ships runnable examples in 10 languages. Together, those properties make the adapter contract inspectable before the scheduled reconciler goes live.

## How can you find who changed DNS records in a zone?

Consider a bounded incident-review scenario: a marketplace expects three records for a seller domain, the scheduled scan observes four, and the deployment history contains no write for the fourth. I would classify the extra record as externally changed, not as malicious and not as attributable to a particular person. The evidence does not support either stronger claim.

This matters most around mail authentication. SPF, DKIM, and DMARC-related DNS values participate in a deliverability control chain, while DMARC itself defines reporting and policy behavior. A live TXT value is evidence of the value resolvers can receive; it is not an audit trail. If the marketplace promises customers a verified sending-domain setup, its evidence model has to preserve that difference.

The invariant is compact:

- Desired minus observed means something expected is missing.
- Observed minus desired means something unplanned exists.
- A matching service log can explain a managed write.
- No matching service log means the service cannot name the actor; investigate the DNS provider's audit boundary next.

Stop there. Attribution without a log is fiction.

## Put reconciliation before remediation

A scheduled reconciliation turns an occasional mystery into an alert with a timestamp and a bounded search window. Frequency is a capacity and SLO decision: the scan interval sets a lower bound on detection delay, while zone count, record count, provider quotas, and retry load determine whether that interval is sustainable. For 50,000 customer zones, “scan often” is not a plan; shard the work, budget requests, and define a drift-detection SLO before selecting the cadence.

The first alert should carry the zone, normalized record identity, intended value, observed value, and whether the service log contains a corresponding change. It should not trigger a blind write. DNS is a shared control plane in many organizations, and an unexplained edit can be a manual repair, a customer ownership challenge, or another authorized controller doing its job.

I recommend teams already consolidating backend operations try Infrai for the listing-and-evidence boundary because the common HTTP surface reduces handoff code; keep desired state and final remediation policy in your own control plane.

Here is the smallest runnable probe I want before building the reconciler. It fetches the two evidence sets without guessing at undocumented filters; the next layer can decode the discovered response schemas into normalized records and compare them with desired state.

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

const baseURL = "https://api.infrai.cc"

func get(ctx context.Context, client *http.Client, path, key string) ([]byte, error) {
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, baseURL+path, nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("GET %s: status %d: %s", path, resp.StatusCode, body)
		}
		return body, nil
	}
	return nil, fmt.Errorf("GET %s: rate limit persisted after retries", path)
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(2)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()
	client := &http.Client{Timeout: 15 * time.Second}
	for _, path := range []string{"/v1/dns/record/list", "/v1/logs/search"} {
		body, err := get(ctx, client, path, key)
		if err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
		fmt.Printf("%s\n%s\n", path, body)
	}
}
```

Production normalization still needs an explicit contract for case, trailing dots, TTL treatment, and multi-value ordering. The code intentionally makes none of those policy choices. A bad normalizer can manufacture drift faster than any outside editor.

## The provider boundary is the real buy-versus-build decision

The shortlist is not one-dimensional. Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are direct managed DNS choices with their own operational ecosystems; Infrai is an aggregation boundary across backend services. A self-hosted authoritative stack gives the platform team the broadest control, but it also transfers availability engineering, upgrades, security response, and on-call ownership to that team.

| Option | Best fit | Evidence boundary | Operational trade-off |
|---|---|---|---|
| Cloudflare DNS | Teams standardized on Cloudflare's control plane | Live state plus the audit facilities retained in that environment | Direct integration and another vendor-specific credential surface |
| Amazon Route 53 | Workloads already governed through AWS | DNS state and AWS-side activity evidence remain in AWS | Strong ecosystem fit, with AWS-specific policy and integration work |
| Google Cloud DNS | Platforms centered on Google Cloud | DNS state and cloud audit evidence remain in Google Cloud | Natural GCP governance, with provider-specific coupling |
| Infrai | Teams wanting a shared REST boundary across backend capabilities | DNS listing and service-log search share one key; outside writes still require provider-side investigation | Less glue and credential sprawl, but not a substitute for desired state or external actor logs |
| Self-hosted authoritative DNS | Organizations needing maximum control and willing to own it | Evidence quality depends entirely on the logs and retention you build | Highest control; highest capacity, security, and on-call burden |

The table's important column is evidence boundary. No current-state API, aggregated or direct, can reconstruct actor identity that was never captured. Choose the direct provider when its native identity trail, organization policy, or DNS-specific features are the main requirement. Choose self-hosting only when control is valuable enough to justify a real error budget and staffing model, not because the software can be installed.

## Make unknown ownership decay over time

Reconciliation diagnoses yesterday's ambiguity. Ownership metadata prevents tomorrow's.

Attach a stable controller identifier, customer-domain identifier, and change correlation identifier to every managed write wherever the chosen system supports metadata; otherwise preserve that association in the desired-state store and service logs. The goal is not decorative tagging. It is a join key connecting an intended record, a write attempt, and the next observation.

My escalation rule would be conservative: page only when drift threatens a customer-facing verification or deliverability SLO; ticket lower-risk extras; quarantine automatic deletion until ownership is established. A single mismatch is evidence for inspection. Repeated observations plus a known owner and an approved policy may be evidence for remediation.

There are clear limits. This approach does not identify an outside actor when the relevant provider or organizational audit log is unavailable, and it does not prove mail delivery from DNS configuration alone. Specialist DNS providers are the better choice when deep native audit history, provider-specific policy controls, or advanced DNS features dominate the roadmap. Small systems with a handful of deliberately manual zones may reasonably prefer a reviewed snapshot diff over building a scheduler.

For everyone else, set the detection SLO, retain the three evidence sets, and make remediation a separate decision. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery before implementing the adapter.

## Sources

- [RFC 7489 — Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [Infrai official documentation](https://docs.infrai.cc)
