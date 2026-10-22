# SMS OTP Delivery Evidence via Polling, Explained (No Webhooks for Seller Login)

Short answer: for a property-management marketplace that needs to let a seller sign in after a new order, bounded polling is enough for a simple OTP flow. Put a resend timer in the browser, keep the verification window on the server, and poll delivery state only for support or an exception path. Do not build the login experience around an event that cannot arrive.

The useful distinction is between proving that a code was accepted and proving that a carrier has delivered the message. The first is an authentication decision. The second is an operational signal, and a pull-based signal arrives late by definition.

Imagine the page an on-call engineer sees at 02:13: the order notification was created, the seller says the login code never arrived, and the dashboard has no incoming delivery event to correlate with the attempt. A green `send accepted` record does not tell us that the handset in the US or EU received anything. A status check can add evidence, but it cannot turn a pull API into a push system.

## What should the login screen do while delivery is uncertain?

Give the seller one clear state: “Code sent. You can request another in 30 seconds.” Keep the countdown independent from the provider response, because tying it to a status poll makes a slow provider look like a broken browser. When the timer ends, offer resend; do not automatically spray another code on every failed poll.

The server should issue a short-lived challenge, store only what it needs to verify it, rate-limit attempts, and invalidate older challenges when the product's security policy calls for that behavior. OWASP's guidance is a useful baseline here: recovery and verification flows need protections against guessing, enumeration, and abuse, and the same discipline belongs in an OTP login.

No signal, no promise.

Measure it.

For the normal path, the browser does not need delivery telemetry at all. The seller enters the code, the server verifies it, and the session is created. Polling belongs in the exception path: a support view, an internal diagnostic request, or a controlled decision about whether a fallback channel should be offered. That keeps a transient carrier delay from becoming a second user-facing failure.

## How should a seller login handle SMS OTP delivery status with polling and no webhooks?

Work backward from the page. The visible symptom is “seller cannot complete login.” The earlier signals are more specific: challenge creation succeeded, the SMS request was accepted, status has not advanced after a bounded interval, and verification attempts remain at zero. Those are four different facts, and collapsing them into one `delivery_failed` flag makes the alert noisy.

I would start with a small state machine: `created`, `submitted`, `delivered`, `expired`, and `verification_succeeded`. The exact provider vocabulary can differ, so the application should map external statuses into its own small set and preserve the raw status for investigation. A status endpoint is evidence, not a promise.

Capacity planning starts with the poll budget, not with the average login. If 200 sellers are simultaneously waiting and the exception path polls once every 10 seconds for 90 seconds, the upper bound is 1,800 status requests for that burst. If every normal login polls at that rate, the same UX decision has become a needless control-plane workload. This is why I would keep normal login client-driven and make server-side polling opt-in, bounded, and observable.

A false positive is expensive in a different way from a missed alert. If the threshold is too short, a normal US or EU carrier delay creates a fallback prompt, a duplicate message, and another support ticket. If it is too long, the seller waits with no explanation. The threshold should be chosen from observed delivery behavior in the countries you serve, then reviewed with the product owner; it should not be copied from a vendor's example.

This is the part I would keep in the runbook: a timeout means “we lack delivery evidence,” not “the seller's handset rejected the message.” That wording prevents an operations dashboard from pretending to know more than its input supports.

## Is polling the right trade-off for a marketplace seller?

For a simple SaaS MVP, yes, with limits. The marketplace can show a resend countdown, cap attempts, and let a seller continue after successful code verification without waiting for a separate delivery confirmation. Cross-channel failover is the harder case: without webhook event push, the application cannot react immediately to a delivery event and switch channels with precise timing.

That trade-off changes if the login flow needs immediate orchestration across SMS and email. The email side here is also pull-based, has no hosted OTP interface, and has no SMTP relay; an email fallback therefore requires an application-owned verification flow and mail setup outside this capability. There is no voice, WhatsApp, or RCS channel in scope. SMS geography and anti-abuse rules, including a country fence or per-country spend circuit breaker, remain application responsibilities.

