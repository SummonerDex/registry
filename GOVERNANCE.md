# Governance

SummonerDex is an open, community-run registry of League of Legends personas. It is open source and anyone can contribute, but at heart it is only a registry. It answers who, never how well: identity, the accounts tied to a persona, name history, socials, esports teams and roles, aliases, and facets saying what kind of persona someone is. It holds no Riot data except the accounts themselves and esports affiliations, and nothing product-related. No rank, stats, match history or live status. Every platform adapts to the schema. No feature is built for any one consumer.

Governance is built on roles, not people, and has no single final arbiter. The people who hold each role today are listed under "Who holds what now". The full rules are in `docs/ARCHITECTURE.md` (sections 3, 4, 5, 6.1, 8.4, 10.3, 10.4 and 11). If this file and the architecture disagree, the architecture wins until one of them is fixed.

## Roles

| Role | May do | How it is granted |
|---|---|---|
| Steward | Merges, splits, relinks and slug changes; publisher keys and publisher independence roots; rubrics; facet overrides and editorial facets; admin; appeals; the entity board once an entity exists | Two charter Stewards at the start; later by nomination (below) |
| Privacy panel | Non-owner hide tickets and disputes, sensitive scrubs, erasure, minor reviews, the active-pro limit, delegates for E3 owners | Appointed by the Stewards |
| Reviewer | Community submissions, platform samples, first-binding sign-offs, social links, re-confirmations, `streamer_observed` | Granted by a Steward |
| Publisher | Keyed writes through the publish API, within its key state and declared predicates | Publisher onboarding (below) |
| Persona | Authority over its own accounts, links, name and visibility, and nothing else | Identity verification (architecture section 6) |
| Trustee | Holds escrow and vault keys | Appointed by the Stewards |

### Steward

- Merge, split, relink and slug-change decisions carry a 48h hold, extended to 72h after an objection.
- An editorial facet needs a written reason and a 7-day comment period, and it expires after 12 months.
- Until three Reviewers exist, a Steward confirms every `create_persona`.
- Stewards review submitter pairs flagged by the weekly cluster check before their auto-approval resumes.
- Appeals are decided within 14 days by Stewards who did not act on the original decision.
- At most 2 of 5 Stewards may share an employer.

### Reviewer

- Nobody approves their own, their employer's or their publisher's submission.
- An `add_claim` of an `account_link` needs a Reviewer unaffiliated with the submitter. A `social_link` always needs a Reviewer, because an approved social channel can become E3 authority.
- Platform trial batches are reviewed in full by two Reviewers unaffiliated with the publisher. Daily samples are reviewed by one unaffiliated Reviewer.
- A Reviewer also samples 10% of auto-approved items each week.

### Publisher

A publisher is a product that sends claims in bulk with a signed key. Kinds: `platform`, `system`, `community` and `persona`, plus a `verifier` flag approved separately by two unaffiliated Stewards. A publisher never declares its own independence. Stewards set its independence root at onboarding. An independence root groups publishers whose data is not independent; publishers in the same root count as one source when claims are corroborated. Publishers run by Stewards count as one root until a third publisher unaffiliated with any Steward is active.

### Persona

What a persona owner may do depends on their verification tier (architecture 6.1). In short: an E3 owner can hide or unhide their own claims instantly, make their own accounts public, and contest third-party claims (a Reviewer decides). First binding needs Reviewer sign-off and a 72h hold. Delisting at E3 waits 48h unless cancelled. Erasure at E3 is a ticket to the privacy panel.

## Reputation

Community submitters are not a governance role. What they may submit is limited by counted reputation states, set in `rubrics/reputation@1.json` (architecture 4.1):

| State | Entry rule | Open submissions at a time |
|---|---|---|
| `new` | Terms accepted; Discord or OAuth account 30+ days old | 3 |
| `queued` | After the first submission | 10 |
| `trusted` | Trailing 180 days: 10+ counted approvals across 5+ personas on 3+ distinct days, first approval 14+ days old; 0 reverts until 100 outcomes, then at most 3% reverted | 30 |
| `suspended` | Demotion (architecture 4.3) | 0 |

