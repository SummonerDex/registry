# Security policy

## Reporting a vulnerability

Email security@summonerdex.gg (planned; the domain is not registered yet). Do not open a public GitHub issue, pull request or Discord message for a security report.

Please include:

- what is affected (component, endpoint or package, and version or commit if known)
- steps to reproduce
- what an attacker could do with it
- whether any persona data, keys or secrets were exposed while you tested

When testing, use your own accounts and keys, keep to the smallest amount of data needed to show the issue, and do not degrade the service for others.

## Response

The response time for security reports is to be confirmed. Fix timelines and disclosure dates are to be confirmed per report. Safe-harbour terms for researchers are to be confirmed.

For now, the two charter Stewards read security reports. Who reads them once a third Steward is seated is to be confirmed.

## In scope

- **The Worker**: the one TypeScript Worker that serves the API, the web app, the owner and admin consoles, the Discord interactions endpoint and the publish API.
- **The site**: the Astro site on Cloudflare Pages, including the submission form and owner console.
- **The publish API**: batch signing and verification, publish and read keys, receipts, and the compliance feed.
- **The idfold packages**: `@summonerdex/idfold` (npm) and `summonerdex-idfold` (PyPI), which produce the Riot ID join key.

Issues we especially want to hear about:

- any way to tell a hidden, delisted, erased, withheld or never-existed record apart from the others from outside
- any way to read unpublished claims, private contacts, submitter identities or suppression results
- any way to re-add a suppressed account, or bypass a hide
- forged or replayed batch signatures, key leakage, or admin access without an admin role
- join-key mismatches between the TypeScript and Python idfold packages

## Out of scope

- Riot Games, Twitch and Discord platforms, and other third-party services the project runs on. Report those to the platform.
- Concerns about persona data: a wrong account, an account you want hidden, or personal information in a record. Send those to privacy@summonerdex.gg (planned), not security@.
- Abuse of the registry or its intake: abuse@summonerdex.gg (planned).
- Code of Conduct reports: abuse@summonerdex.gg (planned), see `CODE_OF_CONDUCT.md`.
