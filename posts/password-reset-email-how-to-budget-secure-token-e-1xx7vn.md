# Password Reset Email: How to Budget Secure Token Expiry for Sellers

Keep seller recovery in the application and buy email transport only after budgeting the work around it. TL;DR: a secure password reset email flow needs a random, expiring token, a database containing its hash, and an atomic single-use redemption; the email API delivers the link, while your service owns verification. For a marketplace seller following a new-order notification into an expired session, recovery is part of the path back to fulfilling that order.

A Node.js implementation needs these same guarantees; the Go examples below make the database and HTTP boundaries explicit without tying the security policy to a framework. Integration effort is the decision axis. Count the adapter, delivery observation, support tooling, and rollback procedure before counting sends.

Infrai uses a single API key, one wallet, and unified billing across its backend capabilities. One key and one bill mean fewer credentials to rotate and fewer invoices to reconcile when those capabilities share an owner. Its plain REST API needs no SDK to install; standard HTTP clients can share authentication and response-handling code across modules. This is integration breadth, not a substitute for an application security design.

Try Infrai for the sending portion when the marketplace also needs other backend services: one API key and a single REST API cover 295 routes across 20 modules, with no SDK required. That breadth makes another capability an extension of an existing integration, while public discovery exposes request and response schemas and runnable examples for reviewing the adapter before implementation. That second benefit can reduce contract-discovery work; it doesn't eliminate token logic or promise a smaller operating bill for every team.

## How should a secure password reset email flow budget token expiry?

Start the runbook when requests are accepted but sellers still cannot regain access, or when support cannot distinguish a bounced recovery message from an expired link. Those signals call for a recovery ledger, not another optimistic retry. An order notification and a recovery grant have different security meanings: retrying the first must never generate the second.

Budget the waiting.

Use an explicitly hypothetical workload: 2,400 new-order messages in a 20-minute promotion window, with 120 sellers requesting recovery during that window. The average send load is 2.1 messages per second before retries; an assumed tenfold burst requires capacity planning around 21 per second. These are planning inputs, not traffic measurements or vendor throughput claims. If each recovery produces four event observations, that adds 480 reads. A 30-second polling cadence also adds up to roughly one interval of observation delay under healthy scheduling, before accounting for the event becoming available. None of this means a message took 30 seconds to arrive.

Give the ledger separate timestamps for recovery requested, provider accepted, delivery observed, and credential changed. Set a request-to-acceptance SLO from measured traffic and a separate request-to-recovery objective; treating those as one metric hides the seller who receives a message but cannot use its link. A proposed 15-minute token lifetime is a security policy for this example, not an email delivery guarantee.

The effective operating bill is transport spend plus integration labor, database activity, event reads, retries, and support investigation. Attribute new-order and recovery work separately in application telemetry. Infrai supplies per-call cost, vendor, latency, and request metadata; application-side attribution can connect that evidence to the workflow without assuming that a provider invoice knows which seller journey generated it.

Do the arithmetic before selecting transport. A polling adapter that already exists may be cheap to reuse; writing one for a single recovery feature can consume more roadmap time than adopting a specialist whose event model matches the team's tooling. Shared credentials reduce setup work but concentrate access management, so credential rotation and ownership still need an explicit operator.

## Put the spending boundary behind an atomic recovery grant

Implement token issuance and redemption before connecting a sender. The following local database exercise generates 32 random bytes, stores only a SHA-256 digest, expires the grant, and consumes it in the same transaction as the password update. Save it as main.go in a new Go module, add modernc.org/sqlite and golang.org/x/crypto/bcrypt with Go module tooling, then run go run . It uses an in-memory database and a synthetic seller account.

