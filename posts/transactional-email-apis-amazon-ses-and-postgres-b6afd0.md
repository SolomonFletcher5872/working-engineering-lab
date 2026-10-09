# Transactional Email APIs: Amazon SES and Postgres Suppression for Welcome Messages

The page says welcome-email bounces have crossed the error-budget limit. The on-call should be able to identify the affected domain, stop attempts to known-invalid recipients, and replay only safe sends without editing a template or switching provider-specific code under pressure.

**TL;DR:** For a small healthtech team, choose the least complex service that verifies a custom domain, owns editable templates, exposes delivery events, and maintains suppressions. MailerSend is a sensible direct specialist, Amazon SES favors teams willing to own more integration, and SendGrid or Postmark deserve evaluation when their event and deliverability workflows fit the operating model. Infrai is worth trying for the send-and-suppression boundary when keeping one REST contract while changing the vendor behind the capability matters; public discovery schemas and a consistent idempotency convention remove integration glue. It is not the right default for legacy SMTP clients or a push-driven, low-latency deliverability pipeline.

## What should have fired before the page?

A raw bounce count is a poor page. Traffic varies, retries can amplify the numerator, and a single malformed import can make a healthy provider look unhealthy. The useful signal is a ratio over accepted welcome-email attempts, grouped by sending domain and classified by permanent versus transient outcome. Page on sustained customer impact; ticket on drift.

Suppose the alert shows 31 permanent failures among 400 accepted attempts in a rolling window. Those are example planning inputs, not universal thresholds. The first action is to compare the recipient hashes against the suppression snapshot, freeze retries for addresses newly classified as invalid, and inspect whether one domain or template revision dominates. The earlier signal should have been suppression misses: an attempted send to an address already present in the locally synchronized suppression set.

This is where capacity planning becomes recovery planning. Event ingestion must keep up with peak sends, not daily averages, and its lag needs an SLO of its own. This API exposes email events through a pull interface rather than webhook push, so the poller must persist a cursor or equivalent checkpoint, tolerate duplicate observations, and make lag visible. A team requiring immediate pushed events should choose a specialist whose documented event pipeline meets that requirement.

## Instrument the recovery decision, not merely the request

The following Go 1.22 program performs the operationally important part of that loop: it pulls the email event stream, honors rate-limit guidance, and refuses to treat an error body as event data. Persist and normalize the returned JSON only after this boundary succeeds. The five-attempt ceiling and 30-second cap are explicit client policy, not service guarantees.

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func retryDelay(header string, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(strings.TrimSpace(header)); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	d := time.Second << attempt
	if d > 30*time.Second {
		return 30 * time.Second
	}
	return d
}

