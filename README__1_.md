# Resolver

The ARK resolver data for Rod Library, University of Northern Iowa (NAAN `32498`). Apache reads [`resolve.csv`](resolve.csv) to send each Archival Resource Key (ARK) to its current location.

The files here are written by scripts in the GitHub repository [rodlibrary/Scripts](https://github.com/rodlibrary/Scripts/tree/main/arks). Don't edit them by hand unless you need to correct a row: the scripts merge with what's here, so a hand edit is kept, but a typo can stop an ARK resolving.

| File | What it holds | Written by |
|---|---|---|
| [`resolve.csv`](resolve.csv) | Every ARK and where it resolves | `generate_arks.py` adds new ARKs; `match_preservica.py` fills in Preservica references and targets |
| [`ark_master_list.csv`](ark_master_list.csv) | Every ARK ever minted, with when and from which file. It keeps new blades unique. | `generate_arks.py` (only ever added to) |
| [`match_report.csv`](match_report.csv) | The latest Preservica match: one row per asset whose MODS has ARK fields, with the outcome | `match_preservica.py` (weekly) |

## resolve.csv

| Column | Meaning |
|---|---|
| `dc_path` | Digital Commons path, if any: a landing-page key (`etd/123`) or a collection key (`etd`) |
| `dc_article` | Digital Commons file key (`etd/1124`), if any |
| `ark` | The ARK without `ark:/`, e.g. `32498/s3x7k9q2mbd` |
| `preservica_ref` | The Preservica asset holding the ARK (`IO_<reference>`) |
| `target_url` | Where the ARK resolves to |
| `status` | `active`: resolve to `target_url`. `withdrawn`: served as HTTP 410 Gone. `unmatched`: not resolved yet. |
| `notes` | Problems reported when minting, and when each ARK was matched in Preservica |

`resolve.csv` replaces `redirects.csv` from [rod-library/redirects](https://git.uni.edu/rod-library/redirects). It has the same columns.

## How it's updated

1. **Minting.** Staff fill in `arks.csv` and run:

   ```bash
   export GITLAB_TOKEN=glpat-...
   python3 generate_arks.py output --push-to-gitlab
   ```

   The script pulls `resolve.csv` and `ark_master_list.csv` from here, mints the new ARKs, adds them to both files and pushes them back.
2. **Cataloguing.** The ARK goes into the asset's MODS in Preservica:
   - the blade in **Archival Resource Key** (`<identifier displayLabel="Archival Resource Key" type="ark">`);
   - the full address in **Location (URL)** (`<physicalLocation displayLabel="Location (URL)" type="url"><url>`).
3. **Matching.** Every Monday a GitHub workflow runs `match_preservica.py`. It reads those fields from every Preservica asset and pushes `resolve.csv` back here when it changed:
   - each ARK it finds gets the asset's `preservica_ref`;
   - an ARK with no `target_url` is pointed at the asset's page in Preservica's portal (`https://uni.access.preservica.com/uncategorized/IO_<reference>/`) and becomes `active`.

   It never changes an existing different reference or target, and leaves out ARKs found in more than one asset. It lists those in `match_report.csv`. Staff can also run it by hand, or add `--match-preservica` to the minting command.

Full instructions: [rodlibrary/Scripts/arks/README.md](https://github.com/rodlibrary/Scripts/blob/main/arks/README.md).

## Access token

The scripts push with a project access token: **Settings > Access tokens**, **Developer** role, **api** scope. It's stored as the `GITLAB_TOKEN` secret in the GitHub repository rodlibrary/Scripts, and staff set it in their terminal when minting. When it expires, create a new one and update both.
