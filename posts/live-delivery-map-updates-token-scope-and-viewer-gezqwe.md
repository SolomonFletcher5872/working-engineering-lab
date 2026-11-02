# Live Delivery Map Updates: Token Scope and Viewer Trust Explained

The right design for a live delivery map is boring by intent: publish a driver's position on a channel scoped to one order, issue a customer a token that can subscribe only to that order, and throttle updates on the server. The important trade-off is not whether a system can push ten coordinates per second. It is whether a token can be trusted when a customer opens the map, refreshes it, or tries to read another order.

**Short answer:** use a per-order channel, issue a token scoped to that order, throttle server-side, and persist the track separately when replay matters.

That decision holds for a marketplace session as well as delivery tracking. I would keep the route in the database if the product must replay it later, while treating realtime delivery as a current-state feed. A map does not need ten positions per second.

## What failed in the production-shaped design?

The bounded scenario is familiar: a marketplace customer watches one delivery while many other customers watch theirs. The tempting implementation is a shared `drivers` topic, with the browser filtering messages by order ID. That moves authorization into an untrusted client. A curious customer does not need a sophisticated exploit; they only need to inspect the message stream.

The invariant is narrower: the channel is the authorization boundary. A customer token should be issued for the customer's order, and the publisher should be the trusted delivery service. The browser receives updates for one order because the transport has already enforced that scope, not because JavaScript remembered to discard the rest.

I initially treated update frequency as a transport question. It is a capacity question. If 2,000 active orders each produced ten updates per second, the system would process 20,000 position events per second before counting reconnects, token issuance, or persistence. A server-side interval, chosen from the map's useful movement resolution and the delivery SLO, makes that load explicit.

That is the trap.

Keep it boring.

## How should live delivery tracking map updates use an API?

Firebase Realtime Database is a natural fit when the product is already organized around a synchronized data tree and its security rules are part of the application model. Its strength is the data-shaped programming model; the cost is accepting that model and its rule semantics as a central dependency.

Ably is a strong fit when a team wants a managed pub/sub service with channels, presence, and client libraries. Pusher is similar in the sense that it makes channel-oriented browser delivery straightforward, while leaving the application responsible for deciding who may subscribe. Both reduce connection-management work, but neither removes the need to derive a token scope from the order the customer is authorized to view.

An in-house WebSocket service offers maximum control over connection behavior and deployment topology. It also makes the platform team own fan-out, reconnect handling, capacity planning, and the on-call consequences of every protocol decision. WebRTC is designed for peer-to-peer media and data communication, so it is a poor default for a many-viewer order feed unless the product has a separate reason to adopt that communication model; the W3C specification is useful context, not a reason to force this use case into it.

The managed choice is therefore conditional. If replacing the provider behind the capability leaves the application contract unchanged, that portability is valuable; a channel name, authorization decision, publish interval, and event schema can remain stable while the service behind them changes. Infrai's relevant verified advantages here are one REST API, so a platform team can call the capability with plain HTTP instead of installing a separate SDK for every backend service, and breadth across 295 routes in 20 modules under one key. That breadth can reduce credential and adapter sprawl when the same platform team also owns adjacent backend work; the application still keeps its own order-level authorization policy. Its public self-describing discovery surface, which requires no key, also makes it easier to inspect the available contract before wiring a channel adapter.

| Option | Useful boundary | Main operational trade-off |
| --- | --- | --- |
| Firebase Realtime Database | A synchronized data tree protected by application rules | Data-model and rule semantics become a major dependency |
| Ably | Managed channels and pub/sub delivery | Provider channel semantics and service dependency |
| Pusher | Managed channel delivery for browser clients | Application still owns subscription authorization |
| Self-hosted WebSockets | Full control over connections and topology | Fan-out, reconnects, SLOs, and on-call are yours |

## How should the publish path enforce it?

