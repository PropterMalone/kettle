# Kettle: Consent-Mediated Paging on AT Protocol

A design for programmatic out-of-band notification ("paging") between AT Protocol identities, without phone-number exchange.

**Status**: Sketch v0.2, April 2026.
**Audience**: feedback from atproto-protocol people before lexicon and implementation are locked.
**Changes since v0.1**: (1) §6 Capabilities added — the value-prop surface (group fanout, threading, page-back without phone exchange, dead-man's switch, anonymous tip line, federation, etc.) is now explicit rather than implicit-in-the-lexicon. (2) **Allow-list moved bridge-side, not on PDS.** Trust graph (who-allows-who-to-page-them) was a subpoenable public record in v0.1; now it's bridge-private state managed by the recipient via authenticated API. PDS only carries a pointer to the recipient's bridge. (3) §7 Encryption clarified. (4) Open questions §9 expanded.

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

**PDS-anchored discovery, bridge-private trust.** The recipient's PDS carries a *pointer* to their bridge (`community.paging.bridgeEndpoint`); the actual allow-list (who can page them, with what rate/category/hours rules) lives bridge-side, never published to the public firehose. Senders post page records to their own PDS; the recipient's bridge subscribes to the firehose, resolves its own private allow-list, and decides whether to dispatch. The trust graph is therefore private to the bridge operator, not subpoenable from public records.

**Sender writes are public.** A sender posting `community.paging.send` to their own PDS is publicly observable on the firehose — the *act* of paging is not secret, even though the *content* (encrypted) and the *recipient's allow-list state* (bridge-private) are. Sealed-sender envelope (§7 option b) is the path to also hiding sender-recipient relationships; v1 ships without it.

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
                                                                   
                                   Consults BRIDGE-PRIVATE         
                                   allow-list (sender_did,         
                                   rate, hours, category)          
                                                                   
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

Sender writes to their own repo. Bridge subscribes to a filtered Jetstream stream (e.g., `wantedCollections=community.paging.send` filtered on `target_did`). The allow-list is bridge-side state — recipient configures it by authenticated API call to *their own* bridge, never via PDS write.

The bridge is the only stateful service-side component, and it's owned by the recipient (Tier 2/3) or a trusted operator chosen by them (Tier 0/1). Bridge-private trust state is the only state worth subpoenaing — that subpoena targets the bridge, not the public PDS infrastructure.

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

### `community.paging.bridgeEndpoint`
Recipient publishes on their PDS. Public record. **Pointer only — the allow-list itself is not on PDS.**

```json
{
  "$type": "community.paging.bridgeEndpoint",
  "bridge_endpoint": "https://bridge.kcdsa.org",
  "bridge_did": "did:plc:..." | null,
  "default_pubkey": "<bytes>",
  "createdAt": "2026-04-29T20:00:00Z"
}
```

Key: `self`. Tells senders' tooling (and the wider network) which bridge handles this recipient. The `default_pubkey` is the recipient's public key for encrypting page bodies.

**Why not on PDS:** the trust graph (who is permitted to reach me) is sensitive metadata — knowing "Kevin allows these 23 DIDs to page him" reveals organizing affiliations, social structure, rapid-response tree topology. v0.1 of this spec put the allow-list on PDS as a public record; v0.2 fixed that. Public PDS records are subpoenable, scrapeable, and indexable. Bridge-side state is not (or at least, not without targeting the bridge specifically).

### Allow-list (bridge-side, not a lexicon)

The allow-list is bridge-private state. Schema is bridge-internal, but the recipient-facing semantics:

```yaml
allow_list:
  global_rate_limit_per_day: 20
  default_quiet_hours_utc: [3, 4, 5, 6, 7]   # UTC hours during which only emergency category fires
  entries:
    - sender_did: did:plc:abc...
      rate_limit_per_day: 3
      allowed_hours_utc: [13, 14, ..., 2]
      categories: [emergency, event]
      expires_at: 2026-05-01T23:59:59Z
      transport_override: signal           # optional per-sender transport
    - sender_did: did:plc:def...
      rate_limit_per_day: unlimited
      categories: [emergency, event, personal]
      expires_at: null
```

The recipient manages the allow-list via authenticated API calls to their own bridge (HTTPS + atproto-signed challenge, or DPoP). Bridges should support export/import of allow-list state for portability when switching bridges.

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

Multi-bridge federation falls out: KC DSA's bridge handles its members; Sonoma DSA's bridge handles its members; no single bridge has the whole graph. The recipient's `community.paging.bridgeEndpoint` record on PDS tells senders' tooling which bridge is authoritative for which recipient.

## 6. Capabilities — what Kettle delivers

Items in the **transport menu** dispatch via integration with the named third-party service (most via documented public APIs, none requiring partnership). Items in **pure-protocol features** require no third-party cooperation — bridge logic and atproto records do the work.

### Transport menu

The bridge dispatches via configured transport. Recipient's allow-list specifies which transport for which sender or category.

- **SMS** — Twilio, Vonage, Plivo, Telnyx, Bandwidth, AWS SNS, or any RFC-compliant SMS provider. ~$0.008/msg + carrier fees. US compliance: A2P 10DLC registration (~$20 + 3-week wallclock).
- **Push notifications** — APNs / FCM via a Kettle-distributed mobile app, or ntfy.sh (open protocol, self-hostable). Free. No phone numbers in system.
- **Signal** — via signal-cli / signald (community-maintained; Signal-org cooperation is a probable bonus, not a requirement). Recipient must have Signal installed.
- **Matrix** — via standard client-server API. E2E encryption via Olm.
- **Discord / Slack / Telegram** — via documented incoming webhooks or bot APIs. No platform partnership needed.
- **Email / email-to-SMS gateway** — SMTP. Free, ubiquitous; carrier gateways are dying.
- **Voice call** — SIP via Asterisk + Telnyx/VoIP.ms trunks. Different reach profile (people ignore unknown numbers).
- **Webhook to user-chosen endpoint** — bridge POSTs to any URL the recipient provides. Recipient wires to IFTTT, Home Assistant, custom bot, etc.
- **XMPP / IRC** — old but open; some organizing communities still on them.

The protocol is transport-pluggable. Adding a transport adds ~50 lines to a bridge implementation; lexicons don't change.

### Pure-protocol features

These are protocol-native — atproto records and bridge logic do the work. They all dispatch via the transport menu when the time comes, but the *coordination semantics* are Kettle's.

- **Page-to-group fanout.** A `community.paging.group` record defines membership; one sender pages the group; all members' bridges fire. The rapid-response primitive: "ICE 5min out at 14th & Wabash" → 50 members notified within seconds.
- **Threaded async conversation.** `replyTo` field enables multi-party async messaging with consent at every edge — replies pass through the original sender's allow-list before delivery. **This is decentralized group DMs without centralized chat infrastructure** — the use case Bluesky's roadmap covers, but federated rather than centralized at `chat.bsky.social`.
- **Page-back without phone exchange.** Recipient gets SMS via Twilio short code; replies SMS; bridge translates the SMS reply into a `community.paging.send` record on recipient's PDS, encrypted to original sender's pubkey. Bidirectional flow, neither end ever learns the other's number.
- **Read/delivery receipts.** Bridge writes `community.paging.received` records back to recipient's PDS. Sender's tooling subscribes — knows "delivered to bridge at 23:42" without knowing if recipient read it.
- **Auto-acknowledgments.** Recipient configures bridge with auto-replies ("in a meeting, will respond after 8pm"). Bridge writes ack records to sender's PDS; sender's tooling surfaces them.
- **Time-locked pages.** `deliver_after` field on page record. Bridge holds, dispatches at scheduled time. Useful for organizing prep ("remind coordinator at 8am about 9am action") and staged escalation.
- **Dead-man's switch.** `community.paging.deadmansSwitch` record: "if I haven't checked in by time T, fire this page to these contacts." Bridge polls; if no check-in, fires. Critical for organizers in physically-risky situations (protest arrests, ICE encounters, traveling through hostile territory). Replaces centralized-SaaS dead-man tools with on-PDS, federated, self-hostable equivalent. (Threat-model open question in §9: who runs the polling bridge if the recipient's own bridge goes offline alongside them.)
- **Anonymous tip line.** Wildcard allow-list ("anyone can page; content held in a moderation queue"). Recipient reviews queue, decides which to dispatch. Use cases: union tip line, mutual aid intake, journalist source contact. Cryptographic sender authentication via atproto signing — but the sender's DID can itself be a fresh, unprofiled identity, preserving practical anonymity.
- **Geofenced paging.** Allow-list entries can specify "only deliver if I'm in geo X." Recipient publishes location records (or keeps last-location at bridge). Filters before dispatch.
- **Per-context routing.** Different bridges for different categories. Work category → Slack webhook; organizing → Signal-via-self-bridge; family → APNs via Kettle app. Routing rules live in the bridge-side allow-list; recipient configures via authenticated API to each of their bridges, or routes everything through one meta-bridge that re-dispatches based on category.
- **Quorum paging / confirmation chains.** Group page; dispatch fires only when N members ack within M minutes ("≥10 confirmed coming, GO"). Bridge logic, no third party.
- **Cross-bridge federation.** KC DSA's bridge ↔ Sonoma DSA's bridge ↔ propter-page-bridge. Members on bridge A can page members on bridge B. Bridges subscribe to firehose records targeting their users regardless of which bridge published them.
- **Bridge-as-a-service for orgs.** A union, mutual-aid org, or DSA chapter runs a single bridge for all members. Members keep PDSes elsewhere. Org gets observability over its rapid-response channel; members get a free federated paging service. No SaaS dependency.
- **Public alert channels.** Any DID can run a write-only-broadcast Kettle account. Subscribers add it to *their own* allow-lists with rate caps. Enables city-wide rapid-response broadcasts (chapter emergency channels, weather/disaster alerts, ICE-spotted reports) without any centralized broadcast service.
- **Atproto-native moderation.** Bridge applies labels: spam-list label → drop; positive-reputation label from a chapter labeler → trust; quiet-hours violation → hold for review. Pure atproto, composable with existing labelers.

The transport menu is what gets paging out to phones; the pure-protocol features are what makes Kettle a paging *system* rather than a notification dispatcher with a wrapper.

## 7. Encryption: open tradeoff

Three options for content/graph privacy:

**(a) Encrypted body in public records (default for v1).** Graph public, content private. Cheap. Good enough for most organizing — the *content* of a rapid-response page is the secret, not the relationship.

**(b) Sealed-sender envelope.** Sender publishes to a paging mailbox without revealing recipient in cleartext. Bridge attempts decryption on records, only succeeds on records meant for it. Graph + content both private. Heavier — bridge attempts decryption per record. Real-world OK because you can shard the mailbox by hash bucket.

**(c) Use `chat.bsky.convo` DM substrate.** Pages flow as structured DM payloads. Private from public firehose but centralized at Bluesky chat service. Loses federation purity; gains centralized-but-private content path. Requires Bluesky-team coordination.

Recommendation: ship (a) for v1; (b) is the principled long-term answer; (c) is the "we want this in production tomorrow" path if Bluesky team will play.

## 8. Composition

**With Smoke Signal events.** RSVP record on Smoke Signal could grant time-bounded auto-entry to the event-organizer's allow-list. RSVP yes to May Day → organizer's DID gets `expires_at = event.end + 1h` paging rights. Event-time reachability without manual allow-list edits. Coordination needed: Smoke Signal lexicon ownership lives in `community.lexicon`; paging should probably live there too.

**With Snug** (sibling project, encrypted MLS group chat for atproto, same author). A Snug group can be the *target* of a Kettle page (page-to-group). Membership in a Snug group implies pageable-by-group-organizer or pageable-by-any-member. Snug's MLS encryption protects in-group conversational context; Kettle extends the reach of any in-group signal out-of-band when the recipient isn't actively in the chat. The two protocols share substrate (atproto DIDs, PDS records) but solve different problems with different cryptographic shapes — Snug is bidirectional persistent encrypted chat; Kettle is one-way ephemeral consent-mediated paging.

**With labelers.** Bridge-side capability becomes a public label (`paging-enabled`, `paging-emergency-allowed`) so other organizers know who to expect a response from. No phone numbers leak.

**As a federated group-DM substrate.** Add `replyTo` and the system becomes async multi-party threading with allow-list-mediated membership. Each user has one inbox (their allow-list defines who can post to it). Threads form when senders reply to pages and the original sender's allow-list permits them. This is decentralized group DMs — the use case Bluesky's roadmap covers, but federated rather than centralized.

## 9. Open questions for atproto-community

1. **Namespace placement.** `community.lexicon.paging.*` vs `chat.kettle.paging.*` vs a new top-level. Smoke Signal team is migrating events into `community.lexicon.*`; coordination is the load-bearing call. *(Lexicon-community input requested before locking schema.)*

2. **Bridge auth pattern.** Is `community.paging.bridgeAuthorization` the right shape? Or should bridges authenticate via standard atproto OAuth scopes against the recipient's PDS? The custom delegation is more federation-pure; OAuth is more pattern-matched to existing infrastructure.

3. **Encryption substrate.** (a) public encrypted body / (b) sealed-sender envelope / (c) `chat.bsky.convo` DM payloads. Each has different federation properties and different coordination requirements. v1 plans (a) but (b) is the right long-term shape — interested in feedback on whether sealed-sender is over- or under-engineered.

4. **Jetstream production-readiness.** Jetstream is documented as "not a stable protocol API." For self-hosted bridges this is acceptable; for hosted public bridges (Tier 0) and app-based clients (Tier 3) it's a production risk. Is there a stable firehose subscription path planned?

5. **Cross-bridge federation.** A user's `bridgeEndpoint` on PDS-A may name a bridge run by a different operator than PDS-A's host. Does the protocol need explicit cross-bridge addressing semantics, or is "the recipient's PDS points at one bridge, that bridge handles all of their pages" sufficient?

6. **Rate limiting and abuse.** Per-sender rate limits and global daily caps are bridge-side state, enforced bridge-side. Adversarial sender posts a million page records; bridge filters, but the records are still on the firehose. Is there a firehose-level abuse mitigation we should care about, or is "ignore at the bridge" sufficient?

7. **Bridge-side state portability.** The allow-list lives at the bridge, not on PDS — that's the trust-graph-privacy fix in v0.2. Cost: switching bridges requires migrating the list. Is the right answer (a) a standard `community.paging.exportAllowList` XRPC the recipient invokes against their old bridge before pointing PDS at a new one, (b) a manual JSON dump via the bridge's admin UI, or (c) a more principled approach like signed allow-list snapshots that any bridge can ingest? Recipients without technical fluency need this to not be a brick wall.

8. **Dead-man's-switch threat model.** Who runs the polling that fires the dead-man's-switch page when recipient stops checking in? If it's the recipient's own bridge, it can't fire after the bridge itself goes offline (recipient's machine seized, network cut, recipient kidnapped *with* their phone that's running the Tier 3 bridge). External-poller designs introduce a third-party trust dependency. Probably the right answer is "designated escalation bridge" — recipient pre-authorizes a chapter or trusted-friend bridge to run the poller, with an attestation flow that can't be silenced by capturing only the recipient's primary device. Wants atproto-community input on the attestation pattern.

9. **Anonymous tip line + atproto signing.** Atproto signing implies the sender's DID is in the record. "Anonymous" in this design means "DID is fresh and unprofiled" rather than "no DID." Is that practical-anonymity guarantee strong enough for high-stakes use cases (whistleblowing, abuse reporting)? Or should the lexicon support a sealed-sender mode that strips the sender DID at the cost of losing source authentication?

## 10. v1 scope

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

## 11. Comparisons

**vs. Signal.** Signal does end-to-end encrypted 1:1 and group messaging; people on Signal are reachable. Kettle-paging is *not a chat replacement* — it's a paging primitive that can dispatch *to* Signal as one of many transports. They compose: Signal handles the conversation, Kettle decides who's allowed to start one and routes the alert.

**vs. Action Network / Spoke / Hustle.** Existing organizer-SMS tools are centralized SaaS with phone-number databases. Kettle-paging puts the directory at the bridge (which can be self-hosted, federated, or never have phone numbers in it at all in Tier 3). Trade-off: existing tools are operational today and have institutional buy-in; Kettle is a sketch.

**vs. Twitter SMS-tweet.** SMS-tweet was opt-in broadcast (public phone-number list, fanout). Kettle-paging is bilateral consent-mediated summon. Different shape; different threat model.

**vs. Bluesky group DMs (planned).** When they ship, Bluesky's group DMs will be centralized at `chat.bsky.social`. Kettle-paging is federated (PDS-anchored consent, pluggable bridges). Different federation properties; same underlying need for "talk to N people privately."

---

## License

This document is released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
