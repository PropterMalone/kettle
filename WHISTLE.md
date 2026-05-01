# Whistle: Public Broadcast Channels on AT Protocol

A design for opt-in, subscriber-private broadcast channels on AT Protocol. A publisher posts once; subscribers receive out-of-band — without the publisher learning who is subscribed.

**Status**: Design v0.1 stub, May 2026. Sibling spec to [Kettle](./PAGING.md) (paged-summon counterpart); pair overview at [README](./README.md). Pre-implementation; lexicon not yet locked.

---

## 1. Problem

A senator's office wants to alert constituents to vote-now moments. A union local wants to send strike-call announcements. A mutual-aid org wants to ping its volunteer pool when a need surfaces. The publisher wants to *broadcast* (one publish, many subscribers) without holding a centralized list of subscribers' phone numbers or identities.

Existing tools fall short:

- **Bluesky posts** (`app.bsky.feed.post`) — public broadcast, but pull-delivered through the app/feed. No out-of-band push when the app is closed; no rate-cap hand to the subscriber.
- **Mass-text services** (CallHub, Hustle, Mobilize) — phone-number databases concentrated at the operator. The publisher knows every recipient by name and phone.
- **Email lists** — same problem; recipient list lives at the publisher.
- **RSS / Atom** — pull, not push; no phones rung.

Whistle inverts the broadcast model: publisher posts; subscribers self-select; *the publisher does not know the subscriber list*. Subscribers' bridges watch the publisher's records and dispatch out-of-band via the subscriber's chosen transport.

**Scope.** Whistle is opt-in broadcast: content public (or shared-key encrypted, optional), subscribers private. Whistle suits use cases where the message is *meant* to be heard widely but the subscriber list shouldn't be a centralized PII pool — civic mobilization, public alerts, organization-wide announcements. It does not suit use cases where message content must stay hidden, or where the publisher must know every recipient by name.

Whistle is *not* targeted paging. Body-private + recipient-targeted semantics — the inverse exposure profile — belong to the sibling primitive [Kettle](./PAGING.md). The two compose for compartmentalized broadcast (§8).

---

## 2. Design principles

**Subscriber-private subscription.** The subscriber list lives at each subscriber's own bridge — never on the publisher's PDS, never on the firehose. An adversary cannot enumerate "who follows publisher X" from the public substrate.

**Content is public by default.** Broadcasts go on the firehose in cleartext. Optional shared-key encryption (§7) wraps content for distribution to a known subscriber set when content audibility shouldn't be wider than the subscriber set.

**Pluggable dispatch.** Same bridge software as Kettle. The subscriber configures their bridge to watch publishers and dispatch via SMS / Signal / push / etc. when broadcasts land.

**Subscriber-side rate cap.** Subscribers configure rate limits, quiet hours, and category filters per publisher. Aggressive publishers cannot bypass.

**Subscription is by-publisher, not by-message.** Once subscribed to publisher X, the subscriber receives every broadcast from X (subject to rate caps and category filters). The subscriber's only knob to refuse a single message is to unsubscribe entirely, the same way mailing-list subscriptions work.

---

## 3. Architecture

```
Publisher's PDS                    Subscriber's bridge          Subscriber
───────────────                    ───────────────────          ──────────
Posts                              Subscribes filtered
community.alert.broadcast          Jetstream
{content, category, channel}       ────────────────────►
                                   sees record from
                                   subscribed publisher

                                   Consults BRIDGE-PRIVATE
                                   subscription list
                                   (rate, hours, category)

                                   Dedups by record CID
                                   (replay protection)

                                   Dispatches via configured
                                   transport ────────────────► SMS / Signal / push / ntfy

                                   Writes                       ◄── subscriber manages
                                   community.alert.delivered        subscription via
                                   to subscriber's PDS              authenticated API
                                   for audit trail                  to bridge
```

The subscriber's bridge filters Jetstream for `community.alert.broadcast` records from publishers in the subscriber's subscription list, applies the subscriber's rate / category / quiet-hour rules, and dispatches.

Subscription is bridge-side state. The subscriber configures it through an authenticated API to *their own* bridge — never via PDS write. The publisher publishes; the publisher does not maintain the subscriber list.

---

## 4. Lexicons

### `community.alert.broadcast` (v1)

Publisher posts to their own PDS. Public.

```json
{
  "$type": "community.alert.broadcast",
  "content": "Vote now — Senate floor 4pm",
  "category": "civic",
  "channel": "senator-x-mobilization",
  "createdAt": "2026-05-01T20:00:00Z"
}
```

