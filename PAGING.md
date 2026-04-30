# Kettle: Consent-Mediated Paging on AT Protocol

A design for programmatic out-of-band notification ("paging") between AT Protocol identities, without phone-number exchange.

**Status**: Design v0.3, April 2026. Pre-implementation; lexicon is not yet locked.

---

## 1. Problem

Organizers need *programmatic out-of-band reach* — a way for trusted senders to summon them via SMS, Signal, or push without phone-number exchange.

- Bluesky's 1:1 DMs (`chat.bsky.convo`) are centralized at `chat.bsky.social` and don't deliver out-of-band when the app is closed.
- Group DMs are roadmap-future and will be centralized at the same chat service.
- Existing organizing tools (Action Network, Spoke, Hustle, raw Twilio) require centralized phone-number directories — high-value subpoena/breach targets in adversarial threat models (ICE, state-level surveillance).

The inversion proposed here is *targeted summon*: each user maintains a private allow-list of DIDs that can summon them via whatever transport they prefer. Senders never learn recipient phone numbers; recipients revoke instantly.

The forcing function is rapid-response coordination in adversarial environments. The primitive generalizes, but the threat model is what shapes the design.

**Constituency note.** This is currently a designer's projection. Organizers in the prompting use case have not been consulted. Field validation is part of the path to lexicon-lock; see §9 Q10.

---

## 2. Design principles

**PDS-anchored discovery, bridge-private trust.** The recipient's PDS carries a *pointer* to their bridge (`community.paging.bridgeEndpoint`); the allow-list itself lives bridge-side, never on the public firehose. The trust graph is private to the bridge operator, not subpoenable from public records.

**Sender writes are public.** Every `community.paging.send` record sits on the firehose: sender DID, recipient DID, timestamp, and `category` are all observable. Body is encrypted; metadata isn't. Sealed-sender (§7 B) closes this in v1.1; v1 accepts the leak.

**Pluggable dispatch.** The protocol doesn't know about phone numbers, SMS, Signal, push, or any specific transport. Bridges are feed-generator-shaped watchers that subscribe to filtered Jetstream and dispatch via configured transport.

**Self-bridging is first-class.** No tier requires a centralized operator. Tier 3 (app-based push) eliminates phone numbers from the system entirely.

**Trust is bridge-bounded.** The bridge operator holds the cleartext allow-list, decryption capability, and dispatch credentials. Tiers 0/1 concentrate this risk; Tiers 2/3 distribute or eliminate it. §13 expands.

The risks v1 doesn't solve — forward secrecy, firehose traffic analysis, bridge-operator trust, recipient-device capture, DID compromise — are named and explained in §13 rather than waved at as "future work."

---

## 3. Architecture

```
Sender's PDS                       Recipient's bridge              Recipient
────────────                       ──────────────────              ─────────
Posts                              Subscribes filtered             
community.paging.send              Jetstream                       
{targetDid, ciphertext,            ────────────────────►           
 nonce, category}                  sees record                     
                                                                   
                                   Consults BRIDGE-PRIVATE         
                                   allow-list (senderDid,          
                                   rate, hours, category)          
                                                                   
                                   Dedups by record CID            
                                   (replay protection)             
                                                                   
                                   If allowed:                     
                                   decrypts body using             
                                   recipient's key                 
                                                                   
                                   Dispatches via configured       
                                   transport ─────────────────────► SMS / Signal / push / ntfy
                                                                   
                                   Writes                          ◄── recipient manages allow-list
                                   community.paging.received           via authenticated API to bridge
                                   record back to recipient's      
                                   PDS for audit trail             
```

Bridge subscribes to filtered Jetstream (`wantedCollections=community.paging.send` filtered on `targetDid`). Allow-list is bridge-side state; recipient configures via authenticated API (atproto-signed challenge or DPoP) to *their own* bridge — never via PDS write.

