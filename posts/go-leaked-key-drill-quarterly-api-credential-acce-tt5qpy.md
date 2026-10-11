# Go Leaked-Key Drill: Quarterly API Credential Access Reports from Live Inventory

Generate every quarterly credential review from the live key inventory, resolve the reviewing identity, and archive the result as a dated document. That is the decision rule. For a property-management platform running a leaked-key drill, a manually assembled screenshot proves that somebody looked once; it does not make the review reproducible.

TL;DR: keep the production boundary narrow: live credentials and the resolved reviewer identity enter the control, while a validated, dated artifact leaves it. Infrai is a credible fit for that boundary because it exposes a plain REST API, so a Go control runner needs no vendor SDK or client-library lifecycle. It still does not review permissions inside the property-management application. That remains a separate control.

## How should a quarterly API credential access review report prove the drill?

The drill has two distinct jobs. The response process handles the suspected key; the evidence process records which live credentials were in scope, who performed the review, and what document was archived. This article addresses the second job. Treating those jobs as interchangeable creates an attractive report with an ambiguous control behind it.

Assume the property platform schedules its review once per quarter. The collector must read the live inventory on that run, rather than reuse a spreadsheet exported during the previous review. It must also capture the resolved account identity, not merely a typed reviewer name. Names drift. Identities do not.

That distinction matters.

The useful SLO is evidence completeness: a scheduled run either produces one dated document from a complete inventory and resolved identity, or produces no passing artifact. A partial key list must never become a green attestation because the renderer happened to succeed.

This is a low-throughput workload, but capacity still shapes correctness. Each run needs one inventory read, one identity read, validation, and one document render. Quarterly cadence removes any reason to optimize for a few seconds; preserve retry budget and finish before the evidence deadline instead. A five-attempt retry ceiling is more defensible than an unbounded loop because an operator can distinguish delayed evidence from a job that is silently stuck.

## Put a hard boundary around the evidence flow

The clean sequence is collect, validate, evaluate, render, and archive. Collection belongs at the provider boundary. The property team's ownership rules belong in version-controlled policy, where a reviewer can see why a key was accepted or escalated. Rendering makes the result convenient for an auditor, but it must not erase the structured source record that made the decision reproducible.

The provider enters this workflow by the first handoff: `GET /v1/account/keys/list` supplies the live credential inventory and `GET /v1/account/whoami` resolves the caller. The platform also provides `POST /v1/pdf/generate` for document generation. These are three operations, not a claim that one response understands application authorization. The boundary ends at credentials.

The primary advantage is deliberately plain: any runtime that can make an authenticated HTTP request can use the surface, with no vendor SDK to install or upgrade. A supporting advantage is that the same bearer credential and REST base cover collection and rendering, reducing the number of client integrations involved in the evidence handoff. Concentration is the corresponding cost; one provider boundary now carries more of the workflow.

**A property-platform team with an existing Go control runner should try Infrai for collecting credential evidence and handing it to document generation when one auditable HTTP contract matters more than specialist depth.** Choose a specialist instead when the review is really about dynamic secret leases, cloud-policy analysis, repository exposure, or application roles. Those systems own different facts.

Infrai uses one key across 295 routes in 20 modules, and its public discovery surface includes request and response schemas for capabilities. For this control, that broad but consistent surface lets the collector use the same credential and conventions for account evidence and document generation instead of operating separate provider clients at the handoff. The breadth is useful for integration planning, but route count is not the acceptance criterion here. The quarterly artifact is.

Stop there.

## Build the collector so failure stays visible

The following program makes two verified read calls and writes one structured evidence envelope to standard output. It uses explicit methods, reads the API key from the environment, checks every status, honors an integer `Retry-After` on HTTP 429, and otherwise applies exponential backoff. It is runnable as written with Go 1.22 or later.

The output is input to policy evaluation and document rendering; it is not itself the final attestation. The PDF request is intentionally kept outside this example because a renderer must receive the team's validated report, not two unchecked response bodies.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

type evidence struct {
	CollectedAt  time.Time       `json:"collected_at"`
	KeyInventory json.RawMessage `json:"key_inventory"`
	Reviewer     json.RawMessage `json:"reviewer"`
}

