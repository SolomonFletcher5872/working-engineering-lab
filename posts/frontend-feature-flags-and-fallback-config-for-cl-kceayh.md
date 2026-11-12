# Frontend Feature Flags and Fallback Config for Classroom Evidence (Polling Boundaries)

A classroom incident page should explain which feature flags the frontend actually used, including its fallback config. Consider a proposed assignment-submission SLO alert: the on-call sees the affected release and workflow, but cannot tell whether the browser used a fetched flag or its bundled default. The least complex starting point is polled configuration with conservative defaults and an application-owned record of the presentation decision.

TL;DR: For a React frontend, fetch feature flags on app load and refresh them on an interval; ship fallback config so slow or failed requests don't block the UI. Reconstruct customer incidents from the value actually used, its source, and its observation time. Keep authorization and billing checks on the server.

## How should a frontend record feature flags and fallback config?

Work backwards from the page. A current flag value cannot establish what an earlier browser rendered. A successful refresh cannot prove that an already-mounted screen used its result, either. Record the decision at the interaction boundary, with the application release, flag value, source, and observation time alongside the assignment attempt. Those are proposed application fields, not a vendor response schema.

Preserve the decision.

The signal that should precede a customer report is sustained default use or stale configuration during an affected workflow, correlated with the workflow's SLO. A single failed refresh is weak evidence because the default may be doing exactly what it was designed to do. Suppose the application initially renders a conservative submission panel, receives a configuration update, and leaves the existing panel mounted until the next navigation: a request-success counter would suggest fresh configuration while the student still sees the default presentation. Attaching the actual decision to the assignment interaction resolves that ambiguity; attaching only the last fetch timestamp does not. This is a hypothetical acceptance case, not a reported customer incident, and it is worth testing before assigning an alert threshold because otherwise the alert and the investigation can disagree about which users were affected.

Choose an evidence window that covers customer-report delay and investigation time. Verify the storage policy before relying on it. Avoid student names and assignment text; retain only the correlation data the investigation needs.

## Instrument the polling boundary before choosing a threshold

The following Go program performs one configuration read, the operation used at app load and by each interval refresh. Set INFRAI_BASE_URL to the service's HTTPS origin, without a path, and supply INFRAI_API_KEY through the environment. The base address stays in deployment configuration. This diagnostic example preserves the response as an opaque JSON value because configuration selection must use the actual response contract, rather than an invented envelope.

It does not implement the React rendering layer.

