# How to Gate Startup Transactional Email API and SMTP Relay Rollouts

Use a replay and rollback acceptance gate before moving seller order notifications between a transactional email API and an SMTP relay. For a fintech marketplace, the deciding constraint is whether the team can account for each notification after a timeout, a worker restart, or a transport change; a convenient send method does not answer those questions.

TL;DR: Keep a durable notification record, distinguish transport acceptance from downstream delivery evidence, and rehearse ambiguous outcomes before changing traffic. Apply the same gate to startup welcome emails. An API-first interface earns its place through demonstrated operational behavior; SMTP earns its place through the same evidence.

This note uses a proposed acceptance policy, not a measured service benchmark: a 60-second dispatch objective, a burst of 120 seller notifications per second, and a 30-second interruption. Those numbers are deliberately explicit so a platform team can replace them with its own workload. None is a provider commitment.

## Should a startup gate transactional email API and SMTP relay rollouts differently?

Start with the signal: notification age rises while send requests appear successful, or a worker times out and the next worker cannot tell whether the previous attempt was accepted. A welcome message sent twice is awkward. A duplicate new-order alert can also make a seller question whether the marketplace created two orders. The order identifier must remain stable across notification attempts.

Acceptance needs a name.

The protocol doesn't settle the release decision.

Use separate states for a notification awaiting dispatch, an attempt awaiting reconciliation, and a notification with transport acceptance recorded. Keep downstream delivery evidence separately, with its source and timestamp. Do not label an accepted request “seller notified,” because that label asserts a recipient outcome the application has not established. This is an operational accounting rule: define the evidence required for each state before writing an adapter.

For this runbook, define the dispatch SLO as 99.9% of eligible seller notifications receiving transport acceptance within 60 seconds, measured from the durable notification creation time. That is a proposed internal target. Track delivery evidence coverage and age alongside it; neither acceptance nor an open event should silently stand in for a seller reading an order. Welcome messages can share the ledger design while using a separate queue and objective, so onboarding traffic does not consume the entire order-alert budget.

Step 1 is to calculate the recovery budget before choosing a transport. Save this standalone Go program as a local capacity worksheet and run it with the standard Go toolchain. It assumes constant rates, no further interruption, and full access to the stated dispatch capacity; use a load test to challenge those assumptions.

```go
package main

import "fmt"

func main() {
	const arrivalPerSecond = 120.0
	const interruptionSeconds = 30.0
	const dispatchPerSecond = 240.0
	const ageLimitSeconds = 60.0

	backlog := arrivalPerSecond * interruptionSeconds
	spare := dispatchPerSecond - arrivalPerSecond
	if spare <= 0 {
		panic("no recovery capacity while new seller orders arrive")
	}
	drainSeconds := backlog / spare
	// This is a conservative worksheet budget, not a latency prediction.
	recoveryBudget := interruptionSeconds + drainSeconds
	fmt.Printf("backlog=%.0f drain_seconds=%.0f budget_seconds=%.0f\n",
		backlog, drainSeconds, recoveryBudget)
	if recoveryBudget >= ageLimitSeconds {
		fmt.Println("HOLD: the proposed budget has no safety margin")
		return
	}
	fmt.Println("Proceed to a workload test with explicit headroom")
}
```

The worksheet produces 3,600 queued notifications and a 30-second drain time. Its conservative budget reaches 60 seconds, so this configuration fails the margin check even though dispatch capacity is twice the arrival rate. That is the useful disagreement: nominal throughput can look comfortable while the recovery budget has no room for retries, variable latency, or a second interruption. The worksheet does not calculate a latency percentile. Measure notification age under the actual queue discipline before claiming the SLO is met.

For the roadmap decision, compare the work the team will own. Require evidence for each candidate implementation, without assuming that an interface category supplies a capability.

| Approach | Evidence required before rollout | Ownership trade-off |
| --- | --- | --- |
| Managed transactional API | Acceptance identifiers, timeout semantics, status reconciliation, and tested throttling behavior | Potentially less transport operation; provider-specific semantics still need an adapter and a runbook |
| Managed SMTP relay | Session outcome capture, handling of uncertain acceptance, and downstream status correlation | Existing SMTP integration may be reusable; the team must prove its ledger can reconcile relay outcomes |
| Self-hosted sending path | The same acceptance tests plus domain authentication, queue recovery, and operational coverage | More control over the sending path; more capacity and on-call work must fit the platform roadmap |

Price belongs in the staffing and capacity review, but it cannot resolve an ambiguous send. A short integration estimate is also incomplete if it excludes reconciliation and rollback.

## Implement a conservative acceptance gate

Step 2 is to make ambiguous outcomes executable in a small contract harness. The program below models one seller notification across two attempts. It is intentionally an in-memory acceptance test, not a production queue: the production implementation must persist the record and attempt state, coordinate concurrent workers, and recover that state after a crash.

The important branch is the timeout. After an uncertain result, this policy requires reconciliation before authorizing another send. If the selected transport cannot resolve that uncertainty, keep the record visible for an explicit operator decision; do not pretend that an adapter can manufacture certainty. A stable application key helps correlate attempts, but it does not by itself make a remote service deduplicate them.

