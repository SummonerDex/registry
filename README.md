# SummonerDex

SummonerDex is an open, community-run registry of League of Legends personas. A persona is a pro player, streamer or notable creator. The registry records who a persona is and which in-game accounts, names, socials and esports affiliations belong to them. Every platform can read this one shared source instead of keeping its own list. Anyone can contribute, and anyone can reuse the data with attribution.

The registry answers who, never how well. It holds identity, the accounts tied to a persona, name history, socials, esports teams and roles, aliases, and facets. A facet is a label for what kind of persona someone is, such as pro or streamer. The registry holds no Riot data except the accounts themselves and esports affiliations, and nothing product-related.

## What it is not

- No rank, stats or match history.
- No live status or live-game data.
- No product features. The registry builds nothing for any one consumer. Every platform adapts to the public schema.

## Licensing

The data is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Credit it with this string:

```
SummonerDex contributors, CC BY 4.0, https://summonerdex.gg/attribution
```

See [DATA-LICENSE.md](DATA-LICENSE.md) and [ATTRIBUTION.md](ATTRIBUTION.md). Keys and dumps will also come under Access Terms, which are not written yet.

The code is licensed under the [Apache License 2.0](LICENSE).

## Status

The design is complete and nothing is built yet. The architecture is in [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md). The summonerdex.gg domain and the GitHub org are planned but not registered yet. A trademark search for the word "summoner" still has to be done before the project is announced.

## Getting involved

Read [CONTRIBUTING.md](CONTRIBUTING.md) to learn how to submit and review. Read [GOVERNANCE.md](GOVERNANCE.md) for roles and how decisions are made.

Planned contacts, all at summonerdex.gg once the domain is registered:

- privacy@ to hide or remove data about you
- abuse@ to report misuse
- publishers@ for publisher keys
- security@ for security issues

## Repo map

- `docs/ARCHITECTURE.md` how the registry works
- `schema/` the public persona schema and an example persona file
- `.github/` issue templates, the pull request template and code owners
- `CONTRIBUTING.md`, `GOVERNANCE.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md` how to take part
- `LICENSE`, `DATA-LICENSE.md`, `ATTRIBUTION.md` licences for code and data
- `CLAUDE.md` working notes for coding agents

## Contributors

TrackDIFF and LouisGameDev are helping contribute.

## Riot Games

SummonerDex isn't endorsed by Riot Games and doesn't reflect the views or opinions of Riot Games or anyone officially involved in producing or managing Riot Games properties. Riot Games, and all associated properties are trademarks or registered trademarks of Riot Games, Inc.

Riot Games does not endorse or sponsor this project.

Riot's "Legal Jibber Jabber" policy (section 6) asks for a two-sentence notice. Its first sentence says the project was made under that policy using assets owned by Riot Games. SummonerDex plans to use no Riot assets, so whether that first sentence belongs here is still to be confirmed.
