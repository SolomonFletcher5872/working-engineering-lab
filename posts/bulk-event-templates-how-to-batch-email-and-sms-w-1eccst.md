# Bulk Event Templates: How to Batch Email and SMS with Status Polling

The page should identify students waiting for signup verification links, the template revision attached to those requests, and the age of the oldest eligible message. For an edtech platform, start with an application-owned template catalog and a Postgres worker that checks recipient preferences and suppression before sending email and SMS batches. This is a proposed operating design, not a production incident report.

**Short answer:** Use batch sends for bulk event notifications, persist recipient-level intent in Postgres, and reconcile delivery through paginated email events and SMS status polling. Keep template revisions in the application so that a provider change does not change the verification-link contract or silently rewrite content already waiting in the queue.

Own the template.

Swapping the vendor behind a capability should leave application code using the same contract, while provider mappings remain configuration. Infrai offers a common REST API across account lookup, email, and SMS under one key; public, self-describing discovery schemas also let maintainers check the handoff without installing another SDK. The trade-off is explicit: one vendor to trust, one bill, and one outage surface.

## 1. What should page when a bulk event notification system batches email and SMS?

Imagine a course enrollment opening: students request verification links, support sees unfinished accounts, and the delivery dashboard shows accepted sends. That is an illustrative failure mode. Acceptance leaves a gap between handing off a message and a student completing verification; an incorrect application link can make a technically successful send operationally useless.

Start with a proposed SLO: 99% of eligible signup intents should leave the local delivery queue within 60 seconds over a rolling 30-day window. These are design targets, not measured performance or provider guarantees. Track verification completion separately because opening a message is outside the worker's control, and record suppression as an explicit outcome rather than letting those recipients disappear from accounting. The delivery SLO should measure dispatch, while the product dashboard measures completion.

Work backward from the page. The earlier signal is increasing oldest-eligible-intent age by channel and template revision; polling lag needs a separate signal because delayed reconciliation can make healthy dispatch look stalled. Record the template revision when creating the intent.

Do not log tokens.

Capacity planning starts before batch-size tuning. At an illustrative 120 signup intents per second, 60 seconds of stalled dispatch creates 7,200 pending intents before retries. That is arithmetic, not a benchmark. Send concurrency must fit the transport quota and the database connection budget while leaving capacity for status reconciliation; batch limits come from current schemas, not a guessed constant in a tutorial.

## 2. Buy transport, decide who owns the verification template

The application catalog should hold a logical purpose such as student-signup-verification, a revision, a channel, allowed variables, and a provider mapping. Persist expiry with the intent and check it again before dispatch. A retry carrying an expired link remains a failed signup experience even if the transport accepts it.

| Buy-versus-build option | What the team gets | Template ownership and limitation |
| --- | --- | --- |
| Application catalog plus managed transport | Transport is bought; revisions, eligibility, and reconciliation are built | Reviewable content history, but the team operates the ledger and worker. |
| Clerk + Resend + Twilio | Separate account, email, and SMS product boundaries | Three signups and three sets of credentials; the team builds account-to-message mapping and consistent suppression decisions. |
| Postmark plus a separate SMS service | A transactional-email option to evaluate using its published guidance | Suitable for an email-centered evaluation; cross-channel fallback remains application policy. |
| Self-managed dispatch and templates | Direct control of worker releases and backlog handling | The team also owns capacity planning, retry behavior, and on-call coverage. |

Clerk occupies the account boundary, Resend the email boundary, and Twilio the SMS boundary in the alternative stack. Those separate relationships can suit a platform that wants independent provider selection. Postmark's transactional-email guidance supplies another evaluation baseline. None of these choices decides who may change the verification link or which content revision a queued retry should use.

Managed template editing can suit teams that prioritize an editorial workflow outside application releases. Application-owned templates instead make link changes reviewable with the signup code, at the cost of maintaining the catalog and deployment process. A content editor should not need queue expertise, but changing the verification destination deserves review. Consider a proposed release in which the course team changes the wording while the account team changes the verification-link destination: if queued intents resolve an unversioned template name at send time, an old intent can acquire new content without either team seeing a new signup request. Pinning the revision makes that interaction explicit. Keep the approved destination policy with the renderer, validate that the required link variable exists before committing an intent, and reject a missing variable before it enters a batch. A rollback then changes the revision assigned to new intents while leaving already-created work intelligible to support. The trade-off is additional catalog and release ownership; the benefit is being able to explain exactly what a student was meant to receive.

