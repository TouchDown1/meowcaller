<!-- Datasheet = three things only: the reference source VERBATIM, the Go envelope
     (signatures, no bodies), and implementation suggestions. No implementation. -->

# Datasheet: `meowcaller/call-lifecycle`

The candidate 1:1 call-state invariant and the engine boundary that must
preserve it when signaling and relay events arrive out of order. The state
machine below is a pinned implementation reference, not an official WhatsApp
protocol specification.

**Validation vector:** focused offline KATs in `engine_lifecycle_test.go` must pin
the event-to-phase sequence and prove that an early relay acknowledgement neither
advances the public call phase nor starts outbound RTP before peer acceptance.

**Reference pinned at:** `41095d4e6ba4610e054e9ede3af1d5e88a83faee`
(`whatsapp-rust/src/voip/session.rs`; verbatim retained by repository commit
`e5283bf`). Protocol-ordering evidence is the draft WACRG SIG-17 outgoing 1:1
flow pinned at `0114515cef5c0344a8a864f6ad5ff58e650550ed`; its own metadata marks
caller-side media promotion as an open question and whatsapp-rust support as
partial.

## Reference source (verbatim — authoritative excerpt)

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum CallDirection {
    Outgoing,
    Incoming,
}

/// Lifecycle phase of a call.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum CallPhase {
    Idle,
    Calling,
    Ringing,
    Connecting,
    Active,
    Ended,
}

/// Per-call signaling state. Transitions are validated so an out-of-order server message
/// can't silently advance a torn-down call.
#[derive(Debug, Clone)]
pub struct CallSession {
    pub call_id: String,
    pub peer_jid: Jid,
    pub call_creator: Jid,
    pub direction: CallDirection,
    pub is_video: bool,
    phase: CallPhase,
}

impl CallSession {
    pub fn new_outgoing(call_id: impl Into<String>, peer_jid: Jid, call_creator: Jid) -> Self {
        Self {
            call_id: call_id.into(),
            peer_jid,
            call_creator,
            direction: CallDirection::Outgoing,
            is_video: false,
            phase: CallPhase::Idle,
        }
    }

    pub fn new_incoming(call_id: impl Into<String>, peer_jid: Jid, call_creator: Jid) -> Self {
        Self {
            call_id: call_id.into(),
            peer_jid,
            call_creator,
            direction: CallDirection::Incoming,
            is_video: false,
            phase: CallPhase::Ringing,
        }
    }

    pub fn phase(&self) -> CallPhase {
        self.phase
    }

    pub fn is_active(&self) -> bool {
        self.phase == CallPhase::Active
    }

    pub fn is_ended(&self) -> bool {
        self.phase == CallPhase::Ended
    }

    /// Attempt a phase transition; returns false (no-op) if it is not legal from the
    /// current phase. `Ended` is reachable from anything except `Ended`.
    pub fn transition_to(&mut self, next: CallPhase) -> bool {
        let ok = match (self.phase, next) {
            (CallPhase::Ended, _) => false,
            (_, CallPhase::Ended) => true,
            (CallPhase::Idle, CallPhase::Calling) => self.direction == CallDirection::Outgoing,
            (CallPhase::Calling, CallPhase::Ringing) => true,
            (CallPhase::Ringing, CallPhase::Connecting) => true,
            (CallPhase::Connecting, CallPhase::Active) => true,
            // Idempotent self-transition is allowed.
            (a, b) if a == b => true,
            _ => false,
        };
        if ok {
            self.phase = next;
        }
        ok
    }
}
```

## Go envelope (signatures only)

```go
package meowcaller

func validCallPhaseTransition(
	direction CallDirection,
	current CallPhase,
	next CallPhase,
) bool

func (c *Call) transitionTo(next CallPhase) bool

func (m *engineCall) outgoingMediaReady() bool
```

## Implementation suggestions (guidance, not authoritative)

- Keep one transition predicate for both `CallSession.TransitionTo` and the live
  `Call`; do not maintain a validated and an unchecked phase machine.
- Preserve the canonical outgoing chain exactly:
  `Idle -> Calling -> Ringing -> Connecting -> Active -> Ended`.
- Map peer `preaccept` to `Ringing`, peer `accept` to `Connecting`, and the first
  authenticated inbound RTP packet to `Active`.
- Treat the WACRG SIG-17 rule that `preaccept` is ringing/early-media only and
  does not authorize protected media as a bounded implementation hypothesis,
  not settled official protocol truth. The specification is draft, its exact
  caller-side RTP promotion trigger is still open, and current whatsapp-rust
  also attaches its media engine on relay readiness rather than peer accept.
- Treat an early relay acknowledgement as retained transport readiness only. It
  must not advance the call phase or authorize outbound RTP.
- For an outgoing call, require peer acceptance in addition to call key and relay
  readiness before starting the media loop. Keep relay allocation separate from
  media emission if early transport preparation is still required.
- Do not copy stale wire constants from the WACRG call-offer page: it still shows
  an older capability byte while the pinned meowcaller and current whatsapp-rust
  offer builders use the newer captured value. This datasheet adopts only the
  lifecycle ordering hypothesis.
- Preserve the peer reject `reason` as redacted diagnostics before changing
  reject policy. A secondary-device `busy` reject and a primary-device non-busy
  reject are not interchangeable; do not ignore the latter.
- Keep `Ended` a sink. Late relay, preaccept, accept, or RTP events must not
  resurrect the call.
- KAT the observed out-of-order case explicitly:
  `offer -> relay ack -> preaccept -> accept -> first inbound RTP`, including
  `outbound RTP count == 0` before accept.
- Add order/idempotence KATs for relay-before-accept, accept-before-relay,
  duplicate relay/accept, and reject/terminate-before-accept. Incoming-call
  behavior must remain unchanged.
