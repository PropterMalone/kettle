# Kettle: Consent-Mediated Paging on AT Protocol

A design for programmatic out-of-band notification ("paging") between AT Protocol identities, without phone-number exchange.

**Status**: Design v0.4, May 2026. Pre-implementation; lexicon not yet locked. Sibling spec [Whistle](./WHISTLE.md) (broadcast announcement channels) ships paired; pair overview at [README](./README.md).

---

## 1. Problem

Organizers need *programmatic out-of-band reach*: a way for trusted senders to summon them via SMS, Signal, or push without phone-number exchange.

- Bluesky's 1:1 DMs (`chat.bsky.convo`) live at `chat.bsky.social` and go silent when the app is closed.
- Group DMs ship later, on the same centralized chat service.
- Existing organizing tools (Action Network, Spoke, Hustle, raw Twilio) keep phone numbers in centralized directories — high-value targets for ICE or state-level surveillance under subpoena or breach.

Kettle proposes *targeted summon*: each user keeps a private allow-list of DIDs that can summon them via the transport they choose. Senders never see recipient phone numbers; recipients revoke instantly.

Rapid-response coordination in adversarial environments forces the design. The primitive generalizes; the threat model shapes it.

**Scope.** Kettle is targeted summon: one named recipient per page, body encrypted to that recipient, consent gating on the receive side. Body privacy holds; the existence and metadata of every page (sender DID, recipient DID, category, timestamp) broadcast to anyone with firehose access. Kettle suits use cases where ephemeral body privacy is enough and metadata exposure is an acceptable cost — emergency rapid-response, mobilization. It does not suit use cases where the *existence* of the message must stay hidden, such as longitudinal sensitive coordination where the metadata trail itself is incriminating.

Kettle is *not* a broadcast announcement channel. Content-public, subscribers-private semantics — the inverse exposure profile — belong to the sibling primitive [Whistle](./WHISTLE.md), with a different lexicon and UI to prevent mode-confusion errors. The two compose for compartmentalized broadcast (§8).

**Constituency note.** No organizer in the prompting use case has reviewed this yet. Field validation gates lexicon-lock (§9 Q10).

---

## 2. Design principles

**PDS-anchored discovery, bridge-private trust.** The recipient's PDS publishes a *pointer* to their bridge (`community.paging.bridgeEndpoint`); the allow-list lives bridge-side, off the public firehose. The trust graph stays with the bridge operator — public records can't surface it under subpoena.

**Sender writes are public.** Every `community.paging.send` record sits on the firehose: sender DID, recipient DID, timestamp, and `category` are all observable. v1 encrypts the body and leaves the metadata in cleartext. Dual-DID identity separation (§7B) closes the leak in v1.1.

**Pluggable dispatch.** The protocol stays transport-agnostic — no phone numbers, SMS, Signal, or push at the protocol layer. Bridges watch a filtered Jetstream feed and dispatch through whatever transport they're configured for.

**Self-bridging is first-class.** Every tier supports self-hosting. Tier 3 (app-based push) drops phone numbers from the system entirely.

**Trust is bridge-bounded.** The bridge operator holds the cleartext allow-list, decryption keys, and dispatch credentials. Tiers 0/1 concentrate this risk; Tiers 2/3 distribute or remove it. §12 expands.

v1 leaves five risks open: forward secrecy, firehose traffic analysis, bridge-operator trust, recipient-device capture, and DID compromise. §12 treats each one rather than handwaving "future work."

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

The bridge subscribes to a filtered Jetstream feed (`wantedCollections=community.paging.send`, scoped to `targetDid`). Allow-list state lives at the bridge. The recipient configures it through an authenticated API to *their own* bridge — atproto-signed challenge or DPoP — never via PDS write.

When the recipient delegates dispatch to a third-party bridge (Tier 0/1), `community.paging.bridgeAuthorization` is the recipient-signed delegation token bridges present on demand. Self-hosted bridges (Tier 2/3) skip it.

---

## 4. Lexicons

Four lexicons in v1; two stubs sketched for v1.1. Field names follow atproto camelCase convention.

### `community.paging.send` (v1)

