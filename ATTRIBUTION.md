# Attribution

SummonerDex data is CC BY 4.0. If you use it, credit it with this exact string:

```
SummonerDex contributors, CC BY 4.0, https://summonerdex.gg/attribution
```

The same string is in every JSON body, in a `Link: rel="license"` header on API responses, in every dump manifest and in the bot's `/about`. The summonerdex.gg domain is planned and not registered yet.

## Where to put it

Current recommendation (still an open question): product-level credit. One credits line in your product is enough. You do not need to repeat it on every profile page or every place a persona appears.

Where the string sits as text, link the URL if your medium allows it.

## Examples

### Website footer

```html
<footer>
  Persona data: <a href="https://summonerdex.gg/attribution">SummonerDex contributors, CC BY 4.0</a>
</footer>
```

A credits or about page works too.

### API consumer

Every response carries an `attribution` object. Keep it when you store or pass the data on, and show `attribution.text` wherever your product credits its data sources. If you republish the data, for example in your own API or export, include the string alongside it.

```json
"attribution": {
  "text": "SummonerDex contributors, CC BY 4.0, https://summonerdex.gg/attribution",
  "license": "CC-BY-4.0",
  "terms": "https://summonerdex.gg/terms/access-1.0"
}
```

### Discord bot

Put the string in the bot's about or help command, or in its profile description. Per the current recommendation you do not need to add it to every embed.

```
/about
Persona data: SummonerDex contributors, CC BY 4.0, https://summonerdex.gg/attribution
```

## Keys and dumps

If you use an API key or download a dump, the Access Terms (to be written) also apply. They make attribution part of the key terms. See [DATA-LICENSE.md](DATA-LICENSE.md).
