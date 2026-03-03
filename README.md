# modrepo-builder-action

GitHub Action that builds a `modrepo.xml` from GitHub release assets (`.zip`) containing `About/About.xml`, then commits and force-pushes the result to a `modrepo` branch.

This is for Stationeers mod authors who want to release their mods on GitHub.

## Basic usage

Put the yaml code below into a workflow file in your github repository (`.github/workflows/build-modrepo.yml`).
After that, every time you create or update a GitHub release, the action will build and publish a `modrepo.xml` for your mod on the `modrepo` branch.
In order to install mods from your repository, you need to add this to the stationeers launchpad repository list:

```
https://github.com/<your_username>/<repo_name>/blob/modrepo/modrepo.xml
```

```yaml
name: Build modrepo

on:
  workflow_dispatch:
  release:
    types: [published, edited]

jobs:
  build-modrepo:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - name: Build and publish modrepo.xml
        uses: StationeersLaunchPad/modrepo-builder-action@v1
```

## Advanced Information

### What this action does

When the action runs, it:

1. Checks out the caller repository.
2. Switches to (or creates) the `modrepo` branch.
3. Scans GitHub releases for one or more repositories.
4. Downloads `.zip` assets and reads `About/About.xml`.
5. Builds `modrepo.xml`.
6. Updates `modrepo_cache.json` (for digest-based caching).
7. Commits and pushes changes to the `modrepo` branch.

### Requirements for mod zip assets

Each release asset must:

- Be a `.zip` file.
- Include `About/About.xml`.
- Include these required XML fields in `About/About.xml`:
  - `ModID`
  - `Version`
  - `Name`
  - `Author`

Optional metadata read from `About/About.xml`:

- `<Branch>`
- `<Tags><Tag>...</Tag></Tags>`
- `<DependsOn ModID="...">` or `<DependsOn WorkshopHandle="...">`

### Inputs

| Name             | Required | Default                | Description                                                                                                    |
| ---------------- | -------- | ---------------------- | -------------------------------------------------------------------------------------------------------------- |
| `debug-output`   | No       | `"false"`              | Verbose debug output. Set to `"true"` to enable.                                                               |
| `commit-message` | No       | `"Update modrepo.xml"` | Commit message used for updates in `modrepo` branch.                                                           |
| `mod-repos`      | No       | `""`                   | Whitespace/newline-separated list of `owner/repo` repositories to scan. If empty, scans only the current repo. |

### Output files (in `modrepo` branch)

- `modrepo.xml` – generated mod repository index.
- `modrepo_cache.json` – cache keyed by asset digest to speed up future runs.

### Usage with multiple repositories

```yaml
name: Build org modrepo

on:
  # run once per day or manually
  workflow_dispatch:
  schedule:
    - cron: "0 0 * * *"

jobs:
  build-modrepo:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - name: Build and publish modrepo.xml from multiple repos
        uses: StationeersLaunchPad/modrepo-builder-action@v1

        with:
          debug-output: "false"
          commit-message: "Update modrepo.xml (automated)"
          mod-repos: |
            tim_is_happy/FreakerMod
            jameson_the_desaster/RedSketches
```

### Notes about branch behavior

- This action **publishes to `modrepo` branch**.
- It uses `git push -f origin modrepo`, so the target branch is force-updated.
- Run this in a repository where force-updating `modrepo` is acceptable.

### Permissions

Your workflow job should include:

```yaml
permissions:
  contents: write
```

This is required to commit and push the generated files.

### How repository selection works

Selection priority in the script is:

1. `_MODREPO_BUILDER_REPOS` (provided by `inputs.mod-repos`) if non-empty.
2. Otherwise `GITHUB_REPOSITORY` (current repository).

So in normal action usage:

- If you provide `mod-repos`, those repos are scanned.
- If you leave it empty, only the current repo is scanned.

### Example `About/About.xml`

```xml
<ModMetadata>
  <ModID>Example.Mod</ModID>
  <Version>1.2.3</Version>
  <Name>Example Mod</Name>
  <Author>Example Author</Author>
  <Tags>
    <Tag>Utility</Tag>
  </Tags>
</ModMetadata>
```

## Troubleshooting

- **No `modrepo.xml` changes committed**  
  Means generated output is unchanged; action exits successfully.

- **Release skipped**  
  A release is skipped if it has no `.zip` assets.

- **Asset skipped**  
  A `.zip` is skipped if:
  - it has no digest metadata in the GitHub release API response, or
  - `About/About.xml` is missing, or
  - required fields (`ModID`, `Version`, `Name`, `Author`) are missing/invalid.

- **Push failed**  
  Ensure workflow has `contents: write` permission and branch protections allow this workflow to update `modrepo`.
