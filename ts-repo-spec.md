You are conforming a TypeScript repo to my standard tooling.

This document specifies the repo's target state: the rules that apply throughout, the archetype the repo belongs to, the files that must exist and their contents, the `package.json` shape, and the code conventions. Then it covers what to remove when migrating from an older setup, and a compact procedure to apply and verify it all. Apply only what's missing, and leave anything already in the target state alone. Even when nothing appears to be missing, still run the procedure's install, bump, lint, and verify steps (in particular, `bask bump-deps` always runs at least twice). If you're unsure about anything, or something seems off, don't skip it. Report it at the end so I can decide.

## Rules

- **Do not release.** Set up and verify the release tooling, but do NOT bump the version, generate `CHANGELOG.md` contents, create a git tag, publish, or trigger the release (`gh workflow run release.yml`, `bask release`, `bask release-run`).
- **Do not `tofu apply`.** Set tofu up fully and report what `tofu plan` wants to change, but leave `tofu apply` to me. Until I apply, `tofu-check` (and with it `bask build`) fails. That is expected: note it in the report instead of trying to fix it.
- **Do not change GitHub directly.** Don't delete or edit Actions secrets or repo settings with `gh`. `tofu apply` owns them (see Stale secrets).
- **The repo is the source of truth, not GitHub.** `github.tofu` and the `package.json` fields the module reads (`description`, `keywords`, `homepage`) define the GitHub repo's state. Never copy GitHub's current state back into the repo to shrink the plan. A difference between the two is a pending change for the report. The one exception is `visibility` when `github.tofu` is first created (see Baseline).
- **Do not commit.** Leave every change uncommitted (staged or unstaged is fine) so the diff can be reviewed.
- **Global tools and credentials are assumed installed.** They are never devDependencies. If one is missing, stop and report.
- **Scope: the repo's own files.** Anything shipped by `@adamhl8/configs` (the base tasks, the reusable workflows and their setup action, the base configs) is out of scope. Don't override or copy it just to make it conform to a rule here.
- **Use judgment, not inertia.** Don't keep something just because the repo has it. Clean up what the target doesn't need, even when no tool flags it (an `as const` on a factory argument). Raise those decisions in the report.
- **Check `@adamhl8/configs` for what's new.** It keeps evolving, so before migrating, look for new features or changes that affect this repo. Its README (`node_modules/@adamhl8/configs/README.md`) and its base files (`node_modules/@adamhl8/configs/dist/configs/`) show what it currently provides. Prefer what it provides over hand-rolled code or other dependencies.
- **Prefer `@adamhl8/configs` defaults.** Call each factory with no arguments, and make the smallest change that makes the tooling pass. Add an argument only when this spec calls for one or a lint failure clearly does.
- **Every factory is a deep merge** (see Config merging in the configs README). Assign the result to `const config` on its own line and `export default` the `config` separately.
- **Compare each config to its base.** Every config built on a configs base (a factory config, or a synced file like `bunfig.toml`) gets compared with its base file. Remove what the base already covers and what overrides it by accident, keeping only what's genuinely repo-specific. A synced file stores the base values from its last sync as its own, and the repo's values win, so a newer base value never replaces an older one by itself. Delete the stale key, then run `bask prepare` again to pick up the current base.

- **Self-dependent packages.** A package that the shared tooling loads by name (a release-it plugin, bask, the configs package itself) depends on itself (`"file:."`) so it can self-install. Such a repo may build before the self-install, through its own `prepare` setup or `bunfig.toml` settings. Those are intentional: leave them alone.
- **The oxlint base is strict, and it forces real code changes.** Expect to fix code, not just to accept autofixes. When a rule genuinely doesn't fit, suppress it as narrowly as the hits allow:
  1. Fix the code. This is the default.
  2. Otherwise, an inline `// oxlint-disable-next-line <rule> - <reason>` at the violation.
  3. A top-of-file `// oxlint-disable <rule> <rule> - <reason>` only when one file hits the same rule over and over.
  4. A scoped `overrides` block in `oxlint.config.ts` (keyed by a file glob) only when many files hit it.
  5. A global `rules` override only for a rule that's irrelevant to the whole project.

  There are good reasons to disable a rule sometimes, so take each case on its merits. Every disable comment ends with its reason after a single dash (` - <reason>`). A block disable that doesn't open the file (`/* oxlint-disable <rule> - <reason> */` around a section) needs a matching `/* oxlint-enable <rule> */` where the section ends. Disable comments use the `oxlint-` prefix. An override looks like this:

  ```ts
  const config = oxlintConfig({
    overrides: [
      {
        files: ["**/*.test.ts"],
        rules: { "node/no-process-env": "off" },
      },
    ],
  })
  ```

- **oxfmt formats the code blocks in Markdown too.** A block it would break (a deliberately badly formatted example in a README, or one whose order matters) takes `<!-- oxfmt-ignore -->` on the line before it.

## Repo archetypes

Several choices below are decided by what the repo _is_. Pick the archetype first, then the rest follows:

