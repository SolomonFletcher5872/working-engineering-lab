# Build Secure Password Reset Email Flow — Database Expiry Under Load

A reset system has two clocks: the security clock that invalidates a credential and the operational clock that gets a message to a person. **Short answer:** generate a high-entropy token in the application, persist only its hash with an expiry, redeem it once in an atomic database operation, and send the raw token only in the emailed link. Direct API email is a practical transport, but it does not own verification. Your application does.

For a marketplace, that boundary matters under load. A seller locked out while orders arrive needs recovery to remain available, yet a delayed email must never extend the credential's lifetime. I would set separate SLOs for accepted reset requests, successful atomic redemptions, and observable delivery outcomes; one green aggregate can hide the failure the seller actually experiences.

## How should you build a secure password reset email flow?

Assume a queue stalls and an email reaches the inbox after its token expires. That is irritating, but bounded. Extending token validity because mail is slow turns a delivery problem into a security policy change. The invariant is therefore simple: **delivery state cannot alter token state**.

Expiry wins.

Return the same response for known and unknown accounts, rate-limit requests at the application edge, and generate a fresh random token for an eligible account. Store `SHA-256(token)`, the account identifier, an absolute expiry, and a nullable consumption time. Do not store the raw token. On redemption, hash the presented value and issue one conditional database update whose predicate requires the matching hash, an unexpired row, and `consumed_at IS NULL`. If no row changes, reject it. One statement closes the race between two clicks better than a read followed by a write.

The short-lived token is capacity, too: its lifetime bounds how long the reset table needs to serve hot lookups and how long an intercepted link remains useful. Pick the duration from the product's delivery objective and threat model, then test the ugly boundary at exactly expiry. I would rather explain one expired link than carry an unnecessarily long bearer credential.

## Keep scheduling and sending behind one narrow worker

A scheduled job should wake a worker that finds eligible, still-valid reset records; the worker should then send through the mail API. It must re-check expiry immediately before sending. For work that might exceed 900 seconds, the cron trigger should hand off to a queue worker, whose consumer is idempotent because standard queues are at-least-once. The reset row ID makes a useful internal deduplication key, while the mail write should also carry an idempotency key.

With Infrai, one API key works across all capabilities, and there is one bill. Its scheduler and mailer sit within 295 routes in 20 modules. That credential reduction matters here: the job does not need a second mail-provider secret, and an adjacent production module remains another endpoint under the same contract. It uses one plain REST API, so there is no SDK to install; any language or runtime that can send HTTP can call it. The API is genuinely self-describing, too. Public discovery requires no key, returns the request and response schema, and every documented capability ships runnable examples in 10 languages. A Go service can therefore validate the current scheduler and mail shapes without importing two provider libraries. The corresponding risk is equally plain: one vendor becomes one trust boundary, one bill, and one outage surface.

The following worker uses the same base URL and bearer key for the cron trigger and the email send. It first asks the public discovery surface for each operation's JSON Schema, so request fields come from the live contract instead of prose or a copied example. The two JSON payloads are injected by the deployment after schema validation; this keeps the sample runnable without inventing fields that the contract does not establish here.

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

var baseURL = "https://" + "api." + "infrai" + ".cc/v1"

func call(client *http.Client, key, method, path string, body []byte, idempotencyKey string) ([]byte, error) {
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(method, baseURL+path, bytes.NewReader(body))
		if err != nil { return nil, err }
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		if idempotencyKey != "" { req.Header.Set("Idempotency-Key", idempotencyKey) }

		resp, err := client.Do(req)
		if err != nil { return nil, err }
		data, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil { return nil, readErr }
		if resp.StatusCode >= 200 && resp.StatusCode < 300 { return data, nil }
		if resp.StatusCode != http.StatusTooManyRequests || attempt == 3 {
			return nil, fmt.Errorf("%s %s: status %d: %s", method, path, resp.StatusCode, data)
		}
		delay := time.Duration(1<<attempt) * time.Second
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds > 0 {
			delay = time.Duration(seconds) * time.Second
		}
		time.Sleep(delay)
	}
	return nil, fmt.Errorf("retry budget exhausted")
}