| Field | Type | Notes |
|---|---|---|
| `content` | string | Cleartext body, or shared-key ciphertext under §7B. Length budget: TBD; longer than Kettle's SMS-segment limit since dispatch isn't always to SMS. |
| `category` | enum | `urgent` / `routine` / `civic` / TBD. Cleartext on firehose. Subscribers filter on this. |
| `channel` | string, optional | Channel slug for publishers running multiple channels under one DID. Subscribers can subscribe per-channel. |
| `createdAt` | datetime | ISO 8601 UTC. |

### `community.alert.channelMetadata` (v1)

Publisher publishes per-channel discovery info on their own PDS. Rkey: channel slug, or `self` for single-channel publishers.

```json
{
  "$type": "community.alert.channelMetadata",
  "name": "Senator X mobilization alerts",
  "description": "Vote-now alerts for moments where constituent action matters",
  "category": "civic",
  "subscriptionPolicy": "open",
  "ratePolicy": { "expectedFrequency": "weekly", "maxPerDay": 5 },
  "createdAt": "2026-05-01T19:00:00Z"
}
```

`subscriptionPolicy`: `open` (anyone may subscribe) or `gated` (subscriber must obtain a capability OOB). `ratePolicy` is publisher-declared expectation; subscriber's bridge enforces actual caps.

### Bridge-side state (not a lexicon)

Subscription schema is bridge-internal. Subscriber-facing semantics:

```yaml
subscriptions:
  - publisherDid: did:plc:senator...
    channel: senator-x-mobilization
    ratePerDay: 3
    quietHoursUtc: { start: 4, end: 11 }
    transportOverride: push
    expiresAt: 2026-12-31T23:59:59Z
```

`channel` may be omitted to subscribe to all channels under that publisher DID. Per-publisher quiet hours intersect with global subscriber quiet hours.

### `community.alert.delivered` (v1, audit trail)

Bridge writes after dispatch.

```json
{
  "$type": "community.alert.delivered",
  "sourceBroadcast": "at://did:plc:.../community.alert.broadcast/...",
  "dispatchedAt": "2026-05-01T20:00:42.123Z",
  "transport": "push"
}
```

### v1.1 stubs

- **`community.alert.channelKey`** — distribution mechanism for shared-key encrypted channels (§7B). Key rotation on member churn is the open question.
- **`community.alert.revoke`** — explicit retraction record bridges should honor (e.g., publisher pushed a wrong alert, wants to suppress future dispatch). Best-effort; some recipients will already have been dispatched.

---

## 5. Bridge tiers

Same tier model as Kettle (see [PAGING.md §5](./PAGING.md)). Same software image; configuration toggles paging vs. broadcast modes. A bridge instance can serve both at once.