| archetype           | npm               | `engines`                                            | `tsdown.config.ts`                                                          | `github.tofu` `actions_secrets`              | `release.yml`                                                       | release artifacts                                                     |
| ------------------- | ----------------- | ---------------------------------------------------- | --------------------------------------------------------------------------- | -------------------------------------------- | ------------------------------------------------------------------- | --------------------------------------------------------------------- |
| **npm library**     | published         | `node`, or `bun` if it requires bun or ships no code | `tsdownConfig`, or none for an asset package (see Published package shapes) | `NPM_CI_TOKEN = true`                        | no inputs, `contents: write`                                        | npm publish                                                           |
| **docker app**      | `"private": true` | `bun`, or `node` if its runtime is node              | none, or `tsdownBundleConfig` for a side bundle                             | omit the block (defaults), unless `GH_TOKEN` | `docker-image: <image-name>`, `contents: write` + `packages: write` | ghcr image                                                            |
| **compiled binary** | `"private": true` | `bun`                                                | none (`bun build --compile` in a bask task)                                 | omit the block (defaults), unless `GH_TOKEN` | no inputs, `contents: write`                                        | release-it hooks + `github.assets`                                    |
| **plugin / bundle** | `"private": true` | `node`                                               | `tsdownBundleConfig`                                                        | omit the block (defaults), unless `GH_TOKEN` | no inputs, `contents: write`                                        | release-it hooks + `github.assets`, or a bundle committed to the repo |

Notes that cut across the table:

- **Archetypes combine.** Each column follows from the part of the repo it describes. A CLI published to npm is an npm library that ships a `bin` (see Published package shapes). A published package that also attaches release artifacts is an npm library plus release-it hooks and `github.assets`.
- **`engines` says where the code the repo ships or runs has to work.** `node` when it must work outside bun: a package consumers install, an app whose runtime is node, and a plugin loaded by a browser-like host (Obsidian, IINA), which is web code. `bun` when it runs on bun. It decides which runtime APIs the code uses (see Where bun APIs stop).
- **`"private": true` is about npm, not GitHub.** It doesn't imply a private GitHub repo: `github.tofu`'s `visibility` is set on its own (see github.tofu). `"private": true` is what makes release-it skip the npm publish step.
- **Release-specific work never gets its own workflow job.** Building and attaching artifacts and post-release publishing go in `.release-it.ts` hooks (see .release-it.ts), and a docker image or system packages go in the `release.yml` inputs. `before-release` (system packages for the release build) fits any archetype, so "no inputs" in the table is only the default. The only job a repo may append is a deploy job (see GitHub Actions).

### Published package shapes

A package published to npm takes one of four shapes, decided by how it's consumed. The three built shapes use `tsdownConfig` (unbundled, publint on). An asset package has no tsdown build:

| shape             | what it is                                                                               | `tsdown.config.ts`                                            | `package.json` `exports`                                                 | types           | attw                                             |
| ----------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------- | ------------------------------------------------------------------------ | --------------- | ------------------------------------------------ |
| **typed library** | something consumers import                                                               | `tsdownConfig({ platform: "node" })` (or `"neutral"`)         | `{ ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js" } }` | `.d.ts` shipped | on, `@arethetypeswrong/core` is a devDependency  |
| **CLI-only**      | a `bin`, no importable API                                                               | `tsdownConfig({ platform: "node", dts: false, attw: false })` | none                                                                     | none            | off, no `@arethetypeswrong/core`                 |
| **host-loaded**   | loaded by name by another tool (e.g. a release-it plugin), no consumer imports its types | `tsdownConfig({ platform: "node", dts: false, attw: false })` | `{ ".": { "import": "./dist/index.js" } }`                               | none            | off, no `@arethetypeswrong/core`                 |
| **asset package** | non-code files (fonts, CSS), generated by a bask task if at all                          | none                                                          | none                                                                     | none            | off, and no publint either (nothing runs tsdown) |

- A typed library can also ship a `bin` (a library with a companion CLI). It stays a typed library. The CLI is a second entry appended to the base's `./src/index.ts`: `tsdownConfig({ platform: "node", entry: ["./src/cli.ts"] })`.
- attw checks only importable, typed entrypoints, which is why it's off for the untyped shapes. publint stays on for the three built shapes and still validates `bin` and `files`.
- An asset package's `files` lists the assets it ships. Files generated at release (e.g. TTFs) come from an `after:bump` hook plus `github.assets` (see .release-it.ts).
- A CLI-only package whose entrypoint exports something only so a test can import it moves that code into its own module that both the entrypoint and the test import.

## Files that must exist

Create these verbatim. Each snippet shows the default, with no overrides. A repo may already have overrides: keep one only if it's still needed (see Rules). If the repo has one of these files in a different shape (say a `lefthook.yaml` with its own `pre-commit:`/`commands:` block), replace it with the form below rather than merging the two.

### bask.ts

```ts
export * from "@adamhl8/configs/bask"
```

That one line provides the base tasks (see bask tasks in the configs README), and bare `bask` lists them. Tasks run as `bask <task>`, and a task's name is its export name in kebab-case (`bask bump-deps` runs `bumpDeps`). Two things about them matter here:

- In CI, `build` fails if the run changed or created any file, so everything the install, `prepare`, lint, and tsdown write must be committed exactly as they leave it, or be gitignored.
- `bump-deps` runs npm-check-updates, so a repo's `.ncurc.yml` (e.g. a `reject` pin) is real config: keep it.

Repo-specific scripts are extra exported tasks below the `export *` (see Scripts). A repo that changes a base task extends it with a hook instead of redefining it or copying its body. To write a task or a hook, read the bask README (`node_modules/@adamhl8/bask/README.md`). On top of what it says:

- Call a local bin by its bare name (`` sh`astro sync` ``), never through `bun` (`` sh`bun astro sync` ``). bun's own subcommands (`bun build`, `bun install`) are not bins and keep their `bun`.
- `.sh` helpers stay as files, called from their task (`` sh`./scripts/render.sh` ``).
- `bask.ts` is internal code: it uses bun APIs (see bun) and follows the same lint rules as the rest of the repo.

### oxlint.config.ts

