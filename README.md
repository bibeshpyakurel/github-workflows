# github-workflows

Reusable GitHub Actions workflows shared across [@bibeshpyakurel](https://github.com/bibeshpyakurel)'s
projects. One definition lives here; each project calls it with a short caller
workflow instead of carrying its own copy.

This repository is **public**, so any repository — public or private — can call
these workflows. A private caller additionally needs
*Settings → Actions → General → Access* to permit "Accessible from repositories
owned by the user", because a private repository does not share its Actions
scope with other repositories by default.

## What lives here

| Workflow | Purpose |
| --- | --- |
| [`.github/workflows/codeql-reusable.yml`](.github/workflows/codeql-reusable.yml) | CodeQL static analysis, one matrix job per language |
| [`.github/workflows/commits-reusable.yml`](.github/workflows/commits-reusable.yml) | Conventional Commits validation for the commits in a pull request |

**Triggers stay in the caller.** A reusable workflow cannot declare `push`,
`pull_request` or `schedule` for its callers, and that is deliberate here: each
project keeps its own branches and its own weekly CodeQL cadence visible in its
own workflow file. Only the *body* of the job is shared.

---

## `codeql-reusable.yml`

Runs `github/codeql-action` `init` → `autobuild` (optional) → `analyze`, one
matrix job per language, with `fail-fast: false` so one language failing does
not hide the others. Actions are pinned to full commit SHAs.

Permissions requested: `contents: read`, `security-events: write`,
`actions: read`.

### Inputs

| Input | Type | Default | Description |
| --- | --- | --- | --- |
| `languages` | string | `'["javascript-typescript"]'` | JSON array of CodeQL languages. One matrix job per entry. |
| `queries` | string | `security-extended` | Query suite passed to `codeql-action/init`. |
| `autobuild` | boolean | `true` | Run `codeql-action/autobuild`. Keep `true` for compiled languages; set `false` for source-analysed ones (`javascript-typescript`, `python`, `ruby`) where Autobuild has nothing to compile and only costs minutes. |
| `runs-on` | string | `ubuntu-latest` | Runner label. |
| `timeout-minutes` | number | `30` | Job timeout. |

`languages` is a string because `workflow_call` inputs cannot be arrays; it is
expanded with `fromJSON` into the matrix.

### Calling it

```yaml
name: CodeQL

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: "27 4 * * 1"

permissions:
  contents: read
  security-events: write
  actions: read

concurrency:
  group: codeql-${{ github.ref }}
  cancel-in-progress: true

jobs:
  codeql:
    uses: bibeshpyakurel/github-workflows/.github/workflows/codeql-reusable.yml@main
    permissions:
      contents: read
      security-events: write
      actions: read
    with:
      languages: '["python"]'
```

Two languages, one compiled:

```yaml
    with:
      languages: '["csharp","javascript-typescript"]'
      autobuild: true
```

---

## `commits-reusable.yml`

Checks out the **calling** repository at full depth and runs its Conventional
Commits validator over the pull request's `base..head` range:

```
bash <script-path> <base sha> <head sha>
```

The script itself stays in the calling repository — it is small, and keeping it
there means a project can tighten its own rules without changing this shared
workflow. Only the plumbing (checkout, full history, SHA wiring) is shared.

Permissions requested: `contents: read`. Must be called from a workflow
triggered by `pull_request`; the base and head SHAs come from that event.

### Inputs

| Input | Type | Default | Description |
| --- | --- | --- | --- |
| `script-path` | string | `scripts/check-conventional-commits.sh` | Validator script path, relative to the calling repository root. |
| `runs-on` | string | `ubuntu-latest` | Runner label. |
| `timeout-minutes` | number | `5` | Job timeout. |

### Calling it

```yaml
name: Commits

on:
  pull_request:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: commits-${{ github.ref }}
  cancel-in-progress: true

jobs:
  commits:
    uses: bibeshpyakurel/github-workflows/.github/workflows/commits-reusable.yml@main
    permissions:
      contents: read
```

---

## Required status checks rename when you migrate

This is the one thing that bites. A job that comes from a reusable workflow is
reported as `<caller job name> / <called job name>`, not as the called job's
name alone. A repository whose branch protection requires the context
`Analyze python` will require `codeql / Analyze python` after migrating, and the
old context will sit forever as "Expected — waiting for status to be reported".

So for each migrated repository, update
*Settings → Branches → main → Require status checks* to the new context names in
the same change. The check is not weakened — same languages, same query suite,
same schedule — only renamed.

| Before | After (with caller job id `codeql` / `commits`) |
| --- | --- |
| `Analyze python` | `codeql / Analyze python` |
| `Analyze javascript-typescript` | `codeql / Analyze javascript-typescript` |
| `Conventional commits` | `commits / Conventional commits` |

## Pinning

Callers use `@main` so a fix lands everywhere at once — which is the point of
this repository. If a project ever needs to freeze, pin to a tag or a commit SHA
instead (`...codeql-reusable.yml@v1`).