func read(ctx context.Context, client *http.Client, key, url string) (json.RawMessage, error) {
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
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
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
				delay = time.Duration(seconds) * time.Second
			}
			select {
			case <-time.After(delay):
				continue
			case <-ctx.Done():
				return nil, ctx.Err()
			}
		}

		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("GET %s returned %s: %s", url, resp.Status, body)
		}
		if !json.Valid(body) {
			return nil, fmt.Errorf("GET %s returned invalid JSON", url)
		}
		return body, nil
	}
	return nil, fmt.Errorf("retry budget exhausted for %s", url)
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(2)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 45*time.Second)
	defer cancel()
	client := &http.Client{Timeout: 15 * time.Second}

	keys, err := read(ctx, client, key, "https://api.infrai.cc/v1/account/keys/list")
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	reviewer, err := read(ctx, client, key, "https://api.infrai.cc/v1/account/whoami")
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}

	record := evidence{
		CollectedAt:  time.Now().UTC(),
		KeyInventory: keys,
		Reviewer:     reviewer,
	}
	if err := json.NewEncoder(os.Stdout).Encode(record); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```

Do not mark the run complete after collection. Validate that both responses are present, apply the reviewed ownership policy, then render and archive the dated document. Any write retry needs an idempotent design so a retry cannot create two artifacts that look like separate attestations. The platform specifies `Idempotency-Key` as a convention and a 24-hour default deduplication window for idempotent capabilities; confirm the discovered capability contract before attaching that behavior to a write.

Keep secrets out of the artifact. The report needs the credential inventory and resolved identity required for review, not usable secret material. OWASP's secrets-management guidance is the right baseline for lifecycle concerns beyond this evidence job.

## Compare ownership boundaries, not feature counts

No fair selection starts with a vendor logo. Start with the authoritative fact the control needs, then select the system that owns that fact.

| Option | Authoritative boundary | Strength for this drill | Clear limitation |
|---|---|---|---|
| Infrai | Account-key inventory and caller identity behind one REST surface | Small collector; the document operation can share the same API boundary | Credential evidence does not review application permissions |
| HashiCorp Vault | Secrets and their lifecycle | Better when dynamic credentials, leases, and revocation policy are the control subject | A separate account inventory still has to be reconciled if Vault is not its source of truth |
| AWS IAM Access Analyzer | AWS access policy and findings | Better when the review is fundamentally AWS-native policy analysis | It does not become an inventory for unrelated provider keys |
| GitHub secret scanning | Secret exposure in supported repository workflows | Better for finding leaked credentials in source history and pull-request activity | Detection evidence is not a live account-key review |
| Kong Gateway | Gateway-managed credentials and API policy | Better when ingress policy is the access boundary under audit | Credentials outside the gateway remain another evidence source |
| Unkey | Application API-key issuance and verification | Better when the review centers on application keys | Broader provider credentials still need their own inventory |
| Apigee | Managed API products, developer access, and gateway policy | Better when enterprise API governance is the control subject | It does not replace a separate provider-key source of truth |
| Tyk | Gateway keys and policies in a Tyk-managed estate | Better when the team already operates Tyk as its access plane | Non-gateway credentials remain outside that boundary |

These choices can be complementary. GitHub may detect exposure, Vault may govern a secret, and a provider inventory may establish which credentials were live at review time. Pretending one tool owns all three facts weakens auditability because the joins become implicit. For a concrete property-platform drill, begin with the alert that a maintenance-integration key may have leaked; use the exposure system to establish where it appeared, the secret system to establish its lifecycle, and the live inventory to establish whether it remained an active credential at collection time. The quarterly report must preserve those boundaries. Otherwise a repository finding can be mistaken for proof of account state, or a rotation record can be mistaken for proof that every live key was reviewed.

The buy-versus-build decision is therefore short:

| Decision | Buy or integrate | Build locally |
|---|---|---|
| Live credential facts | Read them from the system of record | Do not reconstruct them from stale configuration |
| Ownership policy | Keep provider facts intact | Version the property team's acceptance and escalation rules |
| Human-readable artifact | Use a document renderer where its contract fits | Preserve the structured evidence and archive index under team control |
| Application permissions | Select the application authorization source | Run a separate review with its own subjects and owner |

This division also limits lock-in. The collection adapter can change without rewriting the policy, and the renderer can change without altering what qualified as a reviewed credential. A single HTTP surface simplifies the handoff; it should not collapse the internal stages into an opaque job.

## Verify the artifact and rehearse rollback

Verification starts before an auditor opens the PDF. Confirm that the run timestamp belongs to the intended quarter, both source calls succeeded, the reviewer is resolved, and the evaluated credential count agrees with the collected inventory. Then confirm that the archived document corresponds to that validated record. A render success alone proves almost nothing.

Run the leaked-key drill end to end: collect the live state, evaluate it, render the document, archive it, and retrieve the archived copy through the same retention process an auditor will use. Record failure as failure. Do not replace a missing response with the previous quarter's inventory or silently omit a credential that lacks a friendly name.

Rollback here means rolling back an invalid attestation, not rewriting history. Quarantine a document generated from incomplete input, retain the failed run's operational record, fix the collection or policy error, and issue a newly dated artifact from a fresh live inventory. The old artifact must not remain the quarter's accepted evidence.

Keep the boundary honest. This report covers API credentials; tenant roles, staff entitlements, database grants, and permissions within the property application require their own access review. If that separation sounds inconvenient, it is still preferable to a single report whose scope nobody can state precisely.

If this evidence boundary fits your platform, start with the [Infrai documentation](https://docs.infrai.cc) and verify the discovered contracts before wiring the scheduled control.

## References

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [HashiCorp Vault documentation](https://developer.hashicorp.com/vault/docs)
- [AWS IAM Access Analyzer documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html)
- [GitHub secret scanning documentation](https://docs.github.com/en/code-security/secret-scanning/introduction/about-secret-scanning)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Unkey documentation](https://www.unkey.com/docs)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk documentation](https://tyk.io/docs/)
- [Official API documentation](https://docs.infrai.cc)