When dispatch is delegated to a third-party bridge (Tier 0/1), `community.paging.bridgeAuthorization` (signed by the recipient) is the delegation token bridges present on demand. Self-hosted bridges (Tier 2/3) don't need it.

---

## 4. Lexicons

Four lexicons in v1; two stubs sketched for v1.1. Field names are camelCase per atproto convention.

### `community.paging.send` (v1)

Sender posts to their own PDS. Public; body encrypted.

```json
{
  "$type": "community.paging.send",
  "targetDid": "did:plc:abc...",
  "ciphertext": "<multibase string>",
  "nonce": "<base64url string>",
  "category": "emergency",
  "replyTo": "at://did:plc:.../community.paging.send/...",
  "createdAt": "2026-04-29T23:00:00Z"
}
```

| Field | Type | Notes |
|---|---|---|
| `targetDid` | DID | Recipient. |
| `ciphertext` | string | Plaintext body encrypted to recipient's `defaultPubkey`. Real lexicon may prefer `{"type":"bytes"}`; encoding choice is §9 Q14. |
| `nonce` | string | AEAD nonce. |
| `category` | enum | `emergency` / `event` / `personal`. **Cleartext on firehose** — see §2. |
| `replyTo` | AT-URI, optional | Parent page. Resolution rule §9 Q12. |
| `createdAt` | datetime | ISO 8601 UTC. |

Plaintext body (decrypted, not itself an atproto record):

```json
{ "$type": "community.paging.body", "text": "ICE 5min out, 14th & Wabash", "facets": [] }
```

Bounded at 140 GSM-7 characters or 70 UCS-2 to fit one SMS segment; longer bodies segment at the bridge for SMS dispatch and pass through whole on push/Signal.

### `community.paging.bridgeEndpoint` (v1)

Recipient publishes on their PDS. Pointer only — the allow-list is *not* here.

```json
{
  "$type": "community.paging.bridgeEndpoint",
  "bridgeEndpoint": "https://bridge.example.org",
  "bridgeDid": "did:plc:...",
  "defaultPubkey": "<multibase public key>",
  "keyVersion": 1,
  "createdAt": "2026-04-29T20:00:00Z"
}
```

Rkey: `self`. `bridgeDid` optional (omit for self-hosted-without-DID). `keyVersion` enables rotation: a new record with `keyVersion: 2` and a fresh `defaultPubkey` cuts off pages encrypted to the old key once senders' tooling refreshes.

The trust graph (who is permitted to reach me) is sensitive metadata: knowing "X allows these 23 DIDs to page them" reveals organizing affiliations and rapid-response tree topology. Public PDS records are subpoenable; bridge-side state is not — at least, not without targeting the bridge specifically.

### `community.paging.bridgeAuthorization` (v1, optional)

Recipient signs a delegation: "this bridge endpoint may act as my page-receiver." Required when `bridgeDid` is set.

```json
{
  "$type": "community.paging.bridgeAuthorization",
  "bridgeDid": "did:plc:...",
  "scope": ["receive", "decrypt", "dispatch"],
  "expiresAt": "2026-10-29T23:00:00Z",
  "createdAt": "2026-04-29T20:00:00Z"
}
```

Rkey: `bridgeDid` (one record per delegated bridge). Revocation via record deletion is eventually consistent across the firehose; for fast cutoff against a compromised bridge, also unwire `bridgeEndpoint` and rotate `defaultPubkey`.

### `community.paging.received` (v1, audit trail)

Bridge writes after dispatch. Sender's tooling subscribes to learn delivery status.

```json
{
  "$type": "community.paging.received",
  "sourcePage": "at://did:plc:.../community.paging.send/...",
  "dispatchedAt": "2026-04-29T23:00:42.123Z",
  "transport": "sms"
}
```

### Bridge-side state (not a lexicon)

Allow-list schema is bridge-internal. Recipient-facing semantics:

```yaml
allowList:
  globalRateLimitPerDay: 20
  defaultQuietHoursUtc: { start: 3, end: 7 }   # half-open; only emergency category fires
  entries:
    - senderDid: did:plc:abc...
      rateLimitPerDay: 3
      allowedHoursUtc: { start: 13, end: 3 }   # wraps midnight: 13:00–02:59
      categories: [emergency, event]
      expiresAt: 2026-05-01T23:59:59Z
      transportOverride: signal
    - senderDid: did:plc:def...
      categories: [emergency, event, personal]   # rateLimitPerDay omitted = unlimited
```

Per-sender `allowedHoursUtc` is intersected with `defaultQuietHoursUtc`; quiet-hours emergency-only override applies regardless. `transportOverride` beats global category-routing.

Bridges should support allow-list export/import for portability; format is §9 Q7.

### v1.1 stubs (open questions, not v1)

- **`community.paging.group`** — group fanout. Membership-list privacy is the design open question (§9 Q11): on group-creator's PDS / bridge-private / MLS-grouped composing with Snug.
- **`community.paging.deadmansSwitch`** — periodic check-in with payload to fire on miss. Polling architecture is the open question (§9 Q8): own-bridge vs designated escalation bridge.

---

## 5. Bridge tiers

Same software, four deployment shapes.

