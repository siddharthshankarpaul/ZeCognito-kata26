# Architecture Decision Records

Von Digitalis Estates, O'Reilly Architectural Katas 2026.

Scope: how tickets are issued, amended, refunded, and admitted at the gate, on an estate that needs to go from 5,000 to 15,000 daily visitors within three years over patchy WiFi.

## Template

Every record here follows the same structure: Status, Context, Decision, Diagram, Alternatives Considered, Consequences and Tradeoffs, Conclusion.

The brief asks for trade-off analysis, so the Alternatives and Consequences sections carry the real weight. Every rejected option names the specific reason it lost, and every consequence names what we gave up, not just what we gained.

The cryptographic verification problem, how a turnstile trusts a ticket with no network call, is already decided in [platform/ADR-002-offline-ticket-signing.md](../platform/ADR-002-offline-ticket-signing.md): Ed25519 signing, the private key held only in cloud KMS, self-contained signed payloads, family passes as an N-admit token, a pre-signed voucher pool for offline gate sales, and a local redemption ledger for replay prevention. The records here build on that decision rather than repeating it.

## The records

| Number | Decision | The tradeoff |
|---|---|---|
| [001](001-adr-admissions-as-a-modular-monolith.md) | Admissions is a modular monolith, one deployable over one database with enforced internal module boundaries, not microservices. | A single scaling axis and module-boundary discipline to maintain, in exchange for family-pass purchases, entitlements, and refunds that are correct by construction. |
| [002](002-adr-revocation-deny-list.md) | A deny list scoped to only today's valid passes, with two classes (hard `revoked` vs. `superseded` for upgrades), and a bounded staleness tolerance at the gate. | A small, bounded re-use window for a revoked pass, in exchange for a list that stays tiny regardless of growth and never denies someone who paid to upgrade. |
| [003](003-adr-reentry-and-gate-connectivity.md) | Day tickets allow unlimited same-day re-entry with no consumed state, and gates get real, funded connectivity as a deliberate investment rather than best-effort WiFi. | Concurrent use of one QR code isn't software-prevented, and gate wiring is capital spend outside the general hardware budget, in exchange for stateless verification and a deny list that's normally seconds old. |

## The through lines

Three ideas recur across these records.

1. **Ticketing is a correctness problem at a modest scale, not a throughput problem at a huge one.** [001](001-adr-admissions-as-a-modular-monolith.md) exists because tens of admissions a second doesn't justify the coordination cost microservices would add.
2. **Offline capability is a fallback, not the default design target, at the gate specifically.** Unlike the animal enclosures and footfall sensors that genuinely can't all be wired, [003](003-adr-reentry-and-gate-connectivity.md) treats the small, fixed set of gates as worth real investment, so full offline mode (`platform/ADR-002`) is what a gate falls back to, not how it normally runs.
3. **A ticket's signature proves it was issued, not that it's still valid.** [002](002-adr-revocation-deny-list.md) exists specifically to close that gap, without turning the deny list into a record of every ticket ever sold.

## Open questions

- The real revocation rate and average party size, both currently estimates, which [002](002-adr-revocation-deny-list.md)'s deny-list sizing depends on.
- Whether the estate's gate count is genuinely small enough for the conventional wiring [003](003-adr-reentry-and-gate-connectivity.md) assumes, this needs confirming against the actual site plan.
- Whether any product will ever need single-entry-only day tickets, which would require revisiting the re-entry model in [003](003-adr-reentry-and-gate-connectivity.md).
- How disputes over concurrent QR-code use at the gate are handled operationally, since [003](003-adr-reentry-and-gate-connectivity.md) leaves this to physical supervision rather than software.
