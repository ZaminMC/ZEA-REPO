# Contributing to ZEA-REPO

Thank you for helping maintain EarthMC data for
[ZEA](https://github.com/ZaminMC/ZEA). This repository is validated by
automation, so most mistakes are caught before review — but please read the
rules below first.

## What belongs here

ZEA-REPO stores only **curated, stable, community-verified data**:

- Badge symbol meanings (`datasets/badges.json`)
- Server chat channel definitions (`datasets/channels.json`)
- Curated nation and town descriptions, tags and links
  (`datasets/nations/`, `datasets/towns/`)

Anything the official EarthMC API serves at runtime — residents, balances,
coordinates, online status, relationships — is **fetched live by the mod and
never stored here**. If you are unsure whether data belongs here, it
probably does not.

**Never submit:** invented information, personal/private player data,
credentials, API keys, or data you cannot verify in-game.

## File rules

- File names are **kebab-case slugs** of the exact entity name:
  `Yugoslavia` → `datasets/nations/yugoslavia.json`.
  Lowercase, digits, hyphens only.
- JSON keys are camelCase. UTF-8 encoding. Two-space indent.
- `uuid` values must be copied from the official API
  (`https://api.earthmc.net/v4/nations` or `/v4/towns` list endpoints).
- `messageTag` values in `datasets/channels.json` must be copied verbatim
  from observed in-game chat (case included); the town-bearing nation form
  is expressed with `messageTagCarriesTown`, never by editing the tag
  string.
- Keep descriptions neutral and under 1000 characters.
- Set `lastVerified` to the date you actually checked the data.
- Do **not** edit `manifest.json` — CI regenerates it when your PR merges.

## Validating locally

```
pip install jsonschema
python3 scripts/validate.py
```

CI runs the same command on every pull request and rejects invalid data.
Editors pick up the schemas automatically via `.vscode/settings.json`.

## Workflow

1. Fork / branch, edit files under `datasets/`.
2. Run the validator locally.
3. Open a pull request describing what you verified and where
   (in-game observation, official API, nation/town permission).
4. After merge, CI updates `manifest.json` automatically.

## Schema evolution

Every file carries a `version`. Additive changes (new optional fields) bump
it; consumers ignore unknown fields. Incompatible changes are made by
introducing a new dataset, not by reinterpreting an existing one — open an
issue first if you think that is needed.

## License

By contributing you agree your contributions are licensed under the
repository's [MIT license](LICENSE).
