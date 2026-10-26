<!-- This repository holds no persona data. Pull requests that add personas, accounts or claims are closed. Claims enter through the submission form and the Discord bot once intake opens at launch. Until then, only the Stewards' seed data enters. -->

## What and why

<!-- One change. Say why it exists. -->

## Type

- [ ] Code (Apache-2.0)
- [ ] Rules: schema, rubric, reason codes, allow-list, terms or policy (Stewards decide by 72-hour lazy consensus; a tie keeps the status quo)
- [ ] Publisher page (`publishers/{pub_id}.json`)
- [ ] Docs

## Tests

<!-- Which tests you ran, or say none. -->

## Checklist

- [ ] Commits carry a DCO `Signed-off-by:` line (`git commit -s`). Proposed, not yet a merge requirement.
- [ ] No persona data, Riot IDs of real people, PUUIDs, keys or secrets.
- [ ] Nothing stores or serves ranks, LP, match data, live status, legal names, birth dates or addresses.
- [ ] Any new output path goes through the single publication check, and non-public states return identical bytes.
- [ ] Nothing here is built for one consumer.
- [ ] Conflicts declared (a product or team you work for that this change affects):