The sender posts to their own PDS. Public; body encrypted.

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
| `ciphertext` | string | Plaintext body encrypted to recipient's `defaultPubkey`. Real lexicon may switch to `{"type":"bytes"}`; encoding choice is §9 Q14. |
| `nonce` | string | AEAD nonce. |
| `category` | enum | `emergency` / `event` / `personal`. **Cleartext on firehose** — see §2. |
| `replyTo` | AT-URI, optional | Parent page. Resolution rule §9 Q12. |
| `createdAt` | datetime | ISO 8601 UTC. |

Plaintext body (decrypted; not itself an atproto record):

```json
{ "$type": "community.paging.body", "text": "ICE 5min out, 14th & Wabash", "facets": [] }
```

The body fits one SMS segment: 140 GSM-7 or 70 UCS-2 characters. SMS dispatch segments longer bodies at the bridge; push and Signal pass them through whole.

### `community.paging.bridgeEndpoint` (v1)

The recipient publishes this on their PDS. Pointer only — the allow-list stays elsewhere.

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

Rkey: `self`. `bridgeDid` is optional (omit for self-hosted-without-DID). `keyVersion` enables rotation: a new record with `keyVersion: 2` and a fresh `defaultPubkey` cuts off pages encrypted to the old key once senders' tooling refreshes.

The trust graph — who is permitted to reach me — is sensitive metadata. "X allows these 23 DIDs to page them" reveals organizing affiliations and rapid-response tree topology. Public PDS records are subpoenable; bridge-side state is not, except by targeting the bridge directly.

### `community.paging.bridgeAuthorization` (v1, optional)

The recipient signs a delegation: "this bridge endpoint may act as my page-receiver." Required when `bridgeDid` is set.

```json
{
  "$type": "community.paging.bridgeAuthorization",
  "bridgeDid": "did:plc:...",
  "scope": ["receive", "decrypt", "dispatch"],
  "expiresAt": "2026-10-29T23:00:00Z",
  "createdAt": "2026-04-29T20:00:00Z"
}
```

Rkey: `bridgeDid` (one record per delegated bridge). Record deletion revokes — eventually consistent across the firehose. For fast cutoff against a compromised bridge, also unwire `bridgeEndpoint` and rotate `defaultPubkey`.

### `community.paging.received` (v1, audit trail)

The bridge writes this after dispatch. Sender tooling subscribes to learn delivery status.

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

Per-sender `allowedHoursUtc` intersects with `defaultQuietHoursUtc`; the quiet-hours emergency-only override applies regardless. `transportOverride` beats global category-routing.

Bridges should support allow-list export/import for portability; format is §9 Q7.

### v1.1 stubs (open questions, not v1)

- **`community.paging.group`** — group fanout. Membership-list privacy is the open question (§9 Q11): group-creator's PDS, bridge-private, or MLS-grouped composing with Snug.
- **`community.paging.deadmansSwitch`** — periodic check-in with a payload to fire on miss. Polling architecture is the open question (§9 Q8): own-bridge vs designated escalation bridge.

---

## 5. Bridge tiers

Same software, four deployment shapes.