```go
package main

import "fmt"

type Outcome int

const (
	Accepted Outcome = iota
	KnownNotAccepted
	Uncertain
)

type Notice struct {
	Key      string
	OrderID  string
	SellerID string
	State    string
	Receipt  string
}

func apply(n *Notice, result Outcome, receipt string) {
	switch result {
	case Accepted:
		if receipt == "" {
			panic("accepted outcome requires a local receipt reference")
		}
		n.State, n.Receipt = "accepted", receipt
	case KnownNotAccepted:
		n.State = "pending"
	case Uncertain:
		n.State = "reconcile"
	default:
		panic("unknown outcome")
	}
}

func begin(n *Notice) bool {
	if n.State != "pending" {
		return false
	}
	n.State = "inflight"
	return true
}

func require(ok bool, message string) {
	if !ok {
		panic(message)
	}
}

func main() {
	n := Notice{
		Key:      "seller-order:order-1042:seller-17",
		OrderID:  "order-1042",
		SellerID: "seller-17",
		State:    "pending",
	}
	require(begin(&n), "first attempt must start")
	apply(&n, Uncertain, "")
	require(!begin(&n), "uncertain acceptance must block automatic resend")

	// Model reconciliation that establishes the first attempt was accepted.
	apply(&n, Accepted, "attempt-1-evidence")
	require(!begin(&n), "accepted notification must not be sent again")
	require(n.Receipt != "", "acceptance evidence must be retained")
	fmt.Printf("%s state=%s receipt=%s\n", n.Key, n.State, n.Receipt)

	retry := Notice{Key: "seller-order:order-1043:seller-17", State: "pending"}
	require(begin(&retry), "retry example must start")
	apply(&retry, KnownNotAccepted, "")
	require(begin(&retry), "known nonacceptance can permit another attempt")
	fmt.Println("Acceptance contract checks passed")
}
```

Here, a receipt is a local reference to stored attempt evidence; it need not be a provider-issued message identifier. The adapter must document what establishes acceptance and what establishes known nonacceptance. A lost response belongs in the uncertain branch unless additional evidence resolves it. Reusing the same key through a second transport is not proof that only one message will be delivered.

Keep the queue record creation atomic with the business event through an appropriate durable transaction boundary. For the marketplace example, define which committed order event makes a seller notification eligible, and preserve that definition during rollout. This is a design requirement, not an assertion about a particular database. Otherwise the transport comparison begins after the most consequential gap: an order can exist without any notification work being recorded. Consider a controlled test in which order-1042 commits, its alert is claimed, the transport accepts the attempt, and the worker stops before recording the response. On recovery, the order still exists and its notification still needs accounting, but dispatching it again could duplicate the alert. The acceptance gate should demand an explanation for that exact record: which evidence survived, which worker owns reconciliation, and what authorizes the next transition. Run the same test with a welcome message and a seller alert queued together, so the exercise also exposes whether an onboarding burst delays order work.

Do this before cutover.

Domain authentication is another acceptance prerequisite, independent of the adapter. Google's sender guidelines require SPF or DKIM for all senders to personal Gmail accounts, with additional requirements for bulk senders, including SPF, DKIM, and DMARC; the guidelines also require TLS for transmission. Follow the applicable requirements and inspect alignment for the actual sending domain. Do not infer that using an API automatically establishes the domain configuration.

Keep the scope honest. An order alert and a welcome message are notification workflows. They should not become a substitute for an authentication mechanism; NIST's cited digital identity guidance excludes email as an out-of-band authenticator. If onboarding includes verification or account access, review the authentication design separately rather than treating delivery telemetry as proof of identity.

## Verify rollout and rehearse rollback

Step 3 is to turn operational evidence into a release decision. Before live traffic, exercise known nonacceptance, uncertain acceptance, worker interruption after dispatch, duplicate work delivery, and delayed status evidence using controlled test recipients. The harness above checks only state transitions. The deployed system must demonstrate durable recovery and concurrency control, and a real transport test must establish how each outcome maps into those transitions.

For a concrete release worksheet, the following standalone program checks synthetic canary results against a proposed 1,000-notification gate. These are illustrative inputs, not production measurements. Zero ambiguous records is a deliberately strict release condition; set and document the actual observation window before collecting results.

```go
package main

import "fmt"

func main() {
	const eligible = 1000
	const acceptedWithin60Seconds = 999
	const unresolved = 0
	const missingLedgerRecords = 0
	const concurrentTransportOwners = 0

	bad := eligible - acceptedWithin60Seconds
	withinBudget := bad*1000 <= eligible
	safe := withinBudget &&
		unresolved == 0 &&
		missingLedgerRecords == 0 &&
		concurrentTransportOwners == 0

	fmt.Printf("eligible=%d late_or_unaccepted=%d\n", eligible, bad)
	if !safe {
		fmt.Println("HOLD: keep new assignments on the established path")
		return
	}
	fmt.Println("Gate passed: review evidence before expanding the canary")
}
```

A canary with 999 timely acceptances out of 1,000 meets this sample's 99.9% arithmetic. It does not establish long-term reliability. Compare cohorts using the same eligibility rule, include unresolved records in the investigation, and preserve the original creation timestamps when moving work; resetting a timestamp can make a growing backlog look healthy. Monitor oldest pending age, unresolved attempt age, acceptance latency, and authenticated sending configuration alongside the sample ratio.

Rollback needs ownership.

Keep the evidence.

Stop assigning fresh work to the new path, let the established path take new eligible records, and reconcile uncertain attempts before deciding whether any may be retried. Preserve accepted records and their evidence. If two workers can independently claim the same notification during a cutover, fix that ownership boundary before widening traffic.

The decision between a transactional email API and SMTP should survive this rehearsal: the team can explain what happened to a seller's order alert, recover within its capacity budget, and reverse a rollout without casually duplicating messages. That is a defensible acceptance gate for startup welcome email infrastructure as well.

## References

- Google, Email sender guidelines: https://support.google.com/a/answer/81126
- NIST, SP 800-63B Digital Identity Guidelines, Authentication and Lifecycle Management: https://pages.nist.gov/800-63-3/sp800-63b.html