| Tier | Operator | Phone numbers in system? | Threat-model fit |
|---|---|---|---|
| 0 | Hosted public bridge | Yes (Twilio at operator) | Convenience default. Single subpoena/breach target for the user base. |
| 1 | Community / chapter | Yes (chapter's Twilio) | Chapter is trust boundary. Operator faces personal legal exposure (§13). |
| 2 | Self-hosted | Yes (your Twilio sub-account) | Only you and Twilio. Subpoena targets you personally. |
| 3a | App push, FCM/APNs | **No** | Strong cryptographic posture, **platform-dependent**: Apple/Google can revoke. |
| 3b | App push, ntfy.sh self-hosted | **No** | Sovereign. The only tier with no platform kill-switch. |

The reference bridge ships as one Docker image deployable as Tier 0/1/2 via configuration. The Tier 3 client (Expo/React Native) ships with both 3a and 3b as user-selectable transports.

---

## 6. Capabilities

### Transport menu (bridge-internal; lexicons unchanged)

- **SMS** — Twilio, Vonage, Plivo, Telnyx, Bandwidth, AWS SNS. ~$0.008/msg + carrier fees. US: A2P 10DLC registration (~$20 + 3-week wallclock).
- **Push** — APNs/FCM via the Kettle app, or ntfy.sh (open, self-hostable).
- **Signal** — signal-cli / signald (community-maintained; no Signal-org partnership required, cooperation would be a bonus).
- **Matrix** — standard client-server API; E2E via Olm.
- **Discord / Slack / Telegram** — documented incoming webhooks or bot APIs.
- **Email** — SMTP. Email-to-SMS gateways exist but are dying.
- **Voice** — SIP via Asterisk + Telnyx or VoIP.ms.
- **Webhook** — POST to any user-provided URL (IFTTT, Home Assistant, custom bot).
- **XMPP / IRC** — old but open.

Adding a transport is ~50 lines of bridge code.

### v1 ships

- Single-recipient consent-gated paging.
- Allow-list management API (bridge-internal, recipient-managed).
- Per-sender + global rate limiting, quiet hours, category filters, expirations.
- Read/delivery receipts via `community.paging.received`.
- Time-locked pages — sender posts with optional `deliverAfter`; bridge holds.
- Replay protection — bridges MUST dedup by record CID.

### v1 protocol-supports (reference bridge skips)

- **Threaded conversation** via `replyTo` (resolution rule §9 Q12).
- **Cross-bridge federation.** Any bridge can subscribe to firehose for any recipient pointing at it; v1 reference doesn't ship multi-bridge orchestration UX.
- **Atproto-native moderation.** Bridges can apply labelers (drop labeled-spam DIDs, hold quiet-hours violations).

### Future (v1.1+)

- **Page-to-group fanout** — `community.paging.group`. Prompting use case; v1.1 priority.
- **Dead-man's switch** — `community.paging.deadmansSwitch` + polling.
- **Anonymous tip line** — wildcard allow-list + moderation queue UX.
- **Geofenced paging** — coordinates with `community.lexicon.location`.
- **Per-context routing across multiple bridges.**
- **Quorum paging / confirmation chains.**
- **Bridge-as-a-service for orgs** with multi-tenant management.
- **Public alert channels** with subscriber rate caps.
- **Page-back via SMS short code** — bridge translates SMS reply into a `community.paging.send` on the recipient's PDS.

---

## 7. Encryption

Three options for body privacy. v1 ships A.

### A. Encrypted body in public records (v1)

Sender encrypts to recipient's `defaultPubkey` (asymmetric + AEAD). Public firehose carries `{targetDid, ciphertext, nonce, category, createdAt}`; only recipient's bridge decrypts.

- Content private; graph public.
- No forward secrecy: static `defaultPubkey`, retroactive on key compromise.
- Traffic-analysis surface: cleartext sender-DID + cleartext `category` + timing.

### B. Sealed-sender envelope (v1.1+)

Following Signal's [sealed-sender](https://signal.org/blog/sealed-sender/) or IETF [Oblivious Message Delivery](https://datatracker.ietf.org/wg/ohai/about/) — construction choice TBD. Sender publishes to a paging mailbox without revealing recipient in cleartext; bridges shard by `H(targetDid || epoch)` and attempt decryption per record in their bucket.

- Content + graph + category all private.
- Per-session ephemeral keys give forward secrecy.
- Cost: bucketed decryption attempts. With 16-bit bucketing each bridge sees ~1/65k of firehose; benchmarkable.

### C. `chat.bsky.convo` substrate (fast path)

Pages flow as structured DM payloads. Private from public firehose; centralized at `chat.bsky.social`.

- Loses federation purity.
- Coordination required: Bluesky team would need to allow a structured-payload extension to the chat lexicon.

---

## 8. Composition

**With Smoke Signal events.** RSVP record on Smoke Signal grants time-bounded auto-entry to the event-organizer's allow-list. RSVP yes to May Day → organizer's DID gets `expiresAt = event.end + 1h` paging rights. Lexicon coordination needed: Smoke Signal lives in `community.lexicon.*`; paging proposes the same namespace (§9 Q1).

**With Snug** (sibling project: encrypted MLS group chat for atproto, same author). A Snug group can be the *target* of a Kettle page (`community.paging.group`, v1.1). Snug protects in-group conversation; Kettle extends reach when the recipient isn't in the chat. Lexicon-level consistency between the two is pre-v1 work.

**With labelers.** Bridge capabilities surface as public labels (`paging-enabled`, `paging-emergency-allowed`); the bridge URL is already public via `bridgeEndpoint`, so this leaks nothing new.

---

## 9. Open questions for atproto-protocol contributors

1. **Namespace placement.** Recommendation: `community.lexicon.paging.*`, coordinating with Smoke Signal (already in that namespace). Push back if there's a reason to separate.

2. **Bridge auth pattern.** Is `community.paging.bridgeAuthorization` the right shape, or should bridges authenticate via standard atproto OAuth scopes against the recipient's PDS? Custom delegation is more federation-pure; OAuth is more pattern-matched.

3. **Encryption substrate.** §7 A vs B vs C. v1 ships A. Specifically: is sealed-sender's bucket-attempt-decrypt cost acceptable at firehose scale?

4. **Jetstream production-readiness.** Documented as "not a stable protocol API." Acceptable for self-hosted bridges; production risk for hosted Tier 0. Is a stable subscription path planned?

5. **Cross-bridge federation semantics.** Does the protocol need explicit cross-bridge addressing, or is "pointer + firehose subscription" sufficient?

6. **Firehose-level abuse mitigation.** Bridge-side filtering catches dispatched abuse; the records remain on the firehose. Anything to do at the substrate level?

7. **Bridge-side state portability.** Switching bridges means migrating the allow-list. Right answer: standard `community.paging.exportAllowList` XRPC, bridge admin-UI dump, or signed allow-list snapshots? Recipients without technical fluency need this to not be a brick wall.

8. **Dead-man's-switch polling.** Recipient's own bridge can't fire after it goes offline. External-poller introduces third-party trust. Probably "designated escalation bridge" with attestation; pattern wants protocol input.

9. **Anonymous tip line + atproto signing.** "Anonymous" here means "fresh, unprofiled DID." Strong enough for whistleblowing, or should the lexicon support a sealed-sender mode that strips sender DID at the cost of source authentication?

10. **Constituency validation.** Currently a designer's projection. Plan: 2–3 sit-downs with rapid-response organizers before lexicon-lock. Will probably reshuffle the §6 tiering.

11. **`community.paging.group` membership privacy.** Group-creator's PDS as a public list, bridge-private, or MLS-grouped composing with Snug?

12. **`replyTo` allow-list resolution.** Proposed: direct-parent author's allow-list governs each step (A→B uses B's; B's reply→A uses A's); multi-party threads cascade. Alternative: thread-root author. Want pushback.