| Tier | Operator | PII at operator? | Threat-model fit |
|---|---|---|---|
| 0 | Hosted public bridge | Yes (subscriber phone numbers if SMS) | Convenience default. |
| 1 | Community / chapter | Yes (chapter's Twilio) | Chapter is the trust boundary. |
| 2 | Self-hosted | Yes (your Twilio) | Only you and Twilio. |
| 3a | App push, FCM/APNs | **No** | Strong cryptographic posture, platform-dependent. |
| 3b | App push, ntfy.sh self-hosted | **No** | Sovereign. |

---

## 6. Capabilities

### Transport menu

Same as Kettle (SMS, push, Signal, Matrix, Discord/Slack/Telegram, email, voice, webhook, XMPP/IRC). Adding a transport is ~50 lines of bridge code.

### v1 ships

- Subscription management API (bridge-internal, subscriber-managed).
- Per-publisher + global rate limiting, quiet hours, category filters, expirations.
- Delivery receipts via `community.alert.delivered`.
- Replay protection — bridges MUST dedup by record CID.
- Channel discovery via `community.alert.channelMetadata`.

### v1 protocol-supports (reference bridge skips)

- **Cross-bridge federation.** Any bridge can subscribe to the firehose for any publisher.
- **Atproto-native moderation.** Bridges can apply labelers — drop labeled-spam channels, suppress dispatch on flagged content.

### Future (v1.1+)

- **Shared-key encrypted channels** (§7B) — channel content visible only to subscribers holding the key.
- **Subscriber-private opt-in via capability tokens** — for `gated` channels where the subscriber needs an OOB invite to subscribe.
- **Channel-key rotation** on member churn.
- **Geofenced broadcast** — coordinate with `community.lexicon.location`.
- **Bridge-as-a-service for organizations** with multi-channel management.
- **Reply-to-broadcast via Kettle** — subscribers can page the publisher in response, escalating from public broadcast to targeted summon (composes via §8).

---

## 7. Content privacy

Two options. v1 ships A.

### A. Cleartext broadcast (v1)

`content` is cleartext on the firehose. Anyone with firehose access reads the message. The privacy property is *who is subscribed*, not *what is said*.

- Suitable for public-by-design content: civic alerts, public mobilization calls, open organizational announcements.
- Subscriber-list privacy is the only privacy property; content has none.

### B. Shared-key encrypted channel (v1.1+)

Publisher encrypts to a channel-shared key distributed to subscribers OOB during channel onboarding. Adversary on firehose sees an opaque blob.

- Content visible only to subscribers holding the key.
- Trade: subscribers must hold and protect the channel key; key rotation on member churn is a real operational burden.
- Member churn problem: when a subscriber leaves (or is removed), past content remains decryptable to them. Key rotation invalidates future broadcasts; doesn't retroactively re-encrypt history.

### Sealed-publisher mode (research direction)

Hiding the publisher's identity is non-trivial — at minimum, subscribers need a way to authenticate broadcast content as coming from the channel they subscribed to. Out of v1 scope; flagged in §9 Q6.

---

## 8. Composition

**With Kettle (cell-structure relay).** See [PAGING.md §8](./PAGING.md). Whistle channel feeds independent Kettle relays; trust splits across publisher / relays / members. Subpoena topology: publisher subpoena yields a relay list; relay subpoena yields one affinity group's members.

**With Smoke Signal events.** RSVP to a Smoke Signal event auto-subscribes the user to the event organizer's Whistle channel for the event window (TBD: lexicon-level integration).

**With labelers.** Channels surface as labelable subjects — `civic-alert`, `mobilization-channel`, `verified-publisher`. Labelers help subscribers discover trustworthy channels and skip spam.

---

## 9. Open questions

1. **Namespace placement.** Recommend `community.lexicon.alert.*`, alongside Kettle's `community.lexicon.paging.*` and Smoke Signal. Push back if there's a reason to separate.

2. **Channel discovery.** How do new subscribers find channels? Bluesky search, dedicated atproto labelers, out-of-band only, channel directory feed? Probably a mix; ergonomic default for non-technical subscribers is the priority.

3. **Spam mitigation.** Subscriber rate caps protect the recipient. Substrate-level signals (labelers, atproto-native abuse reports) would help with channel-level spam. What's the right shape?

4. **Shared-key channel key distribution and rotation.** Member churn is the hard part. Is `community.alert.channelKey` the right shape, or should this lean on MLS-style group-key trees (composing with Snug)?

5. **Verifying publisher authenticity.** A subscribed channel suddenly publishes incriminating content — was it a publisher key compromise, or a turned publisher? Audit-friendly publisher-rotation patterns wanted.

6. **Sealed-publisher mode.** Hiding publisher identity is research; sketch wanted before v1.1 commits.

7. **Reply-to-broadcast routing.** When a subscriber wants to page the publisher back (compose with Kettle), which DID gets paged — the publisher's persistent DID, a designated `community.alert.replyTarget`, or a channel-specific paging endpoint?

8. **Constituency validation.** Same posture as Kettle — designer's projection until organizers in the prompting use case have weighed in.

---

## 10. v1 scope

**Build:**

- Three lexicons (`broadcast`, `channelMetadata`, `delivered`).
- Reference bridge — same Docker image as Kettle, broadcast mode toggle.
- Tier 3 reference client — same Expo/React Native app as Kettle, with a separate UI surface for Whistle subscriptions and clear visual distinction from Kettle pages.
- Hosted public bridge — single deployment of the reference, Whistle + Kettle modes both enabled.

**Skip for v1:**

- Shared-key encrypted channels (§7B).
- Sealed-publisher mode.
- `community.alert.channelKey`, `community.alert.revoke`.
- Channel directory / dedicated discovery feed.

**Estimate.** Sharing infrastructure with Kettle, the incremental Whistle work is ~1 week of bridge code + ~3-4 days of client UI. The reference bridge ships both primitives in one image. Biggest risks: avoiding mode-confusion in the client UI (cardinal design risk for the pair), and Jetstream stability (same as Kettle).

---

## 11. Comparisons

**vs. Kettle (sibling primitive).** Inverse exposure profile. Kettle = body-private + recipient-public-via-`targetDid`. Whistle = content-public + subscribers-private. Different primitives in the same family, deliberately separated to prevent mode-confusion errors. Composes via the cell-structure relay pattern (§8).

**vs. Bluesky posts + custom feeds.** Bluesky posts are pull-delivered through the app/feed. Whistle adds out-of-band push (SMS, Signal, etc.) and subscriber-private subscriptions. Bluesky's follow graph is public; Whistle's subscription list is bridge-private.

**vs. mass-text services (CallHub, Hustle, Mobilize).** Centralized phone-number databases at the operator. Whistle keeps subscriber lists at each subscriber's bridge — federated, no centralized PII pool, no single-subpoena blast radius.

**vs. RSS / Atom.** Pull, not push. RSS doesn't ring phones, doesn't dispatch out-of-band. Whistle is RSS with bridge-mediated dispatch and subscriber-private subscriptions.

**vs. email lists.** Recipient list at the publisher. Subpoena on the publisher yields the full subscriber list. Whistle's subscriber list lives at subscribers' bridges instead.

**vs. Twitter alert SMS (deprecated).** Twitter SMS-tweet was opt-in broadcast with public phone-number list. Whistle hides the subscriber list.

---

## 12. Threat model

The risks v1 leaves unsolved, named.

### Adversaries considered

- **State-level surveillance with firehose access.** Sees broadcast content (cleartext under §7A) and metadata. Sees publisher DID + content + timing. Cannot enumerate subscribers from the firehose.
- **Subpoena on the publisher.** Yields broadcast content (already public) and channel metadata. Does *not* yield the subscriber list (lives at subscribers' bridges).
- **Subpoena on a subscriber's bridge.** Yields that subscriber's subscription list (which channels *they* subscribe to). Does not yield the channel's other subscribers.
- **Compromised subscriber device.** Reveals that subscriber's subscription state and locally-decrypted alerts (under §7B).

### Unmitigated in v1

- **Coalition subpoenas across multiple subscriber bridges.** Reconstructs partial subscriber lists by aggregation. Privacy is per-subscriber, not collective; a hostile actor with broad subpoena power can reconstruct membership over time.
- **Behavioral correlation.** A subscriber's response to a broadcast (showing up to a rally, posting a quote, retweeting) reveals subscription. Whistle protects substrate-level subscription privacy, not behavioral privacy.
- **Malicious channel operator.** A channel publishes incriminating content and points authorities at the subscriber list. Mitigation: subscribers should use Tier 3 push (no PII at the bridge) or self-hosted bridges. v1 cannot prevent operator hostility entirely.
- **Replay attacks.** Broadcast records are permanent on the firehose. Bridges MUST dedup by record CID; without dedup, an adversary can replay old urgent broadcasts to trigger false dispatches.
- **Publisher key compromise.** A compromised publisher key lets an attacker broadcast as that publisher to all subscribers. Mitigation: publisher-key rotation pattern (TBD; §9 Q5).

### Out of scope

- Quantum-future cryptanalysis (rotate when atproto-substrate does).
- App-store revocation: Apple/Google can pull the Tier 3 mobile app. Sovereign push (Tier 3b ntfy.sh) is the v1 escape.
- Network-level anonymity (Tor-equivalent). Subscribers connecting to their bridge from identifiable IPs is out of scope.

### Legal exposure for publishers

Publishers operating high-stakes channels (organizing for direct action, abortion-network coordination) carry legal exposure for the *content* they publish. Same questions as Kettle Tier 1 operators: [Stored Communications Act](https://www.law.cornell.edu/uscode/text/18/2701), [ECPA](https://www.law.cornell.edu/uscode/text/18/2510), state-level conspiracy / "hindering apprehension" statutes. Whistle's subscriber-private property limits *bridge* operator exposure but does nothing to reduce *publisher* exposure on what gets published.

---

## Appendix A: Project name

A whistle is for public attention — referee whistles, train whistles, dog whistles, whistleblowers. Loud, clear, broadcast. The instrument carries the use case in the metaphor: you blow a whistle when you want to be *heard*.

The pair: Kettle is the small private kettle for trusted-channel paging; Whistle is the loud public whistle for broadcast attention. Same family of communication infrastructure for organizing — different shapes for different needs.

If the name fails constituency validation, rename is cheap — the lexicon namespace is the actual identity.

---

## License

This document is released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Lexicons themselves are intended for [CC0](https://creativecommons.org/publicdomain/zero/1.0/) on adoption into `community.lexicon.*`.
