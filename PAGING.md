# Kettle: Consent-Mediated Paging on AT Protocol

A design for programmatic out-of-band notification ("paging") between AT Protocol identities, without phone-number exchange.

**Status**: Design v0.3, April 2026.
**Audience**: feedback from atproto-protocol contributors before lexicon-lock.
**Changes since v0.2**:
1. **Lexicon fields converted to atproto-idiomatic camelCase** (was snake_case, would have read as "hasn't seen another lexicon").
2. **§4 expanded** with stub schemas for `received` (audit trail), and forward-references for `group` and `deadmansSwitch` (deferred to v1.1, surfaced as open questions).
3. **§7 Encryption: forward-secrecy and traffic-analysis caveats made explicit.** v1 ships with static recipient key + cleartext sender-DID + cleartext `category` on the firehose — these are real graph-reconstruction vectors and the doc now names them. Sealed-sender option B references real constructions (Signal sealed-sender, IETF OMD).
4. **§13 Threat model added** — bridge-operator-as-adversary, recipient-side device capture, legal exposure for chapter bridge operators. The honest version of "what isn't solved."
5. **§6 Capabilities tiered** into v1-ships / v1-protocol-supports / future — was a flat menu in v0.2.
6. **"Decentralized group DMs" framing pulled back** — was overclaim, conflicted with Bluesky group-DM roadmap. Now mentioned only in §11 Comparisons as an emergent property, not the design center.
7. **§9 Q1 (namespace placement) now proposes** `community.lexicon.paging.*` rather than punting.
8. **Project-name discussion moved to Appendix A** — was burying the lede before §1 Problem.

---

## 1. Problem

Organizers need *programmatic out-of-band reach* — a way for trusted senders to summon them via SMS, Signal, or push without phone-number exchange.

- Bluesky's 1:1 DMs (`chat.bsky.convo`) are centralized at `chat.bsky.social` and don't deliver out-of-band when the app is closed.
- Group DMs are roadmap-future and will be centralized at the same chat service. (Kettle solves an adjacent problem; see §11 Comparisons.)
- Existing organizing tools (Action Network, Spoke, Hustle, raw Twilio) require centralized phone-number directories — high-value subpoena/breach targets in adversarial threat models (ICE, state-level surveillance).

The inversion proposed here is *targeted summon*: each user maintains a private allow-list of DIDs that can summon them via whatever transport they prefer. Senders never learn recipient phone numbers. Recipients revoke instantly.

The forcing function is rapid-response coordination in adversarial environments — KC DSA running anti-ICE coordination is the prompting case. The primitive generalizes to any "I want to be reachable by these specific people, no one else" use case, but the threat model is what shapes the design.

**Constituency note.** This is currently a designer's projection, not a co-authored design. KC DSA members have not been consulted on whether this primitive matches their actual rapid-response practice. Field validation is part of the v0.3 → v0.4 path; see §9 Q10.

---

## 2. Design principles

**PDS-anchored discovery, bridge-private trust.** The recipient's PDS carries a *pointer* to their bridge (`community.paging.bridgeEndpoint`); the actual allow-list (who can page them, with what rate/category/hours rules) lives bridge-side, never on the public firehose. The trust graph is private to the bridge operator, not subpoenable from public records.

**Sender writes are public.** Every `community.paging.send` record sits on the public atproto firehose. The *act* of paging — sender DID, recipient DID, timestamp, `category` — is publicly observable; only the body content is encrypted. Sealed-sender envelope (§7 option B) is the principled long-term answer; v1 ships without it and accepts the metadata leak.

**Pluggable dispatch.** The protocol doesn't know about phone numbers, SMS, Signal, push, or any specific transport. That's bridge-internal — bridges are feed-generator-shaped watchers that subscribe to filtered Jetstream and dispatch via configured transport.

**Self-bridging is first-class.** No tier requires a centralized operator. Tier 3 (app-based push) eliminates phone numbers from the system entirely.

**Trust is bridge-bounded.** The bridge operator sees: cleartext allow-list, recipient's decryption capability, dispatch credentials (e.g., Twilio SID), all message content at dispatch time. The bridge is the operative attack surface for the threat model. Tiers 0/1 concentrate this risk; Tiers 2/3 distribute or eliminate it. §13 expands.

