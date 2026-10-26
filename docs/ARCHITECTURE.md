# Architecture: SummonerDex, an open League of Legends persona registry

This is the technical architecture of SummonerDex, an open registry that links League of Legends personas (pros, streamers and other public players) to their Riot accounts. It describes the data model, how data enters and leaves the registry, how privacy and removal work, and the plan for building it.

Three rules shape everything:

1. **D1 is the only write path and the only store of subject data.** Cloudflare D1 is the database. Git holds rules, the publisher registry and the public decision log. No persona, account or claim enters public Git, because forks and clones keep history after any rewrite.
2. **Truth and publication are separate.** `evidence_tier` says how sure we are. `disclosure_basis` records what public tie the person made; it is metadata that consumers can filter on. Claims publish by default and are hidden on request: one `is_public()` predicate (1.9) serves every approved or unreviewed claim once stored, unless it is held, hidden, suppressed, stale or caught by the minor rule.
3. **Every non-public state looks the same from outside.** Hidden, delisted, erased, withheld and never-existed return identical bytes. Compliance events use one action. Suppressed ingest looks like any accepted line.

## 1. Data model

### 1.1 Entities

The main terms:

- A **persona** is the public identity of a person, such as a pro player or streamer (1.10).
- A persona's **shell** is its public record without any claims: ID, slug, display name and status.
- An **account** is what a Riot ID pointed to during a time interval (1.4).
- A **claim** is one fact about a persona, such as "this persona owns this account". Claims never change once written (1.5).
- A **reference** is one source's statement of a claim. One claim can have many references (1.5).
- A **publisher** is a product or person allowed to send claims, such as a stats site (1.13, section 5).
- A **submission** is a request from a community member to add or change something (3.1).
- A **facet** is a label consumers filter on, such as `pro` or `streamer_declared` (7.1).
- A **tombstone** is a signed record that something was removed, with no detail about what it was (1.12).
- A **suppression record** stops a hidden item from coming back through later ingest (1.12).
- An **audit row** records every change (2.2).

Stewards run the project, and Reviewers check submissions (11.1). Section 2.1 says which store owns each entity.

### 1.2 Identifiers and slugs

IDs are prefixed ULIDs (`per_`, `acc_`, `clm_`, `ref_`, `sub_`, `bat_`, `pub_`, `usr_`, `ctr_`, `tmb_`, `vrf_`, `sup_`). ULIDs sort by time, which makes cursors cheap. Publisher IDs are ULIDs like every other ID; the readable name lives in the registry entry (1.13). `usr_` is the private submitter record tied to a Discord, GitHub or Twitch login. `ctr_` is the public contributor pseudonym shown on decisions. The two are minted separately so nobody outside can join them, because Discord's terms bar sharing Discord API data.

Slugs match `^[a-z0-9][a-z0-9-]{2,39}-[0-9a-hjkmnp-tv-z]{4}$`: a base from the public handle plus a four-character random Crockford base32 suffix (digits and lowercase letters, without i, l, o and u) that is always present (`example-mid-7q2k`). If the suffix were added only on a collision, a suffixed slug would show that the bare base is taken, which tells the world that a now non-public persona once held it. That would break the identical-404 rule (rule 3). An always-present random suffix makes every slug's shape tell nothing.

A slug is minted only when a persona first becomes `listed`. Before that, the persona is addressed internally by its `per_` ID, so rejected or provisional proposals use no slug and squatting a famous handle wins nothing. Slugs never contain legal names and are never reused. They never change, with one exception: a Steward may approve one change for a public slug that impersonates someone, which leaves a redirect (2.4). Every minted slug is reserved in `private.slug_reservations`, so the reservation list cannot reveal who delisted. Consumers store `persona_id` as the foreign key and use slugs only for URLs, which is why that one slug change breaks no join.

### 1.3 Riot ID normalization rule

The registry stores `game_name` and `tag_line` exactly as received. The **join key** is the folded form of a Riot ID that the registry uses to match accounts. It is an index only:

```
fold(s)   = remove(full_casefold(NFC(s)), White_Space ∪ General_Category=Cf)
name_key  = fold(game_name) + "#" + fold(tag_line)
```