Choose ownership first.

## 3. Wire the account output into the batch input

The following Go program uses the same key and base URL for account lookup and email batch delivery. It extracts the account email using a configured JSON pointer, then replaces exactly one STUDENT_EMAIL string in a deployment-supplied batch document. The document must contain the approved template and application-generated verification link. This avoids pretending that unverified response fields or batch parameters are a provider contract.

Save it as main.go and run go run main.go. Configure INFRAI_API_KEY, API_BASE_URL, ACCOUNT_ID, ACCOUNT_EMAIL_POINTER, EMAIL_BATCH_FILE, and SIGNUP_INTENT_ID in the environment. API_BASE_URL is the platform's versioned API base from your deployment configuration; the pointer and batch file must match current discovery schemas. SIGNUP_INTENT_ID is a persisted application identifier reused for the same payload. This program dispatches a real message when supplied valid configuration.

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "net/url"
    "os"
    "strconv"
    "strings"
    "time"
)

var client = &http.Client{Timeout: 15 * time.Second}

func env(name string) string {
    value := os.Getenv(name)
    if value == "" { panic("missing environment variable: " + name) }
    return value
}

func call(method, path string, body []byte, intent string) []byte {
    base := strings.TrimRight(env("API_BASE_URL"), "/")
    for attempt := 0; attempt < 5; attempt++ {
        req, err := http.NewRequest(method, base+path, bytes.NewReader(body))
        if err != nil { panic(err) }
        req.Header.Set("Authorization", "Bearer "+env("INFRAI_API_KEY"))
        if body != nil { req.Header.Set("Content-Type", "application/json") }
        if intent != "" { req.Header.Set("Idempotency-Key", intent) }
        resp, err := client.Do(req)
        if err != nil { panic(err) }
        data, readErr := io.ReadAll(resp.Body)
        resp.Body.Close()
        if readErr != nil { panic(readErr) }
        if resp.StatusCode == http.StatusTooManyRequests && attempt < 4 {
            delay := time.Second * time.Duration(1<<attempt)
            retry := resp.Header.Get("Retry-After")
            if seconds, err := strconv.Atoi(retry); err == nil && seconds >= 0 {
                if wait := time.Duration(seconds)*time.Second; wait > delay { delay = wait }
            } else if date, err := http.ParseTime(retry); err == nil {
                if wait := time.Until(date); wait > delay { delay = wait }
            }
            time.Sleep(delay)
            continue
        }
        if resp.StatusCode < 200 || resp.StatusCode >= 300 {
            panic(fmt.Errorf("request failed: status=%d reason=%s", resp.StatusCode, data))
        }
        return data
    }
    panic("retry budget exhausted")
}

func pointer(value any, path string) any {
    if !strings.HasPrefix(path, "/") { panic("expected a nonempty JSON pointer") }
    for _, part := range strings.Split(path[1:], "/") {
        key := strings.ReplaceAll(strings.ReplaceAll(part, "~1", "/"), "~0", "~")
        switch current := value.(type) {
        case map[string]any:
            var ok bool
            value, ok = current[key]
            if !ok { panic("missing pointer member") }
        case []any:
            index, err := strconv.Atoi(key)
            if err != nil || index < 0 || index >= len(current) { panic("invalid pointer index") }
            value = current[index]
        default:
            panic("pointer traverses a scalar")
        }
    }
    return value
}

func bind(value any, email string) (any, int) {
    count := 0
    switch current := value.(type) {
    case string:
        if current == "STUDENT_EMAIL" { return email, 1 }
    case map[string]any:
        for key, item := range current {
            replacement, n := bind(item, email)
            current[key] = replacement
            count += n
        }
    case []any:
        for index, item := range current {
            replacement, n := bind(item, email)
            current[index] = replacement
            count += n
        }
    }
    return value, count
}