**Honest about what isn't solved**, by name (not "etc."):
- **No forward secrecy** — recipient's static `defaultPubkey` decrypts every page ever encrypted to it; key compromise is retroactive.
- **Traffic-analysis on cleartext sender-DID + `category` + timing** — graph and mobilization-event reconstruction without ever decrypting content.
- **Bridge-operator trust** — turned/breached bridges see the entire local trust graph and dispatch destinations.
- **Recipient-side device capture** — Tier 3 push lights up lock screens; bridge credentials live in the keychain. No panic-wipe in v1.
- **DID compromise** — a turned organizer's DID retains paging rights until manually revoked.

These are not hand-waved as "future work" — they are the v1 risk surface. Choose tier accordingly.

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

Sender writes to their own repo. Bridge subscribes to a filtered Jetstream stream (`wantedCollections=community.paging.send` filtered on `targetDid`). The allow-list is bridge-side state; recipient configures it by authenticated API call (atproto-signed challenge or DPoP) to *their own* bridge — never via PDS write.

When a recipient delegates dispatch to a third-party bridge (Tier 0 or 1), `community.paging.bridgeAuthorization` (signed by the recipient's atproto identity) is a delegation token the bridge presents on demand. For self-hosted bridges (Tier 2/3) where the recipient *is* the bridge operator, the authorization record is unnecessary.

The bridge is the only stateful service-side component. Bridge-private trust state is the only state worth subpoenaing — that subpoena targets the bridge, not the public PDS infrastructure.

---

## 4. Lexicons

Three records are sufficient for the v1 protocol; `received` is added as a fourth (audit trail). Two additional lexicons (`group`, `deadmansSwitch`) are sketched as v1.1 follow-ons and surfaced as open questions in §9.

All field names use **camelCase** per atproto lexicon convention (`app.bsky.*`, `chat.bsky.*`, `community.lexicon.*`).

### `community.paging.send` (v1 ships)

Sender posts to their own PDS. Public record on the firehose; body content is encrypted.

```json
{
  "$type": "community.paging.send",
  "targetDid": "did:plc:abc...",
  "ciphertext": "<base64url multibase string>",
  "nonce": "<base64url string>",
  "category": "emergency",
  "replyTo": "at://did:plc:.../community.paging.send/...",
  "createdAt": "2026-04-29T23:00:00Z"
}
```

| Field | Type | Notes |
|---|---|---|
| `targetDid` | string (DID) | Recipient. |
| `ciphertext` | string (multibase) | Plaintext body encrypted to recipient's `defaultPubkey`. Real lexicon will use `{"type":"bytes"}` (CBOR byte string, base64 over JSON) — encoding choice is §9 Q14. |
| `nonce` | string | AEAD nonce. |
| `category` | string (knownValues) | `emergency` / `event` / `personal`. **Cleartext on firehose** — see §2 traffic-analysis caveat. |
| `replyTo` | string (AT-URI), optional | Reference to parent page in a thread; semantics in §9 Q12. |
| `createdAt` | string (datetime) | ISO 8601 UTC. |

Plaintext (the decrypted body) follows a separate wire-format schema, not itself an atproto record:

```json
{
  "$type": "community.paging.body",
  "text": "ICE 5min out, 14th & Wabash",
  "facets": []
}
```

Plaintext text is bounded at 140 characters to fit GSM-7 SMS without segmentation; if the bridge encodes UCS-2 (any non-GSM-7 char), one segment holds 70 characters and the bridge will segment beyond that. v1 reference bridge will document this for SMS dispatch; other transports (Signal, push) carry full text.

### `community.paging.bridgeEndpoint` (v1 ships)

Recipient publishes on their PDS. Public — but a *pointer only*; the allow-list itself is not on PDS.

```json
{
  "$type": "community.paging.bridgeEndpoint",
  "bridgeEndpoint": "https://bridge.kcdsa.org",
  "bridgeDid": "did:plc:...",
  "defaultPubkey": "<multibase public key>",
  "keyVersion": 1,
  "createdAt": "2026-04-29T20:00:00Z"
}
```

Key (rkey): `self`. `bridgeDid` is optional (omit for self-hosted-without-DID). `keyVersion` enables rotation: recipient publishes a new record with `keyVersion: 2` and a new `defaultPubkey`; senders' tooling caches by version and refreshes when bridges report decryption-key-mismatch errors.

**Why not on PDS** (the v0.2 thesis): the trust graph (who is permitted to reach me) is sensitive metadata — knowing "Kevin allows these 23 DIDs to page him" reveals organizing affiliations, social structure, and rapid-response tree topology. Public PDS records are subpoenable, scrapeable, and indexable. Bridge-side state is not, or at least, not without targeting the bridge specifically.

### `community.paging.bridgeAuthorization` (v1 ships, optional)

Recipient signs a delegation: "this bridge endpoint may act as my page-receiver." Required when `bridgeDid` is set; bridges present this to challengers (e.g., a sender's tooling verifying it's talking to a legitimately-delegated bridge).

```json
{
  "$type": "community.paging.bridgeAuthorization",
  "bridgeDid": "did:plc:...",
  "scope": ["receive", "decrypt", "dispatch"],
  "expiresAt": "2026-10-29T23:00:00Z",
  "createdAt": "2026-04-29T20:00:00Z"
}
```

Key (rkey): `bridgeDid` (one record per delegated bridge). Revocation: delete the record (atproto record-deletion semantics — eventually consistent across firehose). For fast-cutoff against compromised bridges, the recipient should also unwire the bridge endpoint and rotate `defaultPubkey`; deletion alone is insufficient.

### `community.paging.received` (v1 ships, audit trail)

Bridge writes to recipient's PDS after dispatch. Public — establishes a delivery record.

```json
{
  "$type": "community.paging.received",
  "sourcePage": "at://did:plc:.../community.paging.send/...",
  "dispatchedAt": "2026-04-29T23:00:42.123Z",
  "transport": "sms"
}
```

Key (rkey): TID. Sender's tooling subscribes to recipient's `received` records to learn delivery status.

### Bridge-side state (not lexicons)

The allow-list is bridge-private state — schema is bridge-internal, not part of the protocol. Recipient-facing semantics:

```yaml
allowList:
  globalRateLimitPerDay: 20
  defaultQuietHoursUtc: { start: 3, end: 7 }   # half-open interval, only emergency category fires during quiet hours
  entries:
    - senderDid: did:plc:abc...
      rateLimitPerDay: 3
      allowedHoursUtc: { start: 13, end: 3 }   # wraps midnight: 13:00-23:59 + 0:00-2:59
      categories: [emergency, event]
      expiresAt: 2026-05-01T23:59:59Z
      transportOverride: signal
    - senderDid: did:plc:def...
      rateLimitPerDay: null                    # field omitted = unlimited
      categories: [emergency, event, personal]
```

Recipient manages via authenticated API to their own bridge (HTTPS + atproto-signed challenge, or DPoP). Bridges should support export/import for portability when switching bridges; export format is §9 Q7.

**Precedence rule**: per-sender `allowedHoursUtc` is intersected with `defaultQuietHoursUtc`; quiet-hours emergency-only override applies regardless of per-sender rules. `transportOverride` takes precedence over any global category-routing.

### Stub lexicons for v1.1 (not v1)

Sketched here so reviewers can see the trajectory; surfaced as open questions in §9.

- **`community.paging.group`** — defines group membership for fanout. Privacy properties of the membership list are the v1.1 design open question (PDS-public list / bridge-private / MLS-grouped composing with Snug). See §9 Q11.
- **`community.paging.deadmansSwitch`** — periodic check-in semantics with payload to fire on miss. Polling architecture is the v1.1 design open question (own-bridge-polls vs designated-escalation-bridge). See §9 Q8.

---

## 5. Bridge tiers

The same bridge software supports four deployment shapes. *Threat-model fit* assumes the v1 crypto choices (no forward secrecy, cleartext metadata on firehose); see §13 for the full picture.

| Tier | Operator | Phone numbers in system? | Threat-model fit |
|------|----------|--------------------------|------------------|
| 0 | Hosted public bridge | Yes (Twilio at the operator) | Convenience default. Single subpoena/breach target for the entire user base. |
| 1 | Community/chapter | Yes (chapter's Twilio) | Chapter is trust boundary. Chapter-bridge operator faces personal legal exposure; see §13. |
| 2 | Self-hosted | Yes (your Twilio sub-account) | Only you and Twilio. Subpoena targets you personally. |
| 3a | App-based push, FCM/APNs | **No** | Strongest *cryptographic* posture but **platform-dependent**: Apple/Google can revoke the app from stores. |
| 3b | App-based push, ntfy.sh self-hosted | **No** | Sovereign push. The only tier with no platform kill-switch. |

The v1 reference bridge ships as a single Docker image deployable for Tiers 0/1/2 (different Twilio account configurations). The v1 reference Tier 3 client is a React Native/Expo app shipping with both FCM/APNs (3a) and ntfy.sh (3b) as user-selectable transports.

---

## 6. Capabilities

What Kettle delivers, tiered by readiness. The transport menu is bridge-internal and grows over time without lexicon changes.

### Transport menu (bridge-internal, no lexicon impact)

The bridge dispatches via configured transport. Recipient's allow-list specifies which transport for which sender or category.

- **SMS** — Twilio, Vonage, Plivo, Telnyx, Bandwidth, AWS SNS. ~$0.008/msg + carrier fees. US compliance: A2P 10DLC registration (~$20 + 3-week wallclock).
- **Push notifications** — APNs / FCM via the Kettle mobile app, or ntfy.sh (open protocol, self-hostable). Free.
- **Signal** — via signal-cli / signald (community-maintained; cooperation from Signal-the-org would be a bonus, not a requirement).
- **Matrix** — standard client-server API. E2E via Olm.
- **Discord / Slack / Telegram** — documented incoming webhooks or bot APIs. No platform partnership.
- **Email / email-to-SMS gateway** — SMTP. (Carrier gateways are dying; reliability dropping yearly.)
- **Voice call** — SIP via Asterisk + Telnyx / VoIP.ms.
- **Webhook to user-chosen endpoint** — bridge POSTs to any URL the recipient provides (IFTTT, Home Assistant, custom bot).
- **XMPP / IRC** — old but open.

Adding a transport is ~50 lines of bridge code; lexicons unchanged.

### v1 ships (the reference implementation delivers these)

- **Single-recipient consent-gated paging.** Sender → recipient with full allow-list enforcement.
- **Allow-list management API** (bridge-internal, recipient-managed via authenticated calls).
- **Per-sender + global rate limiting**, quiet hours, category filters, per-sender expirations.
- **Read/delivery receipts** via `community.paging.received`.
- **Time-locked pages** — sender posts with `deliverAfter`; bridge holds and dispatches at scheduled time. (Adds an optional `deliverAfter` field to `community.paging.send`; documented in v1 lexicon.)
- **Replay protection** — bridges MUST dedup by record CID per recipient (spec mandate, not lexicon-enforced).

### v1 protocol-supports (lexicon shape allows; reference bridge skips for v1)

- **Threaded async conversation** via `replyTo`. Lexicon allows it; bridge dispatch behavior across `replyTo` chains is §9 Q12 (allow-list resolution rule).
- **Cross-bridge federation.** Any bridge can subscribe to firehose for any recipient who points at it via `bridgeEndpoint`. No additional addressing semantics needed in the protocol; v1 reference doesn't ship multi-bridge orchestration tooling.
- **Atproto-native moderation hooks.** Bridges can apply labelers (drop pages from labeled-spam DIDs, hold for review on quiet-hours violation, etc.) — pure atproto, no lexicon impact.

### Future capabilities (require additional lexicons or substantial bridge work; v1.1+)

- **Page-to-group fanout** — `community.paging.group` lexicon; this is the prompting-use-case feature and the v1.1 priority. §9 Q11.
- **Dead-man's switch** — `community.paging.deadmansSwitch` lexicon + polling architecture. §9 Q8.
- **Anonymous tip line** — wildcard allow-list + moderation queue UX; no new lexicon, but reference bridge needs review-queue surface.
- **Geofenced paging** — requires location-record coordination (probably with `community.lexicon.location`).
- **Per-context routing across multiple bridges** (one recipient with multiple bridges, each handling different categories).
- **Quorum paging / confirmation chains** — bridge logic, no new lexicon.
- **Bridge-as-a-service for orgs** with multi-tenant management UX.
- **Public alert channels** with subscriber rate caps and abuse mitigation.
- **Page-back via SMS short code** — bidirectional flow without phone-number exchange (bridge translates SMS reply into `community.paging.send` record on recipient's PDS).

The v1 protocol is sufficient substrate for all of the above; v1 just doesn't deliver them in the reference implementation.

---

## 7. Encryption

Three options for body privacy, with explicit forward-secrecy and traffic-analysis caveats.

### A. Encrypted body in public records (v1 default)

Sender encrypts the page body to recipient's `defaultPubkey` (asymmetric encryption + AEAD). Public firehose sees `{targetDid, ciphertext, nonce, category, createdAt}`; only recipient's bridge can decrypt.

**Properties:**
- Content private. Graph public. Per-page metadata leakage: sender DID, recipient DID, timestamp, category.
- **No forward secrecy.** `defaultPubkey` is static and long-lived; if compromised, all past pages encrypted to it become decryptable. Mitigation: periodic key rotation (`keyVersion` on `bridgeEndpoint` enables this) — but past pages encrypted to retired keys are recoverable until/unless the privkey is destroyed, which the protocol can't enforce.
- **Traffic analysis on the firehose.** Cleartext sender-DID + cleartext `category` + timing reconstructs organizational structure, identifies high-out-degree leaders, detects mobilization spikes, correlates with real-world events. Use of `emergency` category at 23:42 correlated with a protest is direct coordination evidence to a sophisticated adversary, even with content encrypted.

For most "are you free Saturday" use cases this is fine. For ICE-rapid-response use cases with state-level adversaries, option A is **insufficient**; option B is the principled answer; option C is the fast path.

### B. Sealed-sender envelope (principled long-term, v1.1+)

Following Signal's [sealed-sender construction](https://signal.org/blog/sealed-sender/) (or IETF [Oblivious Message Delivery](https://datatracker.ietf.org/doc/draft-irtf-cfrg-vrf-oprf/) — TBD which fits atproto better). Sender publishes to a paging mailbox without revealing recipient in cleartext; bridge attempts decryption per record using sharded mailbox-by-hash-bucket (e.g., 16-bit prefix of `H(targetDid || epoch)`); only the right bridge succeeds.

**Properties:**
- Content + graph + category all private.
- Per-session ephemeral keys give forward secrecy.
- Cost: bridge attempts decryption on every record in its hash bucket(s). 16-bit bucketing means each bridge processes ~1/65k of firehose. Manageable; benchmarkable.
- Construction choice (Signal sealed-sender vs IETF OMD vs Hybrid Public Key Encryption with anonymity tags) is the v1.1 design decision; not picked here.

### C. `chat.bsky.convo` DM substrate (fast path)

Pages flow as structured DM payloads through Bluesky's chat service. Private from public firehose; centralized at `chat.bsky.social`. Loses federation purity.

**Properties:**
- Content + graph private (chat service holds metadata; not on firehose).
- Centralization debt: depends on Bluesky-the-org's chat infrastructure and ToS.
- Coordination required: this option exists only if the Bluesky team is willing to add a structured-payload extension to `chat.bsky.convo` for non-conversational message types. Unknown whether they would.

**v1 ships A.** v1.1 should ship B if the construction choice locks. C is the "we want this in production tomorrow" fallback if Bluesky-team coordination materializes.

---

## 8. Composition

**With Smoke Signal events.** RSVP record on Smoke Signal could grant time-bounded auto-entry to the event-organizer's allow-list. RSVP yes to May Day → organizer's DID gets `expiresAt = event.end + 1h` paging rights. Event-time reachability without manual allow-list edits. Coordination needed: Smoke Signal lexicon ownership lives in `community.lexicon`; paging proposes the same namespace (§9 Q1).

**With Snug** (sibling project, encrypted MLS group chat for atproto, same author). A Snug group can be the *target* of a Kettle page (page-to-group via `community.paging.group`). Snug's MLS encryption protects in-group conversational context; Kettle extends the reach of any in-group signal out-of-band when the recipient isn't actively in the chat. The two protocols share substrate (atproto DIDs, PDS records) but solve different problems with different cryptographic shapes — Snug is bidirectional persistent encrypted chat with forward secrecy; Kettle is one-way ephemeral consent-mediated paging with weaker crypto. Two-way consistency between the docs has not been verified at the lexicon level; that's pre-v1 work.

**With labelers.** Bridge-side capabilities surface as public labels (`paging-enabled`, `paging-emergency-allowed`) so others know who to expect a response from. No phone numbers leak; the bridge URL is already public via `bridgeEndpoint`.

---

## 9. Open questions for atproto-protocol contributors

1. **Namespace placement.** **Recommendation: `community.lexicon.paging.*`**, coordinating with Smoke Signal (already migrating events to `community.lexicon.*`). Push back if there's a reason paging should sit elsewhere.

2. **Bridge auth pattern.** Is `community.paging.bridgeAuthorization` the right shape, or should bridges authenticate via standard atproto OAuth scopes against the recipient's PDS? Custom delegation is more federation-pure; OAuth is more pattern-matched.

3. **Encryption substrate.** §7 A/B/C. v1 ships A; B is the long-term answer. Interested specifically in feedback on whether sealed-sender's bucket-attempt-decrypt cost is acceptable at firehose scale.

4. **Jetstream production-readiness.** Documented as "not a stable protocol API." For self-hosted bridges acceptable; for hosted public Tier 0 it's a production risk. Is a stable subscription path planned?

5. **Cross-bridge federation semantics.** A user's `bridgeEndpoint` may name a bridge run by a different operator than their PDS host. Does the protocol need explicit cross-bridge addressing, or is "pointer + firehose subscription" sufficient?

6. **Rate limiting and abuse at firehose level.** Bridge-side filtering catches dispatched abuse but the records are still on the firehose. Is there a firehose-level abuse mitigation we should care about?

7. **Bridge-side state portability.** Allow-list lives bridge-side (the v0.2 trust-graph-privacy fix). Switching bridges = migrating the list. Right answer: (a) standard `community.paging.exportAllowList` XRPC, (b) bridge-internal admin-UI dump, or (c) signed allow-list snapshots any bridge can ingest? Recipients without technical fluency need this to not be a brick wall.

8. **Dead-man's-switch threat model.** Who runs the polling? Recipient's own bridge can't fire after it goes offline (machine seized, network cut). External-poller introduces third-party trust. Probably the answer is "designated escalation bridge" (recipient pre-authorizes a chapter or trusted-friend bridge with attestation), but the attestation pattern wants atproto-protocol input.

9. **Anonymous tip line + atproto signing.** Atproto signing implies sender DID is in the record. "Anonymous" here means "DID is fresh and unprofiled" rather than no DID. Strong enough for whistleblowing, or should the lexicon support a sealed-sender mode that strips sender DID at the cost of source authentication?

10. **Constituency validation.** Has this been pressure-tested with the constituency it claims to serve? **No, currently a designer's projection.** Plan: 2-3 sit-downs with rapid-response organizers (KC DSA, LA mutual-aid, ICE Out of LA-shape orgs) before v1 lexicon-lock. Field tests will probably move some "v1 protocol-supports" features to "v1 ships" or vice versa.

11. **`community.paging.group` schema.** Group fanout requires a member list. Should that list be (a) on group-creator's PDS as a public record, (b) bridge-private (mirroring v0.2 trust-graph privacy), or (c) MLS-grouped (composing with Snug)? Materially shapes the v1.1 lexicon.

12. **`replyTo` allow-list resolution rule.** When sender A pages recipient B; B replies via `replyTo`; whose allow-list governs the reply's delivery to A? **Proposed rule: the direct-parent author's allow-list governs.** A→B uses B's allow-list (B is the recipient); B's reply→A uses A's allow-list (A is now recipient). Multi-party threads (C replies to B's reply) cascade through each parent step. Want pushback on whether thread-root vs direct-parent is the right scope.

13. **Replay-protection mandatory at bridge?** Lexicon doesn't enforce dedup; bridges MUST track seen record CIDs per recipient. Should "MUST dedup" be in the spec normatively, or left as bridge-implementation guidance?

14. **`<bytes>` encoding choice.** Lexicon convention is multibase-string-over-JSON or CBOR-bytes. Pick one before lexicon-lock. Slight lean toward multibase strings for portability across non-CBOR consumers.

---

## 10. v1 scope

**Build:**
- Five lexicons + JSON schemas (`send`, `bridgeEndpoint`, `bridgeAuthorization`, `received`, `body`-wire-format).
- Reference bridge — Docker image, Twilio + ntfy.sh dispatchers in same binary, deployable as Tier 0 / 1 / 2 from one image.
- Tier 3 reference mobile client — Expo/React Native, push-only, ntfy.sh self-host option for sovereign push.
- Hosted public bridge — single deployment of reference (e.g., `bridge.kettle.org`).
- Smoke Signal RSVP integration — RSVP-yes auto-grants time-bounded paging right.

**Skip for v1:**
- Sealed-sender envelope (§7 B) — v1.1.
- `community.paging.group` and group fanout — v1.1.
- `community.paging.deadmansSwitch` — v1.1.
- MLS-style key rotation — v1 does periodic static-key rotation only (`keyVersion`).
- Encrypted-blob attachments — text-only.
- Multi-device key sync — single device per recipient in v1.
- Cross-bridge orchestration tooling (the protocol supports it; v1 reference doesn't ship multi-bridge UX).

**Estimated engineering**: ~3 weeks for the protocol + reference bridge; the Tier 3 mobile client is ~2 weeks (already-bootstrapped Expo project + Jetstream client + APNs/FCM provisioning + TestFlight/Play submission). A2P 10DLC compliance for Tier 0 is a parallel multi-week wallclock; Tier 3 + ntfy.sh sidesteps it. The estimate's biggest risk is Jetstream stability (§9 Q4) and A2P brand/campaign approval timing.

---

## 11. Comparisons

**vs. Signal.** Signal does end-to-end encrypted 1:1 and group messaging. Kettle is *not a chat replacement* — it's a paging primitive that can dispatch *to* Signal as one of many transports. Signal handles the conversation; Kettle decides who's allowed to start one and routes the alert.

**vs. Action Network / Spoke / Hustle.** Existing organizer-SMS tools are centralized SaaS with phone-number databases. Kettle puts the directory at the bridge (which can be self-hosted, federated, or have no phone numbers in Tier 3). Trade-off: existing tools are operational today and have institutional buy-in; Kettle is a sketch.

**vs. Twitter SMS-tweet (deprecated).** SMS-tweet was opt-in broadcast (public phone-number list, fanout). Kettle is bilateral consent-mediated summon. Different shape; different threat model.

**vs. Bluesky group DMs (planned).** When they ship, Bluesky's group DMs will be centralized at `chat.bsky.social`. Kettle is federated. Different federation properties; adjacent but not overlapping use cases — Bluesky group DMs are *chat*, Kettle is *out-of-band notification*. Threading via `replyTo` makes Kettle group-DM-shaped in some emergent ways, but it's not a chat replacement and shouldn't be pitched as one.

---

## 12. License

This document is released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Lexicons themselves are intended for [CC0](https://creativecommons.org/publicdomain/zero/1.0/) on adoption into `community.lexicon.*` per that namespace's conventions.

---

## 13. Threat model

The honest version of "what isn't solved." This section is load-bearing for chapter operators considering Tier 1 deployment and for individual organizers in adversarial threat models.

### Adversaries explicitly considered

- **State-level surveillance with firehose access** (ICE, DHS, federal LE). Sees every `community.paging.send` record. Without sealed-sender, sees sender DID + recipient DID + cleartext `category` + timing — sufficient to reconstruct organizational structure and detect mobilization events without ever decrypting content.
- **Subpoena targeting the bridge operator.** Yields cleartext allow-list, dispatch destinations (phone numbers / handles), and any retained message content.
- **Compromised individual member device** (post-arrest, lost phone). Reveals that member's allow-list state and any local decrypted page history.
- **Adversarial fellow-organizer or turned member.** A member of an allow-list with paging rights can flood, exfiltrate, or impersonate.

### Adversaries acknowledged but not mitigated in v1

- **Bridge operator turned hostile.** The bridge sees the entire local trust graph, holds decryption capability, and dispatches via authenticated transport credentials. A turned/breached chapter-bridge operator has near-total visibility into the rapid-response network. Tiers 2/3 distribute or eliminate this; Tier 0/1 concentrate it. **Multi-bridge sharding** (no single bridge has the full graph) is a v1.1 mitigation; v1 ships single-bridge-per-recipient.
- **Recipient-device capture by intimate-partner adversary.** Tier 3 push notifications appear on lock screen by default; bridge credentials live in OS keychain. v1 has no panic-wipe, no duress-PIN, no silent-disable mode. For organizers escaping abuse — a realistic overlap with tenant-organizing, mutual-aid, abortion-network contexts — this is more likely than ICE for the median user. v1.1 should ship duress mode; v1 documents the gap.
- **Replay attacks on page records.** Cleartext records are permanent on the firehose. Bridges MUST dedup by record CID (mandate in §6, but lexicon doesn't enforce). Without dedup, an adversary can replay old emergency pages to trigger false mobilizations.
- **DID compromise.** A turned organizer's DID retains paging rights until manually revoked from the allow-list. Mitigation: short `expiresAt` on entries (forces periodic re-confirmation).
- **Fresh-DID anonymity ≠ network-level anonymity.** The anonymous-tip-line use case requires Tor-equivalent network anonymity *in addition to* fresh DID — atproto PDS registration leaks IP, timing, and writing-pattern correlations. v1 doesn't address.

### Adversaries not considered

- **Quantum-future cryptanalysis.** Irrelevant for v1; rotate to PQ schemes when atproto-substrate does.
- **Coordinated DoS at firehose level.** Atproto-substrate concern, not Kettle's.
- **App-store revocation.** Apple/Google can pull the Tier 3 mobile app from stores; sovereign push (ntfy.sh self-hosted, Tier 3b) is the v1 escape hatch and is documented in §5.

### Legal exposure for Tier 1 operators (not legal advice)

A chapter running a Tier 1 bridge for ICE-coordination support is operating an electronic communications service knowingly facilitating anti-enforcement activity. Live legal questions include the [Stored Communications Act](https://www.law.cornell.edu/uscode/text/18/2701), [ECPA](https://www.law.cornell.edu/uscode/text/18/2510), A2P 10DLC carrier disclosure obligations, conspiracy-theory liability for the operator personally, and state-level "hindering apprehension" statutes.

The reference bridge defaults to:

- Zero retention on dispatched message bodies (in-flight only, dispatched-and-deleted).
- Allow-list export-only (no centralized sync to operator-controlled storage).
- Subpoena-response posture: minimum legal disclosure, notification to affected DIDs where permitted.
- A `--paranoid` mode that goes further: ephemeral allow-list (RAM-only, lost on restart), no logs, no telemetry.

Chapter operators considering Tier 1 should consult [EFF](https://www.eff.org) or the [National Lawyers Guild](https://www.nlg.org) before deployment. The reference implementation makes safe defaults but **does not insulate the operator from liability**.

---

## Appendix A: Project name

The name carries two meanings at once and is the better for holding both.

"Kettling" is the police tactic of containing protesters in a defined area — the inversion is *a tool that kettles communication into trusted channels*, used by the people otherwise being kettled.

A kettle is also for tea: the small, private, warm conversations among people who already trust each other — the cozy side of why organizers organize in the first place.

The protocol serves both: a rapid-response paging primitive sharp enough to coordinate against a hostile state, and a quiet inbox for "are you free Saturday?" between comrades. Organizing infrastructure that's only sharp-edged misses the half of community that isn't a fight.

If the name doesn't survive review by the people the tool is for, rename is cheap — the lexicon namespace is the actual identity. v0.4's constituency-validation pass (§9 Q10) will surface whether the name lands.