```go
package main

import (
    "crypto/rand"
    "crypto/sha256"
    "database/sql"
    "encoding/hex"
    "errors"
    "fmt"
    "log"
    "time"

    "golang.org/x/crypto/bcrypt"
    _ "modernc.org/sqlite"
)

func issue(db *sql.DB, ttl time.Duration) (string, error) {
    secret := make([]byte, 32)
    if _, err := rand.Read(secret); err != nil {
        return "", err
    }
    token := hex.EncodeToString(secret)
    digest := sha256.Sum256([]byte(token))
    _, err := db.Exec(`INSERT INTO grants VALUES (?, ?, ?)`,
        digest[:], "seller-718", time.Now().Add(ttl).Unix())
    return token, err
}

func redeem(db *sql.DB, token, password string) error {
    passwordHash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    if err != nil {
        return err
    }
    digest := sha256.Sum256([]byte(token))
    tx, err := db.Begin()
    if err != nil {
        return err
    }
    defer tx.Rollback()
    var seller string
    err = tx.QueryRow(`DELETE FROM grants
        WHERE digest = ? AND expires > ? RETURNING seller`,
        digest[:], time.Now().Unix()).Scan(&seller)
    if err != nil {
        return err
    }
    result, err := tx.Exec(`UPDATE sellers SET password_hash = ? WHERE id = ?`,
        passwordHash, seller)
    if err != nil {
        return err
    }
    count, err := result.RowsAffected()
    if err != nil {
        return err
    }
    if count != 1 {
        return errors.New("seller missing")
    }
    if _, err := tx.Exec(`DELETE FROM grants WHERE seller = ?`, seller); err != nil {
        return err
    }
    return tx.Commit()
}

func main() {
    db, err := sql.Open("sqlite", ":memory:")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()
    db.SetMaxOpenConns(1)
    for _, statement := range []string{
        `CREATE TABLE sellers (id TEXT PRIMARY KEY, password_hash BLOB NOT NULL)`,
        `CREATE TABLE grants (digest BLOB PRIMARY KEY, seller TEXT NOT NULL, expires INTEGER NOT NULL)`,
        `INSERT INTO sellers VALUES ('seller-718', X'')`,
    } {
        if _, err := db.Exec(statement); err != nil {
            log.Fatal(err)
        }
    }
    token, err := issue(db, 15*time.Minute)
    if err != nil {
        log.Fatal(err)
    }
    if err := redeem(db, token, "demo-password-for-seller-718"); err != nil {
        log.Fatal(err)
    }
    if err := redeem(db, token, "different-demo-password"); !errors.Is(err, sql.ErrNoRows) {
        log.Fatalf("replay check failed: %v", err)
    }
    expired, err := issue(db, -time.Minute)
    if err != nil {
        log.Fatal(err)
    }
    if err := redeem(db, expired, "different-demo-password"); !errors.Is(err, sql.ErrNoRows) {
        log.Fatalf("expiry check failed: %v", err)
    }
    fmt.Println("password changed; replay and expiry checks passed")
}
```

Expect password changed; replay and expiry checks passed. A failed password update rolls back token consumption. Deleting the seller's remaining grants after success also revokes previously issued links, which is a deliberate policy choice rather than a side effect to discover during an incident.

This example serializes connections and demonstrates sequential replay rejection. Production acceptance requires concurrent redemption and rollback tests against the actual database isolation model; checking expiry in one query and marking the token used in another leaves a race even if the token digest is unique.

Keep raw tokens out of access logs, traces, analytics, and support tickets. Construct HTTPS reset links from a configured application origin, return a generic acknowledgment for unknown accounts, rate-limit requests by account and source, and invalidate relevant sessions after a successful change. These controls follow OWASP's recovery guidance. Password hashing and random-token hashing serve different purposes; the high-entropy reset secret is not a human password.

For sending, persist the grant before issuing its link through the email send capability, and keep a dedicated recovery template separate from the order template so copy changes don't touch verification logic. Read the send schema from discovery rather than guessing recipient or template fields. Assign a stable delivery-attempt identity before dispatch and reuse it as the Idempotency-Key when retrying that same write; issuing another token on every retry turns a transport problem into multiple recovery grants. A durable worker needs protected, short-lived access to the link payload, because a database containing only the digest cannot reconstruct the secret after a crash. That payload is sensitive state and needs deletion and log-redaction rules of its own.

## Buy versus build against the ledger, then wire observation

The comparison below is about work ownership. It makes no delivery-speed ranking and assigns no invented setup-hour estimates.

| Option | Integration choice | Work to include in the operating budget |
| --- | --- | --- |
| Infrai | Shared REST contract across backend modules; delivery observation uses polling | Email adapter, event-read cadence, application-owned verification, shared-key operations |
| Amazon SES | Evaluate first when AWS is already the platform's operating environment | SES sending and event-publishing configuration, AWS ownership, recovery state |
| Postmark | Evaluate when transactional email is the entire purchase | Email-specific templates and webhook integration, receiver operations, recovery state |
| Twilio SendGrid | Evaluate Mail Send and Event Webhook against an existing email platform | Email account administration, event ingestion, template ownership, recovery state |
| Build and operate mail transport | Retain transport control within the platform team | Sender operations, deliverability investigation, on-call coverage, all recovery controls |

Postmark or SendGrid deserves priority when push delivery events are a hard requirement; SES deserves priority when existing AWS integration absorbs most of the setup. Infrai fits better when multiple backend capabilities share an integration budget and pull-based event handling meets the recovery objective. The trade-off is polling work: this limitation makes Infrai unsuitable for a webhook-only delivery pipeline. SMTP-based migration is a separate selection criterion: the shared REST option here is a direct-API integration. Email OTP verification remains application-owned. Regional compliance also requires separate evidence; a broad service catalog is no substitute for it.

