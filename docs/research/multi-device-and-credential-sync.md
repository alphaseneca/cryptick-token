# Multiple tokens per person: management and credential sync

Status: **research only, nothing implemented.** Written to settle the
architecture decision before any code gets written against it, the same
way the presence-gating note was.

## 1. Multi-device management: doesn't exist today, confirmed in the code

Checked directly rather than assumed. `webauthn-host`'s USB discovery
(`crates/webauthn-host/src/usb.rs`, `UsbClient::try_connect`) enumerates
every HID interface matching the Cryptick VID/PID, sorts them by which
looks most like the real FIDO interface, and connects to whichever sorts
first. That sort is resolving *which interface of one physical token* to
talk to (a token exposes more than one HID interface) — it has no concept
of a second, independent physical token. If you plug in two Cryptick
Tokens at once today, the host will find both devices' interfaces mixed
together in that one list and just connect to whichever wins the sort. No
device identity, no "known devices" list, no "switch to my other key"
action exists anywhere in the daemon, the CLI, or the extension.

### What real support requires, minimum

1. **A stable device identity.** The USB/BLE descriptors already carry a
   serial number field (standard HID/USB descriptor data) — confirm it's
   actually populated and unique per unit on this board (don't assume;
   check a real descriptor dump). That becomes the key for everything
   below.
2. **A known-devices list**, kept by the daemon (not the extension — the
   extension is a thin client over the daemon, per the existing
   architecture), keyed by that identity: a friendly name, last-seen
   transport, maybe which one is "default."
3. **An explicit "active device" concept** exposed over IPC, so `tools/cryptick`
   and the extension both show the same answer to "which key am I talking
   to right now" and both can switch it.
4. **UI**: a device picker (CLI flag, extension dropdown) wherever an
   operation needs to target a specific one, rather than "whichever sorts
   first."

None of this is security-sensitive by itself — it's bookkeeping. It's a
real feature to build, not a quick fix, but there's no hard design
question blocking it the way there is for §2.

## 2. "Encrypted sync between two physical tokens" — the part that needs a decision before any code

### Restating the ask

Register credentials once, then move them (or a usable copy of them) onto
a second physical Cryptick Token, with the transfer itself happening as an
encrypted exchange generated and consumed entirely in hardware — never a
plaintext dump of key material that could be extracted or reverse
engineered off either device or off the wire.

That's a well-posed security requirement, not a naive one. The problem
isn't the ask — it's that it runs directly into what a FIDO2 resident
credential's private key *is*, by design.

### Why this isn't just an engineering gap

A FIDO2 authenticator's whole security claim — the thing that makes it
phishing-resistant and distinguishes it from a password or a TOTP seed —
is that the private key for a credential is generated inside one specific
authenticator and never leaves it, in any form, under any ceremony, full
stop. That's not a missing feature in this firmware; it's the FIDO2
specification's own non-negotiable property, and it's the basis for FIDO
Alliance L1/L2 certification (a certified authenticator must attest that
key material is non-exportable). The moment there's a documented,
supported path for a private key to leave the device — encrypted,
hardware-generated, dual-presence-gated, doesn't matter — the device can
no longer truthfully make that claim, because now the key's security
depends on everything that touches it along that path, not just on the
device's own secure boundary. A wrapped/encrypted export is a smaller
attack surface than a plaintext one, but it is not zero, and "the private
key, under some ceremony, can leave the hardware" and "the private key
cannot leave the hardware" are a different claim to a relying party and to
a certification body, even when the former is implemented well.

This matters for this project specifically because PROJECT.md (the
daemon's own spec) already has a whole settled-dead-ends table (§19) and a
security-invariants section (§15) built on exactly this kind of
non-negotiable property — the pattern here is "don't reopen a hard
boundary without new evidence," and key export is the same category of
boundary as the ones already settled there.

### The industry-standard answer, and why it's not a cop-out

The actual, widely-deployed answer to "what if I lose my key" for
non-syncable hardware authenticators is: **register a second, independent
credential on the second device with every relying party, as its own
backup authenticator.** Not a copy of the first credential — a second
credential, its own keypair, generated on-device the normal way. This is
what YubiKey's own guidance tells customers to do, and it's why most
WebAuthn registration flows support registering more than one authenticator
per account. It fully solves "I lost my primary key" without touching the
non-export guarantee at all, works with zero new protocol or firmware
work (`authenticatorMakeCredential` on each device, run twice, once per
relying party, is all it is), and costs the user nothing but doing the
registration ceremony twice.

**This is my recommendation as the first thing to actually build**, ahead
of anything below — it's secure, it's already fully supported by the
existing firmware and daemon, and it's the backup story most users
actually need.

### If a true hardware-to-hardware transfer is still wanted anyway

It's not impossible to do honestly — some enterprise hardware (and a few
hardware wallets) do offer an explicit "clone to a backup unit" ceremony,
and if this project wants that, here's the shape that doesn't lie about
what it is:

- An authenticated key-agreement handshake (ECDH) directly between the two
  tokens' secure elements — the host/daemon relays bytes but never holds
  key material in a form it can read, matching the zero-trust boundary
  already established for the daemon.
- Explicit physical presence confirmed on **both** devices for the same
  ceremony (not just the source), so a compromised host can't trigger it
  unattended.
- A dedicated, clearly-separate credential class (or an explicit flag on
  this feature) so relying parties and the project's own documentation
  never claim FIDO-L1/L2-equivalent non-export guarantees for credentials
  that went through this path — don't let the marketing get ahead of what
  was actually built.
- Rate-limited / one-shot per ceremony, logged on-device, not something
  that can run silently or repeatedly.

This is a real, multi-week protocol-and-firmware design effort, not a
patch — closer in size to the PIV-over-BLE work than to a UI fix.

### What I'd actually do, in order

1. Ship independent multi-registration (§2, "industry-standard answer")
   now — it's cheap, secure, and solves the real user problem.
2. Ship basic multi-device management (§1) — bookkeeping, not a security
   question.
3. Treat hardware-to-hardware clone as a separate, explicitly-scoped
   future feature, built only with a clear decision already made about
   the certification/claims tradeoff above — not bundled into "sync
   support" as if it were the same thing as #1.