| Tier | Operator | Phone numbers in system? | Threat-model fit |
|---|---|---|---|
| 0 | Hosted public bridge | Yes (Twilio at operator) | Convenience default. Single subpoena/breach target for the user base. |
| 1 | Community / chapter | Yes (chapter's Twilio) | Chapter is the trust boundary. Operator carries personal legal exposure (§12). |
| 2 | Self-hosted | Yes (your Twilio sub-account) | Only you and Twilio. Subpoena targets you personally. |
| 3a | App push, FCM/APNs | **No** | Strong cryptographic posture, **platform-dependent**: Apple/Google can revoke. |
| 3b | App push, ntfy.sh self-hosted | **No** | Sovereign. Only tier with no platform kill-switch. |

The reference bridge ships as one Docker image, deployable as Tier 0/1/2 by configuration. The Tier 3 client (Expo/React Native) ships with both 3a and 3b as user-selectable transports.

---

## 6. Capabilities

### Transport menu (bridge-internal; lexicons unchanged)

- **SMS** — Twilio, Vonage, Plivo, Telnyx, Bandwidth, AWS SNS. ~$0.008/msg + carrier fees. US: A2P 10DLC registration (~$20 + 3-week wallclock).
- **Push** — APNs/FCM via the Kettle app, or ntfy.sh (open, self-hostable).
- **Signal** — signal-cli / signald (community-maintained; runs without Signal-org partnership, though cooperation helps).
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
- **Cross-bridge federation.** Any bridge can subscribe to the firehose for any recipient pointing at it; v1 reference omits multi-bridge orchestration UX.
- **Atproto-native moderation.** Bridges can apply labelers — drop labeled-spam DIDs, hold quiet-hours violations.

### Future (v1.1+)

- **Page-to-group fanout** — `community.paging.group`. Prompting use case; v1.1 priority.
- **Dead-man's switch** — `community.paging.deadmansSwitch` + polling.
- **Anonymous tip line** — wildcard allow-list + moderation queue UX.
- **Geofenced paging** — coordinates with `community.lexicon.location`.
- **Per-context routing across multiple bridges.**
- **Quorum paging / confirmation chains.**
- **Bridge-as-a-service for orgs** with multi-tenant management.
- **Page-back via SMS short code** — bridge translates SMS reply into a `community.paging.send` on the recipient's PDS.

---

## 7. Privacy modes

Four options for content + graph privacy. v1 ships A. §7B is the recommended v1.1 strong-privacy upgrade. §7C is the centralized fast path. §7D was considered and superseded by §7B.

### A. Encrypted body in public records (v1)

The sender encrypts to the recipient's `defaultPubkey` (asymmetric + AEAD). The public firehose carries `{targetDid, ciphertext, nonce, category, createdAt}`; only the recipient's bridge decrypts.

- Content private; graph public.
- No forward secrecy: the static `defaultPubkey` makes key compromise retroactive.
- Traffic-analysis surface: cleartext sender-DID + cleartext recipient-DID + cleartext `category` + timing.

### B. Dual-DID identity separation (v1.1, recommended)

Each user maintains two DIDs: a *persistent* DID (public-facing — Bluesky activity, follows, posts) and a *paging* DID used only as `targetDid` (and, ideally, as the publishing DID for sends). The paging DID is given out-of-band during the consent ceremony and never appears in any record that links it to the persistent DID.

- **Sender** publishes from `paging_did_S`.
- **Recipient** receives at `paging_did_R`, given to allow-listees out-of-band.
- Adversary on firehose sees `paging_did_S → paging_did_R`, encrypted body, category, timestamp. Without side information (former allow-listee, bridge compromise, behavioral correlation), neither party links to a persistent identity.

Properties:

- Anonymity is **structural**, not probabilistic. Privacy holds at any userbase size — fixes §7D's biggest weakness.
- No per-record decryption load on bridges; no DoS-by-adjacency.
- No epoch-rotation protocol; the DID itself is the privacy boundary. Rotate by issuing a new paging DID.
- Sender hidable too — same trick on the publish side.

Costs:

- Two DIDs per user; bridge config holds the `paging_did → persistent_did` mapping privately.
- New senders cannot discover `paging_did_R` via the firehose; consent grants travel a side channel (Signal, in person, org meeting). Fine for rapid-response organizing where grants happen at meetings anyway; un-discoverable for casual use.
- Pseudonym hygiene falls on the user. If `paging_did_S` also publishes random Bluesky content, adversary correlates by timing/style across DIDs. Strong separation requires the paging DID to publish *only* paging records — operational discipline most users won't keep.
- Recovery / rotation: social-recovery patterns developed for the persistent DID don't transfer to a DID the world doesn't know about.

Variants:

- **Single ephemeral paging DID per user** — all of X's pages come from `paging_did_X`. Adversary links X's pages to each other but not to `persistent_did_X`. Cheap baseline.
- **Pairwise DIDs** — fresh DID per (sender, recipient) pair. Adversary can't tell that pages from X are all the same sender. Maximum unlinkability; key explosion.

Wire format unchanged from §7A. Dual-DID ships as a deployment pattern + consent-ceremony documentation, not a new lexicon.

### C. `chat.bsky.convo` substrate (fast path)

Pages flow as structured DM payloads. Private from the public firehose; centralized at `chat.bsky.social`.

- Loses federation purity.
- Coordination required: the Bluesky team would need to allow a structured-payload extension to the chat lexicon.

### D. Sealed-sender bucket envelope (considered, superseded)

Earlier drafts proposed sealed-sender via bucket-sharding: each `community.paging.send` carries a bucket ID = `H(targetDid || epoch)` truncated to N bits; bridges subscribe to records in their user's current bucket and attempt asymmetric decryption per record. The construction would have followed Signal's [sealed-sender](https://signal.org/blog/sealed-sender/) or IETF [Oblivious Message Delivery](https://datatracker.ietf.org/wg/ohai/about/).

Why §7B dual-DID supersedes:

- Anonymity scales with userbase. On small networks (≪65k users with 16-bit buckets), most buckets contain 0 or 1 user, so the bucket ID becomes a near-identifier. §7B is structural; works at any scale.
- Bridges burn compute attempting decryption on records in their bucket. Linear with firehose volume; **DoS-by-adjacency** lets an attacker spam pages to anyone in the bucket and force every bridge in it to decrypt. §7B has no per-record cost.
- Epoch rotation is a non-trivial protocol surface (sender-side epoch lookup, bridge subscription retuning, missed-page risk on offline bridges). §7B uses the DID itself as the boundary — no epochs.
- Sender DID stays cleartext anyway. §7B can hide both ends.

Bucket-sharding survives only as a fallback if dual-DID's out-of-band consent ceremony proves untenable for the actual user base.

---

## 8. Composition

**With Smoke Signal events.** An RSVP record on Smoke Signal grants time-bounded auto-entry to the event-organizer's allow-list. RSVP yes to May Day → the organizer's DID gets `expiresAt = event.end + 1h` paging rights. Lexicon coordination is needed: Smoke Signal lives in `community.lexicon.*`; paging proposes the same namespace (§9 Q1).

**With Snug** (sibling project: encrypted MLS group chat for atproto, same author). A Snug group can be the *target* of a Kettle page (`community.paging.group`, v1.1). Snug protects in-group conversation; Kettle extends reach when the recipient is outside the chat. Lexicon-level consistency between the two is pre-v1 work.

**With Whistle (cell-structure relay).** The Kettle + [Whistle](./WHISTLE.md) pair composes for compartmentalized broadcast. A sender publishes a Whistle alert; one or more independent Kettle relays subscribe and translate the broadcast into per-recipient pages on their own allow-list. Trust splits cleanly:

- The sender doesn't know individual recipients — only the relays.
- Each relay knows its own members but not the sender's other audiences. A relay's compromise exposes only its own affinity group.
- Members trust their relay, not the sender directly.

Subpoena topology: subpoena on the sender yields a relay list, not a member list; subpoena on a relay yields only that affinity group's members. Combined with §7B dual-DID, members appear only as `paging_did`s, not persistent identities. The composition needs **≥2 relays per Whistle channel** for the privacy property to mean anything; a single-relay deployment collapses to "the relay is the audience."

**With labelers.** Bridge capabilities surface as public labels — `paging-enabled`, `paging-emergency-allowed`. The bridge URL is already public via `bridgeEndpoint`, so this leaks nothing new.

---

## 9. Open questions for atproto-protocol contributors

1. **Namespace placement.** Recommend `community.lexicon.paging.*`, alongside Smoke Signal. Push back if there's a reason to separate.

2. **Bridge auth pattern.** Is `community.paging.bridgeAuthorization` the right shape, or should bridges authenticate via standard atproto OAuth scopes against the recipient's PDS? Custom delegation is more federation-pure; OAuth is more pattern-matched.

3. **Privacy mode.** §7 A vs B vs C vs D. v1 ships A; v1.1 plans B (dual-DID). Specifically: is the out-of-band consent ceremony tractable enough to make B the default? If not, §7D bucket-sharding returns as a fallback.

4. **Jetstream production-readiness.** Documented as "not a stable protocol API." Acceptable for self-hosted bridges; production risk for hosted Tier 0. Is a stable subscription path planned?

5. **Cross-bridge federation semantics.** Does the protocol need explicit cross-bridge addressing, or does "pointer + firehose subscription" cover it?

6. **Firehose-level abuse mitigation.** Bridge-side filtering catches dispatched abuse; the records remain on the firehose. Anything to do at the substrate level?

7. **Bridge-side state portability.** Switching bridges means migrating the allow-list. Right answer: standard `community.paging.exportAllowList` XRPC, bridge admin-UI dump, or signed allow-list snapshots? Recipients without technical fluency need a clean migration path.

8. **Dead-man's-switch polling.** A recipient's own bridge can't fire after it goes offline. An external poller introduces third-party trust. Likely answer: "designated escalation bridge" with attestation; pattern wants protocol input.

9. **Anonymous tip line + atproto signing.** "Anonymous" here means "fresh, unprofiled DID." Strong enough for whistleblowing, or should the lexicon support a sealed-sender mode that strips sender DID at the cost of source authentication?

10. **Constituency validation.** Currently a designer's projection. Plan: 2–3 sit-downs with rapid-response organizers before lexicon-lock. The §6 tiering will probably reshuffle.

11. **`community.paging.group` membership privacy.** Group-creator's PDS as a public list, bridge-private, or MLS-grouped composing with Snug?

12. **`replyTo` allow-list resolution.** Proposed: each step uses the direct-parent author's allow-list (A→B uses B's; B's reply→A uses A's); multi-party threads cascade. Alternative: thread-root author. Want pushback.

13. **Replay-protection mandate.** Bridges MUST dedup by record CID. Should that be normative spec or implementation guidance?

14. **`<bytes>` encoding.** Multibase-string-over-JSON or CBOR-bytes? Slight lean toward multibase for portability across non-CBOR consumers.

---

## 10. v1 scope

**Build:**
- Five lexicons (`send`, `bridgeEndpoint`, `bridgeAuthorization`, `received`, `body` wire-format).
- Reference bridge — Docker image, Twilio + ntfy.sh dispatchers, deployable as Tier 0/1/2.
- Tier 3 reference client — Expo/React Native, push-only, ntfy.sh option.
- Hosted public bridge — single deployment of the reference.
- Smoke Signal RSVP integration — RSVP-yes auto-grants time-bounded paging right.

**Skip for v1:**
- Dual-DID identity separation (§7B); sealed-sender bucket fallback (§7D).
- `community.paging.group`, `deadmansSwitch`.
- MLS-style key rotation (v1 ships periodic static-key rotation only).
- Encrypted-blob attachments (text-only).
- Multi-device key sync.
- Multi-bridge orchestration UX.

**Estimate.** ~3 weeks for the protocol + reference bridge; ~2 weeks for the Tier 3 client (assumes already-bootstrapped Expo project). A2P 10DLC compliance for Tier 0 runs parallel — multi-week wallclock; Tier 3b sidesteps it. Biggest risks: Jetstream stability (§9 Q4) and A2P brand/campaign approval timing.

---

## 11. Comparisons

**vs. Signal.** Signal does end-to-end encrypted 1:1 and group messaging. Kettle is a paging primitive, not a chat replacement — it can dispatch *to* Signal as one transport. Signal handles the conversation; Kettle decides who can start one.

**vs. Action Network / Spoke / Hustle.** Centralized SaaS with phone-number databases. Kettle puts the directory at the bridge — self-hostable, federated, or absent in Tier 3. Trade-off: existing tools work today and have institutional buy-in.

**vs. Twitter SMS-tweet (deprecated).** SMS-tweet was opt-in broadcast — public phone-number list, fanout. Kettle is bilateral consent-mediated summon.

**vs. Bluesky group DMs (planned).** Bluesky's will live at `chat.bsky.social`. Kettle is federated. Adjacent, not overlapping: Bluesky's are *chat*; Kettle is *out-of-band notification*. Threading via `replyTo` makes Kettle group-DM-shaped in some emergent ways, but it remains a paging primitive — pitch it as that, not a chat replacement.

**vs. Whistle (sibling primitive).** Inverse exposure profile. Kettle = body-private + recipient-public-via-`targetDid`; Whistle = content-public + subscribers-private. Different primitives in the same family, deliberately separated to prevent mode-confusion errors. Composes cleanly via the cell-structure relay pattern (§8). See [`WHISTLE.md`](./WHISTLE.md).

---

## 12. Threat model

The risks v1 leaves unsolved, named.

### Adversaries considered

- **State-level surveillance with firehose access** (ICE, DHS, federal LE). Sees every `community.paging.send` record. Without §7B dual-DID protection, sees sender DID + recipient DID + cleartext `category` + timing — enough to reconstruct organizational structure and detect mobilization events without ever decrypting content.
- **Subpoena on the bridge operator.** Yields the cleartext allow-list, dispatch destinations, and any retained content.
- **Compromised member device** (post-arrest, lost phone). Reveals that member's allow-list state and locally-decrypted page history.
- **Adversarial fellow-organizer or turned member.** A member with paging rights can flood, exfiltrate, or impersonate.

### Unmitigated in v1

- **Bridge operator turned hostile.** The bridge sees the entire local trust graph and operates dispatch credentials. Tiers 0/1 concentrate this risk. Multi-bridge sharding is a v1.1 mitigation.
- **Recipient-device capture by intimate-partner adversary.** Tier 3 push lights up lock screens; bridge credentials live in the OS keychain. v1 ships no panic-wipe, duress-PIN, or silent-disable. For organizers escaping abuse — overlap with tenant-organizing, mutual-aid, abortion-network contexts — this is more likely than ICE for the median user. v1.1 should ship duress mode.
- **Replay attacks.** Cleartext records are permanent on the firehose. Bridges MUST dedup by record CID; without dedup an adversary can replay old emergency pages to trigger false mobilizations.
- **DID compromise.** A turned organizer's DID retains paging rights until manually revoked. Mitigation: short `expiresAt` on entries.
- **Fresh-DID anonymity ≠ network-level anonymity.** Anonymous tip-line use cases also need Tor-equivalent network anonymity — atproto PDS registration leaks IP, timing, and writing-pattern correlations.

### Out of scope

- Quantum-future cryptanalysis (rotate when atproto-substrate does).
- Coordinated DoS at firehose level (substrate concern).
- App-store revocation: Apple/Google can pull the Tier 3 mobile app. Sovereign push (Tier 3b ntfy.sh) is the v1 escape.

### Legal exposure for Tier 1 operators (not legal advice)

A chapter running a Tier 1 bridge for ICE-coordination support is operating an electronic communications service that knowingly facilitates anti-enforcement activity. Live questions: [Stored Communications Act](https://www.law.cornell.edu/uscode/text/18/2701), [ECPA](https://www.law.cornell.edu/uscode/text/18/2510), A2P 10DLC carrier disclosure, conspiracy liability for the operator personally, state-level "hindering apprehension" statutes.

Reference bridge defaults:

- Zero retention on dispatched message bodies.
- Allow-list export-only — no centralized sync to operator-controlled storage.
- Subpoena-response posture: minimum legal disclosure, notification to affected DIDs where permitted.
- `--paranoid` mode: ephemeral allow-list (RAM-only, lost on restart), no logs, no telemetry.

Operators should consult [EFF](https://www.eff.org) or the [National Lawyers Guild](https://www.nlg.org) before deployment. The operator carries the legal exposure regardless of defaults.

---

## Appendix A: Project name

The name carries two meanings.

"Kettling" is the police tactic of containing protesters in a defined area — the inversion is *a tool that kettles communication into trusted channels*, used by the people otherwise being kettled. A kettle is also for tea: the small, private, warm conversations among people who already trust each other.

The protocol serves both: a paging primitive sharp enough to coordinate against a hostile state, and a quiet inbox for "are you free Saturday?" between comrades. Organizing infrastructure that's only sharp-edged misses the half of community that lives outside the fight.

If the name fails constituency validation (§9 Q10), rename is cheap — the lexicon namespace is the actual identity.

---

## License

This document carries a [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) license. Lexicons themselves will land under [CC0](https://creativecommons.org/publicdomain/zero/1.0/) on adoption into `community.lexicon.*`.