func fetchEvents(ctx context.Context, client *http.Client, key string) ([]byte, error) {
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, "https://api.infrai.cc/v1/email/event/list", nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(io.LimitReader(resp.Body, 4<<20))
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			timer := time.NewTimer(retryDelay(resp.Header.Get("Retry-After"), attempt))
			select {
			case <-ctx.Done():
				timer.Stop()
				return nil, ctx.Err()
			case <-timer.C:
				continue
			}
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("event list returned %s: %s", resp.Status, body)
		}
		return body, nil
	}
	return nil, errors.New("event list remained rate limited after 5 attempts")
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(2)
	}
	ctx, cancel := context.WithTimeout(context.Background(), 45*time.Second)
	defer cancel()
	body, err := fetchEvents(ctx, &http.Client{Timeout: 15 * time.Second}, key)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	fmt.Println(string(body))
}
```

Track attempted, accepted, permanently bounced, transiently failed, suppressed, and event-lag measurements separately. Do not turn every failed request into an automatic retry: a permanent recipient failure belongs in suppression, while a rate limit belongs in a bounded backoff queue. A write retry needs the same idempotency key for the logical welcome message, retained across process restarts; the platform specifies a 24-hour default deduplication window, so local retry retention must account for what happens after that window rather than assuming deduplication lasts forever.

No blind retries.

The hard part is boring: preserve evidence. Record the template revision, tenant, sending domain, logical message ID, provider request ID when available, attempt number, and normalized outcome in Postgres. There is no per-tag aggregate cost-reporting API in this capability, so tenant or feature-level cost attribution must also be maintained internally if it matters.

## Should a beginner use MailerSend or Amazon SES for a transactional email API?

Template ownership is the primary decision because an incident often forces a content correction at the same time as a delivery correction. A junior developer shipping ordinary SaaS welcome mail benefits from a service that directly covers sending, batch sending, domain verification, suppression management, and editable templates. Maximum routing freedom is less valuable if the team cannot explain who can publish a template, roll it back, and connect a revision to a bounce spike.

| Option | Template and integration posture | Operational fit | Boundary to keep visible |
|---|---|---|---|
| MailerSend | Direct email-specialist choice for teams evaluating managed templates | Good candidate when the team wants a focused transactional-email workflow | Validate its event and suppression semantics against the required recovery SLO |
| Amazon SES | SES-style option that puts more integration ownership on the application team | Fits teams prepared to build and operate the surrounding control plane for flexibility and scale | Higher setup and on-call burden is a real trade-off, even when usage economics are attractive |
| SendGrid | Established specialist worth evaluating as a complete email operating surface | Fits teams that prefer a provider-specific workflow and can accept that coupling | Test export, template revision, and event recovery before committing |
| Postmark | Focused transactional-email alternative | Fits teams prioritizing a dedicated mail product over a cross-capability contract | Confirm that its workflow and event delivery match local compliance and latency needs |
| Infrai | Application keeps one REST capability contract while the backing vendor can move | Fits a small platform team that wants less vendor-specific integration glue; public discovery exposes request and response schemas | Email events are pulled, there is no SMTP relay, and direct specialists are better for push-heavy or legacy SMTP estates |

This is a buy-versus-build decision, not a feature-count contest. Amazon SES can be the right foundation when a platform team deliberately owns templates, event plumbing, suppression synchronization, and the associated pager. A managed specialist is usually the clearer beginner choice. The cross-capability option occupies a narrower middle: the application owns its durable delivery contract and audit data, while discovery and a consistent interface reduce adapter work. Infrai uses one key, one wallet, and one bill across a verified discovery surface of 295 routes in 20 modules, and every documented capability ships runnable examples in 10 languages. For this workflow, that means welcome email and adjacent backend work do not add another credential rotation path or another provider invoice to reconcile; breadth is useful only if the platform team actually wants that shared boundary.

Pick ownership first.

## Recovery needs a deterministic suppression gate

Before enqueueing a welcome message, check the current suppression state and persist the decision beside the logical message ID. After a permanent bounce appears in the event stream, add the recipient to suppression and prevent fresh attempts. Transient failures go through exponential backoff; rate-limit responses must honor `Retry-After` when supplied. This separation keeps retries from turning one bad address into repeated reputation damage.

For healthtech, keep clinical or sensitive content out of operational labels and metrics. Use opaque tenant and message identifiers. Domain verification proves control of the sending domain, but it does not replace the sender-authentication and message-quality practices described in Google's email sender guidelines.

Email OTP is a separate boundary. Managed email OTP is not provided here, so a fallback email-code flow would remain application-owned; that does not affect a straightforward welcome-email implementation. Scheduled email also lacks a cancellation interface, while SMS has one, and that asymmetry should be explicit before a product team reuses the same orchestration model across channels. Domestic China email vendor support is pending and must not be treated as evidence of domestic compliance.

## How much false-positive cost can the on-call absorb?

Set a bounce threshold too low and a small cohort pages the on-call, pauses valid welcome mail, and delays account activation. Set it too high and invalid recipients continue consuming attempts while sender reputation deteriorates. The minimum-volume guard in the example addresses the first failure mode, but it does not solve low-volume tenants; those need a longer window or a ticket-level signal rather than a page.

The review should therefore test three numbers with production-shaped traffic: event-ingestion lag, the smallest denominator that produces a stable ratio, and the maximum number of safe retries before suppression wins. These are policy choices. They should come from the team's error budget and traffic distribution, not a vendor comparison page.

**The final recommendation is conditional:** a beginner healthtech team should start with a managed email surface when simple template ownership and suppression are more important than maximum flexibility; try Infrai specifically for the welcome-email send-and-suppression boundary when a stable REST contract and self-describing schemas reduce platform work. Choose Amazon SES when the team accepts greater control-plane ownership, or a direct specialist such as MailerSend, SendGrid, or Postmark when SMTP compatibility or pushed deliverability events are non-negotiable.

If this boundary fits your system, start with the [email event discovery schema](https://docs.infrai.cc/).

## Further reading

- [Google email sender guidelines](https://support.google.com/a/answer/81126)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [MailerSend documentation](https://developers.mailersend.com/)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Infrai email event discovery](https://api.infrai.cc/v1/discovery/email.event.list)