13. **Replay-protection mandate.** Bridges MUST dedup by record CID. Should that be normative spec or implementation guidance?

14. **`<bytes>` encoding.** Multibase-string-over-JSON or CBOR-bytes? Slight lean toward multibase for portability across non-CBOR consumers.

---

## 10. v1 scope

**Build:**
- Five lexicons (`send`, `bridgeEndpoint`, `bridgeAuthorization`, `received`, `body` wire-format).
- Reference bridge — Docker image, Twilio + ntfy.sh dispatchers, deployable as Tier 0/1/2.
- Tier 3 reference client — Expo/React Native, push-only, ntfy.sh option.
- Hosted public bridge — single deployment of reference.
- Smoke Signal RSVP integration — RSVP-yes auto-grants time-bounded paging right.

**Skip for v1:**
- Sealed-sender envelope (§7 B).
- `community.paging.group`, `deadmansSwitch`.
- MLS-style key rotation (v1 does periodic static-key rotation only).
- Encrypted-blob attachments (text-only).
- Multi-device key sync.
- Multi-bridge orchestration UX.

**Estimate.** ~3 weeks for the protocol + reference bridge; ~2 weeks for the Tier 3 client (assumes already-bootstrapped Expo project). A2P 10DLC compliance for Tier 0 is parallel multi-week wallclock; Tier 3b sidesteps it. Biggest risks: Jetstream stability (§9 Q4) and A2P brand/campaign approval timing.

---

## 11. Comparisons

**vs. Signal.** Signal does end-to-end encrypted 1:1 and group messaging. Kettle is *not a chat replacement* — it's a paging primitive that can dispatch *to* Signal as one transport. Signal handles the conversation; Kettle decides who's allowed to start one.

**vs. Action Network / Spoke / Hustle.** Centralized SaaS with phone-number databases. Kettle puts the directory at the bridge (self-hostable, federated, or absent in Tier 3). Trade-off: existing tools are operational today and have institutional buy-in.

**vs. Twitter SMS-tweet (deprecated).** SMS-tweet was opt-in broadcast (public phone-number list, fanout). Kettle is bilateral consent-mediated summon.

**vs. Bluesky group DMs (planned).** Bluesky's will be centralized at `chat.bsky.social`. Kettle is federated. Adjacent, not overlapping: Bluesky's are *chat*; Kettle is *out-of-band notification*. Threading via `replyTo` makes Kettle group-DM-shaped in some emergent ways but it's not a chat replacement and shouldn't be pitched as one.

---

## 12. Threat model

