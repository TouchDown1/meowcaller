# Call lifecycle TDD evidence

## Source and journey

The journey was derived from `datasheets/call-lifecycle.md`: as the caller, the
operator needs relay preparation and `preaccept` to leave protected outbound
media idle until the peer sends `accept`, so the target receives a normal ringing
call before media begins.

## RED and GREEN

- RED checkpoint: `4c88118`.
- RED command: `go test . -run
  '^TestOutgoingMediaWaitsForPeerAcceptWhenRelayArrivesFirst$' -count=1`.
- RED evidence: the test failed because media started from relay readiness before
  peer acceptance.
- GREEN command: `go test ./...`.
- GREEN evidence: every Go package passed after direct outgoing media was gated
  on peer acceptance.

## Test specification

| Guarantee | Test | Type | Result |
|---|---|---|---|
| Relay readiness and `preaccept` do not start direct outgoing media | `TestOutgoingMediaWaitsForPeerAcceptWhenRelayArrivesFirst` | integration | PASS |
| `accept` followed by relay readiness starts media | `TestOutgoingMediaStartsWhenRelayArrivesAfterPeerAccept` | integration | PASS |
| Reject before accept is terminal and late events cannot resurrect media | `TestOutgoingRejectBeforeAcceptPreventsLateMedia` | integration | PASS |
| Incoming media does not inherit the outgoing acceptance gate | `TestIncomingMediaDoesNotRequirePeerAccept` | regression | PASS |
| Group acceptance retains its independent prerequisites | `TestOutgoingGroupAcceptRemainsIndependentOfDirectMediaGate` | regression | PASS |

## Validation and gaps

- `go vet ./...`: PASS.
- `go build ./...`: PASS.
- `go test ./...`: PASS.
- Changed-function coverage: `onAccept` 94.9%; `maybeStartMedia` 80.9%.
- Existing root-package coverage is 43.5%; raising unrelated legacy coverage is
  outside this one-module correction.
- The race build could not run because this workspace currently reports
  `CGO_ENABLED=0`; the readiness flag is accessed under `engine.mu`.
- Live recipient ringing is not claimed by the offline suite and requires the
  guarded single-target canary.