func main() {
    account := call(http.MethodGet, "/auth/user/get/"+url.PathEscape(env("ACCOUNT_ID")), nil, "")
    var decoded any
    if err := json.Unmarshal(account, &decoded); err != nil { panic(err) }
    email, ok := pointer(decoded, env("ACCOUNT_EMAIL_POINTER")).(string)
    if !ok || email == "" { panic("account email must be a nonempty string") }
    document, err := os.ReadFile(env("EMAIL_BATCH_FILE"))
    if err != nil { panic(err) }
    var request any
    if err := json.Unmarshal(document, &request); err != nil { panic(err) }
    request, count := bind(request, email)
    if count != 1 { panic("expected exactly one recipient placeholder") }
    body, err := json.Marshal(request)
    if err != nil { panic(err) }
    call(http.MethodPost, "/email/batch/send", body, env("SIGNUP_INTENT_ID"))
    fmt.Println("Batch accepted; reconcile delivery in the worker.")
}
```

The acceptance criterion is narrow: account output supplied the email used in the accepted batch request. This does not prove delivery or signup verification. Run this handoff only after resolving recipient eligibility, and retain the accepted response in the protected ledger. A changed template revision needs a new intent, not the old idempotency key attached to different content. Error bodies surface real failures; route them through restricted diagnostics rather than public signup responses.

The same credential covers account access, the welcome email carrying the verification link, and SMS fallback. Eligibility does not merge with credentials: email suppression is not permission to text a student. Geographic restrictions and country-based abuse circuit breakers belong in the application's SMS policy, and Twilio's SMS-pumping guidance is relevant even when another transport carries the message.

## 4. Instrument the Postgres worker before increasing concurrency

Persist one recipient-level row per event and channel before dispatch, including template revision, preference decision, suppression decision, expiry, attempt identity, provider receipt, and reconciliation position. These are proposed application fields. They are not provider API parameters. Partition eligible audiences by channel and batch sends while retaining individual rows so support can trace a student's signup without reconstructing an entire fan-out.

Recheck eligibility at dispatch. Preferences captured at enrollment can be stale by the time a worker sends the link, and retries must recheck expiry and suppression. A Node.js application can share this database contract with a Go dispatch worker; language choice does not resolve the preference-versus-retry race.

Instrument oldest eligible intent age, dispatch outcomes, age of the last successful reconciliation pass, and unfinished intents by template revision. Put recipient identifiers in access-controlled records rather than metric labels. Otherwise an investigation key becomes an expensive high-cardinality metric.

Delivery visibility uses polling. Paginate email events in background workers and reconcile SMS status in bounded passes, persisting progress only after the corresponding recipient updates commit. Repeating a page must be safe because a worker can stop between updating rows and advancing its checkpoint. Do not assume that email-event pagination and SMS status checks accept the same cursor shape; use each capability's current schema.

Polling cadence consumes the freshness budget. With an illustrative 20-second poll interval and 5 seconds reserved for local processing, a 15-second delivery-evidence target is already impossible before any provider delay. Increase reconciliation capacity or relax the target.

More send concurrency cannot repair stale evidence.

Maintain the SMS content catalog and provider mappings in the application even when managed templates are available. Record event attribution locally during send time, keeping it connected to the template revision and audience. This gives later analysis a stable event identity without making reporting conventions part of the delivery contract.

## 5. Close the alert loop without paging on every enrollment burst

Begin with a shadow alert and compare eligible-intent age against the ledger. Page after a sustained breach of the chosen age budget, and route stale reconciliation separately so the on-call engineer can distinguish delayed evidence from delayed dispatch. In a test environment, pause and restore a worker: the expected evidence is backlog growth followed by a drain, with original template revisions preserved. This is a validation procedure, not a report of an executed test.

A useful threshold depends on the workload. Below the reconciliation interval it manufactures false positives; above link expiry it can hide a queue that faithfully sends unusable links. Set it against the dispatch SLO, expiry policy, and measured drain capacity, then revisit it before enrollment opens.

Every unnecessary page spends attention the next real signup failure will need. Template ownership makes the response smaller: identify the affected revision, stop creating intents for it, review its content, and issue fresh links where appropriate. That is the practical return on keeping the account-to-message seam under application control.

## References

These primary sources support product boundaries, transactional-email evaluation, abuse considerations, and pointer syntax. Queue budgets and enrollment volumes above are illustrative design inputs.

- Postmark, Transactional Email Best Practices: https://postmarkapp.com/guides/transactional-email-best-practices
- Twilio, Preventing toll fraud and SMS pumping: https://www.twilio.com/docs/verify/preventing-toll-fraud
- Clerk documentation: https://clerk.com/docs
- Resend documentation: https://resend.com/docs
- RFC 6901, JSON Pointer: https://www.rfc-editor.org/rfc/rfc6901

## Further reading

- PostgreSQL documentation: https://www.postgresql.org/docs/current/
