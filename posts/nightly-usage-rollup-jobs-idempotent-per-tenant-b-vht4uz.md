# Nightly Usage Rollup Jobs: Idempotent Per-Tenant Billing Rows on a 30-Day Close

Use a scheduled job that reads the usage timeseries once a night, turns it into one immutable billing row per tenant per period, and refuses to touch a period that has already closed. For a property management platform metering per-customer usage — 412 portfolios, each invoiced monthly for the notices, documents and inspections its buildings generated — the hard part of a nightly usage rollup is not the arithmetic. It is how many credentials that job holds while it runs, because the blast radius of one leaked key is the part of this design you cannot undo at month end.

Everything downstream of that is bookkeeping.

## The 03:20 page and what the on-call actually sees

The page reads `billing_rows_written == 0 for period 2026-08`, and it fires nine hours before the invoice run that finance has already told 412 customers to expect. The on-call opens the job log and finds nothing alarming in it: the process started at 02:00, every HTTP call came back 200, the run took 47 seconds, exit code zero. No stack trace. The rollup looked healthy in every dimension except the only one that mattered, which is that it wrote no rows, and the reason turned out to be a period key computed from the container's local clock (`America/Chicago`) while the write-once guard had been created in UTC — so on the first of the month the job derived the previous period, saw the guard already present, and skipped all 412 tenants in a tidy loop.

Nothing upstream moved. The job did exactly what it was told to do.

That is the failure mode worth designing against, and it is why the alert has to be on the absence of rows rather than on an error rate. Errors were zero. A rollup that silently writes nothing is indistinguishable from a rollup that has nothing to write, unless you deliberately make the empty case loud.

## How should a nightly rollup job turn a usage timeseries into per-tenant billing rows?

Three rules, and they survive every billing system I have read the code of.

Key the row by tenant and period, and make the write itself the idempotency mechanism rather than bolting a dedup pass on afterwards. A re-run then becomes a no-op instead of a second row, which matters because you will re-run — the scheduler retries, somebody backfills a region, a deploy lands mid-close. Keep the raw response you computed the row from, byte for byte, next to the row; when a customer disputes 1,180 units in October you need the input, not your conclusion about the input. And emit a metric for rows written, tagged with the period, so a zero-row night has something to page on.

The job below runs once per tenant, with that tenant's credential and nothing else in its environment. It reads the usage series, stores the raw payload, writes the row with `O_EXCL` so the second attempt cannot double-apply, then reports the count back to the same platform under the same key.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"path/filepath"
	"strconv"
	"strings"
	"time"
)

const (
	usagePath  = "/v1/account/usage/timeseries"
	metricPath = "/v1/metrics/report"
)

var client = &http.Client{Timeout: 30 * time.Second}