```ts
import { oxlintConfig } from "@adamhl8/configs"
import { defineConfig } from "oxlint"

const config = oxlintConfig()

export default defineConfig(config)
```

### oxfmt.config.ts

```ts
import { oxfmtConfig } from "@adamhl8/configs"
import { defineConfig } from "oxfmt"

const config = oxfmtConfig()

export default defineConfig(config)
```

**Tailwind.** Both bases find a Tailwind project's stylesheet on their own. Override it only when detection picks the wrong file, and then in both configs together (see oxfmt and oxlint in the configs README).

### commitlint.config.ts

```ts
import { commitlintConfig } from "@adamhl8/configs"

const config = commitlintConfig()

export default config
```

### knip.ts

```ts
import { knipConfig } from "@adamhl8/configs"

const config = knipConfig()

export default config
```

### lefthook.yaml

```yaml
extends:
  - node_modules/@adamhl8/configs/dist/configs/lefthook.base.yaml
```

### tsconfig.json

```json
{ "extends": "@adamhl8/configs/tsconfig" }
```

No `paths` block (see Modules).

### .release-it.ts

```ts
import { releaseItConfig } from "@adamhl8/configs"

const config = releaseItConfig()

export default config
```

The base registers the `release-it-git-cliff` plugin with a git-cliff config bundled in `@adamhl8/configs`, so there is no per-repo `cliff.toml`. The plugin must be a direct devDependency of the repo (see devDependencies): with bun's isolated linker, release-it can't see a copy that exists only under `@adamhl8/configs`.

A repo with release artifacts adds hooks and assets here rather than to the workflow: an `after:bump` hook builds them (or only packages them, since the base runs `build` before the release starts), `github.assets` uploads them, and post-release work (publishing to a homebrew/scoop tap) runs in an `after:release` hook:

```ts
const config = releaseItConfig({
  hooks: {
    "after:bump": ["bask build-binaries"],
    "after:release": ["bun scripts/publish.ts ${version}"],
  },
  github: {
    assets: ["bin/<name>-*"],
  },
})
```

### tsdown.config.ts

Only for a repo that builds something (see Repo archetypes). Three factories, pick by what you're building:

```ts
// a typed library: unbundled, with type declarations, attw, and publint. `platform` is required.
import { tsdownConfig } from "@adamhl8/configs"
import { defineConfig } from "tsdown"

const config = tsdownConfig({ platform: "node" })

export default defineConfig(config)
```

- `tsdownConfig`: a **published package that builds**. Requires `platform`: `"node"` by default, or `"neutral"` for a library whose shipped code uses no runtime APIs at all (pure logic that runs on node, bun, or in a browser). Its options depend on the package's shape (see Published package shapes).
- `tsdownBundleConfig`: a **single-file bundle that isn't a published package** (an Obsidian plugin, an IINA plugin, an Apps Script file). Bundles everything and drops declarations, sourcemaps, attw, and publint. Requires `platform` **and** `entry`.
- `tsdownBinConfig`: an **extensionless helper bin bundled alongside a package's main output** (a second entry in a multi-config `tsdown.config.ts`). It's a node bundle with no declarations or publish checks. Requires `entry` (`platform` defaults to node). This is not how a published CLI is built.

Reach for `tsdownBundleConfig` before hand-writing `unbundle: false` / `dts: false` / `attw: false` / `publint: false` on top of `tsdownConfig`. That combination _is_ the bundle config. The CLI-only and host-loaded config is different: it stays unbundled and keeps publint.

### github.tofu

Manages the GitHub repo itself declaratively (settings, Actions permissions, vulnerability alerts, and Actions secrets) via the tofu module shipped in `@adamhl8/configs`:

```hcl
terraform {
  backend "s3" {
    profile = "adamhl8"
    bucket  = "tofu-state-269329412317-us-east-1-an"
    key     = "adamhl8/<repo-name>.tfstate"
    region  = "us-east-1"
  }
}

module "github" {
  source    = "./node_modules/@adamhl8/configs/dist/tofu/github"
  repo_name = "<repo-name>"
  actions_secrets = {
    NPM_CI_TOKEN = true
  }
  repo_settings = {
    visibility = "public"
  }
}
```

Substitute `<repo-name>` (from the git remote, see Metadata fields) in both places, the backend `key` and `repo_name`. Set `actions_secrets` and `visibility` per the rules below, and keep everything else verbatim.

**Actions secrets.** This is the one list of what the repo needs. Everything else in this document refers back here:

| secret               | used by                                                                                                                          | needed when                          |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------ |
| `CI_SIGNING_KEY`     | `release.yml`: the signed release commit and tag                                                                                 | always (module default)              |
| `CI_TOKEN`           | `update-deps.yml`: the dependency PR. `release.yml`: the release step (e.g. an `after:release` hook that pushes to another repo) | always (module default)              |
| `NPM_CI_TOKEN`       | `release.yml`: the npm publish                                                                                                   | only if the repo publishes to npm    |
| `GH_TOKEN`           | `ci.yml`: `bunx bask build`, as `GITHUB_TOKEN`. `release.yml`: the docker build, as a BuildKit secret                            | only if a build calls the GitHub API |
| `additional_secrets` | a repo-specific job (e.g. the deploy job's `WATCHTOWER_API_TOKEN`)                                                               | as declared                          |

`actions_secrets` is a set of flags. `CI_SIGNING_KEY` and `CI_TOKEN` default to **true**, so the module always manages those two. `NPM_CI_TOKEN` and `GH_TOKEN` default to false. Keep `NPM_CI_TOKEN = true` only for a repo that publishes to npm. Every other archetype **omits the `actions_secrets` block** and takes the defaults, unless it needs `GH_TOKEN`, and then the block holds just `GH_TOKEN = true`. Anything else goes in the `additional_secrets` set of names.

Other module inputs and behavior:

- `visibility` is required ("public" or "private") and is the only `repo_settings` field. An existing `github.tofu` keeps its value. A new one takes the repo's current visibility (see Baseline). It is unrelated to npm's `"private": true`: a private npm package can live in a public GitHub repo.
- The module reads the repo's `package.json` for the repo description, topics, and homepage (see Metadata fields).
- `tofu init` (run by `bask prepare`) generates `.terraform/` and `.terraform.lock.hcl`, both untracked (the synced `.gitignore` block covers them). `github.tofu` is the only tofu file that gets committed.

### .gitignore

Not written by hand. The base `prepare` task keeps the shared ignore list (`node_modules/@adamhl8/configs/dist/configs/gitignore.base`) at the top of the file above a marker line, and replaces everything above the marker on each run. Everything below the marker is left untouched and is where project-specific entries live.

On the first sync the repo's entire old `.gitignore` lands below the marker, so clean up after it: delete every line below the marker that the managed block already covers (including equivalent spellings like `dist` without the slash), keeping only genuinely project-specific entries.

### bunfig.toml

Committed, but not written by hand from scratch. The base `prepare` task deep-merges the shared base config (`node_modules/@adamhl8/configs/dist/configs/bunfig.base.toml`) into the repo's `bunfig.toml` and writes the result back. The repo's values win over the base's, and the file is regenerated from the merged config, so comments don't survive. Repo-specific settings live in the same file and survive later syncs (e.g. `[test] preload = ["./src/__tests__/test-setup.ts"]`, see Tests use `bun test`). Because the repo's values win, an old `bunfig.toml` can silently override the base (see Compare each config to its base).

### bun.lock

Committed, and the only lockfile. Don't hand-edit it.

### GitHub Actions

Three workflows, each a thin wrapper around one of my shared reusable workflows. Every `uses:` line always points at `adamhl8/configs`, regardless of which repo you're working on, so do not rewrite it. The reusable workflows run the base tasks (`ci.yml` runs `bunx bask build`, `release.yml` runs `bunx bask release-run`, `update-deps.yml` runs `bunx bask bump-deps`). For which secrets each one uses, see Actions secrets.

`.github/workflows/ci.yml` runs on pushes and PRs. On a push it lints the commit message, and on a PR it lints the PR title, because squash merges (the only merge the tofu module allows) use the title as the commit message. The `edited` type re-lints a retitled PR, and the reusable workflow skips the build for it:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    types: [opened, edited, synchronize, reopened]

jobs:
  ci:
    uses: adamhl8/configs/.github/workflows/ci.yml@main
```

`ci.yml` takes no secrets by default. A repo that needs `GH_TOKEN` passes it **by name**, never `secrets: inherit`, since CI runs on every PR and shouldn't be handed the release secrets it never uses. It reaches the build renamed to `GITHUB_TOKEN`, because Actions secrets can't be named `GITHUB_*`. When it's unset, the build falls back to the auto-provisioned token, so fork PRs still build:

```yaml
jobs:
  ci:
    uses: adamhl8/configs/.github/workflows/ci.yml@main
    secrets:
      GH_TOKEN: ${{ secrets.GH_TOKEN }}
```

`.github/workflows/update-deps.yml` (weekly `bump-deps` run that opens a dependency-update PR):

```yaml
name: Update dependencies

on:
  schedule:
    - cron: "0 13 * * 5" # Fridays, 13:00 UTC
  workflow_dispatch: {}

jobs:
  update-deps:
    uses: adamhl8/configs/.github/workflows/update-deps.yml@main
    secrets: inherit
```

`.github/workflows/release.yml` (manually dispatched release):

```yaml
name: Release

on:
  workflow_dispatch: {}

jobs:
  release:
    permissions:
      contents: write
    uses: adamhl8/configs/.github/workflows/release.yml@main
    secrets: inherit
```

The reusable release workflow takes two optional inputs, and they exist precisely so a repo doesn't add its own jobs:

- **`docker-image`**: builds the repo's `Dockerfile` and pushes it to `ghcr.io/adamhl8/<docker-image>:latest` once the release lands, checking out the tag the release just created. The image name is separate from the repo name, since they often differ. The caller must **also** grant `packages: write` (a called workflow's permissions are capped by what the calling job grants). If the Dockerfile's build calls the GitHub API, `GH_TOKEN` reaches it as a BuildKit secret named `GH_TOKEN`, mounted with `RUN --mount=type=secret,id=GH_TOKEN,env=GITHUB_TOKEN`.
- **`before-release`**: shell commands for a build that needs system packages. They run after the install and the git signing setup, just before `bunx bask release-run`.

```yaml
jobs:
  release:
    permissions:
      contents: write
      packages: write
    uses: adamhl8/configs/.github/workflows/release.yml@main
    with:
      docker-image: <image-name>
    secrets: inherit
```

```yaml
jobs:
  release:
    permissions:
      contents: write
    uses: adamhl8/configs/.github/workflows/release.yml@main
    with:
      before-release: |
        sudo apt update
        sudo apt install -y fontforge ttfautohint
    secrets: inherit
```

A repo may append a **deploy** job after the release (it waits for the whole called workflow, so it runs after the image is pushed). Nothing else. Its secret is declared via `additional_secrets`:

```yaml
deploy:
  needs: release
  runs-on: ubuntu-latest
  steps:
    - name: Deploy to watchtower
      env:
        WATCHTOWER_API_TOKEN: ${{ secrets.WATCHTOWER_API_TOKEN }}
      run: |
        curl \
          --fail \
          --request POST \
          --header "Authorization: Bearer $WATCHTOWER_API_TOKEN" \
          https://watchtower.<host>/v1/update
```

**Job and step names.** The three wrappers above have no steps of their own, but any repo-specific job the repo adds does, and its names get normalized: every step has a `name`, and that name starts with a verb describing what the step does, not the tool it reaches for. A step running `bun install` is `Install dependencies` (not "bun install" or "Setup"), an `actions/checkout` step is `Check out repository`, and a `curl` to a deploy hook is `Deploy to watchtower`. Sentence case, no trailing period. Job names follow the same shape (`Deploy image`), and a job whose id already reads clearly can go without one.

### README.md, CHANGELOG.md, and LICENSE

`README.md`: must exist, because `bask lint` runs markdown-toc on it and fails on a missing file. If the repo has none, create a minimal one (a title and the `package.json` description). Add `<!-- toc -->` / `<!-- tocstop -->` markers only if the README wants a table of contents.

`CHANGELOG.md`: an empty file, created only if the repo doesn't already have one. Keep an existing changelog as-is, and either way do not generate its contents (that happens on the first real release, see Rules).

`LICENSE`: set the copyright line to `Copyright (c) <current year> adamhl8` (normalize both the year and the holder, e.g. an "Adam Langbert" holder becomes `adamhl8`), and create the file if the repo has none (MIT text, matching `"license": "MIT"`). A repo under a different license (e.g. OFL for a font) keeps its license text and only gets the copyright line normalized. A repo whose code and shipped assets are licensed differently keeps both files: an MIT `LICENSE` for the code, plus the assets' license (e.g. `OFL.txt`), and `license` in `package.json` names the one for what the package ships.

## `package.json`

### Metadata fields

These have one canonical shape and order. Every repo carries the shared fields, and only a repo published to npm adds the npm-facing ones, marked `†` below (a `"private": true` repo must not have those: delete any it has). The skeleton is annotated, not verbatim, and shows the field order to write:

```jsonc
{
  "name": "<preserve>",
  "version": "<preserve, or \"0.0.0\" if missing (the first release sets the real one)>",
  "private": true, // only for a repo not published to npm
  "description": "<required, see below>",
  "keywords": ["<see below>"],
  "homepage": "https://github.com/adamhl8/<repo-name>", // † published only, except a `"private": true` repo with a real site (see below)
  "bugs": { "url": "https://github.com/adamhl8/<repo-name>/issues" }, // † published only
  "license": "MIT",
  "author": {
    "name": "Adam Langbert",
    "email": "adamhl@pm.me",
    "url": "https://github.com/adamhl8",
  },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/adamhl8/<repo-name>.git",
  },
  "bin": { "<name>": "./dist/index.js" }, // only if the package ships an executable
  "files": ["dist/"], // † published only: "dist/" for a built package (not "./dist/"), a package that ships other files (fonts, CSS) lists those instead
  "type": "module",
  "imports": { "#*": "./src/*" }, // see Modules (repo-specific extras like "#package.json" stay)
  "exports": { ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js" } }, // † published only, form per Published package shapes (repo-specific extra subpaths stay)
  "publishConfig": { "access": "public" }, // † scoped (@adamhl8/...) published packages only
  "scripts": { "prepare": "bask prepare" },
  // then dependencies / devDependencies / peerDependencies
  "engines": { "node": ">=24" }, // always last (see below)
}
```

- `<repo-name>` comes from the actual git remote (`git remote get-url origin`), never from the existing field values: metadata copy-pasted from another repo happens (a `repository`/`homepage`/`bugs` block pointing at a different repo), and correcting it to this repo's URLs is in scope.
- `description` is required: the tofu module sets the GitHub repo description from it (the plan errors when it's missing), so add or fill one if missing or empty. Do not end it with a period unless it is multiple sentences.
- `keywords` become the GitHub topics at `tofu apply`, and a missing field clears existing topics. They come from the repo, never from GitHub's current topics. A published package must have them (draft them from the description if missing). Any other repo keeps the `keywords` it has, and gets them drafted from the description only when the project is one people would search for (a reusable tool or plugin, not a personal app or site). Either way, review them, and report anything you'd add or change. Each entry must be a valid topic slug: lowercase, hyphenated, no spaces (`"error handling"` -> `"error-handling"`).
- `homepage` also feeds `tofu apply`: the module sets the GitHub repo's homepage from it, except that a `homepage` pointing at the repo itself (the canonical value above) is treated as redundant and leaves the GitHub homepage unset. Keep the canonical repo URL unless the package has a real docs/project site.
- On a `"private": true` repo, `homepage` is kept only when it's a real site (e.g. a website repo's own domain), because that is what sets the GitHub homepage. One whose `homepage` is just the repo URL drops it, along with `bugs`.
- `license` is the SPDX id of the `LICENSE` file (see README.md, CHANGELOG.md, and LICENSE): `"MIT"`, unless the repo keeps another license (e.g. `"OFL-1.1"`).
- `engines` is required and always the last field. It names exactly one runtime, `{ "node": ">=24" }` or `{ "bun": ">=1.4" }`, chosen per Repo archetypes. Normalize an existing one to that form.
- No `devEngines` or `packageManager` field (bun uses neither): delete one if present.
- Fields not named here (`trustedDependencies`, `workspaces`, the repo-specific `imports`/`exports` extras) are usually the repo's own. Keep them unless they look stale, and report what you removed.

### Scripts

Every script lives in `bask.ts` (the standard ones come from the base, repo-specific ones are exported tasks), and the only one left in `package.json` is `prepare`:

```json
"scripts": {
  "prepare": "bask prepare"
}
```

- A repo-specific script (e.g. a Prisma app's `db:*`) is an exported task below the `export *` (`db:generate` -> `export const dbGenerate`, run as `bask db-generate`). A repo-specific extra step in a standard flow extends the base task with a hook (see bask.ts).
- The only exception is a genuine npm lifecycle script, something the package manager itself must trigger (like `prepare`): those stay in `package.json`. Anything else goes.

### devDependencies

The repo must have every devDependency that applies to it: `@adamhl8/configs` plus what Installation in the configs README lists. Add a missing one with `bun add --dev <name>`, which writes its latest range. Leave the version of an existing one alone, because `bask bump-deps` moves every dependency to its newest (Apply step 3). The exception is `@adamhl8/configs` (see Apply step 2). On top of what the README says:

- Every repo releases, so `release-it` and `release-it-git-cliff` always apply.
- `tsdown` goes in any repo that builds (see Repo archetypes), bundles included.
- `@arethetypeswrong/core` is only for a typed library (see Published package shapes).
- `@types/bun`, not `@types/node`.

Notes:

- knip can't see a bin that only runs from a task or a test, so a devDependency used that way goes in knip's `ignoreDependencies`. That's also how a global tool becomes a devDependency: only when the repo's own tests run it (e.g. `git-cliff` in release-it-git-cliff).
- If the repo imports `@adamhl8/configs/env` (`parseEnv`/`requireWhen`) from runtime code (an app's `env.ts`, a `prisma.config.ts`), `@adamhl8/configs` belongs in `dependencies`, not `devDependencies`.
- Remove an empty `"dependencies": {}` if present. Leave real runtime `dependencies` untouched.

## Code conventions

### Modules

Internal imports are `#` subpath imports (no slash after the `#`), backed by `"imports": { "#*": "./src/*" }` in `package.json`, with an explicit extension on every local specifier (`#utils/foo.ts`). No relative import survives, sibling and child included, in source and test files. ESM has no directory resolution, so a directory import names its index file (`#utils/index.ts`, not `#utils`). `"#*": "./src/*"` is the only internal-import alias: rewrite any other (a tsconfig `paths` block, a differently named `imports` key) to it. tsdown rewrites them back to relative paths in `dist/`, so this is safe for published libraries.

`export` goes on the declaration itself: no `export { foo }` and no re-exports (`export { x } from "..."`, `export * from "..."`), except in `index.*` barrel files. Lint enforces all of this and autofixes most of it.

**All source lives in `src/`.** That's what the alias points at. A repo with source elsewhere (top-level `plugin/` and `views/` behind a `"#*": "./*"` alias, a root `lib/`) moves it under `src/`, then updates every path that names it. Root config files, `bask.ts`, and `scripts/` helpers stay where they are.

### bun

**Never access bun globally.** The global `Bun` is banned: every file that touches a bun API imports what it uses from `"bun"` first. Which import depends on how distinctive the name is:

- Named imports when the bare name reads unambiguously on its own: `import { $, spawn, sleep, Glob } from "bun"`, then `` $`...` ``, `spawn(...)`, `sleep(...)`, `new Glob(...)`.
- The namespace when the bare name is generic enough to collide with a local or another library (`file`, `write`, `serve`, `env`, `password`, ...): `import bun from "bun"`, then `bun.file(...)`, `bun.write(...)`, `bun.password.hash(...)`.

(Module imports like `bun:test` and `bun:sqlite` are unaffected: they are already explicit imports.)

**Bun APIs.** Wherever bun-only code is fine (see Where bun APIs stop), prefer bun's native APIs over the node ones: `bun.file`/`bun.write` instead of `node:fs` reads and writes, `$` (or `spawn`) instead of `node:child_process`, and in general, when bun has a native API for something the code does manually (`Glob`, `bun:sqlite`, `bun.serve`, `sleep`, `bun.password`), use it. Where bun has no equivalent, the node API stays: directory operations (`rm` with `recursive`, `cp`, `mkdtemp`), `node:path`, and `node:os`. (`bun.write` creates missing parent directories, so it needs no `mkdir` first.)

**Where bun APIs stop.** `engines` decides (see Repo archetypes):

- Under `engines.bun`, everything uses bun APIs.
- Under `engines.node`, the code that has to work outside bun uses node APIs only. That's everything a published package ships in `dist/` (a shipped CLI included: `node:child_process` or `execa` instead of `$` from `"bun"`), and the code a node runtime runs. bun is still the package manager, and the `bunfig.toml` and Dockerfile rules still apply.
- The repo's internal code always uses bun APIs: tests and test support, `bask.ts`, config files, build scripts.
- A plugin loaded by a browser-like host (Obsidian, IINA) runs neither node nor bun, so its shipped code uses neither runtime's APIs.
- A single tool that breaks under bun (e.g. a CLI that segfaults) runs under node in its task (`.bun(false)`, see the bask README), and the repo's code that the tool loads keeps node APIs. The rest of the repo stays on bun.

**Shebangs.** An executable `.ts`/`.js` script's shebang is `#!/usr/bin/env bun`. A `.sh` helper uses a POSIX shell shebang (`#!/usr/bin/env bash`, `#!/bin/sh`), because bun's shell isn't bash. Code that must run on node uses `#!/usr/bin/env node`: a shipped `bin` under `engines.node`, and the scripts a node runtime runs.

**Tests use `bun test`** (only if the repo has tests). Tests import from `"bun:test"`, and test config lives in `bunfig.toml`'s `[test]` block (see bunfig.toml): there is no test config file. Test support code (a setup file, helpers, fixtures) lives under a `__tests__` dir, because the base's coverage skips those dirs. A test setup file is registered via `[test].preload = ["./src/__tests__/test-setup.ts"]`, imports from `"bun:test"`, and registers custom matchers through `expect.extend`. Custom matcher types augment `"bun:test"`:

```ts
declare module "bun:test" {
  interface Matchers<T> {
    // your custom matchers
  }
}
```

### Commands and containers

**Commands are spelled out in full.** Anywhere a command is invoked (a bask task, a workflow step, a `Dockerfile`, the `prepare` script, a `.sh` helper), write the long form for project tooling (`bun`, `bask`, `tofu`, `gh`, `docker`, `git`, `npx`, `curl`, and the local devDependency bins). That means the full subcommand name, never an alias (`bun install`, not `bun i`, and `bun remove`, not `bun rm`), and the full flag name where one exists (`--production`, not `-p`, and `--location`, not `-L`). POSIX and shell utilities (`rm -rf`, `ls`, `grep -q`, `mkdir -p`) are exempt: their short flags are idiomatic, and some have no portable long form. This is about readability in the repo's own checked-in files, so it does not apply to commands you only run in the shell yourself, or to the base tasks (see Rules).

**`bun` over `bunx`.** Anything local (a devDependency's bin, a script in the repo) runs as `bun <command>`, never `bunx <command>`. Inside a bask task, a local bin runs by its bare name instead (see bask.ts). `bunx` is for non-local invocations only, i.e. a package that isn't a dependency and gets fetched on the fly (a one-off code generator like `bunx openapi-typescript`).

**`bask` runs bare**, in the `prepare` script, release-it hooks, and anywhere else: `bun run` and bask's own tasks put `node_modules/.bin` on `PATH`. The exception is a workflow step or a Dockerfile `RUN` line, where it doesn't. Those write `bunx bask <task>`, never `bun bask`, because `bun bask` runs `bask.ts` itself instead of the bask bin.

**Dockerfiles.** bun is the package manager in every container build, even in a repo under `engines.node` (this is build/deploy tooling, not shipped code).

- The base image is `oven/bun:latest`. When the runtime must stay node, copy the bun binary in instead: `COPY --from=oven/bun:latest /usr/local/bin/bun /usr/local/bin/bun`.
- **Any stage that installs dependencies with bun or runs the program with bun copies all three of `package.json`, `bun.lock`, and `bunfig.toml`**, in that order, before the install:

  ```dockerfile
  COPY package.json bun.lock bunfig.toml ./
  ```

  `bunfig.toml` is the one that gets forgotten, and leaving it out silently changes behavior rather than failing: the install loses `[install] linker = "isolated"` and the run loses `[run] bun = true`. A stage that neither installs nor runs with bun (a runtime stage that only `COPY --from`s `node_modules` and executes with node) needs `package.json` alone.

- Everything else is copied by name (`COPY src ./src`, `COPY tsconfig.json ./`), never with `COPY . .`, so the build context's `node_modules/`, `.env`, and `.git` can't leak into the image. With explicit copies a `.dockerignore` is optional.
- A build stage that runs a bask task (`RUN bunx bask build-site`) copies `bask.ts` along with what the task reads, and installs devDependencies, because bask and the base tasks come from the install.
- An image's `bun install` needs `--ignore-scripts`, since `prepare` runs `bask prepare` (git hooks and config syncs off devDependencies) and can't work in a container. Where `--ignore-scripts` would also skip a _needed_ install script (a native module like better-sqlite3), `RUN bun pm pkg delete scripts.prepare` before a plain `bun install` instead.

## Migrating from older setups

**For every role the target stack owns, exactly one tool fills it:** package manager, test runner, lint and format, git hooks, commit-message linting, release and changelog, task runner (with `bask.ts` the only home for scripts), CI, and dependency updates. Delete anything else that fills one of those roles, whatever tool it came from, along with its devDependencies. It should usually be obvious what goes. Report anything you're unsure about.

A file that mixes stack-related lines with unrelated ones (say a `.gitattributes` with an entry for old hook scripts next to real rules) loses only the stack-related lines, and is deleted entirely only when nothing unrelated remains.

- A config for an in-stack tool under another name or format than the one in Files that must exist (`.oxlintrc.json`, `lefthook.yml`, `knip.json`) is deleted in favor of that file, after carrying over any real overrides.
- An `.npmrc` stays only if it holds real config (registry, auth). An empty or install-churn-only one is deleted.

## Apply & verify

### Baseline

Based on this spec, gather the facts you need about the repo before changing anything. Two of them need care:

- Its **archetype**, and for a published package its **shape** (see Repo archetypes). That decides `engines`, the tsdown factory, the `github.tofu` secrets, and the `release.yml` inputs.
- Whether it **exists on GitHub**, and what `visibility` a new `github.tofu` gets. An existing `github.tofu` keeps its own `visibility`. Otherwise `gh repo view --json visibility` gives it. If that fails because the repo doesn't exist (not an auth error), skip the tofu import, set `visibility = "private"` in a new `github.tofu`, and flag it in the report.

### Apply

In this order (some steps are order-sensitive):

1. Apply everything in Migrating from older setups.
2. Write everything in Files that must exist and the `package.json` edits. Leave existing dependency versions alone, except `@adamhl8/configs`, which goes to its latest version so the current base tasks exist. Do both in one command: `bun add --dev @adamhl8/configs@latest` plus every missing devDependency. Apply every Code convention that lint doesn't autofix (everything under bun and Commands and containers). Leave subpath imports, import extensions, and direct exports to the lint autofix in step 5.
3. Run `bask bump-deps` **twice**, even if the repo looked fully migrated at baseline (this step is never skipped). The first pass runs under the base tasks that were installed, so it can bring in newer base tasks and tooling than the ones it ran with. The second pass runs under the new base, and must succeed and be a no-op. If it fails or still upgrades something, fix what it reports and run it again until a pass is clean. Then compare each config to its base (see Rules) and clean up `.gitignore` (see .gitignore).
4. Tofu. The repo's Actions secrets and settings are not pushed by hand: `tofu apply` on `github.tofu` manages them, and the apply is mine (see Rules). `bask prepare` already ran `tofu init`, so from the repo root:
   1. Import the existing repo into state. Skip this if `tofu state list` already shows `module.github.github_repository.current` (a re-run: importing twice fails with "Resource already managed"). Also skip it if the repo doesn't exist on GitHub yet (see Baseline), because the module creates it on apply:

      ```sh
      tofu import module.github.github_repository.current "$(basename -s .git $(git remote get-url origin))"
      ```

   2. `tofu plan`, and include what it wants to change (repo settings, Actions permissions, secrets) in the report. Check its output for stale secrets (see Stale secrets). Do NOT run `tofu apply`.

5. `bask lint` until clean. Apply autofixes, fix remaining oxlint errors in the code (suppressing only per Rules), resolve knip findings (unused deps/exports/files), fix actionlint findings in the repo's workflows, and re-run. The import fixes can take more than one pass to converge. oxlint-tsgolint reports TypeScript errors too, so a clean lint is a full type check. A dependency that `bump-deps` moved to a new major version can break the code: read its README or changelog in `node_modules` to see how to adapt, rather than pinning it back.

   knip's **configuration hints** (e.g. a `project`/`ignore` pattern that no longer matches anything) do not fail the run: knip prints them and still exits 0, so `bask lint` looks clean. Treat them as errors anyway. Fix each one (usually by dropping a stale entry from `knip.ts`), and if one can't be resolved, surface it in the report instead of letting it pass silently.

### Stale secrets

The module flags an undeclared Actions secret on the repo as a **warning**, not a failure: the plan still succeeds, and the warning alone doesn't make `bask tofu-check` fail. `tofu-check`'s `-compact-warnings` also hides the message, so read the output of the plain `tofu plan` from Apply step 4 for `Unmanaged secrets on '<repo>': <NAMES>`. For each name:

- If the repo genuinely needs it, declare it via the `actions_secrets` flags or `additional_secrets`.
- Otherwise it's stale (the common case is a leftover `NPM_CI_TOKEN` on a `"private": true` repo). Do NOT delete it. List it in the report with the command for me to run: `gh secret delete <NAME> --repo <owner/name>`. The exception is a secret the plan also destroys (one `github.tofu` used to declare, like a dropped `NPM_CI_TOKEN = true`): my apply removes it, so report it as a planned change with no delete command.

### Verify green

Without releasing:

- `bask` (bare) lists the base tasks, confirming `bask.ts` loads the base.
- `bask build` passes: lint (including `bun test`, which passes with no tests), then tsdown (with publint for a built published package, and attw for a typed library), then `tofu-check`. While the plan has pending changes `tofu-check` fails (see Rules): confirm everything before it passed, and note it. The output has no knip configuration hints (they exit 0, so check the output, not just the exit code).
- A second `bask build` changes no files (compare `git status --porcelain` and `git diff` before and after). Every untracked file `git status` lists is one to commit, and build output is gitignored. CI's `build` fails on either.
- `tofu plan` runs without errors and shows only the expected changes. It shows no `Unmanaged secrets` warning, or every name it shows is either declared or in the report.
- The Actions secrets match the Actions secrets table for this repo: confirm pending additions and removals in the `tofu plan` output, and the ones already applied with `gh secret list --repo <owner/name>`.
- `.git/hooks/pre-commit` and `.git/hooks/commit-msg` are lefthook-managed (their contents reference lefthook), and `lefthook validate` succeeds.
- Release tooling sanity-checks: `release-it-git-cliff` resolves from the repo root (`bun --print 'import.meta.resolve("release-it-git-cliff")'`), `git-cliff --config node_modules/@adamhl8/configs/dist/configs/cliff.base.toml --bumped-version` prints a version, and all three workflow files exist. Do NOT run `bask release` / `bask release-run` / `gh workflow run release.yml`.

### Report and stop

Leave every change uncommitted. The report covers:

- What changed.
- What was skipped (e.g. no tests).
- Decisions made: suppressions and overrides, manual fixes, and anything you removed or kept on judgment.
- Keywords you'd add or change.
- What `tofu plan` wants to change, pending my apply, and whether `tofu-check` (and so `bask build` and the pre-commit hook) fails until then.
- Stale Actions secrets, each with its `gh secret delete <NAME> --repo <owner/name>` command (except one the plan already destroys).
- Anything that couldn't be resolved (a knip configuration hint, a lint rule that needed judgment).
- Anything the spec doesn't cover, or where the repo seems to diverge on purpose (e.g. an archetype or package shape the tables don't list). Report it rather than forcing it into the nearest fit.

The lefthook `commit-msg` (commitlint) and `pre-commit` (`bask build`) hooks are now installed, so a later commit needs a conventional message (`bask commit-types` lists the allowed types), files `bask build` leaves unchanged, and a drift-free tofu state.
