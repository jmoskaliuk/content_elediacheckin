# content_elediacheckin

Demo content repository for the Moodle plugin [`mod_elediacheckin`](https://github.com/eledia-gmbh/moodle-mod_elediacheckin).

This repository holds the didactic check-in / check-out questions that the plugin pulls at runtime via its **Git content source**. It is deliberately kept in a separate repository so content editors (didactic team) can maintain questions without touching plugin code.

## Layout

```
bundle.json          # The live question bundle — this is what the plugin fetches
schema.json          # JSON Schema (Draft 2020-12) the bundle must conform to
.github/workflows/   # CI that validates bundle.json against schema.json on every push
README.md            # You are here
```

## How the plugin consumes this repo

In Moodle, go to *Site administration → Plugins → Activity modules → Check-in* and set:

- **Active content source** → `Custom git repository`
- **Repository URL** → raw URL of `bundle.json`, e.g.
  `https://raw.githubusercontent.com/eledia-gmbh/content_elediacheckin/main/bundle.json`
- **Access token** → optional, only needed for private repos

Then open the *Sync log* admin report and click **Run sync now**. The plugin will:

1. `curl` the URL
2. Validate the response against the embedded schema
3. Write the questions into a staging table
4. Atomically swap staging → live (failure leaves live data intact)
5. Purge caches

## Editing content

Each entry in `bundle.json → questions[]` requires:

| Field          | Type       | Notes                                                    |
|----------------|------------|----------------------------------------------------------|
| `id`           | string     | Unique, stable. Pattern: `[A-Za-z0-9_.-]+`               |
| `ziel`         | enum       | One of `impuls, checkin, checkout, retro, learning, funfact, zitat` |
| `kategorie`    | string[]   | Category external-ids; allowed values depend on `ziel`   |
| `frage`        | string     | The question text shown to learners                      |
| `hat_antwort`  | boolean    | If `true`, `antwort` must be provided                    |
| `antwort`      | string     | Shown behind *"Show answer"*; required iff `hat_antwort` |
| `sprache`      | string     | ISO-639-1 two-letter language code                       |
| `lizenz`       | string     | e.g. `CC-BY-4.0`, `proprietary`                          |
| `version`      | string     | Freeform, recommend semver (`1.0.0`)                     |
| `status`       | enum       | `draft`, `published`, `deprecated`                       |
| `created_at`   | date-time  | ISO-8601                                                 |
| `updated_at`   | date-time  | ISO-8601                                                 |
| `autor`        | string?    | Optional credit line                                     |
| `quelle`       | string?    | Optional source / reference                              |
| `link`         | url?       | Optional further-reading link                            |
| `media`        | string?    | Optional media reference (URL or relative path)          |

Bump `bundle_version` at the top whenever you release a new version; the plugin
records it in the sync log so admins can see which version is live.

## Local validation

```bash
pip install jsonschema
python -c "
import json
from jsonschema import Draft202012Validator
schema = json.load(open('schema.json'))
bundle = json.load(open('bundle.json'))
errors = sorted(Draft202012Validator(schema).iter_errors(bundle), key=lambda e: e.path)
if errors:
    for e in errors: print('-', list(e.path), e.message)
    raise SystemExit(1)
print(f'OK — {len(bundle[\"questions\"])} questions validate.')
"
```

CI runs the equivalent check on every push to `main` and on pull requests.

## License

Questions are released under the license stated in each question's `lizenz` field. The repository scaffolding (README, CI, schema) is licensed under GPL v3 or later, matching the parent plugin.