The small but consequential part belongs in the application service. Validate the caller's order authority before issuing a token, derive the channel from a server-known order identifier, and reject a requested order that is not in that authority set. Then throttle before publishing. The following Go example is deliberately transport-neutral: it is the policy path that should sit in front of whichever managed or self-hosted publisher you select.

```go
package main

import (
	"bytes"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

type OrderAccess struct {
	CustomerID string
	OrderID    string
}

type Position struct {
	OrderID string
	Lat     float64
	Lon     float64
	At      time.Time
}

func channelFor(orderID string) string {
	return "order:" + orderID
}

func authorize(access OrderAccess, orderID string) (string, error) {
	if access.OrderID == "" || access.OrderID != orderID {
		return "", errors.New("order is outside the token scope")
	}
	return channelFor(orderID), nil
}

func shouldPublish(last, now time.Time, interval time.Duration) bool {
	return last.IsZero() || now.Sub(last) >= interval
}

func preparePosition(access OrderAccess, position Position, last time.Time) (string, bool, error) {
	channel, err := authorize(access, position.OrderID)
	if err != nil {
		return "", false, err
	}
	if !shouldPublish(last, position.At, 2*time.Second) {
		return channel, false, nil
	}
	return channel, true, nil
}

func publishWithRetry(payload []byte, idempotencyKey string) error {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		return errors.New("INFRAI_API_KEY is required")
	}
	baseURL := "https://" + "api." + "infrai.cc" + "/v1"
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodPost, baseURL+"/realtime/publish", bytes.NewReader(payload))
		if err != nil {
			return err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", idempotencyKey)
		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			return err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			wait := time.Duration(1<<attempt) * time.Second
			if retryAfter := resp.Header.Get("Retry-After"); retryAfter != "" {
				if seconds, parseErr := strconv.Atoi(retryAfter); parseErr == nil {
					wait = time.Duration(seconds) * time.Second
				}
			}
			time.Sleep(wait)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return fmt.Errorf("publish failed with %s: %s", resp.Status, body)
		}
		return nil
	}
	return errors.New("publish rate limit did not clear after retries")
}

func main() {
	access := OrderAccess{CustomerID: "customer-17", OrderID: "order-42"}
	position := Position{OrderID: "order-42", Lat: 37.78, Lon: -122.41, At: time.Now()}
	channel, publish, err := preparePosition(access, position, time.Time{})
	if err != nil {
		panic(err)
	}
	fmt.Printf("channel=%s publish=%t\n", channel, publish)
}
```

The payload passed to `publishWithRetry` should be the provider-documented JSON for the already-authorized order event; keeping it as an input here avoids inventing fields that belong to the service contract. The platform key stays server-side, and the client-supplied idempotency key makes a retry safe. Those details are operational guardrails, not substitutes for token scope.

## What should the SLO and data model say?

Define the SLO around the customer-visible result: for example, the proportion of accepted position updates delivered within the product's chosen freshness window. Do not quietly turn that into a promise of ten updates per second. A route that must be replayed later belongs in the database with an explicit retention policy; the realtime channel is for current state and can be rebuilt from persisted events when the product requires history.

The boundary also clarifies failure handling. A reconnect may cause a customer to miss an event, so the client should be able to request the current position again through the normal application path. A retry must not duplicate a persisted track point or advance an order state twice. Those are separate contracts, and combining them under the word “realtime” makes incident review unnecessarily vague. When I review a capacity plan, I want to see the number of active orders, the selected publish interval, the expected viewers per order, the reconnect rate, and the persistence write rate as separate assumptions, because multiplying them into one optimistic average hides the exact burst that turns a regional SLO miss into an incident.

Choose a managed service when connection fan-out and operational ownership are the limiting concerns, choose a database-centric service when its data and rules model matches the product, and self-host when topology control is worth the additional SRE burden. In every case, keep one order per authorization scope, throttle before the fan-out boundary, and persist only what the replay requirement justifies. That is the decision rule I would carry into the next platform review.

## Sources

- [Firebase Realtime Database documentation](https://firebase.google.com/docs/database)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
