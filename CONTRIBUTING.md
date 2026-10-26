# Contributing to SummonerDex

SummonerDex is an open registry of League of Legends personas. A persona is a pro player, streamer or notable creator. The registry records the accounts, names, socials and esports affiliations tied to each persona. It answers who, never how well. It holds no rank, stats, match history or live status, and it builds nothing for any one product. Every platform adapts to the schema like anyone else.

The domain, the GitHub org and the mailboxes named here (privacy@, abuse@, publishers@, security@ at summonerdex.gg) are planned but not registered yet. A trademark search for the name still has to be done.

There are three ways to contribute:

1. [Data](#data): claims about personas.
2. [Code](#code): pull requests to the Worker, site, bot and packages.
3. [Rules](#rules): schema, rubrics and policy, changed by pull request.

If you want something hidden or removed, skip to [Hide and remove](#hide-and-remove).

## Data

A claim is one statement about a persona, for example "this Riot account belongs to this persona" or "this is their Twitch channel". Each claim comes with evidence.

### Where claims go

This repository holds no persona data. Personas, accounts and claims live in the registry's database, never in Git. Forks and clones keep history even after it is rewritten, so data in Git could never be truly hidden. So **never open a pull request that adds persona data**. It will be closed.

Claims go through the submission queue, which takes them from:

- the web form on the site (sign in with Discord, GitHub or Twitch), and
- the Discord bot in the project's own Discord (`/accept-terms`, `/submit`, `/status`).

Neither exists yet, so community intake opens at launch. Until then the only data entering the registry is the launch seed, which a Steward loads through the seed path. A Steward never reviews seed data they supplied themselves.

Please do not file claims or persona proposals as GitHub issues. There are no issue templates for them. An issue is public the moment you file it. The queue keeps every pending, rejected or blocked submission out of public view, and makes them all look the same from outside. That way, a claim about an account its owner already hid does not put that account back in view. An issue cannot do that.

Never include legal names, birth dates, addresses, relatives or contact emails in any claim. The registry never stores them, and the validator rejects them.

### What evidence a claim needs

Every claim needs an evidence URL. Free-text proof does not work. The evidence tier says how strong the proof is. You never pick a tier: the registry computes it from the evidence method and its own checks.

| Tier | What it means | What you provide |
|---|---|---|
| E0 | A bare assertion ("this is their smurf") | A URL to whatever you based it on. A Reviewer decides. |
| E1 | Public association on a page the persona controls, or on a team or league page | A link to that page (their Twitch about page, X profile, personal site, team roster) showing the exact Riot ID, social or role |
| E2 | Confirmed by a Reviewer on the persona's own recorded broadcast, with Streamer Mode off | The VOD link and a timestamp where the Riot ID is visible |
| E3 | The persona proved control of a Reviewer-approved social channel, passed a 72-hour first-binding hold, and asserts the account | Nothing from you. Only the persona can do this, by signing in (Twitch at launch). |
| E4 | E3 plus a profile-icon challenge run by a verifier-publisher | Nothing from you. Not offered until a verifier's Riot audit passes (phase 3). |
| E5 | Riot Sign On under the project's own client | Nothing from you. Comes later, after the project has its own Riot production key. |
| OR | Official roster, for affiliations only | Not used until Riot Competitive Operations answers in writing |

A few more rules:

- **Social links always need a Reviewer**, because an approved social channel can later become proof of identity.
- **Affiliations** count toward the `pro` facet only when a Reviewer confirms them from a team or league page.
- **Disclosure basis** (whether the person tied the account to themselves in public) is recorded on every claim. What you claim is only a suggestion. A Reviewer sets it.
- **Renames** need an evidence page showing the new Riot ID.
- Type Riot IDs exactly as shown, as `GameName#TAG`. Do not guess the region from the tag.
- The registry runs no OCR, no live-game matching and no automated alt discovery, and has no "identify this account" queue. Please do not ask the community who owns an account.

### Review and reputation

At first the Stewards review every submission. Nobody approves their own submission, their employer's or their publisher's. A Steward who also runs a publisher declares it and does not review that publisher's claims. The target for a first decision on a community submission is a 72-hour median and 7 days at most. If a Reviewer asks for more evidence, you have 14 days. Open submissions close after 30 days. Decisions about people never pass by timeout.

Approval decides whether a claim is true. Once approved, a claim on a listed persona is public by default, and anyone can ask to hide it (see below).

Your reputation is counted, not scored. Only claims a Reviewer approved, or the persona's owner upgraded, count. Corroborating someone else's claim is recorded but never counts.

| State | How you get there | Open submissions at a time |
|---|---|---|
| `new` | Accept the Contributor Terms with a Discord or OAuth account at least 30 days old | 3 |
| `queued` | After your first submission | 10 |
| `trusted` | In the last 180 days: 10 or more approvals, across 5 or more personas, on 3 or more distinct days, first approval at least 14 days old; no reverts until you have 100 outcomes, then at most 3% reverted | 30 |
| `suspended` | Demotion | 0 |

The cap is on open submissions, not per day: you can submit more as soon as earlier ones are decided. A limit of one submission per minute applies to everyone.

The thresholds live in `rubrics/reputation@1.json` and change only by pull request.

**What auto-approves** (phase 2, `trusted` submitters only). All of these must hold:

1. It adds a Riot account to a persona. Never a social link.
2. The persona is listed and has an E3 or better owner, who gets notice and can hide or contest it at once.
3. The claimed basis is "public profile", nothing else.
4. The evidence is a page the persona controls, and the exact-string check passes. The Worker reads the page once through a permitted API or plain HTTP, confirms the exact Riot ID appears, and discards the content. Whether this read fits Twitch's developer agreement is to be confirmed. Where no permitted read exists, a Reviewer decides.
5. There is no conflict or suppression. The persona is not in consent mode, not protected as a minor, and has not asked to review claims itself.
6. You are not flagged by the weekly cluster check. It looks for pairs of submitters who touch the same personas within 24 hours on 3 or more occasions.

Persona creation, social links, merges, relinks and contested accounts always need a human. A Reviewer samples 10% of auto-approved items each week.

**Demotion.** If an auto-applied claim is reverted, or an owner dispute is upheld, you are suspended for 30 days and your last 7 days of auto-applied claims go back into the queue. A revert rate over 3% (after 100 outcomes), or 12 months without activity, puts you back to `queued`. Submitting legal names, birth dates, addresses or minors' accounts suspends you for 90 days. Sockpuppetry or vote farming suspends you indefinitely, with appeal to the Stewards.

### Publishers

A publisher is a product that sends claims in bulk with a signed key: a stats site, a bot, any product holding its own Riot key. Publishers follow the same schema and rules as everyone else and get no special rights.

**No publisher key, trial or active, is issued while fewer than three Stewards are seated.** The bulk API is phase 2. You can apply before then with the **Publisher application** issue template, or by writing to publishers@summonerdex.gg (planned).

The onboarding steps:

1. A pull request adding `publishers/{pub_id}.json`: operator, contact, method document, open-source ingest code, predicates, maximum rate, rollback plan, conflicts, and the sources your data is derived from.
2. 14 days of public review on the PR and in `#publisher-onboarding`.
3. Approval by two Stewards unaffiliated with you. They also decide how independent your data is for corroboration. You do not declare your own independence.
4. The Publisher Agreement, accepted by a named officer and countersigned by a Steward. It includes the Contributor Terms, a ban on PUUIDs (Riot's permanent player IDs) and raw Riot data, and consent to sampling.
5. A trial key: 250 lines, all held, reviewed in full by two unaffiliated Reviewers within 14 days. If under 2% are errors, a Steward moves you to `active`.

Active keys start at 5,000 lines a day. After that, each day's lines are sampled. Over 5% errors in a day reverts that day's batches and puts you on probation. Two reverts in 90 days suspend the key. The target for a decision on an application is day 21.

## Hide and remove

The registry publishes by default and hides on request. If something about you, or about someone you represent, is in the registry and you want it hidden or removed, ask. You do not need to give a reason.

**How to ask:**

- **Email privacy@summonerdex.gg** (planned).
- In the project Discord, run `/privacy` or `/hide`. The bot replies with a one-time link, so nothing sensitive is typed in Discord.
- If you are the verified owner of a persona (E3 or better), hide any of your own claims instantly from the owner console.

Do not open a public issue for a hide or removal request. Issues are public, and a request there would show who asked, which account and what they want.

Privacy requests are never submissions and never sit in a queue volunteers can browse. We reply within 72 hours.

| What | Who can ask | Target |
|---|---|---|
| Hide an account | The verified owner (instant), or anyone else with evidence | 72 hours for a ticket |
| Delist a whole persona | The owner or a guardian | 72 hours, 30 days at most |
| Remove a legal name, birth date, address, minor's identity or doxxing | Anyone, verified afterwards | 24 hours |
| Erase everything | The owner, a guardian, or a legal request | 72 hours from a verified request, 30 days at most, backups clean by day 35 |

Consumers holding a key must apply every hide within 24 hours through the compliance feed. A hide that is contested stays hidden. Requests are decided by the privacy panel, which is currently the Stewards. A panel member who also runs a publisher declares it and does not review that publisher's claims. Requester contacts are deleted 90 days after the request closes. We will tell you what copies may remain, for example copies that key holders already downloaded.

Two limits apply. A persona with an active pro facet keeps its public shell, but can hide any account its team or league has not published. A persona confirmed as under 18 publishes at most one primary account, verified by the owner and confirmed by a guardian. If you know a listed person is under 18, tell privacy@ with evidence.

## Code

Code is licensed under Apache-2.0. Pull requests are welcome.

- **Sign off your commits** with the Developer Certificate of Origin (https://developercertificate.org): `git commit -s`, which adds `Signed-off-by: Your Name <you@example.com>`. Sign-off is encouraged but not yet required for merge. Whether it becomes required is still open. The sign-off covers code only; it grants no licence to data.
- Keep a PR to one change, and say why it exists.
- Run the tests and say which you ran.
- PR checks run read-only and never execute PR content.

Changes we will not take, whoever asks:

- Anything that serves PUUIDs, or stores them outside the phase 5 private store.
- Anything that stores or serves ranks, LP, match data, live or last-played status, legal names, birth dates or addresses.
- Features built for one consumer. Platforms adapt to the schema.
- Any output path (API, static JSON, search, dumps, feeds, webhooks, notice page) that skips the single publication check. Every non-public state must return identical bytes.
- Anything that lets a payload set an E3 or higher tier, an owner basis, or a basis on a facet.
- Bots that DM or tag players found through data.
- Riot trademarks in names, handles or package names.

Report security problems to security@summonerdex.gg (planned), not in a public issue.

## Rules

Rules live in Git: the schema, rubrics (`rubrics/{facet}@{n}.json`), reason codes, the league allow-list, publisher pages, terms and `GOVERNANCE.md`. You change them by pull request.

- Stewards decide a rules PR by 72-hour lazy consensus: it stands if no Steward objects within 72 hours. A tie keeps the status quo. The outcome is written to the public decision log.
- A rubric change adds a new version instead of editing a released one. Old and new versions run side by side for 30 days.
- Rules CI runs the same validation module the Worker runs, so a rubric PR is tested against the code that applies it.

## Contributor Terms

The Contributor Terms 1.0 are not written yet. Once they are published, you accept them before you submit anything: by clickwrap on the web form, with `/accept-terms` in Discord, or through the Publisher Agreement. The version you accepted is stored with each submission. The form and the bot both ask you to accept them first, so community intake waits for the published terms.

In short: you give the project a non-exclusive, perpetual, irrevocable, sublicensable licence to your contribution, held by SummonerDex, acting through its maintainers, until a legal entity exists. You keep your own rights to it. You waive moral rights as far as you can, and you confirm you had a lawful basis to submit it. The project may remove any contribution at any time. The attribution lock means the data can only ever be relicensed under a licence that requires attribution, and only with the agreement of two-thirds of active contributors.

The data is published under CC BY 4.0. Anyone using it credits: `SummonerDex contributors, CC BY 4.0, https://summonerdex.gg/attribution`.

## Conduct

The project follows the Contributor Covenant 2.1. Report problems, including griefing and mass reporting, to abuse@summonerdex.gg (planned). The Stewards read it, and a Steward does not handle a report that involves their own product.

SummonerDex isn't endorsed by Riot Games and doesn't reflect the views or opinions of Riot Games or anyone officially involved in producing or managing Riot Games properties. Riot Games, and all associated properties are trademarks or registered trademarks of Riot Games, Inc.