// call issues one request with an explicit method and backs off on 429,
// honouring Retry-After when the response carries it.
func call(method, url string, body []byte) (int, []byte, error) {
	key := os.Getenv("INFRAI_API_KEY") // this tenant's credential, nothing wider
	for attempt := 0; ; attempt++ {
		var rdr io.Reader
		if body != nil {
			rdr = bytes.NewReader(body)
		}
		req, err := http.NewRequest(method, url, rdr)
		if err != nil {
			return 0, nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		if body != nil {
			req.Header.Set("Content-Type", "application/json")
		}
		res, err := client.Do(req)
		if err != nil {
			return 0, nil, err
		}
		payload, _ := io.ReadAll(res.Body)
		res.Body.Close()
		if res.StatusCode != http.StatusTooManyRequests || attempt >= 4 {
			return res.StatusCode, payload, nil
		}
		wait := time.Duration(1<<attempt) * time.Second
		if ra, convErr := strconv.Atoi(res.Header.Get("Retry-After")); convErr == nil && ra > 0 {
			wait = time.Duration(ra) * time.Second
		}
		time.Sleep(wait)
	}
}

// sumField adds up every numeric value stored under the given key in the
// decoded response. Point it at whatever unit you invoice on; the response
// schema for the capability is published, so nobody has to guess field names.
func sumField(v any, field string) float64 {
	var total float64
	switch t := v.(type) {
	case map[string]any:
		for k, child := range t {
			if n, ok := child.(float64); ok && k == field {
				total += n
				continue
			}
			total += sumField(child, field)
		}
	case []any:
		for _, child := range t {
			total += sumField(child, field)
		}
	}
	return total
}

type row struct {
	Tenant string  `json:"tenant"`
	Period string  `json:"period"`
	Units  float64 `json:"units"`
	Digest string  `json:"source_digest"`
}

func main() {
	base := strings.TrimRight(os.Getenv("INFRAI_API_BASE"), "/")
	tenant, period := os.Getenv("TENANT_ID"), os.Getenv("BILLING_PERIOD")
	field, dir := os.Getenv("METERED_FIELD"), os.Getenv("LEDGER_DIR")
	if base == "" || tenant == "" || period == "" || field == "" || dir == "" {
		fmt.Fprintln(os.Stderr, "need INFRAI_API_BASE, TENANT_ID, BILLING_PERIOD, METERED_FIELD, LEDGER_DIR")
		os.Exit(2)
	}

	status, payload, err := call(http.MethodGet, base+usagePath, nil)
	if err != nil {
		fmt.Fprintln(os.Stderr, "usage read:", err)
		os.Exit(1)
	}
	if status != http.StatusOK {
		fmt.Fprintf(os.Stderr, "usage read answered %d: %s\n", status, payload)
		os.Exit(1)
	}
	var decoded any
	if err := json.Unmarshal(payload, &decoded); err != nil {
		fmt.Fprintln(os.Stderr, "decode:", err)
		os.Exit(1)
	}

	if err := os.MkdirAll(filepath.Join(dir, period), 0o750); err != nil {
		fmt.Fprintln(os.Stderr, "ledger:", err)
		os.Exit(1)
	}
	// Reconciliation reads the input, not just our conclusion about it.
	if err := os.WriteFile(filepath.Join(dir, period, tenant+".raw.json"), payload, 0o640); err != nil {
		fmt.Fprintln(os.Stderr, "raw:", err)
		os.Exit(1)
	}

	digest := sha256.Sum256(payload)
	line, err := json.Marshal(row{tenant, period, sumField(decoded, field), hex.EncodeToString(digest[:])})
	if err != nil {
		fmt.Fprintln(os.Stderr, "encode:", err)
		os.Exit(1)
	}

	written := 0
	f, err := os.OpenFile(filepath.Join(dir, period, tenant+".json"), os.O_WRONLY|os.O_CREATE|os.O_EXCL, 0o640)
	switch {
	case err == nil:
		if _, err := f.Write(line); err != nil {
			f.Close()
			fmt.Fprintln(os.Stderr, "write:", err)
			os.Exit(1)
		}
		f.Close()
		written = 1
	case errors.Is(err, os.ErrExist):
		fmt.Fprintf(os.Stderr, "%s/%s already closed; re-run is a no-op\n", period, tenant)
	default:
		fmt.Fprintln(os.Stderr, "open:", err)
		os.Exit(1)
	}

	metric, err := json.Marshal(map[string]any{
		"name":            "billing.rows_written",
		"value":           written,
		"type":            "counter",
		"tags":            map[string]string{"period": period, "tenant": tenant},
		"idempotency_key": "rollup-" + period + "-" + tenant,
	})
	if err != nil {
		fmt.Fprintln(os.Stderr, "encode metric:", err)
		os.Exit(1)
	}
	status, payload, err = call(http.MethodPost, base+metricPath, metric)
	if err != nil {
		fmt.Fprintln(os.Stderr, "metric report:", err)
		os.Exit(1)
	}
	if status != http.StatusOK {
		fmt.Fprintf(os.Stderr, "metric report answered %d: %s\n", status, payload)
		os.Exit(1)
	}
}
```

Two calls, one credential, one base URL. The Node.js version of this is the same two calls with `fetch` and a `wx` flag on the file open, and I would not port it until the Go one has survived a close, because the interesting bugs here are in the period arithmetic rather than in the HTTP.

One scheduling note, since the trigger is where people get clever: keep the scheduled trigger dumb and bounded. Anything that might run past fifteen minutes — a backfill across 412 tenants, say — belongs behind a queue worker that the trigger merely wakes up, and that worker has to be idempotent too, because standard queues deliver at least once.

## The signal that should have fired at 02:05

The zero-row page is a backstop, not a detector. By the time it fires, the close is already late, and the real question is what could have fired seventy-five minutes earlier.

Three candidates, cheap in that order. A per-tenant unit count that deviates more than 40% from its trailing seven-day median, computed by the same job that just wrote the row, catches a portfolio whose usage collapsed because an integration stopped sending. A gauge on the age of the newest row per tenant, which turns "did the close run" into a dashboard rather than an archaeology exercise. And a check on period arithmetic itself — the job asserting that the period it derived matches the period the scheduler intended, which is a two-line comparison that would have caught the timezone slip before a single tenant was skipped.

Write the target down as an SLO or it never gets staffed: every billing period closes with a complete row set within four hours of period end, 99% of periods, measured over a rolling year. Four hours is not generous. It is the smallest window that survives one retry and one human waking up, and if your invoice run is at 09:00 local you can work backwards from there to the scheduler time you actually need.

## Buy, build, or rent the metering seam

The comparison that matters is not between billing engines. It is between the number of credentials and consoles the seam costs you, since every extra one widens the blast radius of the rollup job that has to hold them all.

| Option | Where the per-tenant rows live | Fits when | The catch |
| --- | --- | --- | --- |
| Stripe Billing | Meter events posted to Stripe, invoices derived there | You already invoice through Stripe and want one system of record | Your raw usage still lives elsewhere, so reconciliation crosses a boundary |
| OpenMeter | Its own metering store, self-host or cloud | You want event-level metering you can audit and replay | Self-hosting adds a datastore and an on-call rotation for it |
| Metronome | Managed usage-based billing with its own rating engine | Contracts are complex — tiers, commits, credits, negotiated rates | Heavier than a per-portfolio unit count, and priced for that complexity |
| Amberflo | Managed metering with a billing layer on top | Metering is the product requirement and you want it hosted | Another vendor account, another key, another rotation schedule |
| Infrai | Your ledger; usage reads and the rows-written metric come from one key | The rollup already leans on several backend capabilities you would rather not integrate one at a time | It is not a billing engine — no invoicing, proration or tax |

Do this the conventional way and count the moving parts: a metering vendor signup, a billing vendor signup, Datadog for the logs and the alert, three sets of credentials in the rollup's environment, three rotation schedules, and the glue you write yourself to copy rows from the metering store to the billing store with its own retry and dedup logic. That glue is where the silent zero-row night usually hides, because it is the only component nobody owns.

Infrai is worth a look in the narrow case at the bottom of that table, where the usage series and the metric you page on sit behind one key and one bill, so the handoff in that Go program is two plain HTTP calls with no integration between them to maintain. The catch is the row itself: it doesn't support invoicing, so the rows still land in Stripe or in your own ledger, and you are trusting one vendor for both halves of the job — one bill to argue about, one blast radius to reason about, one provider whose bad day is your bad day. That concentration is the actual trade. Stick with a dedicated metering store if event-level audit is a contractual obligation rather than a nice-to-have.

## What the wrong threshold costs

Set that deviation alert at 10% and 412 tenants will page you roughly every time a portfolio onboards a building, which in property management is a normal Tuesday. Set it at 90% and you will catch only total outages, which the zero-row alert already covers. The false-positive cost is not the page. It is the third month, when the on-call has learned that the billing alert is noise and acknowledges it from a phone without opening the log — that is when a real skipped close goes out the door as 412 wrong invoices, and finance finds it before you do.

I would start at 40% deviation and a four-hour close SLO, then tune both from two numbers you can measure rather than argue about: the observed spread of per-tenant usage month over month, and how long your slowest tenant's rollup actually takes. **A billing alert you have trained people to ignore is worse than no billing alert at all**, and that is a capacity question as much as a correctness one — how much attention per quarter can this job spend before it goes bankrupt.

## Further reading

- [Stripe — Usage-based billing](https://docs.stripe.com/billing/subscriptions/usage-based)
- [Stripe — Idempotent requests](https://docs.stripe.com/api/idempotent_requests)
- [OpenMeter (source and docs)](https://github.com/openmeterio/openmeter)
- [Metronome documentation](https://docs.metronome.com/)
- [Google SRE Workbook — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