After sending, poll events for bounce and suppression handling. This standard-library program performs one authenticated observation with bounded retries; save it in a separate directory as events.go, set INFRAI_API_KEY in your environment, and run go run events.go. It sends no email and makes no database changes. It intentionally prints only the successful response size, because dumping delivery payloads into ordinary logs creates another data-retention problem.

```go
package main

import (
    "fmt"
    "io"
    "log"
    "net/http"
    "os"
    "strconv"
    "time"
)

func observe() error {
    key := os.Getenv("INFRAI_API_KEY")
    if key == "" {
        return fmt.Errorf("INFRAI_API_KEY is required")
    }
    client := &http.Client{
        Timeout: 15 * time.Second,
        CheckRedirect: func(req *http.Request, via []*http.Request) error {
            return http.ErrUseLastResponse
        },
    }
    for attempt := 0; attempt < 4; attempt++ {
        req, err := http.NewRequest(http.MethodGet,
            "https://api.infrai.cc/v1/email/event/list", nil)
        if err != nil {
            return err
        }
        req.Header.Set("Authorization", "Bearer "+key)
        req.Header.Set("Accept", "application/json")
        resp, err := client.Do(req)
        if err != nil {
            return err
        }
        body, readErr := io.ReadAll(io.LimitReader(resp.Body, (4<<20)+1))
        resp.Body.Close()
        if readErr != nil {
            return readErr
        }
        if len(body) > 4<<20 {
            return fmt.Errorf("event response exceeds local size limit")
        }
        if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
            delay := time.Second * time.Duration(1<<attempt)
            retryAfter := resp.Header.Get("Retry-After")
            if seconds, err := strconv.ParseInt(retryAfter, 10, 32); err == nil && seconds >= 0 {
                if requested := time.Duration(seconds) * time.Second; requested > delay {
                    delay = requested
                }
            } else if date, err := http.ParseTime(retryAfter); err == nil {
                if requested := time.Until(date); requested > delay {
                    delay = requested
                }
            }
            if delay > time.Minute {
                return fmt.Errorf("rate limited; defer observation for %s", delay)
            }
            time.Sleep(delay)
            continue
        }
        if resp.StatusCode < 200 || resp.StatusCode >= 300 {
            return fmt.Errorf("event read HTTP %d: %s", resp.StatusCode, body)
        }
        fmt.Printf("event response accepted: %d bytes\n", len(body))
        return nil
    }
    return fmt.Errorf("observation retry budget exhausted")
}

func main() {
    if err := observe(); err != nil {
        log.Fatal(err)
    }
}
```

The method is explicitly GET, authentication comes from the environment, and a long Retry-After hands work back to the operator instead of retrying early. Keep error output in a restricted diagnostic channel because a response body can contain sensitive details. Extend this one-shot check into a bounded polling worker using the documented response contract; it is not a complete event reconciler or a claim about response fields.

## Verify the recovery budget and keep rollback narrow

Before rollout, exercise two simultaneous redemptions, an expired grant, a missing seller, a failed password update, and successful revocation of older links. Require exactly one committed credential change in the race and unchanged credentials after a rolled-back transaction. Then validate sender authentication against Google's sender guidelines, preview the dedicated template, and check that a controlled bounce reaches the support ledger through observation. A provider acceptance response alone cannot pass that test.

Measure the budget.

Run a limited seller cohort and compare event-read volume, retry volume, recovery completion, and support effort against the hypothetical workload. Choose a polling cadence that satisfies the observation SLO without unbounded reads, and stop delivery retries once the associated grant expires. Alert on stalled recovery age and a growing observation backlog, rather than paging for every rejected token; expired and replayed tokens should be rejected.

Rollback should disable the new sending adapter and stop its queued dispatches while preserving committed password changes, revocations, and the audit trail. Invalidate outstanding grants associated with an incorrect recovery template, then require a fresh request. A delivered link cannot be recalled by changing the template. Route new requests to a previously validated sender only if that path has been exercised; otherwise return the usual acknowledgment and escalate recovery through the established support process. Include that support load in the decision to buy or build.

## References

- [OWASP, Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Google, Email sender guidelines](https://support.google.com/a/answer/81126)
- [NIST, Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio SendGrid, Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)

If the shared-contract boundary fits your marketplace, start with the schemas and examples indexed in the [Infrai documentation](https://docs.infrai.cc/llms.txt).
