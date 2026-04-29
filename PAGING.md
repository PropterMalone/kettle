# Kettle: Consent-Mediated Paging on AT Protocol

A design for programmatic out-of-band notification ("paging") between AT Protocol identities, without phone-number exchange.

**Status**: Sketch v0.1, April 2026.
**Audience**: feedback from atproto-protocol people before lexicon and implementation are locked.

**Project name.** The name carries two meanings at once and is the better for holding both. "Kettling" is the police tactic of containing protesters in a defined area — the inversion is *a tool that kettles communication into trusted channels*, used by the people otherwise being kettled. And a kettle is also for tea: the small, private, warm conversations among people who already trust each other — the cozy side of why organizers organize in the first place. The protocol serves both: a rapid-response paging primitive sharp enough to coordinate against a hostile state, and a quiet inbox for "are you free Saturday?" between comrades. Organizing infrastructure that's only sharp-edged misses the half of community that isn't a fight. If the name doesn't survive review by the people the tool is for, rename is cheap — the lexicon namespace is the actual identity.

---

## 1. Problem

Organizers need *programmatic out-of-band reach* — a way for people who trust me to summon me (SMS, Signal, push notification) without phone-number exchange.

- Bluesky's 1:1 DMs (`chat.bsky.convo`) are centralized at Bluesky PBC's chat service and don't deliver out-of-band when the app is closed.
- Group DMs are roadmap-future and will be centralized at the same chat service.
- Existing organizing tools (Action Network, Spoke, Hustle, Twilio direct) require centralized phone-number directories — honeypots in adversarial threat models (ICE, state-level surveillance, subpoena exposure).

Twitter's deprecated SMS-tweet feature was *broadcast*: a public list of phone numbers fanning out from a single sender. The inversion this design proposes is *targeted summon*: each user maintains a private allow-list of DIDs that can summon them via whatever transport they prefer. Senders never learn recipient phone numbers. Recipients revoke instantly via PDS write.

The forcing function is organizing in adversarial environments — KC DSA running rapid-response coordination against ICE was the prompting case. The primitive generalizes to any "I want to be reachable by these specific people, no one else" use case, including federated group DMs as a side effect (see §7).

## 2. Design principles

**PDS-anchored consent.** The allow-list is a record on the recipient's PDS. Senders post page records to their own PDS. The protocol moves through standard atproto firehose + custom lexicons. Nothing requires a centralized operator.

**Pluggable dispatch.** The protocol doesn't know about phone numbers, SMS, Signal, push, or any specific transport. That's bridge-internal. Bridges are feed-generator-shaped: anyone-can-run watchers that subscribe to filtered Jetstream, validate sender against the recipient's allow-list, dispatch via configured transport.

**Self-bridging is first-class.** No tier requires a centralized operator. Self-hosted bridges (Tier 2) and app-based push bridges (Tier 3) eliminate the phone-number honeypot entirely.

**Encryption by default.** Page records carry encrypted bodies. Public firehose sees `{target_did, ciphertext, nonce}`; only the recipient's bridge can decrypt. Graph metadata (who paged whom, when) is public on the firehose; content is not.

**Honest about what isn't solved.** Graph privacy (sender↔recipient relationships) is exposed on the public firehose unless a sealed-sender envelope is used (see §6). For most organizing use cases this is acceptable; for the highest-stakes cases it isn't. The design surfaces this rather than papering over it.

## 3. Architecture

```
Sender's PDS                       Recipient's bridge              Recipient
────────────                       ──────────────────              ─────────
Posts                              Subscribes filtered             
community.paging.send              Jetstream                       
{target_did, ciphertext,           ────────────────────►           
 nonce, category}                  sees record                     
                                                                   
                                   Fetches recipient's             
                                   community.paging.allowList      
                                   from recipient's PDS            
                                                                   
                                   If sender_did is on allow-list  
                                   AND rate/hours/category match:  
                                                                   
                                   Decrypts body using             
                                   recipient's key                 
                                                                   
                                   Dispatches via configured       
                                   transport ─────────────────────► SMS / Signal / push / ntfy
```

Sender writes to their own repo. Bridge subscribes to a filtered Jetstream stream (e.g., `wantedCollections=community.paging.send` + filter on `target_did`). Allow-list is publicly readable on recipient's PDS. Bridge enforces consent and dispatches.

The bridge is the only stateful service-side component, and it's owned by the recipient (or a trusted operator chosen by the recipient).

## 4. Lexicons

Three records are sufficient for the v1 protocol.

### `community.paging.send`
Sender posts to their own PDS. Public record; content is encrypted.