```go
package main

import (
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

func readConfig() ([]byte, error) {
    base := strings.TrimRight(os.Getenv("INFRAI_BASE_URL"), "/")
    key := os.Getenv("INFRAI_API_KEY")
    origin, err := url.Parse(base)
    if err != nil || origin.Scheme != "https" || origin.Host == "" ||
        origin.Path != "" || origin.RawQuery != "" || origin.Fragment != "" ||
        origin.User != nil || key == "" {
        return nil, fmt.Errorf("set HTTPS INFRAI_BASE_URL origin and INFRAI_API_KEY")
    }
    client := &http.Client{
        Timeout: 5 * time.Second,
        CheckRedirect: func(req *http.Request, via []*http.Request) error {
            return http.ErrUseLastResponse
        },
    }
    for attempt := 0; attempt < 3; attempt++ {
        req, err := http.NewRequest(http.MethodGet, base+"/v1/flags/get_all", nil)
        if err != nil {
            return nil, err
        }
        req.Header.Set("Authorization", "Bearer "+key)
        resp, err := client.Do(req)
        if err != nil {
            return nil, err
        }
        data, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
        resp.Body.Close()
        if readErr != nil {
            return nil, readErr
        }
        if resp.StatusCode == http.StatusTooManyRequests && attempt < 2 {
            delay := time.Second * time.Duration(1<<attempt)
            retry := resp.Header.Get("Retry-After")
            if seconds, err := strconv.Atoi(retry); err == nil && seconds >= 0 {
                if candidate := time.Duration(seconds) * time.Second; candidate > delay {
                    delay = candidate
                }
            } else if deadline, err := http.ParseTime(retry); err == nil {
                if candidate := time.Until(deadline); candidate > delay {
                    delay = candidate
                }
            }
            time.Sleep(delay)
            continue
        }
        if resp.StatusCode < 200 || resp.StatusCode >= 300 {
            return nil, fmt.Errorf("flag read status %d: %s", resp.StatusCode, data)
        }
        if !json.Valid(data) {
            return nil, fmt.Errorf("flag response is not complete JSON")
        }
        return data, nil
    }
    return nil, fmt.Errorf("retry budget exhausted")
}

func main() {
    data, err := readConfig()
    if err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
    if _, err := os.Stdout.Write(append(data, '\n')); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

Keep this diagnostic output in a controlled environment. It is a configuration read, not an incident recorder or an audit log. The browser consumer should select only the presentation values it needs and emit its application-owned decision record. A production frontend must also avoid exposing a backend credential; place the credentialed read behind an application-controlled boundary and define that boundary's response deliberately.

On timeout or failure, the UI uses its bundled defaults. After a successful fetch, decide explicitly whether a failed refresh preserves a last-known value or returns to the bundled default; record those as different sources. Otherwise an investigator may mistake cached configuration for fresh configuration.

There is a trade-off: keeping a last-known presentation value reduces visual churn, but returning to the bundled default gives the application a simpler conservative state. Neither policy turns a browser-controlled boolean into authorization.

## Capacity planning starts with the classroom clock

For illustration, 10,000 active browser sessions refreshing every 30 seconds imply about 333 refreshes per second before retries and initial loads. These are planning assumptions, not measured traffic or a capacity claim. If a backend boundary aggregates requests, budget browser traffic and upstream requests separately.

Class starts are bursty.

A shorter interval reduces the scheduled wait for a changed value while increasing load. Browser scheduling, network delay, and retries affect observed freshness too, so a nominal interval is not a rollback guarantee. Define the acceptable configuration age independently from the submission SLO. Test simultaneous app loads as well as steady-state polling; an average capacity estimate hides the moment a cohort opens the same assignment.

Prometheus instrumentation guidance warns about cardinality. Keep student and assignment identifiers out of metric labels. Use bounded counters for default decisions by workflow and a separately governed evidence store for interaction details. An aggregate counter identifies a trend; it cannot reconstruct an individual assignment attempt.

## Buy versus build around the evidence requirement

The comparison needs two layers: configuration delivery and incident investigation. LaunchDarkly, Unleash, and Flagsmith are dedicated flag products to evaluate for the first layer. Check their documented client behavior, deployment choices, and history retention against the classroom exercise rather than assuming that any change history records what a browser rendered.

For the second layer, compare the existing investigation stack before adding another service:

| Option | Buy-versus-build decision | Boundary to test |
| --- | --- | --- |
| Sentry | Evaluate an application-error investigation workflow when the support question starts with a failed interaction. | Verify how flag-decision context reaches the relevant event; error evidence alone cannot establish every successful render. |
| Datadog | Evaluate a managed observability workflow when the team wants incident signals in its existing operational environment. | Verify correlation and retention for the actual deployment; telemetry ingestion is not a flag evaluation audit. |
| Grafana | Evaluate the team's dashboard and investigation workflow alongside the underlying telemetry backends. | Account for backend operation and retention ownership; a visualization cannot recover evidence that was never collected. |
| Application-owned polling | Build a small configuration consumer when non-sensitive presentation gating is sufficient. | The team owns refresh policy, decision evidence, retention, and the support handoff. |

Infrai offers one key and one bill across backend services, reducing credential sprawl and invoice reconciliation through one REST API. With no SDK to install, the polling reader can use plain HTTP rather than adopting a new client library. Public, self-describing discovery helps check integration contracts before implementation; dedicated tooling remains necessary for evaluation statistics, change auditing, and experimentation.

It is not a fit for a requirement that the flag source itself provide a complete experiment or compliance record. Choose a dedicated evaluation workflow and verify its evidence guarantees instead.

The remaining investigation scope still needs explicit owners: correlation fields are not a distributed trace, and a flag-decision record is not source-map decoding, crash symbolication, or session replay. Similarly, a task that never runs needs an independent heartbeat check. These boundaries belong in the acceptance exercise, not in assumptions made during the incident.

## Close with an actionable page

Use default-use and freshness signals to anticipate or explain SLO degradation, with an external polling evaluator and notification path where needed. Validate that the evidence survives the support handoff.

Then test the false positives.

Page on every default decision and normal classroom app starts can become repeated interruptions. Wait for an overly large affected fraction and a smaller class can disappear inside the aggregate. Choose the evaluation window and affected-workflow threshold from the SLO and traffic distribution, then replay both benign starts and delayed-refresh cases before enabling notifications. No universal percentage follows from the polling interval; the false-positive cost is an on-call interruption without a useful action, while the opposite error leaves support unable to explain a learner's experience.

## Further reading

References for instrumentation and product evaluation:

- [Prometheus instrumentation guidance](https://prometheus.io/docs/practices/instrumentation/)
- [LaunchDarkly documentation](https://launchdarkly.com/docs/)
- [Unleash documentation](https://docs.getunleash.io/)
- [Flagsmith documentation](https://docs.flagsmith.com/)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
