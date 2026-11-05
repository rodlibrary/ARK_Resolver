# Resolver

The ARK resolver data for Rod Library, University of Northern Iowa (NAAN `32498`). Apache reads [`resolve.csv`](resolve.csv) to send each Archival Resource Key (ARK) to its current location.

The files here are written by scripts in the GitHub repository [rodlibrary/Scripts](https://github.com/rodlibrary/Scripts/tree/main/arks). Don't edit them by hand unless you need to correct a row: the scripts merge with what's here, so a hand edit is kept, but a typo can stop an ARK resolving.

| File | What it holds | Written by |
|---|---|---|
| [`resolve.csv`](resolve.csv) | Every ARK and where it resolves | `generate_arks.py` adds new ARKs; `match_preservica.py` fills in Preservica references and targets |
| [`ark_master_list.csv`](ark_master_list.csv) | Every ARK ever minted, with when and from which file. It keeps new blades unique. | `generate_arks.py` (only ever added to) |
| [`match_report.csv`](match_report.csv) | The latest Preservica match: one row per asset whose MODS has ARK fields, with the outcome | `match_preservica.py` (weekly) |
| [`.gitlab-ci.yml`](.gitlab-ci.yml) | The weekly matching pipeline, and a check of every change to `resolve.csv` | – |

## resolve.csv

| Column | Meaning |
|---|---|
| `ark` | The full ARK 
| `p_path` | The full URL where the ARK resolves to |


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
3. **Matching.** Every Monday this repository's [pipeline](#weekly-pipeline) runs `match_preservica.py`. It reads those fields from every Preservica asset and pushes `resolve.csv` back here when it changed:
   - each ARK it finds gets the asset's `preservica_ref`;
   - an ARK with no `target_url` is pointed at the asset's page in Preservica's portal (`https://uni.access.preservica.com/portal/en-US/asset/sdb%3AIO%7C<reference>`) and becomes `active`.

   It never changes an existing different reference or target, and leaves out ARKs found in more than one asset. It lists those in `match_report.csv`. Staff can also run it by hand, or add `--match-preservica` to the minting command.

Full instructions: [rodlibrary/Scripts/arks/README.md](https://github.com/rodlibrary/Scripts/blob/main/arks/README.md).

## Weekly pipeline

[`.gitlab-ci.yml`](.gitlab-ci.yml) has three jobs:

| Job | When | What it does |
|---|---|---|
| `match-preservica` | The pipeline schedule (Mondays), or **Run pipeline** | Downloads the ARK scripts from [rodlibrary/Scripts](https://github.com/rodlibrary/Scripts/tree/main/arks) and runs `match_preservica.py --from-gitlab --push-to-gitlab --push-report`. The report is also kept as a job artifact for 30 days. |
| `check-resolve-csv` | Every change to `resolve.csv` | Fails, and GitLab emails you, if the header is wrong, an ARK is listed twice, a status isn't `active`, `withdrawn` or `unmatched`, or an `active` row has no `target_url` |
| `check-gitlab` | By hand, after **Run pipeline** | Runs `check_gitlab.py` from the runner: the token, push permission, and a test push to a temporary branch |

Commits that the scripts make to `resolve.csv` run only `check-resolve-csv`, so a push never starts another push.

**Setup (one time):**

1. **Settings > CI/CD > Variables > Add variable**, each with **Masked** (and **Protected**) ticked:
   - `PRESERVICA_USERNAME`: the Preservica service account;
   - `PRESERVICA_PASSWORD`: its password;
   - `GITLAB_TOKEN`: the project access token.
2. **Build > Pipeline schedules > New schedule**:
   - description: *Match ARKs in Preservica*;
   - interval: custom, `23 7 * * 1` (Mondays 7:23 am);
   - cron timezone: *Central Time (US & Canada)*;
   - target branch: `main`;
   - save.
3. **Test:** use **Build > Pipelines > Run pipeline** with the variable `DRY_RUN` = `true`. Open the `match-preservica` job for the counts, and click the play button on `check-gitlab` to test the token from the runner.

Optional variables:

| Variable | Effect |
|---|---|
| `DRY_RUN` = `true` | Report only; `resolve.csv` isn't changed |
| `OVERWRITE` = `true` | Replace references and targets that differ from Preservica |
| `SCRIPTS_REF` | Run a particular branch, tag or commit of rodlibrary/Scripts (default `main`) |
| `PRESERVICA_PORTAL_URL` | Where matched ARKs are sent (default: the asset's portal page) |

The jobs need a runner with internet access, to download the scripts from GitHub and reach Preservica. If a pipeline stays **pending**, the project has no runner: ask IT, or see **Settings > CI/CD > Runners**.

## Access token

The scripts push with a project access token: **Settings > Access tokens**, **Maintainer** role, **api** scope. `main` is protected, and only Maintainers can push to a protected branch unless **Settings > Repository > Protected branches** allows Developers. `check_gitlab.py` in the Scripts repository tests the token, including a test push to a temporary branch. It's stored as the `GITLAB_TOKEN` secret in the GitHub repository rodlibrary/Scripts, and staff set it in their terminal when minting. When it expires, create a new one and update both.