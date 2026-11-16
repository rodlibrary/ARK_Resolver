# Resolver

The ARK resolver data for Rod Library, University of Northern Iowa (NAAN `32498`). Apache reads [`resolve.csv`](resolve.csv) to send each Archival Resource Key (ARK) to its current location.

It works like the ScholarWorks redirects in [rodlibrary/Redirects](https://github.com/rodlibrary/Redirects): one CSV with two columns of full addresses, checked by the same rules. There, `redirects.csv` sends old ScholarWorks addresses (`dc_path`) to Preservica (`p_path`). Here, `resolve.csv` sends ARKs (`ark`) to Preservica (`p_path`).

The files here are written by the scripts in [rodlibrary/Scripts](https://github.com/rodlibrary/Scripts/tree/main/arks). Edit them by hand only to correct a row: the scripts merge with what's here, so a hand edit is kept, but a typo can stop an ARK resolving, so run the check (below) afterwards.

| File | What it holds | Written by |
|---|---|---|
| [`resolve.csv`](resolve.csv) | Every ARK that has somewhere to go, and where | `generate_arks.py` adds ARKs minted with a url; `match_preservica.py` adds ARKs once they're in an asset's MODS in Preservica |
| [`ark_master_list.csv`](ark_master_list.csv) | Every ARK ever minted, with when and from which file. It keeps new blades unique. | `generate_arks.py` (only ever added to) |
| `match_report.csv` | The latest Preservica match: one row per asset whose MODS has ARK fields, with the outcome | `match_preservica.py` (weekly) |
| [`.github/workflows/check.yml`](.github/workflows/check.yml) | Checks every change to `resolve.csv` | – |

## resolve.csv

| Column | What goes in it | Example |
|---|---|---|
| `ark` | the ARK as a full address on library.uni.edu | `https://library.uni.edu/ark:/32498/xo2gohohwd` |
| `p_path` | where it resolves to, in full, or `410` if the item was withdrawn | `https://uni.access.preservica.com/portal/en-US/asset/sdb%3AIO%7C9dbdd64c-5ab8-4344-ae48-99fc70625610` |

Rules (the same as for `redirects.csv` in rodlibrary/Redirects):

* `ark` must be a full `https://library.uni.edu/ark:/32498/...` address, listed once. Hyphens and upper/lower case don't make a different ARK.
* `p_path` must be a full `https://` address on `uni.access.preservica.com` or `digital.library.uni.edu`, or `410`. Copy it from the portal as it is: `sdb%3AIO%7C` must stay encoded (a raw `|` is rejected).
* Each row on its own line, and the file ends with a line break. Editing in Excel: save as **CSV UTF-8**.

A minted ARK that isn't in Preservica yet is in `ark_master_list.csv` only. It's added to `resolve.csv` when `match_preservica.py` finds it.

## How it's updated

1. **Minting.** Staff fill in `arks.csv` and run:

   ```bash
   export GITHUB_TOKEN=github_pat_...
   python3 generate_arks.py output --push-to-github
   ```

   The script pulls `resolve.csv` and `ark_master_list.csv` from here and mints the new ARKs. It records every new ARK in the master list, adds those with a url to `resolve.csv`, and pushes both files back.
2. **Cataloguing.** The ARK goes into the asset's MODS in Preservica:
   - the blade in **Archival Resource Key** (`<identifier displayLabel="Archival Resource Key" type="ark">`);
   - the full address in **Location (URL)** (`<physicalLocation displayLabel="Location (URL)" type="url"><url>`).
3. **Matching.** Every Monday a [GitHub Actions workflow in rodlibrary/Scripts](https://github.com/rodlibrary/Scripts/blob/main/.github/workflows/arks-match-preservica.yml) runs `match_preservica.py`. It reads those fields from every Preservica asset, adds each minted ARK it finds to `resolve.csv` pointing at the asset's page in Preservica's portal (`https://uni.access.preservica.com/portal/en-US/asset/sdb%3AIO%7C<reference>`), and pushes `resolve.csv` back here when it changed.

   It never changes a row that's already here, and never adds an ARK found in more than one asset. It lists those in `match_report.csv`. Staff can also run it by hand, or add `--match-preservica` to the minting command.

The scripts push directly to `main` with a fine-grained token that has **Contents: Read and write** on this repository. They're stored as the `RESOLVER_TOKEN` secret in rodlibrary/Scripts, and staff set them in their terminal when minting. If `main` gets a rule requiring pull requests, the token's account must be allowed to bypass it.

Full instructions: [rodlibrary/Scripts/arks/README.md](https://github.com/rodlibrary/Scripts/blob/main/arks/README.md).

## Checks

[`.github/workflows/check.yml`](.github/workflows/check.yml) runs `check_resolve.py` from rodlibrary/Scripts on every push and pull request that changes `resolve.csv` or `ark_master_list.csv`. It fails, and GitHub emails whoever made the change, if a row breaks the rules above. The run summary also lists any minted ARKs still waiting for Preservica.

To check by hand, with the Scripts repository's `arks` folder:

```bash
python3 check_resolve.py --from-github
```

## History

Until November 2026 this repository lived in GitLab (`git.uni.edu/rod-library/resolver`), with its weekly pipeline there. `resolve.csv` then had seven columns (`dc_path`, `dc_article`, `ark`, `preservica_ref`, `target_url`, `status`, `notes`). Old ScholarWorks addresses now live in [rodlibrary/Redirects](https://github.com/rodlibrary/Redirects).