```json
{
  "$type": "community.paging.send",
  "target_did": "did:plc:abc...",
  "ciphertext": "<bytes>",
  "nonce": "<bytes>",
  "category": "emergency" | "event" | "personal",
  "replyTo": "at://did:plc:.../community.paging.send/...",
  "createdAt": "2026-04-29T23:00:00Z"
}
```

Plaintext (decrypted body) is a `community.paging.body` schema:

```json
{
  "$type": "community.paging.body",
  "text": "ICE 5min out, 14th & Wabash",
  "facets": []
}
```

Length cap on plaintext text: 160 bytes (compatible with SMS without segmentation when transport is SMS).

### `community.paging.allowList`
Recipient publishes on their PDS. Public record (allow-list membership is not secret — only the *number* is, and that lives at the bridge).

```json
{
  "$type": "community.paging.allowList",
  "bridge_endpoint": "https://bridge.kcdsa.org" | null,
  "bridge_did": "did:plc:..." | null,
  "default_pubkey": "<bytes>",
  "entries": [
    {
      "sender_did": "did:plc:abc...",
      "rate_limit_per_day": 3,
      "allowed_hours_utc": [13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 0, 1, 2],
      "categories": ["emergency", "event"],
      "expires_at": "2026-05-01T23:59:59Z"
    }
  ],
  "global_rate_limit_per_day": 20,
  "createdAt": "2026-04-29T20:00:00Z"
}
```

Key: `self`. Bridge resolves at page-receipt time; updates are picked up on the next page received from any sender.

### `community.paging.bridgeAuthorization`
Recipient signs a delegation: "this bridge endpoint may act as my page-receiver." Bridge presents this when challenged. Standard atproto signing.

```json
{
  "$type": "community.paging.bridgeAuthorization",
  "bridge_did": "did:plc:...",
  "scope": ["receive", "decrypt", "dispatch"],
  "expires_at": "2026-10-29T23:00:00Z",
  "createdAt": "2026-04-29T20:00:00Z"
}
```

Optional for self-hosted bridges where the recipient is the bridge operator.

## 5. Bridge tiers

The same bridge software supports four deployment shapes:

| Tier | Operator | Phone number in system? | Threat-model fit |
|------|----------|-------------------------|------------------|
| 0 | Hosted public bridge (e.g., `propter-page-bridge`) | Yes (Twilio) | Lazy default, dragnet exposure |
| 1 | Community/chapter (KC DSA runs theirs) | Yes (chapter's Twilio) | Chapter is trust boundary |
| 2 | Self-hosted (Pi, VPS, Tailscale) | Yes (your Twilio sub-account) | Only you + Twilio |
| 3 | App-based push (APNs/FCM/ntfy) | **No** — never enters system | Best for adversarial threat models |

Tier 3 is the elegant escape: a mobile app subscribes to filtered Jetstream for the user's DID, lights up a push notification when a valid page arrives. **No phone number ever in the protocol layer or the bridge.**

Multi-bridge federation falls out: KC DSA's bridge handles its members; Sonoma DSA's bridge handles its members; no single bridge has the whole graph. The allow-list record's `bridge_endpoint` field tells senders' tooling which bridge is authoritative for which recipient.

## 6. Encryption: open tradeoff

Three options for content/graph privacy:

**(a) Encrypted body in public records (default for v1).** Graph public, content private. Cheap. Good enough for most organizing — the *content* of a rapid-response page is the secret, not the relationship.

**(b) Sealed-sender envelope.** Sender publishes to a paging mailbox without revealing recipient in cleartext. Bridge attempts decryption on records, only succeeds on records meant for it. Graph + content both private. Heavier — bridge attempts decryption per record. Real-world OK because you can shard the mailbox by hash bucket.

**(c) Use `chat.bsky.convo` DM substrate.** Pages flow as structured DM payloads. Private from public firehose but centralized at Bluesky chat service. Loses federation purity; gains centralized-but-private content path. Requires Bluesky-team coordination.

Recommendation: ship (a) for v1; (b) is the principled long-term answer; (c) is the "we want this in production tomorrow" path if Bluesky team will play.

## 7. Composition

**With Smoke Signal events.** RSVP record on Smoke Signal could grant time-bounded auto-entry to the event-organizer's allow-list. RSVP yes to May Day → organizer's DID gets `expires_at = event.end + 1h` paging rights. Event-time reachability without manual allow-list edits. Coordination needed: Smoke Signal lexicon ownership lives in `community.lexicon`; paging should probably live there too.

**With Snug** (sibling project, encrypted MLS group chat for atproto, same author). A Snug group can be the *target* of a Kettle page (page-to-group). Membership in a Snug group implies pageable-by-group-organizer or pageable-by-any-member. Snug's MLS encryption protects in-group conversational context; Kettle extends the reach of any in-group signal out-of-band when the recipient isn't actively in the chat. The two protocols share substrate (atproto DIDs, PDS records) but solve different problems with different cryptographic shapes — Snug is bidirectional persistent encrypted chat; Kettle is one-way ephemeral consent-mediated paging.

**With labelers.** Bridge-side capability becomes a public label (`paging-enabled`, `paging-emergency-allowed`) so other organizers know who to expect a response from. No phone numbers leak.

**As a federated group-DM substrate.** Add `replyTo` and the system becomes async multi-party threading with allow-list-mediated membership. Each user has one inbox (their allow-list defines who can post to it). Threads form when senders reply to pages and the original sender's allow-list permits them. This is decentralized group DMs — the use case Bluesky's roadmap covers, but federated rather than centralized.

## 8. Open questions for atproto-community

1. **Namespace placement.** `community.lexicon.paging.*` vs `chat.kettle.paging.*` vs a new top-level. Smoke Signal team is migrating events into `community.lexicon.*`; coordination is the load-bearing call. *(Lexicon-community input requested before locking schema.)*

2. **Bridge auth pattern.** Is `community.paging.bridgeAuthorization` the right shape? Or should bridges authenticate via standard atproto OAuth scopes against the recipient's PDS? The custom delegation is more federation-pure; OAuth is more pattern-matched to existing infrastructure.

3. **Encryption substrate.** (a) public encrypted body / (b) sealed-sender envelope / (c) `chat.bsky.convo` DM payloads. Each has different federation properties and different coordination requirements. v1 plans (a) but (b) is the right long-term shape — interested in feedback on whether sealed-sender is over- or under-engineered.

4. **Jetstream production-readiness.** Jetstream is documented as "not a stable protocol API." For self-hosted bridges this is acceptable; for hosted public bridges (Tier 0) and app-based clients (Tier 3) it's a production risk. Is there a stable firehose subscription path planned?

5. **Cross-bridge federation.** A user's allow-list on PDS-A may name a bridge run by a different operator than PDS-A's host. Does the protocol need explicit cross-bridge addressing semantics, or is "the recipient's allow-list points at one bridge, that bridge handles all of their pages" sufficient?

6. **Rate limiting and abuse.** Per-sender rate limits and global daily caps are encoded in the allow-list record, enforced bridge-side. Adversarial sender posts a million page records; bridge filters, but the records are still on the firehose. Is there a firehose-level abuse mitigation we should care about, or is "ignore at the bridge" sufficient?

## 9. v1 scope

**Build:**
- Three lexicons + JSON schemas (above)
- Reference bridge (Docker image, Twilio + ntfy.sh dispatchers in same binary, Tier 0 + Tier 2 deployable from the same image)
- Tier 3 reference mobile client (Expo/React Native, push-only, no phone numbers)
- Hosted public bridge (`bridge.kettle.org` or equivalent — single deployment of reference)
- Smoke Signal RSVP integration (RSVP-yes auto-grants time-bounded paging right)

**Skip for v1:**
- MLS-style key rotation for paging records (overkill for one-shot pages with no history; revisit if paging threads become long-lived)
- Cross-bridge federation
- Encrypted-blob attachments (text-only paging is the v1 contract)
- Multi-device key sync (single device per recipient is the v1 simplification)

**Estimated engineering**: ~3 weeks (1 week lexicons, 1 weekend reference bridge, 1 weekend Tier 3 client, 1 weekend hosted bridge deployment, plus integration work). A2P 10DLC compliance for Tier 0 with Twilio is a parallel multi-week wallclock; Tier 3 sidesteps it entirely.

## 10. Comparisons

**vs. Signal.** Signal does end-to-end encrypted 1:1 and group messaging; people on Signal are reachable. Kettle-paging is *not a chat replacement* — it's a paging primitive that can dispatch *to* Signal as one of many transports. They compose: Signal handles the conversation, Kettle decides who's allowed to start one and routes the alert.

**vs. Action Network / Spoke / Hustle.** Existing organizer-SMS tools are centralized SaaS with phone-number databases. Kettle-paging puts the directory at the bridge (which can be self-hosted, federated, or never have phone numbers in it at all in Tier 3). Trade-off: existing tools are operational today and have institutional buy-in; Kettle is a sketch.

**vs. Twitter SMS-tweet.** SMS-tweet was opt-in broadcast (public phone-number list, fanout). Kettle-paging is bilateral consent-mediated summon. Different shape; different threat model.

**vs. Bluesky group DMs (planned).** When they ship, Bluesky's group DMs will be centralized at `chat.bsky.social`. Kettle-paging is federated (PDS-anchored consent, pluggable bridges). Different federation properties; same underlying need for "talk to N people privately."

---

## License

This document is released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
