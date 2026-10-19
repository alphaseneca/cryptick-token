# Hardware-to-hardware credential sync

Status: **research only, nothing implemented** — this is the one open
question below. Multi-device management (the other half this doc
originally covered) shipped; see "Resolved" below.

## Resolved: multi-device management

This doc originally argued multi-device management (stable device
identity, a known-devices list, an active-device concept, a picker UI)
didn't exist and needed building. It's since been built: `cryptick-token-daemon`'s
`feature/credential-management-and-multidevice` branch implemented it
(`crates/daemon/src/devices.rs`'s `DeviceStore` — device identity,
`known_devices.json`, `record_seen`/`list`/active-device tracking —
plus `select_device` over IPC and a device picker in the extension's
`popup.js`). See that repo's `NOTES.md` for the detail. Nothing left to
decide here; this paragraph is kept only so this doc doesn't look
unfinished to a future reader.

## Hardware-to-hardware clone — the part that still needs a decision before any code

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

1. ~~Ship independent multi-registration ("industry-standard answer" above)
   now — it's cheap, secure, and solves the real user problem.~~ Still the
   right default to point users at; no code change needed, since it's just
   registering a credential twice.
2. ~~Ship basic multi-device management — bookkeeping, not a security
   question.~~ **Done** — see "Resolved" above.
3. Treat hardware-to-hardware clone as a separate, explicitly-scoped
   future feature, built only with a clear decision already made about
   the certification/claims tradeoff above — not bundled into "sync
   support" as if it were the same thing as independent multi-registration.
