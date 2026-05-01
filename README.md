# Kettle & Whistle: Consent-Mediated Communication on AT Protocol

A pair of complementary primitives for organizing-grade messaging on AT Protocol — designed to ship together with shared infrastructure and deliberately distinct semantics.

**Status**: Design v0.4, May 2026. Pre-implementation; lexicons not yet locked.

---

## The pair

**[Kettle](./PAGING.md)** — *targeted summon*. One sender pages one named recipient. Body encrypted to recipient; recipient's bridge dispatches via SMS / Signal / push. Body-private + recipient-public-via-`targetDid`.

**[Whistle](./WHISTLE.md)** — *broadcast announcement*. One publisher broadcasts; subscribers self-pull and self-dispatch. Content-public + subscribers-private.

The two have **inverse privacy profiles**, serving inverse use cases:

| | Kettle | Whistle |
|---|---|---|
| **Sender intent** | Reach this specific person | Get this signal heard widely |
| **Body** | Encrypted; recipient-only | Cleartext (or shared-key) on firehose |
| **Graph privacy** | Recipient on firehose; sender knows recipient | Subscriber list at the subscriber's bridge; publisher does not know subscribers |
| **Threat-model fit** | Acceptable when metadata exposure costs less than missing the page (emergency rapid-response) | Acceptable when audibility is the point and subscriber lists shouldn't be a centralized PII pool (civic mobilization, public alerts) |

The primitives are deliberately separated — different lexicons (`community.paging.*` vs. `community.alert.*`), different UI affordances, different reference clients — to prevent mode-confusion errors. Conflating "I want to send this privately" with "I want to broadcast this widely" leads to either a leaked recipient list or an over-shared message; naming and ABI conflation is how that error happens.

---

## Shared infrastructure

The pair runs on the same bridge software:

- Same Jetstream subscription primitive.
- Same dispatch matrix — SMS, push, Signal, Matrix, Discord/Slack/Telegram, email, voice, webhook, XMPP/IRC.
- Same Tier 0–3 deployment shapes — hosted, chapter, self-hosted, app-push (with sovereign-push variant).
- Same audit-trail records (`community.paging.received`, `community.alert.delivered`).
- Same bridge auth and capability model.

A single bridge instance serves both modes. A chapter running its bridge can dispatch Kettle pages for its members and Whistle alerts for civic channels on the same hardware, with one operator and one set of dispatch credentials.

The reference Tier 3 client (Expo/React Native) shows both Kettle pages and Whistle alerts in one app — clearly distinct UI surfaces, deliberately not interchangeable, sharing the underlying transport layer.

---

## Composition: cell-structure relay

The pair composes for **compartmentalized broadcast**:

- A central organizer publishes to a Whistle channel (alert content public; subscribers private).
- Independent **relays** — Kettle bridges with their own allow-lists — subscribe to the Whistle channel.
- When the organizer broadcasts, each relay translates the broadcast into Kettle pages dispatched to its own member list.
- The organizer doesn't know individual recipients — only relays. Each relay knows its own members but not the others. Subpoena lands on one party at a time; the full graph never lives in one place.

Combined with [Kettle §7B dual-DID identity separation](./PAGING.md), members appear only as paging DIDs, not persistent identities. Subpoena on a relay yields a list of paging DIDs, not real names.

The pattern needs **≥2 relays per Whistle channel** for the privacy property to mean anything. Single-relay deployments collapse to "the relay is the audience."

See [PAGING.md §8](./PAGING.md) for the full composition pattern.

---

## Sibling projects

- **[Snug](#)** — encrypted MLS group chat for atproto. Same author family. Snug protects in-group conversation; Kettle extends reach when a recipient is outside the chat; Whistle broadcasts to subscribers of an organization-level channel. Snug spec lives in its own repo (TBD link).

- **[Smoke Signal](https://smokesignal.events/)** — atproto event RSVP. Composes with Kettle via auto-grant of paging rights to event organizers; composes with Whistle via auto-subscribe to event-organizer channels for the event window.

---

## Repository layout

- `README.md` — this file (overview)
- `PAGING.md` — Kettle spec (current: v0.4)
- `WHISTLE.md` — Whistle spec (current: v0.1 stub)
- `PAGING.html`, `WHISTLE.html` — pandoc-rendered versions

---

## Status and roadmap

**Pre-implementation.** Both specs are design documents; no reference code exists yet. Lexicon names are proposed but not yet submitted to `community.lexicon.*`.

**Next steps** before lexicon-lock:

1. Constituency validation — sit down with rapid-response organizers, civic-mobilization operators, and union locals to test whether the proposed primitives match the actual use cases.
2. Coordinate namespace placement with the `community.lexicon.*` editors.
3. Reference bridge — ~3 weeks of engineering for the Kettle-only path; ~1 additional week for the Whistle layer on top.
4. Reference client (Expo/React Native) — ~2 weeks, assumes a bootstrapped Expo project.

**Decision dependencies.** Some open questions (Kettle §9, Whistle §9) need protocol-level input from atproto contributors before lexicons can lock. The biggest are namespace coordination, bridge auth pattern (custom delegation vs. atproto OAuth scopes), and Jetstream production-readiness for Tier 0 hosted bridges.

---

## License

This documentation is released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). The lexicons themselves are intended for [CC0](https://creativecommons.org/publicdomain/zero/1.0/) on adoption into `community.lexicon.*`.
