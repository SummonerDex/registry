# SummonerDex

SummonerDex is an open registry of League of Legends personas. A persona is a pro player, a streamer, or another well-known player. The registry records who a persona is and which game accounts, names, social links, and esports teams belong to them.

Apps and bots read the registry instead of each keeping their own list.

## What it holds

- Identity: a stable id, a display name, and known aliases.
- Accounts: every Riot ID the persona has used, with the dates it was in use.
- Social links: Twitch, YouTube, X, Kick, and similar.
- Esports: teams, leagues, and roles, with dates.
- Facets: short labels that say what kind of persona this is, such as pro or streamer.

## What it does not hold

- No rank, stats, match history, or live game data.
- No features for any one app or bot. Every app reads the same data and adapts to it.
- No private data. Only what a person has shown in public, or what they add themselves.

A person can claim their own entry and hide any part of it. Hiding is done on request.

## How data gets in

- Anyone can submit a claim through the web form or the Discord bot. A claim states one fact and links to evidence.
- Platforms with an API key send claims in bulk.
- A persona can verify their own entry, then add or hide accounts.
- Reviewers approve claims. Submitters with a good record get simple claims approved automatically.

Every claim records who sent it, when, and what evidence it had. Nothing is edited in place. A new claim replaces an old one.

## How data gets out

- One JSON file per persona, served from a CDN. No key needed.
- A free API key for search, lookups by Riot ID, and change feeds. Keys have rate limits.
- Full data dumps for download.
- A removal feed. Apps that use the data must apply removals within one day.

## Ids

- Each persona has a stable id. The URL slug is only for display.
- Each account has a stable id. Riot IDs change, so the registry keeps the history and joins on the id.
- PUUIDs are never shared. They are different for every Riot API key.

## Licence

- Data: CC BY 4.0. Credit it as "SummonerDex contributors, CC BY 4.0, https://summonerdex.gg/attribution".
- Code: Apache-2.0. See LICENSE.

SummonerDex is not endorsed by Riot Games.

## Stack

- One Cloudflare Worker in TypeScript: the API, the forms, and the Discord bot.
- D1 (SQLite) for storage. R2 for persona JSON files and dumps.
- Astro on Cloudflare Pages for the site.
- The packages `@summonerdex/idfold` (npm) and `summonerdex-idfold` (PyPI) normalize Riot IDs the same way everywhere.

The persona record is defined in `schema/persona.schema.json`.

## How to write here

- Simple English. Short sentences. Common words. People from many countries read this.
- Small functions with clear names. No clever tricks.
- Comments explain why, not what. No notes about how the code was written or which tools were used.
- One change per pull request.

## Contributors

TrackDIFF and LouisGameDev are helping contribute.