The risks v1 doesn't fully solve, named.

### Adversaries explicitly considered

- **State-level surveillance with firehose access** (ICE, DHS, federal LE). Sees every `community.paging.send` record. Without sealed-sender, sees sender DID + recipient DID + cleartext `category` + timing — sufficient to reconstruct organizational structure and detect mobilization events without ever decrypting content.
- **Subpoena on the bridge operator.** Yields cleartext allow-list, dispatch destinations, and any retained content.
- **Compromised member device** (post-arrest, lost phone). Reveals that member's allow-list state and locally-decrypted page history.
- **Adversarial fellow-organizer or turned member.** A member with paging rights can flood, exfiltrate, or impersonate.

### Acknowledged but not mitigated in v1

- **Bridge operator turned hostile.** The bridge sees the entire local trust graph and operates dispatch credentials. Tiers 0/1 concentrate this risk. Multi-bridge sharding is a v1.1 mitigation.
- **Recipient-device capture by intimate-partner adversary.** Tier 3 push lights up lock screens; bridge credentials live in OS keychain. v1 has no panic-wipe, duress-PIN, or silent-disable. For organizers escaping abuse — overlap with tenant-organizing, mutual-aid, abortion-network contexts — this is more likely than ICE for the median user. v1.1 should ship duress mode.
- **Replay attacks.** Cleartext records are permanent on the firehose. Bridges MUST dedup by record CID; without dedup an adversary can replay old emergency pages to trigger false mobilizations.
- **DID compromise.** A turned organizer's DID retains paging rights until manually revoked. Mitigation: short `expiresAt` on entries.
- **Fresh-DID anonymity ≠ network-level anonymity.** Anonymous tip-line use cases require Tor-equivalent network anonymity in addition to a fresh DID — atproto PDS registration leaks IP, timing, and writing-pattern correlations.

### Not considered

- Quantum-future cryptanalysis (rotate when atproto-substrate does).
- Coordinated DoS at firehose level (substrate concern).
- App-store revocation: Apple/Google can pull the Tier 3 mobile app. Sovereign push (Tier 3b ntfy.sh) is the v1 escape.

### Legal exposure for Tier 1 operators (not legal advice)

A chapter running a Tier 1 bridge for ICE-coordination support is operating an electronic communications service knowingly facilitating anti-enforcement activity. Live questions: [Stored Communications Act](https://www.law.cornell.edu/uscode/text/18/2701), [ECPA](https://www.law.cornell.edu/uscode/text/18/2510), A2P 10DLC carrier disclosure, conspiracy-theory liability for the operator personally, state-level "hindering apprehension" statutes.

Reference bridge defaults:

- Zero retention on dispatched message bodies.
- Allow-list export-only (no centralized sync to operator-controlled storage).
- Subpoena-response posture: minimum legal disclosure, notification to affected DIDs where permitted.
- `--paranoid` mode: ephemeral allow-list (RAM-only, lost on restart), no logs, no telemetry.

Operators should consult [EFF](https://www.eff.org) or the [National Lawyers Guild](https://www.nlg.org) before deployment. Defaults don't insulate the operator from liability.

---

## Appendix A: Project name

The name carries two meanings.

"Kettling" is the police tactic of containing protesters in a defined area — the inversion is *a tool that kettles communication into trusted channels*, used by the people otherwise being kettled. A kettle is also for tea: the small, private, warm conversations among people who already trust each other.

The protocol serves both: a paging primitive sharp enough to coordinate against a hostile state, and a quiet inbox for "are you free Saturday?" between comrades. Organizing infrastructure that's only sharp-edged misses the half of community that isn't a fight.

If the name doesn't survive the constituency-validation pass (§9 Q10), rename is cheap — the lexicon namespace is the actual identity.

---

## License

This document is released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Lexicons themselves are intended for [CC0](https://creativecommons.org/publicdomain/zero/1.0/) on adoption into `community.lexicon.*`.