Only `approved_by_review` and `upgraded_by_owner` outcomes count toward promotion. Auto-approval for `trusted` submitters starts in phase 2, never before day 14. It covers only additive `account_link` claims that pass every check in architecture 4.2. Persona creation, social links, merges, relinks and contested accounts always need a human. No reputation state can set an owner basis, re-add a suppressed account or contest a privacy decision. Sockpuppetry or vote farming suspends indefinitely, with appeal to the Stewards.

## Two-Steward phase

The project starts with two charter Stewards. Until a third Steward is seated:

- Steward decisions use 72h lazy consensus: a proposal stands if no Steward objects within 72 hours. A tie keeps the status quo.
- A Steward who also runs a publisher declares it and does not review that publisher's claims, rows, keys, batches or samples. No Steward approves their own claims or their own seed rows. Stewards review each other's seed data.
- The Stewards approve every submission at first.
- No publisher key, trial or active, is issued. Stewards' own data enters only through the Steward CLI seed path (architecture 3.8), landing as `pending_review`.
- Products run by Stewards are ordinary publishers and consumers with no special rights. The registry builds nothing for them.

## The third Steward

- The third Steward should be unaffiliated with any publisher. An EU resident is preferred.
- The third Steward is seated before any publisher key is issued. Publisher keys and the signed publish API (phase 2) wait on this.
- When the third Steward is seated, the Cloudflare account moves from 2 admins to 3.

Who the third Steward will be is still open.

## Nomination and removal

**Stewards.** After the charter Stewards, a Steward is nominated, needs two seconds within 7 days, and is confirmed by a 7-day majority.

**Reviewers.** A Steward grants the Reviewer role. Trusted submitters are invited after 60 days.

**Privacy panel and Trustees.** Appointed by the Stewards.

**Unreachable Stewards.** A Steward unreachable for 30 days triggers the dead-man notice to trustees and `#governance`. At 60 days the other Stewards appoint a replacement.

**Removal.** There is no removal process for Reviewers or Stewards yet. It is still open.

## Publisher onboarding

1. A PR adds `publishers/{pub_id}.json`: operator, contact, method document, open-source ingest code, predicates, maximum rate, rollback plan, conflicts and declared `derived_from`.
2. 14 days of public review on the PR and in `#publisher-onboarding`.
3. Approval by two Stewards unaffiliated with the applicant, who also set its independence root.
4. Publisher Agreement clickwrap by a named officer, with Steward countersignature.
5. Trial key: 250 lines, all held, reviewed within 14 days. If under 2% are errors, a Steward sets the publisher to `active` by PR.

A decision on an application is due by day 21. Revocation runs warning, probation, suspension, revocation. The reason class is shown on the publisher's public page (architecture 5.5). Applications go to publishers@summonerdex.gg (planned).

## Privacy panel

**Members.** The two charter Stewards. A panel member who also runs a publisher declares it and does not review that publisher's claims. A third member unaffiliated with any publisher is added when available. This is not a launch gate.

**Intake.** Through `/privacy`, the bot's link, or privacy@summonerdex.gg (planned). Hide, delist, scrub and erase are not submissions and never sit in a queue volunteers browse. Requests are acknowledged within 72 hours. Requester contacts are deleted 90 days after closure.

**Removal ladder and SLAs** (architecture 8.4 and 11.2):

| Level | Who decides | Registry target | Consumers |
|---|---|---|---|
| 1 Hide account | The owner (instant); for tickets, any non-recused panel member | Owner instant, ticket 72h | 24h |
| 2 Delist persona | Automatic for owners; panel for guardians and active-pro disputes | 72h, 30-day cap | 24h |
| 3 Sensitive scrub | Any panel member acts first; verified afterwards | 24h | 24h |
| 4 Erase | Two non-recused panel members, or the quorum fallback (E4 self-service from phase 3) | 72h from a verified request, 30-day cap, backups clean by day 35 | 24h |