- Use NFC, never NFKC, because fullwidth Latin accounts exist.
- Use full case folding, never locale lowercasing, because the dotted capital İ exists.
- Do no length or character validation. Riot's own example has a space, and legacy IDs break the FAQ limits.
- Never put the region in the key or parse it from the tag. Region is a per-game attribute of the PUUID (Riot's permanent player ID, 1.4).

Whitespace stripping is the open risk: Riot has said a space collision bug is fixed, which is to be confirmed. When two different raw strings fold to the same key, both accounts are marked `ambiguous` and never bound automatically.

| Raw input | `name_key` | Pins |
|---|---|---|
| `Example Mid#EUW` | `examplemid#euw` | Space removal, case fold |
| `ＥＸＡＭＰＬＥ#7777` | `ｅｘａｍｐｌｅ#7777` | Fullwidth stays distinct from ASCII |
| `İzmir Example#TR1` | `i̇zmirexample#tr1` | İ folds to `i` plus U+0307 |
| `Zero<U+200B>Width#NA1` | `zerowidth#na1` | Cf removal |

The fold ships as `@summonerdex/idfold` (npm) and `summonerdex-idfold` (PyPI), sharing one `vectors.json` test file. The names carry no Riot trademark, because Riot's legal terms bar Riot marks in handles and as keywords. JavaScript has no full case folding, and Python's `str.casefold()` follows the interpreter's Unicode version. So both packages use tables generated from one pinned Unicode version. CI runs the vectors against the Worker and both packages, so a consumer in either language cannot drift from the registry. Both packages ship at launch; only the Python port of the resolver waits for phase 2 (1.14).

### 1.4 Accounts and binding

In the served record an account is one object with append-only `names[]` and `regions[]` arrays. Each entry carries `from` and `to` dates; the open entry (`to` is null) is the current value, copied into a `current` object so the common case needs no scan. A rename or a transfer closes the open entry and appends a new one. Nothing in the arrays is ever edited or removed: a wrong entry is closed by a retraction and a new entry appended. Each name entry carries its own provenance, because each observation came from a specific publisher at a specific time; freshness and corroboration sit on the account, because they say whether the account still belongs to the persona. The identity is `account_id`; `name_key` is a matching index only.

A **PUUID** is Riot's permanent player ID. Riot encrypts it differently for each API key, so two products see different PUUIDs for the same player. Binding is the step that attaches an incoming account claim to an existing account or creates a new one.

Riot IDs can be renamed for free every 90 days, and Riot says nothing about reuse of freed IDs, so the design assumes a freed ID can be taken at once. One question to Riot DevRel (Riot's developer relations team) is still open: may Riot-ID-to-persona mappings and PUUID-derived references pass from a publisher's product to the registry and on into dumps? Riot's policy bars apps "serving as a data broker between our API and another third-party company", and whether Riot IDs count is to be confirmed. The registry does not wait for the answer. From launch, publishers may send `subject_ref`, `riot_id_observed` (renames seen through the API) and an API-read `lol_region`. Binding uses the publisher's `subject_ref`, compared within that publisher only, together with the name and time window. `subject_ref` is dropped only if Riot says no in writing. Reliable reuse detection across publishers still waits for the project's own Riot key.

| Phase | Anchor | Rule |
|---|---|---|
| Launch | `name_key` and time window, plus a publisher's `subject_ref`, compared within that publisher only | Bind if `observed_at` is in `[first_seen, last_confirmed + 90d]` of an account already on that persona. Otherwise mint a `provisional` account, and open `contested` review if the `name_key` is bound to another persona. A rename is a new `account_link` from the owner, or from a Reviewer with an evidence page showing the new Riot ID, or a publisher's `riot_id_observed` under the same `subject_ref`. The rename sets the old account's `valid_to`. Same `subject_ref`, new `name_key`: rename. Same `name_key`, different `subject_ref` from one publisher: reuse, which mints a new `acc_` that inherits no persona links |
| Project production key (phase 5) | Registry PUUID in `private` | Resolve at ingest and refresh weekly. A PUUID mismatch mints a new account. Re-resolve everything after any key migration |

From launch, the publisher computes `subject_ref = base32(HMAC-SHA256(publisher_secret, puuid))[0:26]`, so the registry never sees a PUUID (5.4).

**Freshness without a key.** Bindings with a third-party basis become `stale_soft` 30 days after `last_confirmed` and `stale_hard` (out of default output) after 90 days, matching the rename cooldown, because reuse is not confirmed. Until publishers exist, only human observation moves `last_confirmed`. Without it, every seed account would drop out about 90 days after it was observed, and lookups would silently return nothing. So the registry offers three confirmation paths that need no Riot key:

1. **Owner re-confirmation**: an E3+ owner confirms "still mine" for each account (`POST /v1/owner/accounts/{acc_}/confirm`). Owner-basis accounts follow the owner re-verification clock (6.4: 12 months, with an email nudge every 90 days) instead of the 90-day name limit. This is because the person vouching is the account holder, who also gets every notice.
2. **Reviewer re-confirmation**: a Reviewer opens the live evidence URL, sees the same raw Riot ID, and records a new reference with `origin.kind: reviewer` and a fresh `observed_at`. This is a new observation, not a re-import, so it may move `last_confirmed` (1.5).
3. **Weekly queue**: a cron lists public third-party-basis accounts within 30 days of `stale_hard` and puts each in the Reviewer queue. For team, league and personal-site evidence it records only the HTTP status of the evidence URL and keeps no content. It never fetches platform pages such as twitch.tv, whose terms bar robots and scrapers.

A dashboard alert fires in `#stewards` when more than 10% of public accounts are within 30 days of `stale_hard`. A slow phase 2 then shows up as an alert at day 60, not as empty lookups at day 90.

### 1.5 Claims and references

A claim is unique per `(subject, predicate, object_key)`. When a second source confirms a claim (corroboration), the registry adds a reference instead of a duplicate claim. A batch revert deletes only that batch's references. A retraction is an explicit `asserts: false`; a missing claim never means "false".

`observed_at` means "confirmed against the publisher's own source". It may not be later than `received_at` plus 5 minutes or more than 2 years old, and a re-import never refreshes it. Relayed references without a date never outrank primary ones.

Predicates:

- `account_link`: persona to `{riot_id, lol_region?}`. `lol_region` comes from the evidence page or is read by a publisher from the Riot API, with its `source` recorded.
- `riot_id_observed`: a rename a publisher saw through the API, matched by `subject_ref` within that publisher (1.4).
- `social_link`: a URL typed by a user. Only X numeric IDs may come from an API.
- `affiliation`: team, league, role class, validity and raw contract end.
- `facet_attestation`: a boolean outcome.
- `external_id`: a Leaguepedia key or Liquipedia ID. Their content is never copied.
- `display_name`, `minor_status`, and `persona_merge` (48h hold).

One `account_link` per persona may carry `primary: true`, set by the owner or by a Reviewer from a source naming the account as the persona's main. The minor rule uses it (1.9).

**API-derived values in public output**: a Riot ID seen through `riot_id_observed` becomes the account's public `riot_id`, and an API-read `lol_region` is served as `accounts[].lol_region`. Each keeps its `source` on the claim, so consumers can filter them. `subject_ref` is never served (1.8). These two fields are the only platform-API-derived values in public output apart from X numeric IDs (9.2).

**Account claim with evidence.** This is the stored form for the seed persona used in every example. The example timeline is: seed load 2027-01-05, launch 2027-01-25, E3 owner 2027-02-04, E4 owner 2027-03-10. These dates are examples only; the real plan is in 13.3. The reference cites the contributing Steward's seed publisher record (3.8, 1.13). Public outputs drop the private fields listed in 1.8.

```json
{
  "claim_id": "clm_01MBJH7BJGRHEM3E2MSH4DMDZS",
  "v": 1,
  "subject": "per_01MBJH79M071HFE865V215DE1C",
  "predicate": "account_link",
  "object": {
    "riot_id": { "game_name": "Example Mid", "tag_line": "EUW" },
    "lol_region": { "value": "euw1", "source": "evidence_page", "observed_at": "2027-01-04T14:03:00Z" }
  },
  "asserts": true,
  "references": [
    {
      "ref_id": "ref_01MBJH7BJGX9G8FGVSQE8Z5379",
      "publisher": "pub_01MAX9T4Q0V7N2C8RJ5WZB3KHE",
      "batch_id": "bat_01MBJGY4N0AV4RRXG07HN7JVAX",
      "origin": { "kind": "seed", "derived_from": [] },
      "evidence": { "tier": "E1", "method": "persona_page", "url": "https://www.twitch.tv/examplemid/about", "captured_at": "2027-01-04T14:00:00Z" },
      "disclosure": { "basis": "public_profile", "assessed_by": "reviewer" },
      "observed_at": "2027-01-04T14:03:00Z",
      "source_updated_at": null,
      "supersedes": null,
      "expected_prior": null
    }
  ],
  "registry": {
    "received_at": "2027-01-05T09:00:02Z",
    "account_id": "acc_01MBJH7AK8E6RHXQAQPDH4AFAX",
    "name_key": "examplemid#euw",
    "bind_status": "name_only",
    "primary": true,
    "evidence_tier": "E1",
    "effective_basis": "public_profile",
    "review_status": "approved",
    "visibility": "public",
    "notice_delivered_at": "2027-01-07T09:40:00Z",
    "audit": { "seq": 1204, "hash": "sha256:6f1d0c9a2b" }
  }
}
```

The validator re-derives the `evidence.tier` a payload carries from `method` and its own checks (3.6). The chainer fills `audit.hash` after the write (2.2); receipts carry `seq` at once. The registry never creates third-party archive copies of evidence pages, because an archive snapshot outlives a hide.

### 1.6 Evidence tiers (truth axis)

An **evidence tier** says how strong the proof behind a claim is, from E0 (someone said so) to E5 (Riot sign-in).

| Tier | Meaning |
|---|---|
| E0 | Bare assertion ("this is their smurf") |
| E1 | Public association on a persona-controlled page or a team or league page |
| E2 | Reviewer-confirmed on the persona's own recorded broadcast (VOD timestamp, Streamer Mode off) |
| E3 | Persona proved control of a Reviewer-approved social channel, completed the first-binding hold (6.2), and asserts the account |
| E4 | E3 plus a profile-icon challenge in the same session, run by a verifier-publisher |
| E5 | Riot Sign On under the project's own client |
| OR | Official roster, affiliations only (Global Contract Database (GCD), ERL (European Regional League) Player Database) |

The registry computes every tier from the method and its own checks. E3 and above come only from an owner session or a verifier attestation, never from a submitted or published payload. The registry itself never discovers accounts: it runs no OCR, no live correlation and no automated alt discovery. A claim a submitter bases on such an observation is stored with its method recorded and shown like any other (3.5).

### 1.7 Disclosure basis (publication axis)

The **disclosure basis** records what public tie the person made between themselves and the item.

| `disclosure_basis` | Set by | Attaches to |
|---|---|---|
| `owner_verified` | E4/E5 owner choosing "show"; public at once | Account; survives renames that `subject_ref` or the project key tracks |
| `owner_declared` | E3 owner choosing "show"; public at once, with notice to every verified channel (6.1) | Account |
| `league_published` | Reviewer, from a team or league publication; the `pub_gcd` adapter once cleared (7.2); a platform publisher's line provisionally, checked by the daily sample (4.4) | Observed string or affiliation |
| `public_profile` | Reviewer, from the persona's own page; the exact-string auto-check in 4.2, only on a persona with an E3+ owner; a platform publisher's line provisionally, checked by the daily sample (4.4) | Observed string |
| `broadcast_visible` | Reviewer only, from the persona's own VOD | Observed string |
| `none` | Default for any other observation, including every community submitter's `disclosure_claimed` until a Reviewer or the 4.2 check confirms it; an ordinary servable value | |

Each reference carries a basis. A claim's effective basis is the strongest among its active references (`owner_verified` > `owner_declared` > `league_published` > `public_profile` > `broadcast_visible`), so owners upgrade claims without changing them.

The basis does not gate publication. It is metadata on every claim that consumers can filter on (`basis=`, 12.1), and `none` is an ordinary servable value. Publication follows `is_public()` alone (1.9). The public persona schema, `schema/persona.schema.json`, lists `none` among the servable `disclosure_basis` values.

Validation (3.6) rejects any submission or publish line that supplies an `owner_*` basis, an E3+ tier, or any basis on a `facet_attestation`, and gate G1 tests this. Owner bases come only from an owner session, and a facet's basis is derived from the claims it rests on (1.11). A platform line's `public_profile` or `league_published` is recorded with `assessed_by: publisher`, and counts as a sample error if the Reviewer disagrees. So platform claims land at once, but a publisher cannot set any other basis. Reputation, platform volume and reviewer approval raise `evidence_tier`, never an `owner_*` basis. A third-party basis attaches to the observed string, so a rename needs a fresh basis. An owner basis follows the account, because the owner chose the account.

### 1.8 Review status, visibility and persona status

Review and visibility are separate fields, so an unreviewed claim can still be public.

| Field | Values |
|---|---|
| `review_status` | `unreviewed` (live platform line or auto-approved, not yet sampled), `approved`, `contested`, `rejected` (kept with a reason code to block re-adds), `reverted` |
| Claim `visibility` | `public` (the default once stored), `held` (persona in consent mode, until the owner approves the claim), `withheld` (minor rule, a minor report under review, or an E4 owner's contest pending; staleness is not a visibility state, `stale_hard()` in 1.9 computes it), `hidden` (owner or panel, plus suppression), `erased` |
| Batch hold (not a claim visibility) | A batch in a trial, on probation or caught by the backlog rule (4.4) is held until its day is sampled or the backlog clears. Meanwhile `batch_held()` in 1.9 keeps a claim that rests only on such batches out of output |
| Persona `status` | `provisional`, `listed`, `historical`, `delisted`, `merged`, `erased`; flags `minor_protected`, `consent_mode`, `review_owner_claims`, `first_binding_hold` |
| Persona `age_status` (private) | `unknown` (default), `adult_attested` (E3+ owner attested 18+), `minor` (confirmed under 18, sets `minor_protected`) |

Private fields never leave D1: `subject_ref`, `publisher_ref`, `visibility`, notice delivery state, `age_status`, `guardian_attested`, reviewer notes, contacts, submitter identities, suppression state and privacy-ticket counts.

### 1.9 The single publication predicate

```
is_public(c) =
     c.asserts
  && c.review_status in {approved, unreviewed}
  && not batch_reverted(c)
  && not batch_held(c)                -- every active reference comes from a trial, probation or backlog hold (4.4)
  && c.visibility == public          -- the default once stored; changed only by a hide, consent mode, the minor rule or a minor report, or an E4 contest (8.3)
  && p.status in {listed, historical}  -- no basis condition on either status (1.7)
  && not suppressed(c.name_key, p.persona_id)
  && account_rule(c, p)
where p = persona(c)

account_rule(c, p) =
     c.predicate != account_link
  || (   not stale_hard(c)
      && (not p.minor_protected
            or (c.primary and effective_basis(c) == owner_verified and p.guardian_attested)))

stale_hard(c) =
  if effective_basis(c) in {owner_declared, owner_verified}
  then now > owner(p).reverify_due              -- 12 months (6.4)
  else now > c.last_confirmed + 90 days
```

A `minor_protected` persona is one confirmed as a minor (8.8). A report also withholds its accounts that have no owner basis while the report is reviewed (8.4). Such a persona publishes at most its one primary account, owner-verified with guardian attestation, and never any other account, because the registry never publishes minors' alts. Unknown age publishes: the minor rule applies only to confirmed or reported minors.

A facet publishes only if every claim it rests on is public, so no facet can reveal a hidden account. Every output path calls this one function: API, static JSON, search shards, dumps, feeds, webhooks and the public notice page. A path that skipped the check would leak opt-outs. Launch gate G1 is a CI suite that asserts byte-identical 404s for every non-public state on every path.

### 1.10 Persona file

`https://cdn.summonerdex.gg/p/{slug}.json`, shown on 2027-03-15, after the owner verified at E4 in phase 3 and after the owner hid an alt on 2027-03-01 (1.12):

```json
{
  "persona_id": "per_01MBJH79M071HFE865V215DE1C",
  "slug": "example-mid-7q2k",
  "display_name": "ExampleMid",
  "status": "listed",
  "games": ["lol"],
  "owner_verified": { "tier": "E4", "since": "2027-03-10T18:20:00Z", "reverify_due": "2028-03-10" },
  "facets": [
    { "facet": "pro", "rubric": "pro@1", "value": true, "league": "LEC", "team": "Example Esports", "role_class": "player",
      "valid_until": "2027-11-15", "basis_claims": ["clm_01MBJH7CHR0GB6KC9QA0MZ17QK"], "computed_at": "2027-03-15T06:00:00Z" },
    { "facet": "streamer_declared", "rubric": "streamer_declared@1", "value": true,
      "valid_until": "2028-02-04", "basis_claims": ["clm_01ME0MWXK0RBKS0KJDVMXWWDYA", "clm_01MBJH7DH0F3F55Z488YAGVDCK"], "computed_at": "2027-03-15T06:00:00Z" }
  ],
  "accounts": [
    {
      "account_id": "acc_01MBJH7AK8E6RHXQAQPDH4AFAX",
      "primary": true,
      "current": { "game_name": "Example Mid", "tag_line": "EUW", "name_key": "examplemid#euw", "region": "euw1" },
      "names": [
        {
          "game_name": "Example Mid", "tag_line": "EUW", "name_key": "examplemid#euw",
          "from": "2027-01-04T14:03:00Z", "to": null,
          "evidence_tier": "E4", "disclosure_basis": "owner_verified", "review_status": "approved",
          "publishers": ["pub_01MAX9T4Q0V7N2C8RJ5WZB3KHE"], "claim_ids": ["clm_01MBJH7BJGRHEM3E2MSH4DMDZS"]
        }
      ],
      "regions": [
        { "region": "euw1", "from": "2027-01-04T14:03:00Z", "to": null, "publishers": ["pub_01MAX9T4Q0V7N2C8RJ5WZB3KHE"] }
      ],
      "last_confirmed": "2027-03-10T18:20:00Z",
      "freshness": "fresh",
      "corroboration": 1
    }
  ],
  "socials": [
    { "platform": "twitch", "url": "https://www.twitch.tv/examplemid", "evidence_tier": "E3", "disclosure_basis": "league_published",
      "review_status": "approved", "publishers": ["pub_01MAX9T4Q0V7N2C8RJ5WZB3KHE"], "last_confirmed": "2027-02-04T17:00:00Z",
      "claim_id": "clm_01MBJH7DH0F3F55Z488YAGVDCK" }
  ],
  "affiliations": [
    { "team": "Example Esports", "league": "LEC", "role_class": "player", "valid_from": "2027-01-01", "valid_to": null,
      "evidence_tier": "E1", "disclosure_basis": "league_published", "review_status": "approved",
      "publishers": ["pub_01MAX9T4Q0V7N2C8RJ5WZB3KHE"], "last_confirmed": "2027-01-05T09:00:03Z",
      "claim_id": "clm_01MBJH7CHR0GB6KC9QA0MZ17QK" }
  ],
  "external_ids": { "leaguepedia": "ExampleMid", "liquipedia": "ExampleMid" },
  "redirected_from": [],
  "policy": "registry-default@1",
  "seq": 6102,
  "attribution": {
    "text": "SummonerDex contributors, CC BY 4.0, https://summonerdex.gg/attribution",
    "license": "CC-BY-4.0",
    "terms": "https://summonerdex.gg/terms/access-1.0"
  }
}
```

Every claim-bearing object (a name entry on an account, a social, an affiliation) carries `evidence_tier`, `disclosure_basis`, `review_status` and `publishers`; every account carries `last_confirmed` (12.2). The Twitch social is E3 because the owner proved control of it on 2027-02-04. Its basis is `league_published` because the team page links it. Socials show the tier, not the method, until Twitch says whether a public "verified via Twitch" label counts as derived Twitch Data. Age status is never published.

### 1.11 Metrics facet claim

The publisher attests the rubric outcome and keeps the hours to itself, because Twitch bars sharing API data and insights derived from it, and YouTube bars derived metrics. `channels` names the owner-verified social claims the outcome was computed over. The endpoint accepts `streamer` only after Twitch answers in writing (7.3). Until then it returns `rejected:facet_not_enabled` and stores nothing. The example below assumes a written yes before mid-2027.

The publisher's reference carries no `disclosure` and no `evidence.tier`. Validation rejects a facet line that supplies either. The registry derives both from the claims in `channels`, and the weakest wins, so a metrics publisher can never mark a facet as owner-disclosed.

```json
{
  "claim_id": "clm_01MSPRNV509V82KQ2PDF6PT9FA",
  "v": 1,
  "subject": "per_01MBJH79M071HFE865V215DE1C",
  "predicate": "facet_attestation",
  "object": {
    "facet": "streamer", "rubric": "streamer@1", "value": true,
    "window_start": "2027-01-01T00:00:00Z", "window_end": "2027-06-30T00:00:00Z",
    "channels": ["clm_01MBJH7DH0F3F55Z488YAGVDCK"]
  },
  "asserts": true,
  "references": [
    {
      "ref_id": "ref_01MSPRNV50BT1BNTFHA6R2YECX",
      "publisher": "pub_01M72ECX80TEWH0AVNH9DN65R6",
      "batch_id": "bat_01MSPRM0J0PKEE1CSNH4HPSN17",
      "origin": { "kind": "primary", "derived_from": [] },
      "evidence": { "method": "metrics_attestation", "url": null, "captured_at": "2027-06-30T00:05:00Z" },
      "observed_at": "2027-06-30T00:05:00Z",
      "source_updated_at": null,
      "supersedes": "ref_01MQ9GR950GBG26VPMV7RCG2V0",
      "expected_prior": "ref_01MQ9GR950GBG26VPMV7RCG2V0"
    }
  ],
  "registry": {
    "received_at": "2027-06-30T00:07:00Z",
    "evidence_tier": "E3",
    "effective_basis": "league_published",
    "derived_from_claims": ["clm_01MBJH7DH0F3F55Z488YAGVDCK"],
    "valid_until": "2027-08-04T00:00:00Z",
    "review_status": "unreviewed",
    "audit": { "seq": 15880, "hash": "sha256:0b7e41c2d9" }
  }
}
```

### 1.12 Tombstone and suppression record

A tombstone is a signed compliance event with opaque IDs, one action and one deadline (8.6). It never names the Riot ID, because that would leak what was hidden. This one records the owner hiding the alt account claim submitted in 3.1, which had been public since its approval on 2027-02-08.

```json
{
  "seq": 5390,
  "tombstone_id": "tmb_01MG08GV4868G769HJDNDGZG35",
  "action": "withdraw",
  "scope": "claim",
  "target_id": "clm_01ME7Q6QW025TMGDMFGS8KXM40",
  "issued_at": "2027-03-01T10:00:05Z",
  "apply_by": "2027-03-02T10:00:05Z",
  "kid": "reg-2027",
  "sig": "ed25519:Vh3QmS1o2kq8b0PzN7cX4yT6fRw9eLjA"
}
```

`seq` is the one global change sequence shared with `/v1/changes` (2.2), so a single cursor orders both feeds. The registry signing key's `kid` changes yearly (5.3). Persona scope uses `"scope": "persona"` with the `persona_id`.

Suppression records stay in `private`: `{sup_id, scope, hmac, key_version, created_at, request_ref}`, where `hmac = HMAC-SHA256(K_supp_v, "riot|" + name_key)`, `"slug|" + slug` or `"social|" + platform + "|" + fold(handle)`. The hash is keyed because Riot IDs are easy to guess. The secret is versioned and rotates only if it is compromised, because erased Riot IDs cannot be hashed again.

### 1.13 Publisher registry entry

`publishers/pub_01M72ECX80TEWH0AVNH9DN65R6.json`, reviewed by PR. `state` and `caps` mirror D1 nightly. Shown during an unaffiliated platform publisher's trial week in phase 2.

```json
{
  "id": "pub_01M72ECX80TEWH0AVNH9DN65R6",
  "kind": "platform",
  "name": "Example Stats",
  "operator_contact": "https://stats.example/contact",
  "method_url": "https://github.com/summonerdex/registry/blob/main/publishers/pub_01M72ECX80TEWH0AVNH9DN65R6/METHOD.md",
  "ingest_code_url": "https://github.com/example-stats/registry-publisher",
  "predicates": ["account_link", "riot_id_observed", "social_link", "affiliation"],
  "riot_derived_fields": { "enabled": true, "fields": ["subject_ref", "lol_region"] },
  "verifier": { "enabled": false, "riot_production_key": "unconfirmed", "product_change_audit": "none" },
  "independence": {
    "root": "pub_01M72ECX80TEWH0AVNH9DN65R6",
    "set_by": "steward",
    "decision": "log/0007-example-stats-onboarding.md",
    "declared_derived_from": [],
    "note": "Own root: no declared derivation and no shared control with any other active publisher"
  },
  "declared_conflicts": [],
  "signing_keys": [
    { "kid": "pub_01M72ECX80TEWH0AVNH9DN65R6-2027-02", "alg": "Ed25519", "public": "MCowBQYDK2VwAyEA3m1n0QxR8vT2kLp9sW4yZc7eHb5uJf1gNd6aXo0iPq8=", "not_after": "2028-02-15" }
  ],
  "caps": { "lines_total": 250, "batches_per_hour": 1 },
  "agreement": { "version": "publisher-1.0", "accepted_at": "2027-02-15T12:00:00Z", "countersigned_by": "steward" },
  "rollback_plan": "Revert by batch_id, then re-send a corrected full_sync.",
  "last_active_at": "2027-02-16T09:00:00Z",
  "state": "trial"
}
```

Stewards set `independence` at onboarding from the declared `derived_from` and from who controls the publisher. The publisher never sets it, because self-declared independence would let two related publishers corroborate each other (4.3). `facet_attestation` is missing from `predicates` because no metrics facet is enabled yet (7.3).

Seed references (3.8) cite a separate record for each Steward, not this one; `pub_01MAX9T4Q0V7N2C8RJ5WZB3KHE` in 1.5, 1.10 and 12.2 is such a record. A Steward record has no `signing_keys`, `caps` or `agreement`, because no key exists before phase 2 (5.1). Its `declared_conflicts` list the products that Steward runs or works for (11.1, 11.3), and its `independence.root` is the one shared Steward root (4.3). It is added by PR before that Steward's first seed load, and the other Steward approves it.

### 1.14 Read-time resolution, `registry-default@1`

Claims are stored raw and resolved when read, so a policy change needs no migration. The resolver works in five steps:

1. For each `(publisher, subject, predicate, object)`, the latest `observed_at` or an explicit `supersedes` wins. This is the only place where newest-wins applies.
2. Set-valued fields are unions of public claims, each with its own interval. One account on two personas is `contested` and is never resolved by recency.
3. Scalars sort by `(fresh, tier, as_of, publisher_rank, claim_id)` under a field-authority table: the official roster or a reviewer-confirmed team page for affiliations, the owner for display name.
4. Distinct independence roots, capped at 3, break ties within a tier.
5. Outputs carry `policy` and `seq`.

The resolver ships as a TypeScript library over `claims.jsonl` at launch and as a Python port in phase 2. At launch a consumer needs only `idfold` and the API.

## 2. Canonical store and distribution

### 2.1 What is canonical where

| Store | Canonical for | Why |
|---|---|---|
| D1 `core` | Personas, accounts, claims, references, submissions, batches, facets, publisher state, ledger, `changes`, `compliance`, notice and delivery log, audit rows, outbox | Facts change faster than GitHub allows: about 80 content requests a minute and one recommended merge a minute |
| D1 `private` | Suppression HMACs, slug reservations, submitter identities, verification sessions, confirmed owner emails, `publisher_ref` and `subject_ref`, key hashes, contacts (90-day retention), platform-API reads (24h TTL), privacy-ticket counters, age status, later registry PUUIDs | Limits the damage of a breach and gives platform data a hard time-to-live boundary |
| R2 | Persona JSON, search shards, dumps, ingest uploads, encrypted backups | Free egress; static reads skip billable Worker calls |
| Git `summonerdex/registry` | Schema, rubrics, reason codes, league allow-list, publisher pages, terms, GOVERNANCE.md, public decision log, signed audit heads (phase 2), stats | Rules need review and public history; none of it is subject data |

No snapshot of persona shells goes into Git. A delisted shell in any public commit survives in forks, and GitHub Support may decline to purge data that is not sensitive. Dumps cover browsing and continuity.

D1 has no transactions across databases. So every flow writes `private` first and makes that write fail-safe: a suppression without its `core` change blocks too much ingest but publishes nothing. A reconciler cron retries any suppression row older than 60 seconds that lacks a compliance event, and alerts `#stewards` after three failures.

### 2.2 Write path, audit and feeds

1. Each channel writes a `submission` and its drafts in one `core` batch.
2. A decision writes, in one `core` batch: the claim or reference, derived rows, a `changes` row with the next global `seq`, an `audit_log` row, and for privacy actions a `compliance` row (same `seq`) and `outbox` rows.
3. Privacy actions rewrite R2 objects inline and purge the CDN. A per-minute projector handles all other `changes`.
4. From phase 2, an outbox cron sends signed webhooks (8.6).

**Audit without exposure.** The table is `audit_log(seq INTEGER PRIMARY KEY AUTOINCREMENT, at, actor_role, actor_pseudonym, action_class, object_id, row_json, prev_hash, hash)`. Writers insert only `seq` (assigned by SQLite) and the canonical `row_json`, inside the decision's batch.

Hashing at write time would fork the chain. There are three reasons:

- D1's SQLite has no SHA-256 function.
- A D1 batch cannot pass a value computed in the Worker from one statement into the next.
- The ingest cron, bot, web form and privacy actions run in parallel isolates (separate running copies of the Worker), so they would read the same `prev_hash`.

Instead, one chainer runs each minute under a lease (`UPDATE chain_lease SET holder = ?, until = ? WHERE until < ?`, 50-second lease). It reads unhashed rows in `seq` order, computes `hash = SHA-256(prev_hash || row_json)` in the Worker, and writes `prev_hash` and `hash` back. A second run that fails to take the lease exits.

Receipts and decision notices return `audit_seq` at once. `GET /v1/publish/batches/{bat_}` shows each row's `hash` once it is chained, normally within two minutes. Rows are written from day one. The chainer, the signed daily head in Git and the head in every dump manifest ship in phase 2 with publisher receipts (13.4), and they hash the backlog from `seq` 1. Erasure replaces `row_json` with `{"erased": true}` and keeps the hash. Per-row audit is not published, because "hide clm_X" in public history would undo the single withdraw action.

**Feeds**: `GET /v1/changes?since=&limit=1000` returns `{seq, persona_id, slug, etag}`. `etag: null` means "no longer public", for every reason alike, and `include=projection` inlines the new projection. Change and compliance events share one rising `seq`, so a consumer orders both feeds with one cursor. Cursors live 90 days. After that they get `410 cursor_expired` with the latest dump's `cut_seq`. The compliance feed is in 8.6.

### 2.3 Dumps

A cron at 03:00 UTC pages through D1 by key into an R2 multipart upload. It does not use `wrangler d1 export`, because that blocks other requests.

```
dumps/2027-02-20/
  manifest.json        license, attribution, cut_seq, compliance_since (= cut_seq), audit_head (phase 2),
                       valid_until (+8d), sha256 per file, Ed25519 sig
  personas.jsonl.gz    public projections
  claims.jsonl.gz      public claims with references
  facets.jsonl.gz  redirects.jsonl.gz  publishers.jsonl.gz
  withdrawn.jsonl.gz   cumulative compliance events, opaque IDs
  LICENSE.txt  ATTRIBUTION.txt
```

`GET /v1/dumps/latest` serves only the newest dump. Seven daily dumps are kept privately for rollback and are never downloadable. Serving older dumps would hand a new key holder a copy from before a hide or delist. Instead, the Access Terms require every downloader to apply `/v1/compliance?since={cut_seq}` right after loading, and then at least hourly. This brings every hide, delist, scrub and erase to every key holder within the same 24-hour deadline, whether or not they download again (8.6).

Every withdrawn item is missing from the next nightly dump. An erase or scrub also deletes every retained daily dump that contains the target, and rebuilds the latest within 24 hours. Download needs a free read key with clickwrap (terms accepted with a click), because an anonymous download binds only through the licence.

### 2.4 Merges, splits and redirects

A `persona_merge` needs a Steward and waits 48 hours, or 72 after an objection, with both owners notified. A merge involving any persona that is not listed is refused, because the redirect would reveal it.

When a merge is applied:

- the losing persona becomes `merged`, and its claims are re-pointed with their original IDs kept;
- a `redirects` row and a `changes` event are written;
- the API answers `301` with `canonical_slug`;
- the loser's static file becomes a `{"status":"merged","canonical_slug":...}` stub;
- the surviving persona lists `redirected_from`.

A split within 90 days restores the loser from the merge record. The one Steward-approved slug change for an impersonating slug (1.2) uses the same redirect row, stub and `301`.

### 2.5 Git sync and consistency

Inbound: a push to `main` posts changed rules to `POST /admin/config/sync` with an HMAC, and rubric changes trigger a recompute. PR validation runs under `pull_request` with read-only rights and never executes PR content. Outbound: from phase 2, the signed audit head and manifest hashes daily; a bot PR for publisher `state` daily; and `log/stats/YYYY-MM.json` monthly (11.4).

Targets:

- Keyed API: up to date immediately after any write.
- R2 JSON: within 60 seconds of privacy actions and 2 minutes otherwise (`max-age=600`, purge on change).
- Feeds: immediate.
- Webhooks: 5 minutes, with retries up to 24 hours.
- Dumps: nightly; erase and scrub purged from retained dumps within 24 hours.

## 3. Submission flows and approval queue

### 3.1 One queue object

All four channels produce this shape, so review, reputation and audit share one code path. This is the stored record that Reviewers see in the admin view. The example is the alt account a community member submitted for the example persona on 2027-02-06.

```json
{
  "submission_id": "sub_01ME51C76G33M33YA35B44FS7F",
  "channel": "discord",
  "action": "add_claim",
  "submitter": { "usr_id": "usr_01MD8XVZG0VBHXF4W6305FATZY", "contributor_id": "ctr_01MD8XVZG0DS3AR0RGXJVZ9CJD", "trust_state": "queued", "terms_version": "contributor-1.0" },
  "idempotency_key": "discord:interaction:1597734400000000000",
  "claims": [
    {
      "subject": "per_01MBJH79M071HFE865V215DE1C",
      "predicate": "account_link",
      "object": { "riot_id": { "game_name": "Example Alt", "tag_line": "0001" } },
      "evidence": { "method": "persona_page", "url": "https://x.com/examplemid", "captured_at": "2027-02-06T09:50:00Z" },
      "disclosure_claimed": "public_profile",
      "observed_at": "2027-02-06T09:50:00Z"
    }
  ],
  "checks": { "schema": "pass", "pii_lint": "pass", "suppression": "clear", "exact_string": "not_run", "conflicts": [] },
  "state": "pending_review",
  "required_approver": "reviewer_unaffiliated",
  "hold_until": null,
  "decision": { "by_role": null, "by_contributor": null, "at": null, "reason_code": null },
  "created_at": "2027-02-06T10:00:02Z"
}
```

`GET /v1/submissions/{sub_}` returns only the submitter view: `submission_id`, `public_state` (`received`, `in_review`, `needs_info`, `approved`, `rejected`, `withdrawn`, `expired`), `reason_code` and `created_at`. `checks`, `required_approver`, `trust_state` and every suppression result stay in the admin view. A visible suppression result would tell a submitter which accounts were hidden, which is exactly what the platform receipt is designed not to reveal (3.7).

The Discord ID lives only in `private.submitter_identities`. `ctr_` is the public pseudonym, and `usr_` never leaves D1, because Discord API data may not be shared. A submitter's `disclosure_claimed` is only advice: the reference records basis `none` with `assessed_by: submitter` until a Reviewer, or the 4.2 auto-check, sets a basis (1.7).

### 3.2 Actions and approvers

| Action | Approver | Hold |
|---|---|---|
| `add_claim` of `account_link`, additive | A Reviewer unaffiliated with the submitter; automatic only for `trusted` submitters under 4.2 | None |
| `add_claim` of `social_link` | A Reviewer, always, because an approved social channel can become E3 authority (6.2) | None |
| `create_persona` | A Reviewer confirming a public role, or confirming the persona's channel so its E3 owner can declare one (7.1); a Steward until three Reviewers exist | None; best-effort shell notice (8.3) |
| First owner binding (E3) | Reviewer sign-off (6.2) | 72h, with notice to every approved channel and contact |
| Delegate for an E3 owner (until E4 exists) | Privacy panel | 72h |
| `retract_claim` (own) | Automatic | None |
| `contest` | A Reviewer decides. An E3 owner's contest marks the claim `contested`; an E4 owner's contest withholds it until the Reviewer decides; only an E5 owner decides alone (6.1) | None |
| Merge, split, relink, slug change | Steward | 48h, 72h after objection |
| Editorial facet | Steward, with a written reason | 7-day comment, 12-month expiry |

Hide, delist, scrub and erase are not submissions. They use the private privacy intake (8.4), so they never sit in a queue that volunteers browse. Nobody approves their own submission, their employer's or their publisher's.

### 3.3 Submission state machine

Approval decides truth only; publication follows the visibility machine in 8.3. The `Submitter sees` column is the `public_state` mapping.

| From | Event | To | Submitter sees |
|---|---|---|---|
| `received` | Invalid schema or evidence | `rejected_invalid`, content discarded | `rejected` with the validation reason |
| `received` | Suppression hit | `blocked` | `in_review`, then `rejected: insufficient_evidence` (below) |
| `received` | Valid platform line / trusted and auto-eligible / otherwise | `live_unreviewed` / `auto_approved` / `pending_review` | n/a / `approved` / `in_review` |
| `pending_review` | Approve / reject with reason code / ask for evidence | `approved` / `rejected` / `needs_info` (14 days) | Same |
| `pending_review` | Destructive approve, then objection | `scheduled`, then back to `pending_review` with 72h hold | `in_review` |
| `live_unreviewed` | Daily sample passes / day over 5% bad | `approved` / `reverted` | n/a |
| `auto_approved` | Overturned | `reverted`; submitter demoted (4.3) | `rejected` with reason |
| Any open | Cancelled / 30 days stale | `withdrawn` / `expired` | Same |

A `blocked` submission goes through the normal queue lifecycle. It shows `in_review` and closes as `rejected` with `insufficient_evidence`, at a time drawn from the past 30 days' decision times for the same action. So neither its timing nor its outcome differs from an ordinary rejection with the same code.

An approved submission always shows `approved` with the same text, "publication depends on visibility rules", whether its claim publishes, is held (consent mode), is withheld (minor rule) or is suppressed. The response says nothing about the persona's state. Identity claims never auto-accept on timeout.

Rejections carry a code from `reason_codes.json`, a fixed list of deprecation reasons plus `minor` and `insufficient_evidence`. `minor` appears only in the admin view; the submitter sees `insufficient_evidence`.

### 3.4 Channels

- **Web form**: Discord, GitHub or Twitch OAuth, used only to identify the submitter; Turnstile (Cloudflare's bot check); versioned Contributor Terms clickwrap; raw Riot ID fields; and a required evidence URL, because free-text proof does not work. Persona proposals ask for a public business contact, stored privately for notice.
- **Discord bot**, on the same Worker over HTTP interactions: an Ed25519 check on each request, defer within 3 seconds, finish within the 15-minute token, no Gateway connection. Commands: `/accept-terms`, `/submit` (modal), `/status`, and `/claim-me`, `/hide`, `/privacy`, which return one-time links so nothing sensitive is typed in Discord. Server verification level HIGH. Accounts must be at least 30 days old, checked from the snowflake ID: `(id >> 22) + 1420070400000`. The bot answers only the person who invoked it, and never DMs or tags anyone found through data.
- **Platform bulk API** (3.7, phase 2) and **Steward CLI seed** (3.8).

### 3.5 Evidence methods and what is not built

- The `evidence.method` enum includes `live_game`, `spectator`, `streamer_mode_state`, `match_cooccurrence`, `ocr` and `behavioral_inference`. Claims from any method, including observation of hidden-player state, are stored and shown like any other. They carry their method so consumers can filter, and they are hidden on request.
- There is no "identify this account" queue, no `/identify` command and no `#identify` channel. Asking the crowd who owns unmatched high-elo accounts de-anonymizes players who cannot reasonably be identified from visible information.
- The registry itself runs no OCR, live-game correlation or automated alt discovery, and contacts nobody found through API data.

### 3.6 Validation pipeline

One TypeScript module runs in the Worker and in rules CI, so rubric PRs are tested with the same code that applies them. It runs these steps:

1. JSON Schema.
2. Tier re-derivation from `method` (E3+ is never accepted from a payload).
3. Rejection of any supplied `owner_*` basis, or any basis on a `facet_attestation` (`rejected:basis_not_assertable`).
4. Lint: fold collisions, implausible `observed_at`, 78-character PUUID-shaped strings, and legal-name, birth-date, address and email patterns.
5. Suppression.
6. Binding (1.4).
7. Conflicts.
8. The exact-string check, where 4.2 applies.
9. Routing.

G1 includes a test for each rejection rule.

### 3.7 Platform bulk API (phase 2)

1. `POST /v1/publish/batches`: NDJSON, up to the key state's per-batch cap (5.3) or 5 MB. Headers: `X-Batch-Id: bat_{ULID}` minted by the publisher, `X-Sent-At`, and `X-Signature: kid=...; alg=Ed25519; sig=base64(sign(sha256(body) || batch_id || sent_at))`. The publisher mints `batch_id` so it can sign it before sending. The server rejects a `batch_id` it has already seen with `409 duplicate_batch`, and a `sent_at` more than 5 minutes from server time with `400 clock_skew`, so `batch_id` also serves as the idempotency key. The body is stored in R2, and the response is `202 {batch_id}`.
2. Mode is `delta` (adds, supersedes, retractions) or `full_sync` (the complete current set for the declared predicates).
3. **Ingest budget.** A per-minute cron takes one chunk of 150 lines per run. For each chunk it pre-fetches suppression HMACs with one `IN` query on `private`, and bindings and `content_hash` values with two `IN` queries on `core`. It then writes about four statements per changed line (claim or reference, `changes` row, `audit_log` row, batch or ledger row). A worst-case chunk with every line changed is about 600 statements, inside D1's limit of 1,000 queries per invocation on the Paid plan. Whether a D1 `batch()` counts as one query or one per statement is to be confirmed, so the budget counts one per statement. Capacity is 150 lines a minute, about 216,000 a day, far above the default daily cap of 5,000 lines (5.3). The week-7 load test (13.3) measures statements per line and resizes the chunk.
4. Per line: validate, fold, check suppression, bind, and compare by `content_hash`. Unchanged lines write nothing. In `full_sync`, the batch row records `observed_at` once and missing lines become `lapsed` (not negated). So the heartbeat costs one row per batch, which avoids the roughly $300 a month that an hourly refresh of every row would cost.
5. A line that only corroborates a public claim adds an `unreviewed` reference at once. A new claim is public at once as `unreviewed`, basis `none` included, and triggers a best-effort notice (8.3). The exception is trial, probation and backlog holds (4.4): those lines publish once sampled (`batch_held()`, 1.9). The batch enters the publisher's daily sample (4.4).
6. The receipt gives, per line, `{line, status, claim_id, ref_id, publisher_ref, audit_seq}`, with status `accepted | unchanged | lapsed | rejected:{code} | suppressed | contested:{clm_}`. Row hashes follow through `GET /v1/publish/batches/{bat_}` once chained (2.2).

Publisher lines may carry `subject_ref`, `riot_id_observed` and an API-read `lol_region` from the first batch (1.4).

`suppressed` appears only where this publisher's own earlier reference on that account was withdrawn. The publisher already received that event, so it learns nothing new and can stop resending. Every other suppression, delisted, minor or consent case returns `accepted` with a minted `claim_id` that never becomes public, so the receipt reveals nothing. `contested` is reported only against public claims.

### 3.8 Seed path

No publisher key exists while there are only two Stewards (5.1). So the Stewards' data enters as one-off `origin.kind: seed` batches, loaded by a Steward CLI and attributed to the contributing Steward's publisher record. These batches land as `pending_review`. Seed loads need no legal entity: until one exists, the two charter Stewards jointly are the controllers named in notices and the counterparty of the Contributor Terms (9.3).

The launch seed is capped so two part-time people can review it: 300 to 500 personas, all rostered pros in allow-listed leagues. Each has a team or league page or a persona-controlled page as evidence, one primary account, and a notice contact where one is known (the persona's own business contact, else the team's press contact). Seed lines may carry `subject_ref`, API-observed renames and API-read regions like any publisher line (1.4). The loader drops lines that lack evidence, because the seed is reviewed against its evidence pages.

Stewards review each other's seed data, never their own (11.1). Review is time-boxed at 3 minutes a row, about 25 hours for 500 rows. Approved seed claims publish when public reads open, with a best-effort notice where a contact is known (8.3).

## 4. Reputation

### 4.1 Ledger and states

`ledger(usr_id, claim_id, submission_id, persona_id, outcome, outcome_at, decided_by_role, reason_code, auto)`, with outcomes `approved_by_review`, `auto_applied`, `corroborated`, `upgraded_by_owner`, `rejected`, `reverted`, `contradicted_by_owner`.

Only `approved_by_review` and `upgraded_by_owner` count toward promotion, because both are human judgements by someone other than the submitter. `corroborated` is recorded but never counts. Otherwise two sockpuppets corroborating each other would reach `trusted` with no human looking, and 30-day-old Discord accounts are cheap to buy. Platforms have no ledger; sampling governs them (4.4).

The registry uses counted states, not points, because counts explain themselves in a rejection message and need no tuning data:

| State | Entry rule | Per day |
|---|---|---|
| `new` | Terms accepted; Discord or OAuth account 30+ days old | 3 |
| `queued` | After the first submission | 10 |
| `trusted` | In the last 180 days: 10+ counted approvals across 5+ personas on 3+ different days, with the first approval 14+ days old; 0 reverts until the submitter has 100 outcomes, then at most 3% reverted | 30 |
| `suspended` | Demotion | 0 |

Thresholds live in `rubrics/reputation@1.json`, so changes go through review. A weekly cluster check flags pairs of submitters whose submissions touch the same personas within 24 hours of each other on 3 or more occasions. Flagged pairs lose auto-approval until a Steward reviews them.

### 4.2 What auto-approves for `trusted` (phase 2)

All of these must hold:

1. The action is `add_claim` of `account_link`. Never `social_link`, which always needs a Reviewer because an approved channel can become E3 authority (6.2).
2. The persona is `listed` and has an E3+ owner. The owner receives the notice and can hide or contest at once, so a wrong attachment reaches someone able to stop it.
3. The claimed basis is `public_profile` only. `broadcast_visible` and `league_published` on a community submission always go to a Reviewer, because judging a VOD or a team page is Reviewer work (1.7).
4. The evidence is a persona-controlled page, and the **exact-string check** passes. The Worker reads the evidence once. It uses the platform's API where one exists and is permitted, or plain HTTP for a personal or team site. The APIs are Twitch Get Users `description` with an app token, and the YouTube channel description from phase 2. It confirms the exact raw Riot ID appears, then discards the content, as the bio-nonce check does. It never scrapes platform HTML. Whether this read fits Twitch's agreement is to be confirmed, as for the bio nonce. Where no permitted read exists, the item goes to a Reviewer.
5. There is no conflict or suppression. The persona is not in `consent_mode`, not `minor_protected`, and does not have `review_owner_claims` set (if it does, the owner reviews, and a rejection counts as `contradicted_by_owner`).
6. The submitter is not flagged by the cluster check.

A passing check records basis `public_profile` with `assessed_by: auto_check`. This is the only community-path basis not set by a Reviewer, and the weekly 10% sample (4.4) covers these items. Auto-approval decides truth only; publication follows 1.9, and the best-effort notice still goes out (8.3). Persona creation, social links, merges, relinks and contested accounts always need a human.

### 4.3 Demotion and independence

| Trigger | Effect |
|---|---|
| Auto-applied claim reverted, or an owner dispute upheld | `suspended` for 30 days, the last 7 days of auto-applied claims re-queued, then back to `queued` |
| Revert rate over 3% in 180 days (after 100 outcomes), or 12 months idle | Back to `queued` |
| Flagged by the cluster check | Auto-approval paused until a Steward reviews |
| Legal names, birth dates, addresses, minors' accounts | `suspended` for 90 days; the Discord ID may be banned |
| Sockpuppetry or vote farming | Indefinite; appeal to Stewards |

Corroboration counts independence roots. An **independence root** is one independent source; publishers under related control share a root. Stewards set each publisher's root at onboarding (1.13), from its declared `derived_from` and from who controls it. A publisher never declares its own independence.

Publishers run by the Stewards count as one root until a third publisher unaffiliated with any Steward is active, because Steward-run publishers are under related control and platform claims must never corroborate themselves. Declared aggregators count zero, all community submitters together count as one root, and nobody corroborates themselves. Roots are capped at 3. They set `corroborated` and break ties within a tier. They never raise a tier, supply a basis or grant a facet. No reputation state can set an `owner_*` basis, re-add a suppressed account or contest a privacy decision.

### 4.4 Platform batch sampling and revert

Sampling is budgeted per publisher per day, not per batch. With a per-batch sample, reviewer load would grow with batch count, while reviewer hours do not.

| Stage | Rule |
|---|---|
| Trial | At most 250 lines, all held, reviewed in full within 14 days by two Reviewers unaffiliated with the publisher |
| Daily sample | For each UTC day with new or changed lines: the larger of 10 lines or 2% of the day's lines, capped at 50, spread across that day's batches; reviewed within 7 days by an unaffiliated Reviewer |
| Error under 2% | The day's lines become `approved` |
| Error 2-5% | Lines stay `unreviewed`; the next day's sample is doubled, up to 100 lines (the only case above 50) |
| Error over 5% | Every batch of that day is reverted by `batch_id`, and the publisher goes on probation: batches are held until their day is sampled |
| Backlog | More than 7 unsampled publisher-days: new batches are stored but held, not live, until the backlog clears |
| Two reverts in 90 days | Key suspended (5.5) |

Load at the default caps: two active publishers at 5,000 lines a day give 2 x 50 = 100 sample lines a day. That is about 700 a week, roughly 6 reviewer-hours a week at 30 seconds a line. Caps rise only when that capacity exists (5.3). A Reviewer also samples 10% of auto-approved items weekly, because a few Reviewers tend to do most of the reviews and trust must be audited cheaply.

## 5. Publisher key issuance

### 5.1 Gate and kinds

Publisher kinds:

- `platform`: any product holding its own Riot key, such as a stats site or a bot. Governed by daily sampling.
- `system`: the Global Contract Database adapter, called `pub_gcd` here (its real ID is a ULID like every other), gated in 7.2.
- `community`: governed by the ledger.
- `persona`: owner authority over the persona's own data.

A separate `verifier` flag needs approval by two unaffiliated Stewards. **No publisher key, trial or active, is issued while fewer than three Stewards are seated.** Forming the legal entity before third-party publishers join is recommended, not required (9.3). Until then the seed path (3.8) carries the Stewards' data. Every key may send `subject_ref` and the Riot-derived fields in 1.4 from its first batch (3.7).

### 5.2 Issuance flow

1. A PR adds `publishers/{pub_id}.json` (1.13): operator, contact, method document, open-source ingest code, predicates, maximum rate, rollback plan, conflicts and declared `derived_from`.
2. 14 days of public review on the PR and in `#publisher-onboarding`.
3. Approval by two Stewards unaffiliated with the applicant, who also set the `independence` root.
4. Publisher Agreement clickwrap by a named officer, Steward countersignature, version stored.
5. Trial key: 250 lines, all held (4.4). With under 2% errors, a Steward sets `active` by PR, and the publisher page shows the trial pass rate.

### 5.3 Keys and limits

| Key | Format and storage |
|---|---|
| Read | `rk_read_{ULID}.{32-byte base64url}`, stored as SHA-256 |
| Publish | `rk_pub_{ULID}.{secret}`, one publisher, SHA-256 |
| Publisher signing | Ed25519, public half in Git, `kid` = `{pub_id}-{yyyy-mm}` |
| Registry signing | Ed25519 in a Worker secret and the vault, yearly `kid` (`reg-2027`), public half in Git |
| Admin | OAuth session plus role row; no static keys |

Keys are shown once, write `last_used_day` at most daily and record `terms_version`. Keyed calls need a `User-Agent` with app name and contact; default library agents get `403`. Keys expire after 12 months. A publisher silent for 180 days is suspended until renewal, because registries edited by PR collect dead entries. A publisher suspended for silence keeps its references, but they are flagged `publisher_inactive` and stop counting toward corroboration and freshness. Its name-only accounts fall back to the key-free confirmation paths (1.4). Owner E4 status it verified stays valid until `reverify_due` (6.3).

| State | Writes | Reads |
|---|---|---|
| `trial` | 250 lines total, 1 batch an hour | 10 req/s |
| `active` | 5,000 lines a day, 2,500 per batch, 4 batches an hour; 20,000 a day after 8 clean sampled days in a row; more only by Steward decision once reviewer capacity is shown | 50 req/s, 2M/day |
| `probation` | 1,000 lines a day, every batch held until its day is sampled | 50 req/s |
| `inactive` | None until renewal | 10 req/s |

Limits are counted inside the isolate and flushed to D1 each minute, because the price of Cloudflare's Rate Limiting binding is to be confirmed.

### 5.4 PUUID namespace and Riot erasure

The registry never receives PUUIDs, which Riot encrypts per application key; PUUID-shaped strings are rejected. It does receive `subject_ref`, a publisher-keyed HMAC of the PUUID that it cannot reverse or join across publishers (1.4).

Publishers may send `publisher_ref`, their own opaque row key of at most 64 characters. The Publisher Agreement forbids deriving it from a PUUID or Riot ID. It lives in `private`, is compared only within one publisher, and appears only in that publisher's own receipts, so a withdraw event's `claim_id` maps to the publisher's row in one lookup.

Riot sends deletion requests as identifier lists that only the key holder can match. So until the project has its own key, a publisher that receives one matches its own rows and calls `POST /v1/publish/erasures {publisher_ref}`. The registry erases the account claims carrying that publisher's references under that `publisher_ref`, writes suppression and emits withdraw events. The call may name a `subject_ref` instead. Registry erasures reach publishers through the compliance feed.

### 5.5 Revocation

The steps are: a warning in the publisher's private thread, probation, suspension, then revocation with the reason class on the public page. Triggers:

- failed samples;
- PUUIDs or raw Riot data in batches;
- missing attribution after a 30-day cure period (CC BY 4.0 section 6(b));
- ignoring compliance after outreach (8.6).

Key caches last 60 seconds. On revocation, references stay but are flagged `publisher_revoked`, batches may be reverted, and `publisher_ref` and any `subject_ref` rows are deleted, because Riot requires deleting Game Information on termination.

## 6. Identity verification ladder

### 6.1 What each tier permits

| Capability | E0-E2 | E3 owner | E4 owner | E5 owner |
|---|---|---|---|---|
| Become the persona's owner | No | First binding: Reviewer sign-off and a 72h hold with notice to every approved channel and contact (6.2) | Instant once E3-bound | Instant |
| Make own account public | No | `owner_declared`, at once, with notice to every verified channel; an account with no E1+ reference first needs a Reviewer check | `owner_verified`, at once | `owner_verified`, at once |
| Hide or unhide own claims; consent mode; `review_owner_claims` | No | Instant, including during the first-binding hold | Instant | Instant |
| Display name, socials | No | After the first-binding hold; later changes instant | Instant | Instant |
| Delist own persona | No | After 48h unless cancelled from any verified channel; never during the first-binding hold | Instant | Instant |
| Erase own persona | No | Ticket to the panel | Self-service, 72h cancellable | Same |
| Contest third-party claims | No | Marks `contested`; a Reviewer decides | Contested claim withheld until the Reviewer decides | Decides alone |
| Delegates (manager) | No | Panel-granted, 72h hold | Yes | Yes |

Owner authority covers only the persona's own accounts, links, name and visibility. E3 rests on a social account, and social accounts get hijacked. So the holds sit on the high-impact actions, binding and delisting, while a wrongful hide costs little and stays instant. A hijacker who passes E3 meets a 72h first-binding hold, notices to the real channels, and a 48h hold on any delist, and the real person can cancel any of them. Until the phase 3 verifier audit passes (6.3), E3 is the highest tier anyone has, and no page offers E4.

### 6.2 E3: social-channel proof, run by the registry

**Channel eligibility.** E3 proof counts only for a channel that is already an `approved` `social_link` on that persona. A Reviewer must have approved it from evidence older than the E3 attempt: a team or league page, or another channel already approved. `social_link` never auto-approves (4.2), so an attacker cannot create `examplemid_official`, attach it to the persona and then prove control of it.

| Platform | Method | Ships | Retention |
|---|---|---|---|
| Twitch | OIDC | Launch | Only `{method, platform, observed_at}` |
| Twitch | Bio nonce via Get Users (app token) | Phase 2, if Twitch confirms the fit | Compared and discarded within 24h |
| YouTube | OAuth `channels.list mine=true`; description nonce fallback | Phase 2, after Google OAuth verification for the scope | Discarded; 30-day cap |
| X | OAuth ($0.010 per user read) | Phase 2, with the paid developer account | Numeric user ID only |
| Kick, CHZZK, SOOP | Not offered until their terms are read | | |

Launch ships Twitch OIDC (OpenID Connect sign-in) only. The YouTube scope needs Google's OAuth verification, and X needs a paid developer account; the MVP does not need either. OIDC is the default because whether the bio nonce fits Twitch's agreement is to be confirmed. Nonces (`SummonerDex-verify-7Q2K91XD`) are single-use, bound to the session and persona, and expire in 30 minutes. A leaked nonce binds only to the session that requested it.

**First binding** (the persona has no owner yet):

1. OIDC proof of an eligible channel moves the `vrf_` session to `social_proven`.
2. The owner gives a contact email. A confirmation link verifies it, and the confirmed address is stored in `private`, with the consent recorded, for notices, holds and re-verification nudges (8.3).
3. The registry emails every other approved channel's contact, and the business contact on file, that an ownership claim is pending, with a cancel link.
4. A Reviewer signs off (the channel's approval evidence is older than the attempt; no other owner exists), and the 72h hold runs with `first_binding_hold` set. A cancel from any notified contact sends the case to the panel.
5. On binding, the owner may attest they are 18+ (`age_status: adult_attested`) or state they are under 18 (`minor_protected`, 8.8).

Later E3 sessions by a bound owner skip steps 3 and 4. Every hold notification goes to every verified channel's confirmed contact, so the real person hears about any action taken through one compromised channel.

### 6.3 E4: profile-icon challenge through a verifier-publisher (phase 3)

The project cannot run a public icon challenge without its own production key, and cannot borrow another product's, because Riot allows one product per key. So an active publisher with the `verifier` flag runs the challenge inside its own product.

Phase 3 waits for two facts about at least one verifier-publisher: it holds its own production key, and Riot has audited the verifier flow as a change to its registered product. Riot's Product Registration rule says new features or changes must be audited through the product's page. Whether that means a new audit of an approved app is to be confirmed, so the plan treats the audit as required. Each verifier candidate confirms its own key status and files its own product change, and Stewards check both before granting the flag (5.1). E4 is offered to nobody until an audit passes; until then every E4 capability uses its E3 fallback in 6.1.

1. After E3, the owner enters a Riot ID. The registry creates `vrf_` (single-use, 10 minutes, bound to the session and `name_key`) and redirects to the verifier with a signed request.
2. The verifier resolves the PUUID under its own key and shows an icon different from the current one, with the persona's and project's names on the page. It waits for `profileIconId` to change after the challenge is issued; `revisionDate` is only extra support.
3. It posts a signed `{vrf_, riot_id_submitted, result: "passed", method: "icon_challenge", observed_at}` to `/v1/verify/attestations`. This is a boolean about the string the owner typed, with no PUUID and no API payload. It may add `subject_ref` (1.4).
4. The registry checks the session, Riot ID, suppression and conflicts, writes an E4 reference, and notifies the persona's verified channels. An attacker might trick the owner into passing a challenge for the attacker's own session. Because the notice goes to the persona's verified channels, the real owner sees it.

Session states: `created` and `social_proven` (30 minutes each), `game_pending` (10 minutes), then `bound`, `failed`, `expired`, or `locked`. `locked` means another persona already owner-verified the account, and the case goes to the privacy panel.

**Second verifier.** Before phase 3 opens, the Stewards name at least two verifier-publisher candidates. Each holds its own production key and is unaffiliated with the others, so E4 does not rest on one product. The flag still goes only to a candidate that passes the checks above. If the only verifier stops or goes `inactive` (5.3), existing E4 status stays valid until `reverify_due`, and owners re-verify at E3 until another verifier exists.

### 6.4 Consent mode, re-verification, RSO

- **Consent mode**: a persona flag. No third-party account claim publishes without the owner approving it; such claims go `held`.
- **Re-verification**: E3 and E4 owners re-verify yearly, or on any conflicting claim. Owners get an email nudge every 90 days to re-confirm their accounts (1.4). Overdue owners keep their hide powers but lose the badge, and their owner-basis accounts go `stale_hard` (1.9). Renames tracked by `subject_ref` carry E4 forward. A rename no publisher tracks needs a fresh E3 declaration or E4 challenge.
- **RSO (E5)**: Riot Sign On. Keep the site, ToS, privacy page, riot.txt and flow screenshots live. Register the product under a Developer Portal group the project owns, and present it as persona-owner tooling (claim, verify, hide), because rejected applications often cite a site the reviewer could not view. After approval, resolve every Riot ID to registry PUUIDs, take Riot deletion lists directly, then request RSO and retire the icon hand-off. Whether RSO and production-key PUUIDs are equal is to be confirmed; the Stewards test it first (13.3, week 2).

## 7. Inclusion facets and rubric

### 7.1 Facets

Anyone can have a persona. Facets are what consumers filter on to decide what they display. A persona can have several, because one person can be a coach, a caster and a streamer. A persona is `listed` with a public-role facet, a reviewer-confirmed public role, or an E3+ owner declaring a public role (pro, streamer, creator). The last route matters for high elo: consumers holding their own Riot keys can compute rank over the accounts owners chose to publish, without the registry touching rank data.

| Facet | Rule (v1) | Granted by | Expiry | Ships |
|---|---|---|---|---|
| `pro` | `affiliation` with `league_published` basis in an allow-listed league, from a reviewer-confirmed team or league page; publisher assertions alone never grant it | Rules engine over Reviewer-confirmed claims | `valid_to` or season end, then `former_pro` | Day one |
| `semi_pro` | Same rule for second-tier leagues (LDL, LCK CL, NACL, non-accredited ERLs, Game Changers) | Rules engine | Season end | Day one |
| `former_pro` | Past `pro` plus a second signal (announcement, no re-listing after the season-start grace period) | Rules engine | Permanent | Day one |
| `streamer_declared` | E3 owner declares they stream LoL on a verified channel; no metrics | Owner | 12 months | Day one |
| `streamer_observed` | A Reviewer saw LoL broadcasts in the past 90 days on the persona's own public channel page, in a browser (not through an API); boolean only, no hours | Reviewer | 6 months | Day one |
| `editorial` | Named Steward, written reason | Steward | 12 months | Day one |
| `community_notable` | Nomination, two independent sources, 7-day comment, two Reviewers other than the nominator | Reviewers | 12-month review | Phase 2 |
| `pro_history` | Points score of 60+ (100 in NA) computed from Reviewer-confirmed affiliation claims; 50-59 reviewed | Rules engine | Permanent, `active` decays | Phase 4 |
| `pro_contract` | Live GCD row in an allow-listed league (tier OR) | `pub_gcd` | Contract end | After Riot Competitive Operations answers |
| `streamer` | Attested `streamer@1` | Metrics publisher | 35 days | After Twitch answers |
| `high_elo` | Never in the registry: it records who a player is, never how well they play | | | Never |

`pro` needs a Reviewer-confirmed page because publisher lines are unreviewed until sampled. Without that, related publishers could create pro facets with no human check, and platform claims must never corroborate themselves.

`streamer_observed` gives a streamer filter without platform data. A Reviewer looking at a public page is not an API read, and the Twitch agreement defines Twitch Data by the API and insights derived from it. Whether a human looking at a Twitch page falls outside that is untested, so it goes to counsel (9.4).

All facets:

- compute only from public claims;
- publish booleans with rubric, `valid_until`, `basis_claims` and `computed_at`, never raw platform or Riot numbers;
- live in `rubrics/{facet}@{n}.json` with `effective_from`;
- drop only when a persona falls below 60% of the entry thresholds;
- run old and new rubric versions side by side for 30 days.

### 7.2 `pro_contract` from the Global Contract Database, once cleared

The Global Contract Database (GCD) is a published Google Sheet with no licence, a different schema per tab, and legal names and team emails on every row. Nothing derived from it ships before Riot Competitive Operations answers in writing. Until then, announcements carry the `pro` facet.

Once cleared, `pub_gcd` works like this:

- It polls each tab's CSV four times a day and detects change by cell B1 plus a content hash.
- It maps columns by header, and alerts on header changes or a row drop over 10%.
- A column allow-list applied while parsing (League, Team, handle, role, End Date, Residency, Status) skips legal-name and contact columns, so they are never held in memory, logged or stored.
- It matches on handle plus team history, because handles collide (Cube, Doran, Zeus), and sends ambiguous rows to review.
- It keeps no raw snapshots, and stores team, league, role class, residency and the raw and parsed `contract_end` ("Day After Worlds 2027").

States run `listed`, `expired_listed`, `unlisted` (absent 3 days), and `former_pro` (with a second signal). Absence never means deletion, and the GCD never produces account links.

### 7.3 Streamer and high elo

`streamer_declared` and `streamer_observed` give consumers a filter at launch without touching platform data. `streamer@1` counts only owner-verified channels. It requires: at least 100 hours streamed in the last 180 days; 10 hours in the last 30 days; 4 streaming days in the last 30 days; an average of 3 or more concurrent viewers. Streams are sampled every 5 minutes, and gaps under 30 minutes are merged. `streamer@1` is accepted only after Twitch answers in writing whether such a boolean is "Twitch Data" under section VI.C of its agreement. YouTube-derived attestations are never accepted. Until Twitch answers, nothing is stored, because holding a Twitch-derived insight already shares it with a third party. So no publisher's `streamer` attestation produces anything until that answer (13.1).

No rank-derived facet enters the registry, ever: the registry answers who, never how well. Rank is also Riot Game Information. Consumers with their own keys compute high elo over published accounts (7.1).

Apart from the Riot-derived fields in 1.4, no gate in this design opens because Riot stays silent; silence is tolerance, not approval. The Twitch answer gates `streamer@1` but not launch. One Steward owns the outside questions and follows up every three weeks (13.3).

### 7.4 How consumers use it

- **Facet list**: `GET /v1/personas?facet=pro,semi_pro&league=LEC&fields=slug,display_name,accounts,facets`, paged by cursor. At launch this lists pros only; streamers appear as owners declare (E3) or Reviewers observe their channels. A consumer that wants other inclusion rules applies them on its own side. The registry publishes facets and their rubrics, not any product's list.
- **Lookup**: `GET /v1/lookup?riot_id=Name%23TAG&at=2027-02-20T00:00:00Z&facets=pro,semi_pro,streamer_declared,streamer_observed`. For a whole lobby or any short list, `POST /v1/lookup/batch` takes up to 10 Riot IDs with one shared `at` and counts as one call against the key's limits.
- **Mirror**: a high-volume consumer should mirror the data instead of calling per request. Load the latest dump, apply `/v1/changes` and `/v1/compliance` from `cut_seq`, and poll both at least hourly (2.2, 8.6). One mirror serves every instance of that consumer within the 100k-calls-a-day read key.

Consumers may match the Riot IDs that Riot's API returns for a live game against public registry data. The Access Terms forbid matching against non-public or Streamer-Mode-hidden accounts, and storing or republishing live status (9.1). A persona with no facet is still found by lookup but never shows in a facet-filtered list: inclusion is filtering, not gatekeeping.

## 8. Privacy and removal

### 8.1 Never stored, never published

- **Never stored**: legal names, birth dates, addresses, relatives; live or last-played status; GCD legal names and emails; PUUIDs (until the project key, then in `private` only); Twitch bio text after the nonce or exact-string check.
- **Never published**: ranks, LP, match data; minors' accounts other than one guardian-attested primary (1.9); age status; Discord IDs; platform-API values, except X numeric IDs and the API-observed Riot IDs and API-read regions in 1.5; contacts; removal reasons.

Intake lint enforces both lists.

### 8.2 Publication defaults

| Claim | Default |
|---|---|
| Any approved or unreviewed claim on a listed or historical persona, whatever its basis (`none` included) or method | Public once stored, except trial, probation and backlog holds (4.4), whose lines publish once sampled; a best-effort notice goes to any known contact (8.3) |
| Any claim on a persona in consent mode | `held` until the owner approves that claim (6.4) |
| Any account of a `minor_protected` persona | `withheld`, except the primary account once `owner_verified` with guardian attestation |
| Accounts without an owner basis on a persona reported as a minor | `withheld` during the panel review (8.4) |
| Any claim an E3+ owner or the panel hides | `hidden` plus suppression, instantly for owners (8.4, 8.5) |

The rule is: publish by default, hide on request. It covers accounts the person never publicly tied to themselves, third-party "this is their alt" submissions, personas of unknown age, and claims from any method.

This default carries known risks, which the project accepts:

- Riot's developer policy bars de-anonymizing players who cannot reasonably be identified from visible information, and VALORANT's RSO opt-in may come to LoL.
- Streamer Mode exists so players can play without being recognized.
- GDPR Art 14 asks for individual notice no later than first disclosure.

The hide and removal machinery applies in full. Owner hide is instant, delist and erase follow the ladder (8.4), and suppression, tombstones, the compliance feed and its 24-hour propagation apply to every hide (8.5, 8.6).

### 8.3 Notice

Notice is best effort: it is sent when a contact is known, logged, and never blocks or delays publication.

1. **Trigger**: storing any claim about a person: a new shell, an account claim of any basis, a rename, a social link, an affiliation. A corroboration of a claim that was already noticed triggers nothing.
2. **Contact**: contacts are stored in `private` and deleted 90 days after the last notice. The registry tries these in order:
   1. the owner's confirmed email from E3 (6.2);
   2. a public business contact that a Reviewer copied from the persona's own page;
   3. for a rostered pro, the team's press contact from the team's own site, asked to forward the notice.

   With no known contact, nothing is sent and the gap is logged.
3. **Delivery log**: a notice counts as delivered when the email provider accepts it and no bounce or complaint arrives within 48 hours (provider webhooks to the Worker). A bounce or complaint marks the contact dead. Delivery state is logged per claim. Nothing is held or purged because delivery failed.
4. **Content**: one digest per persona per 24h, with one-click hide, claim and object links. The digest lists the Art 14 items: controller (the two charter Stewards jointly until an entity exists, 9.3), purposes, basis, categories, sources, recipients, retention, rights, and the right to complain. For a held or withheld claim, the digest says only "a claim about you was received; it is not published", with a link that shows the claim after E3 proof. So the email itself reveals no account to whoever reads the inbox.
5. **Public notice page** at `https://summonerdex.gg/notices`: the general Art 14 notice (what the registry processes, sources, rights, how to object), plus a list that mirrors only items that are already public (slug, display name, role class, date listed). An entry appears when its item becomes public and disappears when it stops being public. So the page shows nothing the API does not show, and an item that never publishes never appears (rule 3).
6. **No waiting period**: publication does not wait for notice, and a hide takes effect whenever it arrives (8.5). Public pages never mention pending claims.

Notices are project email. They are never Discord DMs, and never sent to anyone found only through Riot API data.

| From | Event | To |
|---|---|---|
| New claim | Default / persona in consent mode / minor rule, or a minor report under review | `public` / `held` / `withheld` |
| `held` | Owner approves / owner hides | `public` / `hidden` |
| `withheld` | Panel restores after a minor report, the 1.9 minor exception is met, or an E4 contest is decided for the claim | `public` |
| `public` | Owner or panel hide | `hidden` |
| `hidden` | Owner unhides / panel reverses a wrongful hide of a third-party claim | `public` / `public`, with a fresh best-effort notice |
| Any | Erase | `erased` |

### 8.4 Removal ladder and SLAs

| Level | What | Who asks | Decides | Registry / consumers |
|---|---|---|---|---|
| 1 Hide account | Claim to `hidden` plus suppression | Owner E3+; anyone else by ticket with evidence | Owner; for tickets, any non-recused panel member | Owner instant, ticket 72h / 24h |
| 2 Delist persona | Shell and all claims out; slug suppressed | Owner (6.1), guardian | Automatic for owners; panel for guardians and active-pro disputes | 72h, 30-day cap / 24h |
| 3 Sensitive scrub | Legal name, birth date, address, minor's identity, doxxing | Anyone, verified afterwards | Any panel member acts first | 24h / 24h |
| 4 Erase | Content deleted; tombstone and HMAC kept | Owner (6.1), guardian, legal request | E4 owner self-service (phase 3); otherwise two non-recused panel members, or the quorum fallback | 72h from a verified request, 30-day cap; backups by day 35 / 24h |

The erase SLA is one rule everywhere (11.2): done within 72 hours of a verified request, never later than 30 days from receipt, with backups clean by day 35 (D1 Time Travel keeps 30 days).

**Adjudication**: the two charter Stewards decide, and each recuses where they have a conflict. A third member unaffiliated with any publisher is added when available; this is not a launch gate. A steward who also runs a publisher declares it and does not review that publisher's claims. Intake is through `/privacy`, the bot's link or `privacy@summonerdex.gg`. Requests are acknowledged within 72 hours, and requester contacts are deleted 90 days after closure.

**Quorum fallback**: when fewer than two non-recused members are available, any single non-recused member decides after a 72-hour private notice to the others, and logs the decision for review at the next panel meeting. An erase never waits on quorum, because the GDPR one-month limit runs regardless. If no member is non-recused, the request resolves to erase on day 21. A contested hide resolves to hide, because a wrongful hide costs little and a wrongful publication cannot be recalled.

**Minor reports**: a report that a persona is under 18 must carry evidence (a statement from the person or a guardian, or a public source). It withholds every account without an owner basis at once, and opens a panel review within 72 hours. A persona with a public `league_published` affiliation keeps its shell during the review, so one anonymous report cannot remove a rostered pro from facet-filtered lists. Any other persona is delisted at once. The panel then confirms `minor_protected` or restores the persona.

**Ticket limits**: each requester may open at most 5 privacy tickets a day. A target persona with 3 or more open tickets from different requesters is triaged by a panel member before any further action. Ticket volume per target is kept in private stats, so mass reporting is visible. A requester with three tickets rejected as false in 90 days loses intake access, except for requests about themselves; a person's own rights requests are never refused at intake.

**Active-pro limit**: a persona with an active `pro` or `pro_contract` facet keeps its shell public, but may hide any account its team or league has not published. Retired personas may move to `historical` 12 months after `former_pro`. The limit never applies to a `minor_protected` persona, whose shell follows 8.8.

### 8.5 Hide propagation order

1. Authorize the owner (E3+) or panel role.
2. `private`: insert suppression HMACs first, so a concurrent batch cannot re-add the claim.
3. `core` batch: visibility `hidden`, compliance event, `changes` row with `etag: null` (same `seq`), audit row, outbox rows.
4. Rewrite or delete the R2 persona JSON, search shard and any public notice-page entry; purge the CDN.
5. From phase 2, send signed webhooks to key holders with compliance endpoints.
6. Later ingest of that join key returns `suppressed` only to a publisher whose own reference was withdrawn, and `accepted`, storing nothing, to everyone else (3.7).

### 8.6 Compliance feed

`GET /v1/compliance?since=` returns signed pages of withdraw events (1.12). Hide, delist, scrub and erase share one `withdraw` action, with `scope: claim | persona` and the same 24-hour `apply_by`. Typed actions or different deadlines would tell every consumer that doxxing material existed for a named persona. X's 24-hour deletion rule is the benchmark.

The Access Terms impose the deadline, including the catch-up from a dump's `cut_seq` (2.3). They are enforced by revoking keys, not by the CC BY text, which bars extra downstream terms (CC BY 4.0 section 2(a)(5)(B)). `POST /v1/compliance/ack {through_seq}` is optional. Acks feed transparency stats. A key holder silent for 30 days gets an email from a person before the 5.5 steps start, and nothing is suspended automatically, so one broken cron cannot break a consumer's product. Restores and merges are ordinary `changes` events. Dumps ship the cumulative `withdrawn.jsonl.gz`, and webhooks retry with backoff for 24 hours.

### 8.7 Erasure across stores

| Store | Mechanism | Residual |
|---|---|---|
| D1 `core` and `private` | Row delete; HMAC and tombstone kept | None |
| D1 Time Travel | Ages out after 30 days | SLA: complete by day 35 |
| R2 JSON, search shards, notice page | Rewritten inline, CDN purged | None |
| R2 dumps | Downloads serve only the latest; retained dailies containing the target deleted, latest rebuilt within 24h | Copies already downloaded, bound by the Access Terms and the compliance catch-up |
| Stewards' encrypted backup (launch) and trustee escrow (phase 2, 10.4) | Three weekly copies at most; erasure journal replayed on restore | Gone by day 21 |
| Git / audit chain | No subject data / content replaced, hash kept | None |
| Key holders | Withdraw event, 24h | Revocation for non-compliance |
| Keyless static readers | File deleted, `max-age=600` | 10 minutes of cache; bound only by law |

The requester is told about these residuals.

### 8.8 Minors, Streamer Mode, inbound deletion

- **Minors**: a persona confirmed under 18 (by the owner at binding, or by the panel after a report, 8.4) is `minor_protected`. Its shell is delisted and lists again only on self or guardian attestation. After that it publishes at most its primary account, `owner_verified` with guardian attestation, and never any other account, because the registry never publishes minors' alts. The rules are the same here and in 1.9: self or guardian attestation relists the shell, and guardian attestation is required for the account. No birth date is stored. A `minor_review_after` date asks again later, and adulthood is never inferred from rosters.
- **Unknown age** publishes: every account of a persona not confirmed or reported as a minor follows the default in 8.2.
- **Streamer Mode and live data**: no API flag exists apart from spectator-v5's null `puuid`, so the registry relies on hide on request. Claims from hidden-player observation are stored and shown, owners can hide anything instantly, and no live status is stored. The Access Terms forbid matching live games against non-public or Streamer-Mode-hidden accounts, re-identifying anonymous players, and storing or republishing live status. Matching the visible Riot IDs that Riot's API returns against public registry data is allowed, because Riot's live results already respect Streamer Mode.
- **Inbound deletion**: Twitch `user.authorization.revoke` webhook on the Worker; YouTube within 7 days; X within 24 hours of a request by X or the owner; Discord on request; Riot lists through 5.4.

## 9. Licensing and legal

### 9.1 Instruments

| Instrument | Covers | Accepted at |
|---|---|---|
| CC BY 4.0 | The dataset | Every response and dump |
| Access Terms 1.0 | Attribution; 24h compliance, including the catch-up from a dump's `cut_seq`; `User-Agent`; no matching against non-public or Streamer-Mode-hidden accounts, no storing or republishing live status, no re-identification of anonymous players; throttling and revocation; never restricts the CC BY grant | Clickwrap at key creation and dump download |
| Contributor Terms 1.0 | Non-exclusive, perpetual, irrevocable, sublicensable grant to the licensor (the two charter Stewards jointly until an entity exists, 9.3); moral-rights waiver; lawful-basis warranty; removal at will; attribution lock (relicensing only to licences that require attribution, by two-thirds of active contributors) | Web clickwrap, `/accept-terms`, Publisher Agreement; version stored per submission |
| Publisher Agreement 1.0 | Contributor Terms plus batch accountability, no PUUIDs or raw Riot data, `subject_ref` only as the keyed HMAC in 1.4, `publisher_ref` never derived from a PUUID or Riot ID, consent to sampling, Art 26 joint-controller wording | Clickwrap plus Steward countersignature |
| Apache-2.0 | Worker, adapters, `idfold` and resolver | Repo |

CC BY 4.0 licenses the EU database right as well as copyright, and lets the licensor name one attribution party, so consumers never owe credit to each publisher. A DCO alone grants no licence. Share-alike licences are not used, because they would deter platforms from adopting the data. The live-game clause is narrow on purpose: the concern is hidden accounts, and a broader clause would forbid a consumer's ordinary lookup of the visible players in a game.

### 9.2 Attribution and exclusions

One string, `SummonerDex contributors, CC BY 4.0, https://summonerdex.gg/attribution`, appears in every JSON body, a `Link: rel="license"` header, every manifest and static file, `NOTICE` and the bot's `/about`. Product-level credit satisfies the Access Terms. CC BY applies only when a substantial portion is used, so any duty to show credit rests on the contract. Every response also carries a terms URL. Enforcement runs: credit in the payload, a template letter, a 30-day cure period, throttling, revocation, counsel, and a public "who uses this" page.

Excluded beyond 8.1: raw Riot responses, GCD rows, platform-API handles, bios, counts and stream hours, and Leaguepedia or Liquipedia text or lists. Those wikis are CC BY-SA, so they enter only as evidence URLs. The licence page says the dataset is not endorsed by Riot and holds no Riot Game Information apart from the API-observed Riot IDs and API-read regions, which are served with their source recorded (1.5). Whether a Riot ID checked through account-v1 is Game Information is to be confirmed. So every API-resolved value (`subject_ref`, `riot_id_observed`, an API-read `lol_region`) sits in a separate column that can be deleted (1.4).

### 9.3 Legal entity

Until a legal entity exists, the two charter Stewards jointly hold these roles:

- licensor under CC BY 4.0;
- counterparty of the Access Terms, Contributor Terms and Publisher Agreement;
- holder and enforcer of the database right;
- the controllers named in every notice.

Without an entity the Stewards are personally exposed, so an entity is a recommended follow-up once the project has money or third-party publishers. It is not a launch gate.

| Option | EU database right | Cost |
|---|---|---|
| Commons Conservancy programme (Dutch stichting, a non-profit foundation) | Plausible; whether it accepts data-only projects is to be confirmed | None stated |
| Own Dutch stichting | Yes, if the investment test is met | EUR 500-1,500 notary plus EUR 85.15 KVK (Dutch chamber of commerce) fee; director-residency rules to be confirmed |
| Open Source Collective | No; holds money only | 10% of income |
| US 501(c)(3) or UK CIC | No (a CIC gets the UK right only) | $275-600 or GBP 100-115 |

When the follow-up starts: write to Commons Conservancy, and if there is no written yes within three weeks, instruct a notary. Notary and KVK lead time is to be confirmed, and nothing in the plan (13.3) waits on it. A Dutch bank account is not needed for launch, since the project spends about $15 a month (10.2). It comes with a fiscal host (an organisation that holds and spends money on the project's behalf) when money arrives. IP and domains move to the entity once it exists, because a fiscal host can close and strand the projects it holds.

### 9.4 GDPR, Riot and counsel

- **GDPR**: assume GDPR and UK GDPR apply (Art 3(2)(b), monitoring). A Dutch stichting would likely be an EU establishment, which brings GDPR in through Art 3(1). Art 27's duty to appoint an EU representative binds only controllers caught by Art 3(2), so that question would fall away. Counsel must confirm both steps, including whether non-EU directors change this. A UK representative is still needed (cost to be confirmed). At launch: a privacy notice for listed people naming the Stewards as controllers (G3), Art 26 wording, and a single intake. Art 30 records and a DPIA (data protection impact assessment) are recommended follow-ups once the project has money or third-party publishers, not launch gates. Stay free: Vermont's data-broker law covers non-profits that "sell or license", with exclusions counsel must read.
- **Riot**: disclaimers on every page and the README from day one. The questions to Riot DevRel go through the Developer Portal, the official channel. The GCD question goes to Competitive Operations and the facet question to Twitch. One Steward owns all of these outside questions; they are sent in week 1, with a follow-up every three weeks.
- **Counsel** (low four to low five figures, to be confirmed) is a recommended follow-up once the project has money or third-party publishers, not a launch gate. Topics:
  - the Access Terms against CC BY section 2(a)(5)(B);
  - the attribution lock;
  - who holds the database right under Directive Art 11;
  - Riot IDs as Game Information;
  - whether a stichting with non-EU directors is an Art 3(1) establishment;
  - team press contacts as an Art 14 notice channel;
  - Reviewer-observed `streamer_observed` against Twitch's agreement;
  - US data-broker triggers;
  - user-typed handles;
  - stichting director rules.

## 10. Infrastructure and cost

### 10.1 Components

- **Worker**: one Worker on Workers Paid serves the API, web app, owner and admin consoles, Discord endpoint and publish API. The Free plan caps at 100k requests a day and fails hard.
- **Site**: Astro on Cloudflare Pages, with static persona pages, terms and notices, and islands for the submission form and owner console that call the Worker.
- **Databases**: D1 `core` and `private`, each estimated under 1 GB at 50k personas and 500k claims (10 GB cap).
- **R2** behind `cdn.summonerdex.gg` (r2.dev is for development only), for persona JSON, shards, dumps, ingest and encrypted backups.
- **Crons**: every minute, the projector, ingest, reconciler, outbox and audit chainer (chainer from phase 2); hourly, notices and delivery and bounce logging; daily, dumps, dead-man check, stale-account queue and alert; weekly, backup.
- **Search**: indexed exact `name_key` and display-name prefix with `LIMIT 20`. No FTS5, because D1 cannot export virtual tables.
- **Abuse**: Turnstile, the free WAF rule and in-isolate metering.
- **Email**: Email Routing inbound for privacy@, abuse@, publishers@ and security@. Outbound notice email goes through a transactional provider with SPF, DKIM and DMARC on `summonerdex.gg`, and bounce and complaint webhooks to the Worker (provider to be chosen, price to be confirmed).
- A GitHub Free org.

### 10.2 Monthly cost

Assumptions: 70% of requests are static R2 JSON, 30% hit the Worker at about 5 ms CPU, full-sync writes only changed rows, and logs are sampled at 10%. The table is the steady state after phase 2. At launch the X line is $0, because X verification ships in phase 2.

| Line (USD) | 1M req | 10M req | 50M req |
|---|---|---|---|
| Workers Paid base (10M requests, 30M CPU-ms included) | 5.00 | 5.00 | 5.00 |
| Worker requests over 10M at $0.30/M (0.3M / 3M / 15M) | 0 | 0 | 1.50 |
| Worker CPU over 30M ms at $0.02/M (1.5M / 15M / 75M ms) | 0 | 0 | 0.90 |
| D1 (25B reads, 50M writes, 5 GB included) | 0 | 0 | 0 |
| R2 storage under 10 GB, Class A under 1M | 0 | 0 | 0 |
| R2 Class B, 10M free (0.7M / 7M / 35M static reads) | 0 | 0 | 0 to 9.00 |
| Workers Logs sampled; Turnstile; Email Routing; crons; GitHub; Discord; npm | 0 | 0 | 0 |
| PyPI project, under a project-owned user account (organization accounts are free only for qualifying Community Organizations; to be confirmed) | 0 | 0 | 0 |
| X API reads for E3 (about 200 at $0.010), from phase 2 | 2.00 | 2.00 | 2.00 |
| Outbound notice email (2k-5k digests) | To be confirmed | To be confirmed | To be confirmed |
| Domains .gg, .dev, .org ($76.51/yr renewal; $68.53 first year) | 6.38 | 6.38 | 6.38 |
| **Total, excluding outbound email** | **13.38** | **13.38** | **15.78 to 24.78** |

The 50M R2 range depends on whether cache hits bill as Class B, which is to be confirmed. Its upper bound uses Cloudflare's $0.36/M list price, also to be confirmed. Twitch app registration is assumed free (to be confirmed). Revisit hosting near 17M Worker requests a month (about 57M total), where a dedicated server becomes cheaper. Outside infrastructure: a stichting at EUR 585-1,585 one-off (zero through Commons Conservancy), counsel and a UK representative (both to be confirmed), and later 8-10% of any income to a fiscal host.

**Guardrails**:

- No unindexed D1 statement passes tests, because one unindexed scan of 50k personas bills 50k rows.
- `observed_at` is never refreshed per claim by re-import.
- Ingest pre-fetches with `IN` queries (3.7).
- Keys write `last_used_day` only.
- A daily cron alerts `#stewards` when any product passes 50% of its quota before mid-month.

### 10.3 Accounts and credentials the project must own

| Asset | Held by | Notes |
|---|---|---|
| Recovery mailbox | A free mailbox in the project's name, not on `summonerdex.gg` | Avoids circular recovery through the registrar; hardware 2FA keys for every trustee (two at launch, three from phase 2) |
| Registrar (.gg, .dev, .org) | Recovery mailbox | Auto-renew |
| Cloudflare account | Recovery mailbox; 2 admins, 3 once the third Steward is seated | Scoped API tokens per job |
| GitHub org `summonerdex` | Two owners at launch (the Stewards), three from phase 2; 2FA required | Config-sync HMAC and daily-commit token as Actions secrets |
| npm org; PyPI project | Org; PyPI under a project-owned user account until the entity qualifies as a Community Organization (pricing to be confirmed) | `idfold` packages; no Riot mark in any package name |
| Transactional email provider | Recovery mailbox | DKIM keys and webhook secret in Worker secrets |
| Discord server owner role and application | Project | Bot token only in Worker secrets; credentials may not sit in open source |
| Riot Developer Portal group and product | A group owned by the project, one Riot account per member | Not a personal product |
| Twitch app (launch); Google Cloud project, X developer app (phase 2) | Project | OAuth clients, revoke webhook, X billing |
| Registry Ed25519 key; versioned suppression HMAC secret | Worker secrets plus vault | Yearly `kid` rotation; the HMAC secret rotates only if compromised |
| Backup key (launch); escrow keys (phase 2); shared vault | Backup key in the vault held by both Stewards; from phase 2 each trustee holds an RSA-OAEP private key; the vault is held by the trustees (two at launch, three from phase 2) | Recovery codes for every row above |
| Later | Entity | KVK registration, bank account, fiscal host |

### 10.4 Escrow and continuity

| Asset | Takeover path |
|---|---|
| Public data and rules | Dumps (any free key) and public Git |
| Code | Apache-2.0; `bootstrap.sh` creates both databases, buckets, crons and secret placeholders, then runs `wrangler deploy` |
| Private state, launch | Weekly keyset export of both databases, encrypted with a random AES-256-GCM key through WebCrypto, with the key in the Stewards' shared vault; three copies in R2; a restore drill passes before launch (G4) |
| Private state, phase 2 | The same export, with the AES key wrapped to each trustee's RSA-OAEP key (no new dependency); three copies in R2 plus one trustee-held object overwritten weekly; released only to a successor who accepts the existing terms, because a fork without the suppression list re-publishes hidden accounts |
| Erasures after a snapshot | Daily encrypted erasure journal (opaque IDs, HMACs), replayed by `bootstrap.sh` before serving |
| Control | Dead-man cron: 30 days without a Steward login alerts the trustees and `#governance`; at 60 days the succession clause applies |
| Money | About $15 a month on any successor's card |

The backup runs from launch, and only the three-trustee key wrapping waits for phase 2, because losing the suppression list is the one failure that re-publishes hidden accounts.

## 11. Governance and administration playbook

### 11.1 Roles and two-Steward rules

| Role | Powers | Granted | Now |
|---|---|---|---|
| Steward | Merges, publisher keys, publisher independence roots, rubrics, facet overrides, admin, entity board | Charter; later by nomination, two seconds in 7 days, and a 7-day majority | 2; a third before any key; at most 2 of 5 from one employer |
| Privacy panel | Hide tickets and disputes, scrub, erase, minor reviews, active-pro limit, E3 delegates | Stewards appoint; a third member unaffiliated with any publisher is added when available | The two charter Stewards, each recusing where conflicted; the third member is not a launch gate; quorum fallback in 8.4 |
| Reviewer | Submissions, samples, first-binding sign-offs, social links, re-confirmations, `streamer_observed` | Steward grant; trusted submitters invited after 60 days | Stewards, the panel's third member, recruits |
| Publisher / Persona | Keyed writes / own accounts | Sections 5 / 6 | None until phase 2 / any |
| Trustee | Escrow and vault keys | Stewards appoint | 2 at launch (Stewards), 3 from phase 2 |

- Ties keep the status quo. Steward decisions use 72-hour lazy consensus and are recorded in the public decision log in Git.
- No publisher keys are issued yet; the Stewards' data enters only through the seed path (3.8).
- A steward who also runs a publisher declares it and does not review that publisher's claims, keys, batches or samples (11.3).
- Stewards review each other's seed data. No Steward approves their own claims.
- Steward-run publishers are ordinary publishers and consumers with no special rights. The registry builds nothing for any one product.
- Stewards approve every submission at first. Auto-approval starts per submitter at the thresholds in phase 2, never before day 14.

### 11.2 SLAs

| Queue | Target |
|---|---|
| Community submission | First decision 72h median, 7 days maximum; digest of items older than 48h |
| First-binding sign-off | Within the 72h hold |
| Publisher application / trial / daily sample | Decision by day 21 / 14 days / each publisher-day sampled within 7 days; more than 7 unsampled publisher-days hold new batches (4.4) |
| Privacy: non-owner hide ticket / scrub / delist / erase | 72h / 24h / 72h, 30-day cap / 72h from a verified request, 30-day cap, backups by day 35 |
| Minor report review | 72h |
| Appeals | 14 days, by Stewards who did not act |

Stale submissions close at 30 days. Decisions about people never pass by timeout.

### 11.3 Conflicts and Discord

A Steward who works for or runs a publisher cannot approve its keys, batches, samples, verifier flag, independence root or disputes. Nobody reviews a persona they are, represent or manage. Conflicts are listed in `log/coi.md`, and recusals are logged.

Discord is the front desk. Decisions live in D1 and Git, because Discord content is neither archived nor licensed.

- Public channels: `#start-here` (pinned Start button, terms), `#announcements`, `#api-help`, `#api-announcements` (30-day deprecation notices), `#publisher-onboarding`, `#appeals`, `#decisions` (mirror of the public decision log).
- Private channels: `#review-queue`, with one thread per submission carrying only its ID, action and a link to the signed-in admin view where Riot IDs and evidence are shown. So unpublished claims never sit in Discord, which cannot erase on demand. Also `#sampling`, `#privacy-panel` (links only) and `#stewards`.
- No `#identify` channel and no submission threads visible to members.
- Verification level HIGH. Contributor Covenant 2.1, with a reporting contact: either Steward, with recusal, until the panel's third member is added, who then becomes the non-Steward contact.

### 11.4 What is logged publicly

- The public decision log: policy, publisher onboarding, independence roots and revocation, rubrics, recusals, and quorum-fallback decisions in redacted form.
- Publisher pages with reason classes.
- Monthly stats: submissions, approval rate, median decision time, revert rate, withdrawals as one total, SLA attainment, compliance acks, panel workload.
- Signed audit heads (phase 2).
- The public notice page, which mirrors public items only (8.3).

Never logged publicly: who asked for a removal, why, which account, any Discord ID, or privacy-ticket volume per target, which stays in private stats.

### 11.5 Succession and drills

GOVERNANCE.md names roles, not people, and has no single final arbiter, because a Steward-for-life rule can outlive its first Stewards. A Steward unreachable for 30 days triggers the dead-man notice; at 60 days the others appoint a replacement. If a Steward leaves, the panel runs on the quorum fallback (8.4) until a replacement is seated, so erasures never stall. Every six months: restore the backup or escrow into a scratch account, rotate one secret, confirm three people can reach the registrar, and review GOVERNANCE.md.

## 12. API surface

### 12.1 Endpoints

Base `https://api.summonerdex.gg/v1`, static `https://cdn.summonerdex.gg/p/{slug}.json`. JSON, RFC 3339 UTC timestamps, `ETag` on every GET, cursors, a contact `User-Agent` on keyed calls, and breaking changes as `/v2` with 90 days of overlap.

| Method and path | Auth | Purpose |
|---|---|---|
| `GET cdn/p/{slug}.json` | None | Persona (1.10) |
| `GET /personas/{persona_id or slug}` | Read key | Persona; merged returns `301` |
| `GET /personas?facet=&league=&q=&fields=&basis=&min_tier=&review=&cursor=` | Read key | Filtered list, prefix search |
| `GET /lookup?riot_id=&at=&facets=` | Read key | Join key plus window, facet filter |
| `POST /lookup/batch` | Read key | Up to 10 Riot IDs with one shared `at` and `facets`; one call against limits (7.4) |
| `GET /claims/{clm_}`, `GET /personas/{id}/claims` | Read key | Raw public claims |
| `GET /changes`, `GET /compliance`, `POST /compliance/ack` | Read key | Feeds (2.2, 8.6) |
| `GET /dumps/latest` | Read key | Signed manifest of the newest dump only, 15-minute R2 URLs |
| `GET /publishers[/{id}]`, `/facets/rubrics`, `/meta` | None | Metadata, signing keys |
| `POST /keys` | OAuth plus clickwrap | Read key |
| `POST /webhooks`, `DELETE /webhooks/{id}` | Key | Phase 2 |
| `POST /publish/batches` (`X-Batch-Id`, `X-Sent-At`, `X-Signature`), `GET /publish/batches/{bat_}` (status, receipts, chained hashes), `POST /publish/claims/{clm_}/retract`, `POST /publish/erasures` | Publish key plus signature | Phase 2 (3.7, 5.4) |
| `POST /verify/attestations` | Verifier key plus signature | E4, phase 3 |
| `POST /submissions`, `GET /submissions/{sub_}` (submitter view only, 3.1), `POST /verify/sessions[/{vrf_}/check]` | Session plus Turnstile | Intake, verification |
| `POST /privacy/requests` | Session or email token | Returns a ticket ID only |
| `PATCH /owner/claims/{clm_}`, `POST /owner/accounts/{acc_}/confirm`, `POST /owner/personas/{persona_id}/visibility` | Owner session | Show, hide, re-confirm, consent mode, delist, erase |
| `POST /discord/interactions`; `POST /admin/...` | Ed25519; Steward or Reviewer session | Bot; queue, keys, merges, config |

### 12.2 Response envelope

`GET /v1/lookup?riot_id=Example%20Mid%23EUW&at=2027-02-20T00:00:00Z`, after the owner bound at E3 on 2027-02-04 and before E4:

```json
{
  "data": {
    "riot_id": { "game_name": "Example Mid", "tag_line": "EUW" },
    "name_key": "examplemid#euw",
    "at": "2027-02-20T00:00:00Z",
    "persona": {
      "persona_id": "per_01MBJH79M071HFE865V215DE1C",
      "slug": "example-mid-7q2k",
      "display_name": "ExampleMid",
      "facets": [
        { "facet": "pro", "rubric": "pro@1", "valid_until": "2027-11-15" },
        { "facet": "streamer_declared", "rubric": "streamer_declared@1", "valid_until": "2028-02-04" }
      ]
    },
    "account": {
      "account_id": "acc_01MBJH7AK8E6RHXQAQPDH4AFAX",
      "primary": true,
      "current": { "game_name": "Example Mid", "tag_line": "EUW", "name_key": "examplemid#euw", "region": "euw1" },
      "matched_name": {
        "game_name": "Example Mid", "tag_line": "EUW", "name_key": "examplemid#euw",
        "from": "2027-01-04T14:03:00Z", "to": null,
        "evidence_tier": "E3", "disclosure_basis": "owner_declared", "review_status": "approved",
        "publishers": ["pub_01MAX9T4Q0V7N2C8RJ5WZB3KHE"], "claim_ids": ["clm_01MBJH7BJGRHEM3E2MSH4DMDZS"]
      },
      "last_confirmed": "2027-02-04T17:00:00Z",
      "freshness": "fresh",
      "corroboration": 1
    }
  },
  "meta": {
    "seq": 4812,
    "policy": "registry-default@1",
    "review": { "unreviewed_claims": 0, "contested_claims": 0 },
    "freshness": { "oldest_last_confirmed": "2027-02-04T17:00:00Z", "stale_claims_hidden": 0 }
  },
  "attribution": { "text": "SummonerDex contributors, CC BY 4.0, https://summonerdex.gg/attribution", "license": "CC-BY-4.0", "terms": "https://summonerdex.gg/terms/access-1.0" }
}
```

Every claim-bearing object (a name entry on an account, a social, an affiliation) carries `review_status`, `evidence_tier`, `disclosure_basis` and `publishers`, and every account carries `last_confirmed`, and every facet carries `rubric` and `valid_until`. This holds in API responses, static files and dumps alike, so consumers can filter (`review=approved`, `min_tier=E3`, `basis=owner_verified`) or badge `unreviewed` items. `seq` grows over time: 4812 here, 5390 for the 2027-03-01 tombstone (1.12), 6102 for the 2027-03-15 persona file (1.10).

### 12.3 Errors and rate limits

| Case | Response |
|---|---|
| Malformed query; invalid submission or line | `400 invalid_request`; `422 validation_failed` with per-field or per-line reasons |
| Missing or revoked key; outdated terms, suspended key, default `User-Agent` | `401 key_required`, `key_revoked`; `403 terms_update_required`, `key_suspended`, `user_agent_required` |
| Unknown, hidden, delisted, erased, withheld or held persona or Riot ID | `404 not_found`, byte-identical bodies |
| Merged persona; failed `expected_prior`; cursor over 90 days | `301` with `canonical_slug`; `409 expected_prior_mismatch`; `410 cursor_expired` with `cut_seq` |
| Reused `batch_id`; `sent_at` over 5 minutes off | `409 duplicate_batch`; `400 clock_skew` |
| Rate limit | `429` with `Retry-After` |

Limits:

- Keyless reads exist only as static single-persona JSON through the CDN and the free WAF rule. Every Worker read of subject data needs a key. The metadata endpoints (`/publishers`, `/facets/rubrics`, `/meta`) stay keyless because they carry no subject data (12.1).
- Read keys: 10 req/s burst and 100k a day, or 1M a day on request. Bulk users use dumps plus feeds. `POST /lookup/batch` counts as one call.
- Publisher keys: per 5.3.
- Feeds: polled at most every 10 seconds.
- Community submissions: 3, 10 or 30 open at a time by trust state, and at most one per minute.
- Privacy tickets: per 8.4.

## 13. MVP cutline and roadmap

### 13.1 Day-one minimum

Day one is public launch, targeted for around week 6 to 8 with two people. The example dates in sections 1 and 12 are not the plan. Launch serves the public schema (in the rules repo, 2.1), and any consumer adapts to it. The minimum is the schema, resolver packages, lookup, list, dumps, feeds, hide and remove:

1. 300 to 500 seed personas, all rostered pros in allow-listed leagues, from the Stewards' curated data (3.8). Both Stewards' rows are loaded in the same week and reviewed by the other Steward (11.1). Each claim has its disclosure basis recorded, and a best-effort notice is sent where a contact is known (8.3).
2. `GET /v1/lookup`, `POST /v1/lookup/batch`, `GET /v1/personas/{id}` and static JSON for persona-to-accounts lookup.
3. `GET /v1/personas?facet=&league=` over `pro@1`, `semi_pro@1`, `former_pro`, `streamer_declared@1`, `streamer_observed@1` and `editorial`.
4. `@summonerdex/idfold` and `summonerdex-idfold` for join-key parity, and the TypeScript resolver (1.14).
5. A clickwrap read key, the latest dump plus the `/v1/changes` and `/v1/compliance` feeds, and hide, delist, scrub and erase working end to end.

What launch does not serve: facet lists launch with pros only. Streamers enter as owners declare (E3) or Reviewers observe their channels. High-elo players appear only as published accounts that consumers rank with their own keys. Metrics attestations produce nothing from any publisher until Twitch and DevRel answer (7.3).

### 13.2 Launch gates (public reads stay off until all pass)

Public reads open when the engineering gates G1, G2, G3 and G4 pass.

| Gate | Condition |
|---|---|
| G1 | `is_public` CI suite: byte-identical responses for every non-public state on every output path, plus tests that supplied `owner_*` bases, E3+ tiers and facet bases are rejected, and that a suppressed submission's submitter view matches an ordinary rejection |
| G2 | Hide, delist, scrub and erase end to end, including dump rebuild, the compliance feed and the quorum fallback |
| G3 | A working site with ToS (Access Terms, Contributor Terms), a privacy notice naming the Stewards as controllers, and Riot disclaimers on every page and the README; the DevRel, Competitive Operations and Twitch questions sent, with an owner and follow-up dates (answers not required) |
| G4 | Restore drill of the encrypted launch backup passed (10.4) |

A legal entity, counsel review, a DPIA, Art 30 records and a third panel member are recommended follow-ups, not gates (8.4, 9.3, 9.4). Notice delivery is not a gate either, because notice never blocks publication (8.3).

### 13.3 Weeks 1 to 7

The Stewards build from week 1 in two equal lanes, A and B, and neither lane waits on the other. The Lane A Steward owns the outside questions. Both Stewards' seed rows load in week 4, and each Steward reviews the other's (3.8, 11.1). The table plans G1, G2, G3 and G4 for week 7, inside the week 6 to 8 target. Public reads open earlier only if that work is pulled forward and the gates pass.

| Week | Lane A | Lane B | Exit check |
|---|---|---|---|
| 1 | Name clearance (15), domains, recovery mailbox, Cloudflare, GitHub org, vault; send the Riot, Competitive Operations and Twitch questions | Rules repo (schema, reason codes, GOVERNANCE.md with recusal rules and quorum fallback, conflicts); `idfold` in TypeScript and Python with generated case-fold tables and vectors | Questions sent |
| 2 | D1 schemas; email provider with SPF, DKIM, DMARC; Twitch app; run the PUUID and Streamer Mode test (about 45 minutes, single-key and cross-key steps) with the Stewards' existing Riot keys | Steward CLI seed loader; admin console (GitHub OAuth, role rows); audit rows (no chainer) | Vectors pass in all three CIs |
| 3 | Validation module (tier re-derivation, owner and facet basis rejection, `subject_ref` and Riot-derived fields), claims and references, binding, suppression, `is_public` with the identical-bytes harness; prepare this lane's seed rows (half of up to 500) with evidence, a primary account and contacts where known; PR this Steward's publisher record (1.13) | Same seed and publisher-record work for this lane; reconciler | Harness green; both Steward records approved |
| 4 | Privacy intake, the four removal levels in 8.5 order, quorum fallback, ticket limits, minor-report flow; load this lane's seed rows | Load this lane's seed rows; E3 Twitch OIDC with channel eligibility, email capture and the first-binding hold; owner console with re-confirmation and instant hide | Removal levels pass on fixtures; both Stewards' seed loaded |
| 5 | Best-effort notice: digest, delivery log, bounce and complaint webhooks, public notice page; projector to R2; review Lane B's seed rows | Web form; Discord bot with ID-only review threads; review Lane A's seed rows | E3 on a Steward's own channel; seed review under way |
| 6 | Persona, lookup, batch lookup and list endpoints; changes and compliance feeds; read keys, latest-only dumps; finish reviewing Lane B's seed rows | Facet engine with `streamer_observed`, stale-account queue and alert; site with disclaimers, ToS, privacy notice; finish reviewing Lane A's seed rows | Feeds match D1; seed reviewed |
| 7 | G1 suite on every path; cost alerts; ingest and lookup load test measuring statements per line; launch checklist | Read-path tests on staging against the reviewed seed: lookup, batch lookup, facet list, dump load with feed catch-up; encrypted backup job; restore drill (G4); panel runbook; first-binding dry run with a seed persona that volunteers | Launch if G1, G2, G3 and G4 pass; otherwise public reads stay off and the rules do not change; first stats |

**External dependencies and lead times**

| Dependency | Owner | Starts | Lead time | Blocks |
|---|---|---|---|---|
| Legal entity (Conservancy, or notary plus KVK) | Lane A | When the project has money or third-party publishers | Conservancy answer in about 3 weeks; notary and KVK to be confirmed | Nothing; recommended follow-up (9.3) |
| Counsel review of terms, DPIA, records | Lane A | With the entity follow-up | Assumed 3 weeks (to be confirmed) | Nothing; recommended follow-up (9.4) |
| Transactional email provider and domain authentication | Lane A | Week 2 | Days; deliverability warm-up before week 5 | Best-effort notices, which block nothing |
| Twitch app registration | Lane A | Week 2 | Days | E3 |
| Third panel member and third Steward | Both | Week 2 | When available / before phase 2 | Nothing / publisher keys |
| Riot DevRel, Competitive Operations, Twitch | Lane A | Week 1 | Unknown; follow-up every 3 weeks | `pro_contract`, `streamer`; the DevRel data-flow question stays open (1.4); none block launch |
| A verifier-publisher's own production key and verifier product audit | Each verifier candidate | Before phase 3 | Unknown | Phase 3 |
| Google OAuth verification, X developer account | Lane A | Phase 2 | Weeks (to be confirmed) | YouTube and X E3 |

Not in launch:

- Publisher keys and the signed publish API (a third Steward is needed)
- Webhooks
- E4
- Auto-approval (no one qualifies before day 14; the ledger records from day one)
- YouTube and X E3
- The audit chainer and signed heads
- Trustee escrow
- The Python resolver
- `community_notable`
- `pro_history`
- The GCD adapter
- `streamer@1`
- Anything rank-derived

### 13.4 Phases after launch

| Phase | Earliest | Ships | Gate |
|---|---|---|---|
| 2 Publishers | 1 to 6 weeks after launch | Trial keys for applicants, Steward-run publishers included, each approved only by Stewards unaffiliated with it (5.2, 11.3), with `subject_ref` and the Riot-derived fields in 1.4; signed publish API, daily sampling, webhooks, auto-approval, `community_notable`; audit chainer, signed heads and hashed receipts; trustee escrow; Python resolver; YouTube and X E3 | Third Steward (entity recommended first, 9.3) |
| 3 E4 | 3+ weeks after launch | Verifier flag for each candidate that passes 6.3; E4 self-erase and delegates | At least two verifier candidates named before the phase opens (6.3); at least one verifier-publisher's own production key confirmed and Riot's audit of its verifier feature passed |
| 4 Riot application | Month 4 | Project production-key application as persona-owner tooling; `pro_history` | Stable site |
| Contract data / streamer metrics | On written answers | `pro_contract` and the ERL Player Database route / `streamer@1` | Competitive Operations / Twitch |
| 5 Project key | Months 7-10 (Riot's review typically takes about two months and can take up to a year) | Registry PUUIDs, re-resolve all, reliable reuse detection, direct Riot deletion lists | Production key |
| 6 RSO | After phase 5 | E5; retire the icon hand-off | RSO client |
| Scale | About 17M Worker requests a month | Revisit hosting | Cost alerts |

### 13.5 Risks

| Risk | Mitigation |
|---|---|
| Engineering gates delay the launch | Public reads stay off until G1, G2, G3 and G4 pass; entity, counsel and panel are follow-ups that block nothing (13.3) |
| E3 hijack of a persona | Reviewer-approved channels only, first-binding hold with notices, delist hold, instant hide (6.1, 6.2) |
| Riot never audits the verifier change | E3 fallbacks for every E4 capability; second verifier candidate; project key in phase 5 |
| Twitch objects to attestations or observation | Nothing Twitch-derived is stored first; `streamer_declared`; counsel on `streamer_observed` |
| Reviewer bottleneck | Capped seed (25 review hours), daily sample budget sized to 6 hours a week, caps that rise only with capacity, published SLAs |
| Data decays to `stale_hard` before publishers exist | Key-free confirmation paths and the day-60 alert (1.4) |
| Griefing through reports or tickets | Evidence-required minor reports, shell kept for rostered pros during review, ticket limits (8.4) |
| Re-ingest brings back hidden data; a Steward disappears | Suppression first, reconciler, launch backup then trustee escrow with erasure journal, dead-man cron, panel quorum fallback |

## 14. Open questions

Each line gives the question and the current preferred answer, which the sections above already apply.

- **Riot DevRel data-flow question** (may Riot-ID mappings and PUUID-derived references flow from a publisher to the registry and into dumps?): keep accepting `subject_ref`, API-observed renames and API-read regions; drop `subject_ref` only on a written no (1.4).
- **GCD contract data**: nothing ships until Riot Competitive Operations answers in writing (7.2).
- **Streamer facet before Twitch answers**: `streamer_declared` plus Reviewer-observed `streamer_observed`, with counsel on observation (7.1, 7.3).
- **Who the third Steward is**: unaffiliated with any publisher, EU resident preferred, seated before phase 2; until then no publisher keys, and the Stewards' data uses the seed path (3.8, 5.1).
- **Paid tier or donations**: neither until counsel reviews US data-broker laws and Riot's legal terms.
- **Attribution scope**: product-level credit (9.2).

## 15. Name

The project is called SummonerDex. The planned domains are summonerdex.gg (the main domain, used for the site, API and CDN), summonerdex.org and summonerdex.dev, and the GitHub org is `summonerdex`. The packages are `@summonerdex/idfold` on npm and `summonerdex-idfold` on PyPI. "Summoner" is Riot's own term, so a trademark search for the word "summoner" (and for "summonerdex") still has to be done before the project is announced.