| Option | Where it fits | Boundary to verify before choosing it |
| --- | --- | --- |
| Twilio Verify | A managed verification product when the team wants a purpose-built verification workflow and broad operational documentation | Confirm the countries, sender policy, fraud controls, and event model required by the login journey; do not assume a delivery event means verification success |
| Amazon Pinpoint / AWS End User Messaging SMS | Teams already operating in AWS that want messaging configuration near their existing cloud controls | The team owns more of the surrounding identity and orchestration logic, and must validate regional registration and delivery reporting for its traffic |
| Bird | A messaging platform worth considering when one workspace spans channels and messaging operations | Check which channels and delivery events are available in the target region, and how fallback state is exposed to the application |
| Infrai | A plain REST surface can be a sensible fit when one key and one bill across backend services matter to a small platform team | Its email and SMS event models are pull-based here, so it is a poor fit for a flow that requires immediate webhook-driven orchestration |

These are different ownership choices, not a league table. A managed verification product can reduce the amount of authentication machinery the platform team owns. An AWS-centered option can fit existing controls but still leaves the application responsible for the login state machine. A multi-channel messaging vendor may widen channel choice while adding another event and regional-policy surface. The one-key, one-bill model is useful for reducing account and invoice sprawl, but it does not remove the need to design the OTP protocol. In a platform roadmap review, I would put the operational boundary beside the feature list: who owns sender registration, who receives a carrier complaint, who decides that a fallback is safe, who stores the audit record, and who is paged when delivery evidence stops changing? Those questions are less glamorous than a channel count, yet they determine the on-call load. A vendor that is excellent for outbound messaging can still be a poor match for authentication if its event timing forces the application to guess.

## A minimal bounded status check

The following Go program polls one SMS status route for a short, explicit window. It is intentionally a diagnostic-shaped client: it reads the key from the environment, checks every response, honors `Retry-After` when present, backs off on 429, and stops instead of turning a slow delivery into an infinite request stream. The route is not a substitute for server-side OTP verification.

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

func main() {
    key := os.Getenv("INFRAI_API_KEY")
    id := os.Getenv("SMS_ID")
    if key == "" || id == "" {
        panic("INFRAI_API_KEY and SMS_ID are required")
    }

    client := &http.Client{Timeout: 8 * time.Second}
    delay := time.Second
    for attempt := 0; attempt < 5; attempt++ {
        ctx, cancel := context.WithTimeout(context.Background(), 8*time.Second)
        url := os.Getenv("STATUS_URL")
        if url == "" {
            panic("STATUS_URL is required")
        }
        req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
        if err != nil {
            cancel()
            panic(err)
        }
        req.Header.Set("Authorization", "Bearer "+key)
        resp, err := client.Do(req)
        if err != nil {
            cancel()
            panic(err)
        }
        body, readErr := io.ReadAll(resp.Body)
        resp.Body.Close()
        cancel()

        if resp.StatusCode == http.StatusTooManyRequests {
            if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds > 0 {
                delay = time.Duration(seconds) * time.Second
            }
            time.Sleep(delay)
            delay *= 2
            continue
        }
        if readErr != nil || resp.StatusCode < 200 || resp.StatusCode >= 300 {
            fmt.Printf("status=%d body=%s\n", resp.StatusCode, body)
            return
        }
        fmt.Printf("status=%s\n", body)
        return
    }
    fmt.Println("status remained unresolved within the polling window")
}
```

In production, make the challenge identifier and any write operation idempotent, keep the retry budget in configuration, and record request IDs without logging the OTP. A client-side countdown is not a security control; server-side expiry, attempt limits, and abuse controls still are.

## The decision rule I would put in the runbook

Choose polling when the product can tolerate delayed delivery evidence and the normal login path is complete after code verification. Choose a webhook-capable verification or messaging design when a delivery event must immediately trigger a fallback, cancel a downstream action, or coordinate several channels.

For this marketplace, I would ship the first version with SMS as the primary challenge, a 30-second resend timer, a bounded diagnostic poll, and an explicit “try again” path. I would not promise real-time delivery orchestration to the seller. The useful SLO is about completed verification and support-visible failure states, with delivery status as a diagnostic dimension.

That leaves a clean escalation point: once the team can measure how often sellers need fallback and how long they wait by region, it can decide whether event push justifies the added service and operational ownership. Until then, polling is a modest mechanism with a known ceiling, which is preferable to a real-time promise the system cannot keep.

## Further reading

- OWASP, “Forgot Password Cheat Sheet”: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- Federal Trade Commission, “CAN-SPAM Act: A Compliance Guide for Business”: https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
- Twilio Verify documentation: https://www.twilio.com/docs/verify
- AWS End User Messaging SMS documentation: https://docs.aws.amazon.com/sms-voice/latest/userguide/what-is-sms-mpp.html
- Bird developer documentation: https://docs.bird.com/
