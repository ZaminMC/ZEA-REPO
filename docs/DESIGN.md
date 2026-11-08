# ZEA-REPO Design Assessment

Engineering report preceding implementation: what NEU-REPO does, what is
worth adapting, what ZEA-REPO does differently, and the resulting structure.

## 1. NEU-REPO as it exists

Analyzed at master (`80ad2ca`, ~18,200 tracked files):

```
NotEnoughUpdates-REPO/
├── items/            8,958 JSON  — flat, one file per entity, filename = internal ID
├── itemsOverlay/     8,868 SNBT  — per-MC-data-version item component overrides
├── mobs/               294 files — JSON descriptors + PNG skins
├── constants/           35 JSON  — shared topical data (leveling, enchants, ...)
├── .github/             10 files — 4 workflows + Python automation bots
└── README.md, LICENSE, .gitattributes, .vscode/ (schema mappings)
```

Key mechanics:

- **Identity = filename.** `items/ASPECT_OF_THE_END.json` carries
  `internalname`; CI enforces filename == `internalname`.
- **References are ID strings**, never paths: recipe grids use
  `"ITEM_ID:COUNT"`, resolved by the mod to `items/<ID>.json`.
- **Consumption:** the mod queries the GitHub API for the branch's latest
  commit sha, compares against a locally stored sha, and if changed
  downloads the **commit-pinned zip archive**, deletes the old copy, and
  extracts. A backup zip is bundled inside the mod jar.
- **CI:** a separate workflow repository validates **only changed files**
  (JSON parseability, filename/ID match, referential integrity of recipes,
  constants coherence). Bots open PRs for API/wiki-derived updates — never
  direct pushes.
- **Contribution:** unusual — contributors edit data in-game with the mod's
  dev tools, then open normal GitHub PRs (~3,250 to date).

## 2. What is good (worth adapting)

1. **One file per entity** — clean diffs, trivially reviewable PRs,
   conflict-resistant, granular downloads.
2. **Filename-as-ID enforced by CI** — the client can locate any entity by
   name alone.
3. **Automated referential-integrity checks** — broken references never
   reach users.
4. **Commit-pinned immutable downloads** — the client never sees a torn
   state.
5. **Changed-file-only CI** — keeps validation cheap at 18k files.
6. **Bots work through PRs** — generated data follows the same review path
   as human data.

## 3. What must not be copied

1. **Redundancy.** Display text is stored 2–3× per item (filename,
   `displayname`, lore inside `nbttag`) and every item exists twice (JSON +
   SNBT overlay). ZEA-REPO stores each fact exactly once.
2. **A flat 9k-file directory.** Slow to clone and browse. ZEA-REPO splits
   by dataset (and stays small because it only holds curated data).
3. **Loose typing.** `"16"` vs `16`, percent-strings, mixed casing. ZEA-REPO
   enforces JSON Schema.
4. **No manifest.** NEU clients re-download the entire zip for any change.
   ZEA clients must be able to fetch only what changed.
5. **Schemas on an external site**, covering 3 of ~20 real shapes. ZEA-REPO
   keeps schemas in-repo, covering 100% of files.
6. **Mixed entity kinds in one namespace** (items/NPCs/monsters in
   `items/`). ZEA-REPO separates datasets per entity type.

## 4. The fundamental difference: what belongs here

NEU-REPO exists because Hypixel has no public API — the repo **is** the
database. EarthMC has a live official API (v4), so ZEA-REPO inverts the
model:

- **Volatile live data** (residents, balances, coordinates, online status,
  relationships) is **never stored here** — ZEA fetches it from the API at
  runtime.
- **ZEA-REPO stores only what the API cannot serve**: community-verified
  constants (badge meanings, channel definitions), curated descriptions and
  links for nations/towns, and (later) historical records.

This keeps the repo small, reviewable, and slow-changing — and avoids
duplicating a source of truth we do not own.

**Players are not stored** (privacy + churn + full API coverage). Community
"notable players" data can be a future dataset if a feature ever needs it.

## 5. ZEA-REPO structure