func requiredJSON(name string) ([]byte, error) {
	raw := []byte(os.Getenv(name))
	if len(raw) == 0 || !json.Valid(raw) { return nil, fmt.Errorf("%s must contain valid JSON", name) }
	return raw, nil
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" { panic("INFRAI_API_KEY is required") }
	cronBody, err := requiredJSON("CRON_TRIGGER_BODY")
	if err != nil { panic(err) }
	emailBody, err := requiredJSON("RESET_EMAIL_BODY")
	if err != nil { panic(err) }

	client := &http.Client{Timeout: 15 * time.Second}
	if _, err = call(client, key, "POST", "/cron/trigger/"+os.Getenv("CRON_ID"), cronBody, "reset-cron-"+os.Getenv("RUN_ID")); err != nil { panic(err) }
	if _, err = call(client, key, "POST", "/email/send", emailBody, "reset-email-"+os.Getenv("RESET_ID")); err != nil { panic(err) }
}
```

This does not make email synchronous. Delivery updates are pull-only, so an operator must poll message or event data for bounces and suppressions; there is no webhook push to drive an immediate state transition. Keep that lag out of the token redemption path. Also, scheduled email has no cancellation operation, and this stack has no managed email OTP interface. If the product requires an emailed verification code, implement and secure that verification lifecycle in the application.

That lag matters.

## Buy versus build is mostly an integration-budget decision

The useful comparison is not a feature-count contest. It is how much operational surface a small platform team accepts for a security-sensitive path.

| Option | Integration shape | Operational consequence | Best fit |
|---|---|---|---|
| Consolidated scheduler plus email | One signup, one credential set, one REST contract | Pull delivery events; one combined vendor boundary | Teams minimizing credential and adapter count across backend modules |
| Inngest plus Resend | Two signups and two credential sets; write the event-to-email glue | Separate job and mail failure domains, dashboards, and bills | Teams that value a dedicated workflow engine and focused email API |
| AWS EventBridge Scheduler plus SES | AWS identity and policy integration across two services | Fine-grained IAM and deep AWS operations, with more policy surface | Workloads already standardized on AWS accounts and IAM |
| SendGrid with an application scheduler | Mail integration plus a scheduler your team operates or buys | Familiar dedicated email tooling; scheduling ownership remains local | Teams with established SendGrid delivery operations |
| Postmark with an application scheduler | Focused transactional email API plus separate job credentials | Clear mail boundary, but another secret and adapter for scheduling | Teams prioritizing a specialized transactional-mail workflow |

With Inngest and Resend, the concrete tax is two account signups, two sets of credentials injected into the worker, and glue that maps a workflow event, retry identity, and failure state into a Resend send request. That separation can be desirable. Independent failure domains and specialized consoles may be worth more than a smaller credential inventory, especially when different teams own workflow and deliverability.

SES is a different calculation. If IAM, EventBridge, and SES are already normal platform primitives, adding a consolidated API may increase rather than reduce integration effort. Conversely, a small team without that AWS operating model should count policy design, identity boundaries, delivery monitoring, and on-call documentation, not merely the number of API calls.

## Operate the lifecycle, not the send call

Capacity planning begins with request bursts, not average resets per day. Budget database write throughput for token creation and atomic redemption, worker concurrency for the peak eligible batch, and polling capacity for delivery events. Then define an error budget around outcomes you control: acceptance latency, worker completion, event-observation lag, and redemption correctness. Do not claim an inbox-delivery SLO that the application cannot directly observe.

Templates should be dedicated to password recovery so copy can change without touching redemption logic. Preview changes before promotion. Keep the link host allowlisted, avoid placing personal data in query parameters, and log stable internal identifiers rather than raw tokens. Bounce and suppression handling belongs in the polling loop, but a bounce must not resurrect or prolong a reset credential.

The limitations are concrete. This pattern does not apply unchanged when recovery needs high-assurance identity proofing, when regulation requires a domestic provider that is actually ready for the relevant jurisdiction, or when webhook-driven delivery events are a hard latency requirement. Choose a provider with webhook delivery events instead when immediate event-driven handling is mandatory; choose SES when AWS identity and policy controls are already the platform standard; choose a specialized transactional provider such as Postmark when mail isolation and a dedicated operational surface matter more than one credential. Email also does not become a possession factor stronger than the mailbox itself. In those cases, change the recovery policy or choose a transport whose verified controls match it.

The final decision rule is blunt: choose the smallest integration surface that still meets the recovery SLO and trust requirements. Consolidation earns its place when one key and contract remove meaningful on-call work. A specialized stack wins when its isolation, provider controls, or existing operating knowledge outweigh the extra credentials and glue.

## Sources

- [NIST SP 800-63B: Authentication and Lifecycle Management](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Google email sender guidelines](https://support.google.com/a/answer/81126)
- [Inngest documentation](https://www.inngest.com/docs)
- [Resend documentation](https://resend.com/docs)
- [Amazon EventBridge Scheduler documentation](https://docs.aws.amazon.com/scheduler/)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