Minor reports are reviewed within 72 hours.

**Quorum fallback.** When fewer than two non-recused members are available, any single non-recused member decides after a 72h private notice to the others. They log the decision for review at the next panel meeting. An erase never waits on quorum. If no member is non-recused, the request resolves to erase on day 21. A contested hide resolves to hide.

**Ticket limits.** 5 privacy tickets per requester per day. A persona with 3 or more open tickets from different requesters is triaged by a panel member before further action. A requester with three tickets rejected as false in 90 days loses intake access, except for requests about themselves.

## Other queue targets

| Queue | Target |
|---|---|
| Community submission | First decision 72h median, 7 days maximum; digest of items over 48h |
| First-binding sign-off | Within the 72h hold |
| Publisher application / trial / daily sample | Decision by day 21 / 14 days / each publisher-day sampled within 7 days |
| Appeals | 14 days, by Stewards who did not act |

Stale submissions close at 30 days. Decisions about people never pass by timeout.

## Decision log

- Every Steward decision is written to the public decision log in Git, one file per decision, and mirrored to `#decisions` on Discord.
- Recorded publicly: policy, publisher onboarding, independence roots and revocation, rubrics, recusals, and quorum-fallback decisions in redacted form.
- Never recorded publicly: who asked for a removal, why, which account, any Discord ID, or privacy-ticket volume per target.
- Discord is the front desk. Decisions live in the registry database (Cloudflare D1) and in Git, because Discord content is neither archived nor licensed.

Monthly public stats cover submissions, approval rate, median decision time, revert rate, withdrawals as one total, SLA attainment, compliance acks and panel workload.

## Conflicts of interest

A Steward who works for or runs a publisher cannot approve its keys, batches, samples, verifier flag, independence root or disputes. Nobody reviews a persona they are, represent or manage. Every Steward, panel member and Reviewer declares any publisher they run or work for. Declarations will sit in `log/coi.md` (planned, not yet created), and every recusal is logged.

## Who holds what now

| Role | Now |
|---|---|
| Steward | The two charter Stewards; a third before any publisher key |
| Privacy panel | The two charter Stewards; a third member when available |
| Reviewer | The Stewards, the panel's third member once added, and recruits |
| Publisher | None until phase 2 |
| Persona | Any verified owner |
| Trustee | 2 at launch (the Stewards), 3 from phase 2 |

## Succession

- **Shared vault.** Recovery codes for every project account sit in a vault held by the trustees (two at launch, three from phase 2). The launch backup key is in the vault, held by both Stewards.
- **GitHub and Cloudflare owners.** The GitHub org has two owners at launch and three from phase 2, with 2FA required. The Cloudflare account has 2 admins, and 3 once the third Steward is seated. Every trustee has hardware 2FA keys. The recovery mailbox is a free mailbox in the project's name, not on `summonerdex.gg`. The full list of accounts is in architecture 10.3.
- **Dead-man rule.** 30 days without a Steward login alerts the trustees and `#governance`. At 60 days the succession clause applies and the others appoint a replacement.
- **If a Steward leaves**, the privacy panel runs on the quorum fallback until a replacement is seated, so erasures never stall.
- **Every six months**: restore the backup or escrow into a scratch account, rotate one secret, confirm three people can reach the registrar, and review this file.

## Changing these rules

A change to this file needs an entry in the decision log and 72h lazy consensus among the Stewards. A tie keeps the current text. The file is also reviewed every six months as part of the succession drill.

## Contacts

All planned; the domain is not registered yet.

- privacy@summonerdex.gg: hide, delist, scrub, erase, minor reports
- abuse@summonerdex.gg: abuse of the registry or its intake, and Code of Conduct reports (see `CODE_OF_CONDUCT.md`)
- publishers@summonerdex.gg: publisher applications and key questions
- security@summonerdex.gg: vulnerabilities (see `SECURITY.md`)
