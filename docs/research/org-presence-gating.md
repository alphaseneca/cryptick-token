# Organization presence-gating — research notes

Status: **research only, nothing implemented.** Written to be discussed and
decided, not built against yet. If any of this moves forward it should get
its own PROJECT.md-style spec before code is written, the way the daemon's
architecture did.

## 1. The ask, restated

A designated "host" radio (the example given: a BLE dongle, brand "Haiku")
represents an organization being "open." Individual Cryptick Tokens belong
to members of that organization, but a token should only produce sensitive
output — OTP codes, FIDO assertions, PIV signatures — while the host is
present and online (e.g. the admin's laptop is unlocked and running). A
valid fingerprint match on the token alone is explicitly **not** meant to be
sufficient; presence of the host is a second, independent gate. If the host
goes dark, every member token in the fleet goes dark with it, regardless of
biometric match.

This is a fleet-wide, centrally-revocable presence requirement layered on
top of the existing per-device biometric gate — not a replacement for it.

## 2. Feasibility verdict

**Feasible as an additive gate, not as a drop-in to the existing daemon.**
The daemon's own PROJECT.md §21 rules out binding a network socket and rules
out anything that resembles fleet management living in the per-user,
zero-trust daemon — that daemon is deliberately scoped to one person's own
token. Org presence-gating is a different trust domain (an organization
admin controlling other people's hardware) and belongs in a **separate
component**, not bolted onto `cryptick-daemon`. The right shape is closer to
a second, small, clearly-labeled service plus a new firmware precondition —
described below — not a redesign of anything already settled.

## 3. Two architectures, in order of how hard they are to get right

### 3a. Connectionless BLE presence beacon (the obvious approach)

The host continuously **advertises** (not connects) a short manufacturer-data
payload while it considers the organization "open." Each token does a
periodic **passive scan** (no pairing, no connection) for that payload; if a
fresh, validly-signed beacon was seen within the last N seconds, privileged
operations are allowed, gated by fingerprint as today. No beacon → no
operations, independent of what the fingerprint sensor says.

This is the right instinct and it scales well: passive scanning has no
practical limit on simultaneous listeners, unlike BLE connections, which cap
out at a handful per radio. A fleet of 200 tokens all passively listening
for one beacon is a non-issue; 200 tokens trying to hold 200 simultaneous
GATT connections to one host radio is not realistic.

**The catch, and it's a real one:** BLE proximity by signal presence alone
is relay-attackable. This is the identical class of attack that has been
used for years against passive-entry car keys — an attacker with two radios
extends the beacon's effective range arbitrarily, so "I can hear the host"
stops meaning "I am physically near the host." A payload that's just
"signed bytes, present or not" is relayable over Wi-Fi or the internet in
milliseconds; signing the payload stops forgery, not relay, because the
attacker never needs to forge it — only retransmit it. Real distance-bounding
needs round-trip timing (UWB does this; plain BLE RSSI does not, RSSI is
trivially spoofable by changing TX power). **Nothing on the current PCB
does distance-bounding.** If "in the building" has to mean something an
attacker can't fake from the parking lot, this beacon model alone does not
get there — it gets to "the host is *somewhere* reachable," not "the token
is near the host."

Whether that matters depends on what's actually being defended. If the goal
is operational/administrative (stop people from generating codes after the
office officially closes, make revocation trivial by turning off one box)
rather than adversarial (stop a targeted attacker from faking presence),
plain beacon presence is a reasonable, cheap, implementable answer. If it
has to resist a motivated attacker, it isn't, as-is.

### 3b. Time-boxed "badge-in" capability (the hardened alternative)

Instead of continuous passive listening, the token gets a short-lived,
signed capability grant at a **deliberate** moment — tap the token against
a reader at the door, or an explicit BLE exchange when the user first shows
up — valid until a fixed expiry (end of shift, N hours, whatever policy
says) and stored in the token's own secure storage. No continuous radio
contact with the host is needed after that; the token just checks its own
RTC against the grant's expiry, which is consistent with the project's
existing "the device owns time" rule (ARCHITECTURE.md §2) rather than
fighting it.

This is strictly more implementable on current hardware (no new radio
behavior beyond what already exists for enrollment-style exchanges) and is
the standard enterprise badge-in pattern, which is also a point in its
favor — it's a well-understood trust model for the people who'd be using it,
not a novel one. It's still relayable in principle (someone could relay the
one-time badge exchange itself), but only during the narrow deliberate
window, not continuously and passively — a materially smaller attack
surface than 3a for a comparable amount of engineering effort.

**My recommendation if this gets built: start with 3b for the actual grant
mechanism, and treat continuous beacon presence (3a) as an optional
*additional* tightening for people who want "leaves the building == locks
immediately" rather than "grant expires at end of day."** The two aren't
mutually exclusive — a badge-in grant with a short TTL plus a beacon-refresh
that extends it while present is a reasonable middle ground.

## 4. What either option requires, concretely

- **A new, separate host-side service**, not `cryptick-daemon`. Something
  like a small "presence authority" app that runs on the admin's machine (or
  better, a dedicated always-on box — see §5) and either advertises (3a) or
  issues signed grants over a direct exchange (3b). It needs its own key
  material, separate from anything in the per-user daemon's trust boundary.
- **A new firmware precondition**, checked before CALCULATE / GET_ASSERTION
  / PIV sign operations, in addition to — not instead of — the existing
  fingerprint UV gate. Needs a per-org key or shared secret provisioned onto
  the token at enrollment; that provisioning flow is itself an open question
  (who signs it, how does a lost/revoked token get un-provisioned).
- **A freshness/replay mechanism** for whichever payload is used: a
  monotonic counter or timestamp the token checks against its own clock
  (again: device owns time), not just "did I see valid bytes."
- **An explicit policy for hysteresis** — a brief host dropout (laptop lid
  closed for 10 seconds, one missed advertisement) should not instantly
  lock out everyone in the building. Needs a deliberate grace window, not
  an accident of whatever polling interval gets picked.

## 5. Things worth deciding before building anything

- **Is the host meant to be a single person's laptop, or a dedicated
  always-on box?** As specified ("the boss's laptop is online"), the whole
  org's hardware keys go dark whenever that one laptop sleeps, reboots, or
  the boss is on a flight. That may be exactly the intended kill-switch
  behavior — it reads that way — but it's worth confirming it's intentional
  and not just the easiest thing to prototype with, since it's a real single
  point of failure for everyone else's authentication.
- **What exactly is "Haiku"?** If it's a specific chipset/module, its
  capabilities matter: does it support simultaneous central+peripheral
  (needs to talk to host software *and* advertise to the fleet at once),
  BLE 5.0 Long Range/Coded PHY (affects real building-scale range), and
  whatever's needed for whichever of 3a/3b gets picked. Worth pinning down
  before the host-side service gets designed around it.
- **Employment/legal angle**, flagged briefly since it's a real
  consideration and not just a technical one: a design where an employer's
  infrastructure can remotely and instantly disable an individual's
  authentication hardware is worth a conscious policy decision (and
  probably a lawyer's eyes, depending on jurisdiction), separate from
  whether it's technically buildable.

## 6. Suggested next step

Not implementation. A short written policy answering §5's three questions,
then a PROJECT.md-style spec for whichever of 3a/3b (or the hybrid) gets
picked, reviewed the same way the existing daemon architecture was before
any firmware or host-service code gets written against it.