```
ZEA-REPO/
├── README.md                     what this is, structure, consumption model
├── LICENSE                       MIT
├── CONTRIBUTING.md               how to add data, rules, validation
├── manifest.json                 CI-generated index (do not edit by hand)
├── docs/DESIGN.md                this document
├── schemas/                      JSON Schema (draft 2020-12), one per shape
│   ├── manifest.schema.json
│   ├── badges.schema.json
│   ├── channels.schema.json
│   ├── nation.schema.json
│   └── town.schema.json
├── datasets/
│   ├── badges.json               badge symbol → verified meaning
│   ├── channels.json             writable server channels + read-only feeds
│   ├── nations/<slug>.json       curated nation entries (empty initially)
│   └── towns/<slug>.json         curated town entries (empty initially)
├── scripts/validate.py           local + CI validation, manifest generator
└── .github/workflows/validate.yml
```

Decisions behind the seven evaluated items:

| Item | Decision | Why |
|---|---|---|
| Root manifest | **Yes, CI-generated** | enables delta downloads; humans never hand-maintain hashes |
| JSON Schema files | **Yes, in-repo** | CI validation + editor autocomplete via `.vscode/` mapping |
| Stable identifiers | **Yes: slug filename + `uuid` field** | uuid joins to API data; slug is human-facing |
| Naming convention | **Yes: kebab-case slugs, camelCase keys** | documented in CONTRIBUTING |
| Dataset versioning | **Yes: content-hash manifest + `version` field per file** | no manual version bumps, explicit schema version |
| Contribution guide | **Yes** | PR-based, schema-validated workflow |
| Automated validation | **Yes: GitHub Action** | all files, every PR and push; manifest regenerates on main |

Future datasets (map/landmark data, history, notable players) slot into
`datasets/<name>/` with a new schema + manifest entry — existing consumers
ignore unknown datasets by design.

## 6. Example documents

`datasets/nations/yugoslavia.json` (illustrative shape; real entries are
added by contributors with verified data):

```json
{
  "version": 1,
  "name": "Yugoslavia",
  "uuid": "00000000-0000-0000-0000-000000000000",
  "description": "Curated, community-written description.",
  "tags": ["historical"],
  "links": {
    "website": null,
    "discord": null
  },
  "lastVerified": "2026-11-08"
}
```

`datasets/badges.json` (seeded — symbols and meanings are verified
observations from EarthMC chat):

```json
{
  "format": "zea-repo/badges",
  "version": 1,
  "badges": [
    {
      "symbol": "🌎",
      "id": "anniversary-globe",
      "name": "EarthMC 10th anniversary Globe",
      "verified": true,
      "source": "observed in global chat, 2026-11"
    }
  ]
}
```

`manifest.json` (generated):

```json
{
  "format": "zea-repo/manifest",
  "version": 1,
  "generated": "2026-11-08T12:00:00Z",
  "datasets": {
    "badges": {
      "files": {
        "datasets/badges.json": "<sha256>"
      }
    },
    "nations": {
      "files": {
        "datasets/nations/yugoslavia.json": "<sha256>"
      }
    }
  }
}
```

## 7. Update and versioning strategy

- **Client update flow:** fetch `manifest.json` (raw URL, HTTP caching),
  compare per-file sha256 against the local cache, download only changed
  files, delete files that vanished. Initial sync = manifest + all files,
  or the branch zip archive as a bulk alternative.
- **No manual version numbers.** The manifest's content hashes are the
  version; `generated` is informational.
- **Schema evolution:** each dataset file carries `version`; additive
  changes bump it, consumers ignore unknown fields, and incompatible
  changes require a new dataset rather than a silent reinterpretation.

## 8. Contribution and validation workflow

1. Contributor edits/creates files under `datasets/` (never `manifest.json`).
2. PR opens; CI validates **every** file: JSON parseability, schema
   conformance, dataset-location rules (no stray paths), slug/uuid format.
3. On merge to main, CI regenerates `manifest.json` and commits it if
   anything changed (bot commit; contributors never touch it).
4. Local validation: `pip install jsonschema && python3 scripts/validate.py`.

Data rules (enforced socially, checked by schema where possible): no
invented information, no personal/private data, no credentials, `uuid`
values must come from the official API, curated text must be neutral.
