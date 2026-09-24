You are conforming a TypeScript repo to my standard tooling. The target: **bun** (package manager, script runner, and test runner), **just** (task runner, a one-line justfile importing the shared base from `@adamhl8/configs`), **oxlint + oxfmt** (lint/format), **lefthook** (git hooks), **commitlint** (conventional commits), **release-it + `release-it-git-cliff`** (release/changelog, a release-it plugin that drives git-cliff), **tofu** (an OpenTofu `github.tofu` that manages the GitHub repo's settings and Actions secrets via a module shipped in `@adamhl8/configs`), **`bun test`** configured through a synced **`bunfig.toml`**, and **`#*` subpath imports** for internal source.

Almost everything the repo needs comes from **`@adamhl8/configs`**, so the version it's on determines how much work this is. Two realistic starting points:

- **The common case**: the repo is already on bun/just/oxlint/lefthook but on an older `@adamhl8/configs`. It looks migrated. It is not: the base changes underneath it between versions (dead config-merge APIs, removed bins, new required devDependencies). The real work is the bump and its fallout, and Leftovers from older `@adamhl8/configs` (under Migrating from older setups) lists what to hunt for.
- **The rare case**: the repo is on some older stack entirely (a different package manager, linter, test runner, or hook setup). Migrating from older setups covers it by role, whatever the tool was.

This document specifies the repo's target state (the archetype it belongs to, the files that must exist and their contents, the `package.json` shape, and the code conventions), then what to remove or rewrite when migrating from an older setup, then a compact procedure to apply and verify it all. Apply only what's missing, and leave anything already in the target state alone. Even when nothing appears to be missing, still run the procedure's install/bump/lint/verify steps (in particular, `just bump-deps` always runs at least twice).

## Invariants

- Prefer `@adamhl8/configs` defaults: call each factory with no arguments, and make the smallest change that makes the tooling pass. Add an argument only when this spec calls for one (a required key, a package shape, release artifacts, a generated directory) or a lint failure clearly does.
- **Do not release.** Set up and verify the release tooling, but do NOT bump the version, generate `CHANGELOG.md` contents, create a git tag, publish, or trigger the release (`gh workflow run release.yml`, `just release`, `just release-run`). An empty `CHANGELOG.md` is created now only if the repo doesn't already have one (see README.md, CHANGELOG.md, and LICENSE). Its contents are generated on the first real release.
- **Do not `tofu apply`.** Set tofu up fully and report what `tofu plan` wants to change, but leave `tofu apply` to me: I review and apply it manually. Whenever the plan has pending changes (always on a first setup, and after any change to `github.tofu` or to `description`, `keywords`, or `homepage` in `package.json`, which the module reads), `tofu-check` (and with it `just build` and the pre-commit hook) fails until I apply. That is expected: note it in the report instead of trying to fix it.
- **Do not change GitHub directly.** Don't delete or edit Actions secrets or repo settings with `gh`. `tofu apply` owns them, and a stale secret goes in the report for me to delete or for my apply to remove (see Stale secrets).
- **The repo is the source of truth, not GitHub.** `github.tofu` and the `package.json` fields the module reads (`description`, `keywords`, `homepage`) define the GitHub repo's state, and `tofu apply` pushes them there. Never copy GitHub's current state back into the repo to shrink the plan. A difference between the two is a pending change for the report. The one exception is `visibility` when `github.tofu` is first created (see Baseline (f)).
- **Do not commit.** Leave every change uncommitted (staged or unstaged is fine) so the diff can be reviewed.
- **Global tools**, assumed already installed (NOT project deps): `bun`, `just`, `lefthook`, `git-cliff`, `actionlint`, `gh`, `tofu`, `aws`. Also assumed: an AWS credentials profile named `adamhl8` (tofu's S3 state backend, and the base justfile gates its tofu recipes on `aws configure list-profiles` finding it) and the sops age key for `~/homelab/secrets/credentials.yaml` (the tofu module decrypts it to read secret values). If any is missing, stop and report. Everything else is a local devDependency. (`git-cliff` is global because `release-it-git-cliff` shells out to the `git-cliff` on `PATH`.)
- Preserve the repo's identity: `name`, `version`, real dependencies, and repo-specific `imports`/`exports` entries. The metadata fields around them have one canonical shape (see Metadata fields), and normalizing to it is in scope. Change only what this spec describes.
- **Scope: the repo's own files.** These rules govern what the repo itself checks in. Anything shipped by `@adamhl8/configs` (the base justfile, the reusable workflows and their setup action, the base configs) is out of scope. Don't override or copy it just to make it conform to a rule here (e.g. to rewrite its flags or step names).
- **Self-dependent packages.** A package that the shared tooling loads by name (a release-it plugin, an oxlint plugin, the configs package itself) depends on itself (`"file:."`) so its own tooling can find it. Such a repo may override `prepare` to build before the self-install, and may import the base justfile from its own source instead of `node_modules`. Those overrides are intentional: leave them alone.

## Three facts that shape everything

**1. Every `@adamhl8/configs` factory is a deep merge, and the second argument is a second config.**

`fooConfig(userConfig)` deep-merges your config into the base. **Arrays append** to the base array. To use an array as-is instead, put it in a **second config object**, which merges the same way except its arrays **replace** the base array:

```ts
const config = knipConfig(
  {
    // appended to the base ignoreBinaries
    ignoreBinaries: ["fd"],
  },
  {
    // used as-is, replacing the base project globs
    project: ["src/**/*.ts"],
  },
)
```

The second config works at any depth: only the arrays you name are replaced, sibling keys still merge normally, and a key present in both gets the second config's value. Required keys (e.g. `platform` for `tsdownConfig`) must go in the **first** config. No `as const` on either argument (drop it if an older config has one): the factories use `const` type parameters.

Older `@adamhl8/configs` versions had two other forms for this. Both are dead now and silently wrong (see Migrating from older setups).

House style for every factory call: assign the result to `const config` on its own line and `export default` the `config` separately.

**2. The oxlint base is strict, and it will force real source changes.**

It enables `correctness`/`pedantic`/`perf`/`restriction`/`style`/`suspicious`/`nursery` as errors plus type-aware rules, and bans patterns real code hits: `import * as` (`import/no-namespace`) and function declarations (`func-style`, arrow expressions only). It also loads `@adamhl8/eslint-plugin-clean-modules`, whose three rules enforce the module conventions and are the ones that bite:

- `clean-modules/require-subpath-imports`: **every relative import is banned**, not just parent-relative. `./foo.ts` is as illegal as `../shared/foo.ts`. Both become `#`-prefixed subpath imports.
- `clean-modules/require-direct-exports`: `export { foo }` and re-exports (`export { x } from "..."`, `export * from "..."`) are banned. Put `export` on the declaration itself. `index.*` files are exempt, since barrel re-exports legitimately live there.
- `clean-modules/require-import-extensions`: explicit `.ts`/`.tsx` extensions on every local import specifier.

All three autofix, but `require-subpath-imports` and `require-import-extensions` can rewrite the same import (`./foo` -> `#foo` -> `#foo.ts`), so convergence may take more than one `--fix` pass.

What the base already allows: it exempts the config files (`.release-it.ts`, `astro.config.ts`, `commitlint.config.ts`, `knip.ts`, `oxfmt.config.ts`, `oxlint.config.ts`, `prisma.config.ts`, `tsdown.config.ts`) from `import/no-default-export`, and `.release-it.ts` from `no-template-curly-in-string` (for `${version}` hooks). It turns `node/no-top-level-await` and `one-var` off. In `.tsx` files, `func-style` allows function declarations that are named exports (React components).

Beyond autofixes, expect real source changes: function declarations -> arrow expressions (`func-style`), interface method signatures -> property signatures (`method-signature-style`), `import * as x` -> named imports (`import/no-namespace`), and removing conditions the type-aware rules flag as redundant.

When a rule genuinely doesn't fit, suppress it as narrowly as the hits allow:

1. Fix the code. This is the default.
2. Otherwise, an inline `// oxlint-disable-next-line <rule>` at the violation.
3. A top-of-file `// oxlint-disable <rule> <rule>` only when one file hits the same rule over and over. Common offenders in test files: `typescript/require-await`, `no-throw-literal`, `typescript/only-throw-error`, `unicorn/no-null`, `unicorn/error-message`, `vitest/no-conditional-in-test`.
4. A scoped `overrides` block in `oxlint.config.ts` (keyed by a file glob) only when many files hit it.
5. A global `rules` override only for a rule that's irrelevant to the whole project.

The base sets `reportUnusedDisableDirectives: "error"`, so a disable that no longer suppresses anything fails lint. An `overrides` block goes in the **first** config, so it appends to the base's overrides (which hold the config-file exemptions). In the second config it would drop them:

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

Real examples:

- A plugin/entry file that must `export default`: an inline `import/no-default-export` disable on the export (the base already exempts the config files, but not arbitrary source).
- A loop that is sequential by necessity (retry backoff, cursor pagination, interactive prompts): an inline `no-await-in-loop` disable on the loop.
- Tests (or a test harness) that set or read env vars across many files: `node/no-process-env` off via an override for the test files.
- Ambient declaration files (`**/*.d.ts`) and Astro components (`**/*.astro`): `import/unambiguous` off via an override, since every one of them is legitimately a script.
- A repo whose domain legitimately traffics in `null` everywhere (a query language with null literals, Prisma's nullable model types): `unicorn/no-null` off globally via the `rules` override.
- A generated directory (Prisma's client): not a suppression. Add it to `ignorePatterns` in both `oxlint.config.ts` and `oxfmt.config.ts`, and exclude it from knip via `project: ["!./src/generated/**/*"]`.

**3. No semicolons.** The oxfmt base sets `semi: false` and `printWidth: 120`. Every snippet in this document is written the way its formatter leaves it (oxfmt, or `just --fmt` and `tofu fmt` for the justfile and tofu snippets, which `just lint` also runs). Don't add semicolons back: `just lint` strips them anyway.

## Repo archetypes

Several choices below are decided by what the repo *is*. Pick the archetype first, then the rest follows:

| archetype | npm | `tsdown.config.ts` | `github.tofu` `actions_secrets` | `release.yml` | release artifacts |
| --- | --- | --- | --- | --- | --- |
| **npm library** | published | `tsdownConfig`, or none for an asset package (see Published package shapes) | `NPM_CI_TOKEN = true` | no inputs, `contents: write` | npm publish |
| **docker app** | `"private": true` | none, or `tsdownBundleConfig` for a side bundle | omit the block (defaults), unless `GH_TOKEN` | `docker-image: <image-name>`, `contents: write` + `packages: write` | ghcr image |
| **compiled binary** | `"private": true` | none (`bun build --compile` in a justfile recipe) | omit the block (defaults), unless `GH_TOKEN` | no inputs, `contents: write` | release-it hooks + `github.assets` |
| **plugin / bundle** | `"private": true` | `tsdownBundleConfig` | omit the block (defaults), unless `GH_TOKEN` | no inputs, `contents: write` | release-it hooks + `github.assets` |

Notes that cut across the table:

- **Archetypes combine.** Each column follows from the part of the repo it describes. A CLI published to npm is an npm library that ships a `bin` (see Published package shapes). A published package that also attaches release artifacts is an npm library plus release-it hooks and `github.assets`.
- **`"private": true` is about npm, not GitHub.** It doesn't imply a private GitHub repo: `github.tofu`'s `visibility` is set on its own (see github.tofu). `"private": true` is what makes release-it skip the npm publish step.
- **Release-specific work never gets its own workflow job.** Building and attaching artifacts and post-release publishing go in `.release-it.ts` hooks (see .release-it.ts), and a docker image or system packages go in the `release.yml` inputs. `before-release` (system packages for the release build) fits any archetype, so "no inputs" in the table is only the default. The only job a repo may append is a deploy job (see GitHub Actions).

### Published package shapes

A package published to npm takes one of four shapes, decided by how it's consumed. The three built shapes use `tsdownConfig` (unbundled, publint on). An asset package has no tsdown build:

| shape | what it is | `tsdown.config.ts` | `package.json` `exports` | types | attw |
| --- | --- | --- | --- | --- | --- |
| **typed library** | something consumers import | `tsdownConfig({ platform: "node" })` | `{ ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js" } }` | `.d.ts` shipped | on, `@arethetypeswrong/core` is a devDependency |
| **CLI-only** | a `bin`, no importable API | `tsdownConfig({ platform: "node", dts: false, attw: false })` | none | none | off, no `@arethetypeswrong/core` |
| **host-loaded** | loaded by name by another tool (e.g. a release-it plugin), no consumer imports its types | `tsdownConfig({ platform: "node", dts: false, attw: false })` | `{ ".": { "import": "./dist/index.js" } }` | none | off, no `@arethetypeswrong/core` |
| **asset package** | non-code files (fonts, CSS), generated by a justfile recipe if at all | none | none | none | off, and no publint either (nothing runs tsdown) |

- A typed library can also ship a `bin` (a library with a companion CLI). It stays a typed library.
- attw checks only importable, typed entrypoints. On a package with no `exports` it fails with "No resolution", and on `exports` without types it fails with "Package has no types". That's why it's off for the untyped shapes, not a workaround. publint stays on for the three built shapes and still validates `bin` and `files`.
- An asset package's `files` lists the assets it ships. Files generated at release (e.g. TTFs) come from an `after:bump` hook plus `github.assets` (see .release-it.ts).
- A CLI-only package whose entrypoint exports something only so a test can import it moves that code into its own module that both the entrypoint and the test import.

## Files that must exist

Create these verbatim (each wrapper just calls a factory from `@adamhl8/configs`). Follow the house style (fact 1) for any that need overrides. If the repo already has one of these files in a different shape (say a `lefthook.yaml` with its own inline `pre-commit:`/`commands:` block from an earlier partial setup), replace it with the form below. Do not merge the two. A config for the same tool under another filename is covered in Migrating from older setups.

### justfile

The standard scripts live in a shared base justfile, not in `package.json`:

```just
import "node_modules/@adamhl8/configs/dist/configs/justfile.base.just"
```

That one line provides the recipes `lint`, `build`, `test`, `bump-deps`, `tofu-check`, `release`, `release-run` (private, run by the release workflow), `commit-types`, and `prepare`. Bare `just` lists the recipes and then prints the allowed commit types. What each does:

- `prepare`: `lefthook install` (skipped in CI, i.e. when `CI` is set, so commits made by the workflows don't run the hooks), then the `.gitignore` and `bunfig.toml` syncs (see Synced files), then `tofu init -upgrade`. The tofu step is gated on the same condition as `tofu-check`, so it's a no-op where it doesn't apply.
- `lint`: markdown-toc (regenerates the README TOC), `oxlint --fix --fix-suggestions --fix-dangerously`, `oxfmt`, `just --fmt` on every justfile and `tofu fmt` on every tofu file (both skip gitignored files), `knip-bun`, `actionlint` (only when `.github/workflows/` exists), then `just test`.
- `test`: `bun test --pass-with-no-tests`, so a repo with no tests passes.
- `build`: `lint`, then `tsdown` only if `tsdown.config.ts` exists, then `tofu-check` last. In CI it also fails if the run changed or created any file (`git add --intent-to-add --all && git diff --exit-code`). So everything the install, `prepare`, lint, and tsdown write must be committed exactly as they leave it, or be gitignored.
- `tofu-check`: `tofu plan -detailed-exitcode -concise -compact-warnings`, a drift check that fails on any pending change. Runs only outside CI, only when a root `*.tofu` file exists, and only when the `adamhl8` AWS profile is available. So CI never needs tofu credentials, and someone without my AWS profile can still run `build`.
- `bump-deps`: see Apply step 3.
- `release`: `gh workflow run release.yml`. `release-run` is the actual `release-it` invocation (`bun release-it -VV --ci`), run by the workflow in CI. Do NOT run either.
- `commit-types`: prints the commit types commitlint allows.

Recipes invoke local bins through bun (`bun oxlint`, `bun knip-bun`, ...), and the base does not put `node_modules/.bin` on `PATH`. Repo-specific scripts also live in the justfile as extra recipes below the `import` (see Scripts).

Each public recipe in the base is a thin wrapper over a private `_recipe` that holds the actual body. A repo that modifies a standard recipe extends it below the `import` with this pattern, so the override still runs the original (never copy the body):

```just
# add steps after the original: dependencies run first, then the body
bump-deps: _bump-deps
    bun prisma generate

# add steps before and after: deps before `&&` run first, deps after run last
lint: my-setup _lint && my-cleanup
```

### Config wrappers

`oxlint.config.ts`:

```ts
import { oxlintConfig } from "@adamhl8/configs"
import { defineConfig } from "oxlint"

const config = oxlintConfig()

export default defineConfig(config)
```

(Overrides default to none. When a lint failure calls for one, follow the suppression order in fact 2, which also shows the `overrides` shape.)

`oxfmt.config.ts`:

```ts
import { oxfmtConfig } from "@adamhl8/configs"
import { defineConfig } from "oxfmt"

const config = oxfmtConfig()

export default defineConfig(config)
```

`commitlint.config.ts`:

```ts
import { commitlintConfig } from "@adamhl8/configs"

const config = commitlintConfig()

export default config
```

`knip.ts`:

```ts
import { knipConfig } from "@adamhl8/configs"

const config = knipConfig()

export default config
```

(Overrides are usually a stray binary knip can't resolve, e.g. `knipConfig({ ignoreBinaries: ["fd"] })` when a bun shell template shells out to `fd`.)

`lefthook.yaml`:

```yaml
extends:
  - node_modules/@adamhl8/configs/dist/configs/lefthook.base.yaml
```

(The base hooks run `just build` on pre-commit and commitlint on commit-msg. The pre-commit hook fails when `just build` changes any file, such as a lint autofix or reformat, so the fixes have to be staged and the commit retried.)

`tsconfig.json`:

```json
{ "extends": "@adamhl8/configs/tsconfig" }
```

No `paths` block. The base sets `types: ["bun"]`, which is why `@types/bun` is a devDependency. Add other keys only when the repo genuinely needs them: an `exclude` (a fixtures directory, a generated file), an `include` for generated types, or extra `compilerOptions.types`. They replace the base's values instead of merging, so re-list what the base has: an `exclude` keeps `dist/`, and a `types` list keeps `bun`.

### .release-it.ts

```ts
import { releaseItConfig } from "@adamhl8/configs"

const config = releaseItConfig()

export default config
```

The base registers the `release-it-git-cliff` plugin with the git-cliff config bundled in `@adamhl8/configs` (so there is no per-repo `cliff.toml`). The plugin picks the next version from the commits, writes `CHANGELOG.md` (the base formats it with oxfmt), and sets the GitHub release notes. The base also wires up the signed commit (`release: ${tagName}`) and tag (`v${version}`), the GitHub release, and `bun publish`. The plugin must be a direct devDependency of the repo (see devDependencies). release-it resolves plugins from its own location or the working directory, and with bun's isolated linker a copy that exists only under `@adamhl8/configs` is invisible to it.

A repo with release artifacts adds hooks and assets here rather than to the workflow: an `after:bump` hook builds them, `github.assets` uploads them, and post-release work (publishing to a homebrew/scoop tap) runs in an `after:release` hook:

```ts
const config = releaseItConfig({
  hooks: {
    "after:bump": ["just build-binaries"],
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

- `tsdownConfig`: a **published package that builds**. Requires `platform`. Its options depend on the package's shape (see Published package shapes).
- `tsdownBundleConfig`: a **single-file bundle that isn't a published package** (an Obsidian plugin, an IINA plugin, an Apps Script file). Bundles everything and drops declarations, sourcemaps, attw, and publint. Requires `platform` **and** `entry`.
- `tsdownBinConfig`: an **extensionless helper bin bundled alongside a package's main output** (a second entry in a multi-config `tsdown.config.ts`). It's a node bundle with no declarations or publish checks. Requires `entry` (`platform` is fixed to node). This is not how a published CLI is built.

Reach for `tsdownBundleConfig` before hand-writing `unbundle: false` / `dts: false` / `attw: false` / `publint: false` on top of `tsdownConfig`. That combination *is* the bundle config. The CLI-only and host-loaded config is different: it stays unbundled and keeps publint.

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

Substitute `<repo-name>` in both places (the backend `key` and `repo_name`), set `actions_secrets` and `visibility` per the rules below, and keep everything else verbatim.

**Actions secrets.** This is the one list of what the repo needs. Everything else in this document refers back here:

| secret | used by | needed when |
| --- | --- | --- |
| `CI_SIGNING_KEY` | `release.yml`: the signed release commit and tag | always (module default) |
| `CI_TOKEN` | `update-deps.yml`: the dependency PR. `release.yml`: the release step (e.g. an `after:release` hook that pushes to another repo) | always (module default) |
| `NPM_CI_TOKEN` | `release.yml`: the npm publish | only if the repo publishes to npm |
| `GH_TOKEN` | `ci.yml`: `just build`, as `GITHUB_TOKEN`. `release.yml`: the docker build, as a BuildKit secret | only if a build calls the GitHub API |
| `additional_secrets` | a repo-specific job (e.g. the deploy job's `WATCHTOWER_API_TOKEN`) | as declared |

- `actions_secrets` is a set of flags. `CI_SIGNING_KEY` and `CI_TOKEN` default to **true**, so the module always manages those two. `NPM_CI_TOKEN` and `GH_TOKEN` default to false. Keep `NPM_CI_TOKEN = true` only for a repo that publishes to npm. Every other archetype **omits the `actions_secrets` block** and takes the defaults, unless it needs `GH_TOKEN`, and then the block holds just `GH_TOKEN = true`. Anything else goes in the `additional_secrets` set of names.
- Every secret's value is read from the sops-encrypted `~/homelab/secrets/credentials.yaml`, keyed by the lowercased secret name (`ci_token`, `npm_ci_token`, ...).
- The module's `no_unmanaged_secrets` check flags any Actions secret on the repo that isn't declared here (see Stale secrets).

Other module inputs and behavior:

- `visibility` is required ("public" or "private") and is the only `repo_settings` field. An existing `github.tofu` keeps its value. A new one takes the repo's current visibility (see Baseline (f)). It is unrelated to npm's `"private": true`: a private npm package can live in a public GitHub repo.
- The module reads the repo's `package.json` for the repo description, topics, and homepage (see Metadata fields).
- `tofu init` (run by `just prepare`) generates `.terraform/` and `.terraform.lock.hcl`, both untracked (the synced `.gitignore` block covers them). `github.tofu` is the only tofu file that gets committed.

### Synced files

`.gitignore`: not written by hand. The base justfile's `prepare` recipe runs the `adamhl8-gitignore` bin, which keeps the shared ignore list at the top of the file above a marker line (the first run prepends the block, later runs replace everything above the marker, so updates propagate with package bumps). Everything below the marker is left untouched and is where project-specific entries live:

```gitignore
node_modules/
dist/
.terraform
.terraform.lock.hcl
# The above patterns are managed by @adamhl8/configs. Anything manually added above this line will be replaced.

# project-specific entries
.env.local
```

On the first sync the repo's entire old `.gitignore` lands below the marker, so clean up after it: delete every line below the marker that the managed block above already covers (the old hand-maintained `node_modules/`, `dist/`, tofu entries, including equivalent spellings like `dist` without the slash), keeping only genuinely project-specific entries.

`bunfig.toml`: committed, but not written by hand from scratch. bun has no extends mechanism for it, so the base justfile's `prepare` recipe runs the `adamhl8-bunfig` bin, which deep-merges the shared base config into the repo's `bunfig.toml` and writes the result back (a repo without one gets the base as-is), then formats it with oxfmt. The repo's values win over the base's on the merge, and the file is regenerated from the merged config, so comments don't survive. The base:

```toml
[test]
pathIgnorePatterns = ["fixtures/**", "fixture/**"]
coverage = true
coverageSkipTestFiles = true
coveragePathIgnorePatterns = ["src/test-setup.ts"]
concurrentTestGlob = "**/*.test.ts"
onlyFailures = true

[install]
linker = "isolated"

[run]
bun = true
```

Repo-specific settings live in the same file and survive later syncs. The one this setup adds when called for is a test setup file (see Tests use `bun test`):

```toml
[test]
preload = ["./src/test-setup.ts"]
```

`coveragePathIgnorePatterns` already covers `src/test-setup.ts` in the base, so extend it only for something else (a generated directory, a differently-placed test dir). Because the repo's values win, a `bunfig.toml` left from an earlier era can silently override the base, so clean up after the first sync: diff the file against `node_modules/@adamhl8/configs/dist/configs/bunfig.base.toml` and remove anything that overrides the base unintentionally (a `preload` pointing at a deleted file, a `coveragePathIgnorePatterns` that merely restates the base's), keeping only genuinely repo-specific settings.

### GitHub Actions

Three workflows, each a thin wrapper around one of my shared reusable workflows. Every `uses:` line always points at `adamhl8/configs`, regardless of which repo you're working on, so do not rewrite it. The reusable workflows share a setup action (checkout, bun, `just`, actionlint, OpenTofu, `bun install`), and lefthook is never installed in CI. They run the base justfile recipes (`ci.yml` runs `just build`, `release.yml` runs `just release-run`), so on a repo whose workflows already point here, CI stays red until the `justfile` above exists. For which secrets each one uses, see Actions secrets.

`.github/workflows/ci.yml` runs `just build` (lint + build) on pushes and PRs. On a push it lints the commit message, and on a PR it lints the PR title, because squash merges (the only merge the tofu module allows) use the title as the commit message. The `edited` type re-lints a retitled PR, and the reusable workflow skips the build for it:

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

`ci.yml` takes no secrets by default. A repo that needs `GH_TOKEN` passes it **by name**, never `secrets: inherit`, since CI runs on every PR and shouldn't be handed the release secrets it never uses. It reaches `just build` renamed to `GITHUB_TOKEN`, because Actions secrets can't be named `GITHUB_*`. When it's unset, the build falls back to the auto-provisioned token, so fork PRs still build:

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
- **`before-release`**: shell commands for a build that needs system packages. They run after the install and the git signing setup, just before `just release-run`.

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

`README.md`: must exist, because `just lint` runs `markdown-toc -i ./README.md` and fails on a missing file. If the repo has none, create a minimal one (a title and the `package.json` description). Add `<!-- toc -->` / `<!-- tocstop -->` markers only if the README wants a table of contents.

`CHANGELOG.md`: an empty file, created only if the repo doesn't already have one. Keep an existing changelog as-is, and either way do not generate its contents (that happens on the first real release, see Invariants).

`LICENSE`: set the copyright line to `Copyright (c) <current year> adamhl8` (normalize both the year and the holder, e.g. an "Adam Langbert" holder becomes `adamhl8`), and create the file if the repo has none (MIT text, matching `"license": "MIT"`). A repo under a different license (e.g. OFL for a font) keeps its license text and only gets the copyright line normalized.

## `package.json`

### Metadata fields

These have one canonical shape and order. Every repo carries the shared fields, and only a repo published to npm adds the npm-facing ones, marked `†` below (a `"private": true` repo must not have those: delete any it has). The skeleton is annotated, not verbatim, and shows the field order to write:

```jsonc
{
  "name": "<preserve>",
  "version": "<preserve, or \"0.0.0\" if missing (the first release sets the real one)>",
  "private": true, // only for a repo not published to npm (this is what makes release-it skip the npm publish step)
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
  "imports": { "#*": "./src/*" }, // see Code conventions (repo-specific extras like "#package.json" stay)
  "exports": { ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js" } }, // † published only, form per Published package shapes (repo-specific extra subpaths stay)
  "publishConfig": { "access": "public" }, // † scoped (@adamhl8/...) published packages only
  "scripts": { "prepare": "just prepare" },
  // then dependencies / devDependencies / peerDependencies
}
```

- `<repo-name>` comes from the actual git remote (`git remote get-url origin`), never from the existing field values: metadata copy-pasted from another repo happens (a `repository`/`homepage`/`bugs` block pointing at a different repo), and correcting it to this repo's URLs is in scope.
- `description` is required: the tofu module sets the GitHub repo description from it (the plan errors when it's missing), so add or fill one if missing or empty. Do not end it with a period unless it is multiple sentences.
- `keywords` become the GitHub topics at `tofu apply`, and a missing field clears existing topics. They come from the repo, never from GitHub's current topics. Any repo may have them, published or `"private": true`, depending on the kind of project. A published package must have them (draft them from the description if missing). Any other repo keeps the `keywords` it has, and gets them drafted from the description only when the project is one people would search for (a reusable tool or plugin, not a personal app or site). Either way each entry must be a valid topic slug: lowercase, hyphenated, no spaces (`"error handling"` -> `"error-handling"`).
- `homepage` also feeds `tofu apply`: the module sets the GitHub repo's homepage from it, except that a `homepage` pointing at the repo itself (the canonical value above) is treated as redundant and leaves the GitHub homepage unset. Keep the canonical repo URL unless the package has a real docs/project site.
- On a `"private": true` repo, `homepage` is kept only when it's a real site (e.g. a website repo's own domain), because that is what sets the GitHub homepage. One whose `homepage` is just the repo URL drops it, along with `bugs`.
- `license` is the SPDX id of the `LICENSE` file (see README.md, CHANGELOG.md, and LICENSE): `"MIT"`, unless the repo keeps another license (e.g. `"OFL-1.1"`).
- No `devEngines` or `packageManager` field (bun uses neither): delete one if present.
- Fields not named here (`engines`, `trustedDependencies`, `workspaces`, the repo-specific `imports`/`exports` extras) are the repo's own: preserve them.

### Scripts

Every script lives in the justfile (the standard ones come from the base, repo-specific ones are recipes), and the only one left in `package.json` is `prepare`:

```json
"scripts": {
  "prepare": "just prepare"
}
```

- A repo-specific script (e.g. a Prisma app's `db:*`) is a recipe below the `import` (just recipe names can't contain `:`, so `db:generate` -> `db-generate`). A repo-specific extra step in a standard flow extends the base recipe via the `_recipe` wrapper pattern (see justfile).
- The only exception is a genuine npm lifecycle script, something the package manager itself must trigger (like `prepare`): those stay in `package.json`. Anything else goes, including `prepublishOnly` and any script that existed only to run replaced tooling.

### devDependencies

The repo must have each of these. The versions are only the range to write when adding a missing entry. Never change the version of an entry the repo already has: `just bump-deps` (Apply step 3) moves every dependency to its newest.

```
@adamhl8/configs                ^2.8.0
oxlint                          ^1.85.0
oxlint-tsgolint                 ^7.0.2003
oxfmt                           ^0.70.0
@commitlint/cli                 ^21.2.3
@commitlint/config-conventional ^21.2.3
release-it                      ^21.1.0
release-it-git-cliff            ^0.2.0
@types/bun                      ^1.4.2    (replaces @types/node, the base tsconfig sets types: ["bun"])
knip                            ^6.38.0
typescript                      ^7.0.2
markdown-toc                    ^1.2.0
tsdown                          ^0.23.0   (only if the repo builds, see Repo archetypes)
publint                         ^0.3.24   (published packages that build)
@arethetypeswrong/core          ^0.18.5   (typed libraries only, see Published package shapes)
```

`release-it-git-cliff` is the one addition that the weekly update-deps PR can't make. `bump-deps` upgrades `@adamhl8/configs` on its own, but it never adds a new devDependency, so the gap may only show up when release-it fails to load the plugin. Check for it even on a repo that already looks current.

- Do NOT add `lefthook`, `bun`, `just`, `git-cliff`, or `actionlint` (global tools).
- If the repo imports `@adamhl8/configs/env` (`parseEnv`/`requireWhen`) from runtime code (an app's `env.ts`, a `prisma.config.ts`), `@adamhl8/configs` belongs in `dependencies`, not `devDependencies`.
- Remove an empty `"dependencies": {}` if present. Leave real runtime `dependencies` untouched.

## Code conventions

### Modules

**Subpath imports (`#*`).** Internal imports are written as `#...` (no slash after the `#`), backed by `"imports": { "#*": "./src/*" }` in `package.json`. `tsdown` rewrites them back to relative paths in `dist/`, so this is safe for published libraries. **No relative import survives**: `clean-modules/require-subpath-imports` bans every one of them, sibling and child included, in source **and** test files:

```ts
import { foo } from "./foo.ts" // error
import { foo } from "#foo.ts" // autofixed
```

The rule reads the `imports` map and autofixes, and `require-import-extensions` adds the `.ts`. ESM has no directory resolution, so a directory import names its index file explicitly (`#utils/index.ts`, not `#utils`). `"#*": "./src/*"` is the only internal-import alias: rewrite any other (a tsconfig `paths` block, a differently named `imports` key) to it.

**Direct exports.** `clean-modules/require-direct-exports` bans `export { foo }` and re-exports: put `export` on the declaration. `index.*` files are exempt, so barrel files keep their `export ... from`.

```ts
const foo = 1
export { foo } // error

export const foo = 1 // autofixed
```

### bun

**Never access bun globally.** The global `Bun` is banned: every file that touches a bun API imports what it uses from `"bun"` first. Rewrite any existing global `Bun.*` usage to an import. Which import depends on how distinctive the name is:

- Named imports when the bare name reads unambiguously on its own: `import { $, spawn, sleep, Glob } from "bun"`, then `` $`...` ``, `spawn(...)`, `sleep(...)`, `new Glob(...)`.
- The namespace when the bare name is generic enough to collide with a local or another library (`file`, `write`, `serve`, `env`, `password`, ...): `import bun from "bun"`, then `bun.file(...)`, `bun.write(...)`, `bun.password.hash(...)`.

(Module imports like `bun:test` and `bun:sqlite` are unaffected: they are already explicit imports.)

**Bun APIs.** bun is the default runtime (`bun run` and `bun test` both execute on bun), so wherever bun-only code is fine, prefer bun's native APIs over the node ones: `bun.file`/`bun.write` instead of `node:fs` reads and writes, `$` (or `spawn`) instead of `node:child_process`, and in general, when bun has a native API for something the code does manually (`Glob`, `bun:sqlite`, `bun.serve`, `sleep`, `bun.password`), use it. Rewrite existing node imports to the bun equivalents.

**Where bun APIs stop.** The boundary is npm compatibility, plus one runtime exception:

- A repo that publishes to npm keeps its shipped code (everything that lands in `dist/` for consumers) on node APIs **only**, so the package stays node-compatible. Do not introduce bun APIs there. This includes a published CLI's `bin` entrypoint, even for a tool you only ever run locally: it keeps a node shebang (see Shebangs) and uses node APIs (`node:child_process`, or `execa` per the exception below) instead of `$` from `"bun"`.
- Everything else can and should use bun APIs: repos not published to npm (a `"private": true` app, a project shipped as a binary), and the internal code of published packages (tests, `src/test-setup.ts`, config files, build scripts).
- A repo whose runtime genuinely has to be node (e.g. a dependency with a native module that doesn't load under bun, run from a `node:*` image with a `node` entrypoint) keeps node APIs in the code node runs. bun is still its package manager, and its `bunfig.toml` and Dockerfile rules still apply.

**`execa` is the exception to `$`.** Plain shell-outs use bun's `$`. `execa` is fine where the call needs what `$` doesn't do well, like a test harness that needs `preferLocal`, `extendEnv: false`, or merged `all` output. A shipped CLI in a published package can also use it, since it's node-compatible. Don't use `execa` just to set a `cwd` or run a single command, since `` $`...`.cwd(dir) `` covers that.

**Shebangs.** An executable `.ts`/`.js` script's shebang is `#!/usr/bin/env bun`: repo scripts, helpers, and a `bin` entry on a non-published repo. A `.sh` helper keeps its POSIX shell shebang (`#!/usr/bin/env bash`, `#!/bin/sh`), because bun's shell isn't bash. Rewrite `#!/usr/bin/env node` (including the `-S` forms) to the bun shebang, with two exceptions from Where bun APIs stop. A shipped `bin` that lands in `dist/` for npm consumers keeps its node shebang so it still runs without bun installed. A repo whose runtime has to be node keeps `#!/usr/bin/env node` on the scripts node runs.

**Tests use `bun test`** (only if the repo has tests). Tests import from `"bun:test"`, and test config lives in `bunfig.toml`'s `[test]` block (see Synced files): there is no test config file. Tests are internal code, so they use bun APIs even in a published package (see Where bun APIs stop). A test setup file is registered via `[test].preload = ["./src/test-setup.ts"]`, imports from `"bun:test"`, and registers custom matchers through `expect.extend`. Custom matcher types augment `"bun:test"`:

```ts
declare module "bun:test" {
  interface Matchers<T> {
    // your custom matchers
  }
}
```

### Commands and containers

**Commands are spelled out in full.** Anywhere a command is invoked (a justfile recipe, a workflow step, a `Dockerfile`, the `prepare` script, a `.sh` helper), write the long form for project tooling (`bun`, `just`, `tofu`, `gh`, `docker`, `git`, and the local devDependency bins). That means the full subcommand name, never an alias (`bun install`, not `bun i`, and `bun remove`, not `bun rm`), and the full flag name where one exists (`--production`, not `-p`, and `--file`, not `-f`). Normalize any abbreviated command the repo already has. POSIX and shell utilities (`rm -rf`, `ls`, `grep -q`, `curl -LsSf`, `mkdir -p`) are exempt. Their short flags are idiomatic, and some have no portable long form (BSD `rm` on macOS has none). This is about readability in the repo's own checked-in files, so it does not apply to commands you only run in the shell yourself, or to recipes inherited from the base justfile (see Invariants).

**`bun` over `bunx`.** Anything local (a devDependency's bin, a script in the repo) runs as `bun <command>`, never `bunx <command>`. `bunx` is for non-local invocations only, i.e. a package that isn't a dependency and gets fetched on the fly (the base justfile's `bunx npm-check-updates`, or a one-off code generator like `bunx openapi-typescript`). Rewrite any `bunx <local-bin>` the repo has to `bun <local-bin>`.

**Dockerfiles.** bun is the package manager in every container build, and the npm boundary doesn't apply here (this is build/deploy tooling, not shipped code, so use bun even in a published repo).

- The base image is `oven/bun:latest`. When the runtime must stay node, copy the bun binary in instead: `COPY --from=oven/bun:latest /usr/local/bin/bun /usr/local/bin/bun`.
- **Any stage that installs dependencies with bun or runs the program with bun copies all three of `package.json`, `bun.lock`, and `bunfig.toml`**, in that order, before the install:

  ```dockerfile
  COPY package.json bun.lock bunfig.toml ./
  ```

  `bunfig.toml` is the one that gets forgotten, and leaving it out silently changes behavior rather than failing: the install loses `[install] linker = "isolated"` and the run loses `[run] bun = true`, so the container resolves and executes differently from the dev machine. A stage that neither installs nor runs with bun (a runtime stage that only `COPY --from`s `node_modules` and executes with node) needs `package.json` alone.

- Everything else is copied by name (`COPY src ./src`, `COPY tsconfig.json ./`), never with `COPY . .`, so the build context's `node_modules/`, `.env`, and `.git` can't leak into the image. With explicit copies a `.dockerignore` is optional.
- A build stage that runs a `just` recipe (e.g. `just build-site`) copies the binary in with `COPY --from=ghcr.io/casey/just:latest /just /usr/local/bin/` rather than installing it another way.
- An image's `bun install` needs `--ignore-scripts`, since `prepare` runs `just prepare` (git hooks and config syncs off devDependencies) and can't work in a container. Where `--ignore-scripts` would also skip a *needed* install script (a native module like better-sqlite3), `RUN bun pm pkg delete scripts.prepare` before a plain `bun install` instead.

**Stable bun.** Every project tracks the latest stable bun, never canary or a pinned version: `oven/bun:latest` in a Dockerfile (never `oven/bun:canary`), and a workflow that sets up bun itself uses `oven-sh/setup-bun@v2` with no `with: bun-version:` block (the action's default is stable). The three reusable workflows already do this, so a repo on the thin wrappers has nothing to change. A file that pins the bun version (`.bun-version`, or a bun entry in `.tool-versions` or `mise.toml`) is deleted, or loses just that entry if it pins other tools too.

## Migrating from older setups

End state: **for every role the target stack owns, exactly one tool fills it.** bun is the only package manager (`bun.lock` the only lockfile) and the only test runner, oxlint + oxfmt the only lint/format tooling, lefthook the only git-hook mechanism, commitlint the only commit-message tooling, `.release-it.ts` + `release-it-git-cliff` the only release/changelog config, the justfile the only home for scripts (`package.json` keeps only `prepare`, see Scripts), the three wrapper workflows (plus a deploy job) the only CI, and `update-deps.yml` the only dependency-update mechanism.

Anything else that fills one of those roles gets deleted, whatever tool it came from and whether or not this doc names it: old tool configs (`biome.json`, `vitest.config.ts`, a prettier config), any lockfile other than `bun.lock` (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `lock.yaml`), hook directories (`.githooks/`, a husky setup), any other workflow that builds, tests, lints, or releases (e.g. a `test.yml`), a renovate or dependabot config, release configs (including a `"release-it"` key in `package.json`, which conflicts with `.release-it.ts`), stale `package.json` scripts, and the matching devDependencies. A file that mixes stack-related lines with unrelated ones (say a `.gitattributes` with an entry for old hook scripts next to real rules) loses only the stack-related lines, and is deleted entirely only when nothing unrelated remains.

- Disable comments for a tool that's no longer in the stack (e.g. `biome-ignore`) are deleted, or converted to `// oxlint-disable-next-line <rule>` if oxlint flags the same line.
- An `.npmrc` stays only if it holds real config (registry, auth). An empty or install-churn-only one is deleted.
- A config for an in-stack tool under another name or format than the one in Files that must exist (`.oxlintrc.json`, `.oxfmtrc.json`, `lefthook.yml`, `.release-it.json`, `commitlint.config.js`, `.commitlintrc*`, `knip.json`) is deleted in favor of that file, after carrying over any real overrides.

The items below need extra care, because each one is easy to mistake for target state.

### Leftovers from older `@adamhl8/configs`

- **The old `setup` release input.** The reusable `release.yml` input for pre-release commands was renamed `before-release`. Rename any `setup:` under a `release.yml` job's `with:`.
- **`adamhl8-cliff` leftovers.** The bin was removed from `@adamhl8/configs` when the `release-it-git-cliff` plugin replaced it. Grep for `adamhl8-cliff` (justfile recipes, `.release-it.ts` hooks, README, scripts) and delete each use. Changelog and release notes now come from the plugin. To preview the next version, call `git-cliff` directly with the bundled config (see Verify green). Also delete a per-repo `cliff.toml` (the plugin uses the bundled config).
- **The dead config-merge APIs.** `fooConfig(cfg, { arrays: "replace" })` (an options object) and `fooConfig({ key: { value: [...], mode: "replace" } })` (a per-key wrapper) were both valid in earlier `@adamhl8/configs` versions. Neither works now: the second argument is read as a config (see fact 1), so an `{ arrays: "replace" }` left in place silently sets a bogus `arrays` key instead of switching merge mode. Grep every config file for `arrays:` and `mode:` and rewrite to the two-config form.
- **Overrides for things the base now allows.** `reportUnusedDisableDirectives` already makes lint fail on a stale inline disable. What lint doesn't flag is a `rules`/`overrides` entry in `oxlint.config.ts` for `no-template-curly-in-string` on `.release-it.ts`, `node/no-top-level-await`, `one-var`, or named-export function declarations in `.tsx` (see fact 2). Delete each one.
- **Repo-local release jobs.** A docker build/push or asset-upload job in the repo's `release.yml` is superseded by the `docker-image` input or release-it hooks (see GitHub Actions and .release-it.ts). Delete the job.
- **A `ci.yml` without the `edited` trigger.** A wrapper whose `pull_request:` has no `types` never re-lints a retitled PR. Add the `types` list shown in GitHub Actions.
- **Checks the base now runs.** `just lint` runs `just --fmt`, `tofu fmt`, and `actionlint`, and the reusable `ci.yml` lints the PR title. Delete a repo's own recipe, hook command, or workflow step that does the same (e.g. an `actionlint` job or a PR-title lint action).

## Apply & verify

### Baseline

Read `package.json`, `tsconfig.json`, whatever configs are present, and the `src/` layout, and note:

- (a) which **archetype** the repo is (see Repo archetypes), which decides the tsdown factory, the `github.tofu` secrets, and the `release.yml` inputs, and for a published package, which shape (see Published package shapes).
- (b) what `@adamhl8/configs` version it's on, whether any factory call uses a dead merge API, and whether `release-it-git-cliff` is a direct devDependency.
- (c) does it have tests, and are they on `bun:test`?
- (d) does it have relative imports, or an internal-import alias other than `#*`?
- (e) what still fills a role the target stack owns?
- (f) does the repo exist on GitHub, and what `visibility` does a new `github.tofu` get? An existing `github.tofu` keeps its own `visibility`. Otherwise `gh repo view --json visibility` gives it. If that fails because the repo doesn't exist (not an auth error), skip the tofu import, set `visibility = "private"` in a new `github.tofu`, and flag it in the report.

### Apply

In this order (some steps are order-sensitive):

1. Apply everything in Migrating from older setups that the baseline turned up.
2. Write everything in Files that must exist and the `package.json` edits (the `imports` map, and any missing devDependency at its listed range). Leave existing dependency versions alone, because step 3 bumps them. Apply every Code convention that lint doesn't autofix (everything under bun and Commands and containers), minding Where bun APIs stop. Leave subpath imports, import extensions, and direct exports to the autofix in step 5.
3. `bun install`, then **immediately** run `just bump-deps` **twice**, even if the repo looked fully migrated at baseline (this step is never skipped). Every later step assumes deps are already at their newest.

   `bump-deps` runs `bunx npm-check-updates --upgrade`, which goes past the ranges in `package.json` and rewrites them (this, not the plain install, is what moves `@adamhl8/configs` past the `^2.8.0` floor to whatever is newest), then reinstalls from scratch (`rm -f bun.lock`, `rm -rf node_modules/`, `bun install --no-cache`). The reinstall runs lifecycle scripts, so `prepare` -> `just prepare` -> `lefthook install` + `tofu init` + the `.gitignore` and `bunfig.toml` syncs all happen as part of it (watch for a `sync hooks` line confirming the `pre-commit`/`commit-msg` hooks were installed).

   The first pass runs against the `@adamhl8/configs` that `bun install` resolved, so it can bring in a newer base justfile (and newer tooling) than the one it executed with. The second pass runs under the new base with everything already in place. It must succeed and must be a no-op (nothing left to upgrade): if it fails or still upgrades something, the first pass hadn't settled, so fix what it reports and run it again until a pass is clean.

   The reinstall writes `bun.lock` with current stable bun, so expect its `"lockfileVersion"` line to change (a `1` becomes `2`). Don't hand-edit the lockfile.

   Once the marker line is in `.gitignore`, dedupe below it, and clean up `bunfig.toml` against the base (see Synced files).
4. Tofu. The repo's Actions secrets (see Actions secrets) and settings are not pushed by hand: `tofu apply` on `github.tofu` manages them. Your part is setup and review only, and the apply itself is mine (see Invariants). `just prepare` already ran `tofu init`, so from the repo root:

   1. Import the existing repo into state. Skip this if `tofu state list` already shows `module.github.github_repository.current` (a re-run: importing twice fails with "Resource already managed"). Also skip it if the repo doesn't exist on GitHub yet (Baseline (f)), because the module creates it on apply:

      ```sh
      tofu import module.github.github_repository.current "$(basename -s .git $(git remote get-url origin))"
      ```

   2. `tofu plan`, and include what it wants to change (repo settings, Actions permissions, secrets) in the report. Check its output for stale secrets (see Stale secrets). Do NOT run `tofu apply`.

   None of this setup triggers a release: `ci.yml` runs only on pushes/PRs, `release.yml` is `workflow_dispatch`-only, and `update-deps.yml` runs weekly (or manual dispatch) and only ever opens a PR.
5. `just lint` until clean. Apply autofixes, fix remaining oxlint errors in the code (suppressing only per the order in fact 2), resolve knip findings (unused deps/exports/files), fix actionlint findings in the repo's workflows, and re-run. Expect formatting churn (markdown reflow from oxfmt, plus `just --fmt` and `tofu fmt`), and expect the import rules to need more than one pass to converge. The base enables oxlint's `typeAware` and `typeCheck` options, so oxlint-tsgolint reports TypeScript errors too, and a clean lint is a full type check (this is why there's no standalone `tsc --noEmit` script).

   knip's **configuration hints** (e.g. a `project`/`ignore` pattern that no longer matches anything) do not fail the run: knip prints them and still exits 0, so `just lint` looks clean. Treat them as errors anyway. Fix each one (usually by dropping a stale entry from `knip.ts`), and if one can't be resolved, surface it in the report instead of letting it pass silently.

### Stale secrets

`no_unmanaged_secrets` is a tofu `check` block, so an undeclared Actions secret on the repo shows up as a **warning**, not a failure: the plan still succeeds, and the warning alone doesn't make `just tofu-check` fail. Its `-compact-warnings` also hides the message, so read the output of the plain `tofu plan` from Apply step 4 for `Unmanaged secrets on '<repo>': <NAMES>`. For each name:

- If the repo genuinely needs it, declare it via the `actions_secrets` flags or `additional_secrets` (its value must exist in `~/homelab/secrets/credentials.yaml` under the lowercased name).
- Otherwise it's stale (the common case is a leftover `NPM_CI_TOKEN` on a `"private": true` repo). Do NOT delete it. List it in the report with the command for me to run: `gh secret delete <NAME> --repo <owner/name>`. The exception is a secret the plan also destroys (one `github.tofu` used to declare, like a dropped `NPM_CI_TOKEN = true`): my apply removes it, so report it as a planned change with no delete command.

### Verify green

Without releasing:

- `just` (bare) lists the base recipes and then prints the commit types, confirming the justfile import resolves. The reusable CI depends on this (`ci.yml` runs `just build`).
- `just build` passes: lint (including `bun test`, which passes with no tests via `--pass-with-no-tests`), then tsdown (with publint for a built published package, and attw for a typed library), then `tofu-check`. While the plan has pending changes `tofu-check` fails (see Invariants): confirm everything before it passed, and note it. The output has no knip configuration hints (they exit 0, so check the output, not just the exit code).
- A second `just build` changes no files (compare `git status --porcelain` and `git diff` before and after). Every untracked file `git status` lists is one to commit, and build output is gitignored. CI's `build` fails on either.
- `tofu plan` runs without errors and shows only the expected changes. It shows no `Unmanaged secrets` warning, or every name it shows is either declared or in the report.
- The Actions secrets match the Actions secrets table for this repo: confirm pending additions and removals in the `tofu plan` output, and the ones already applied with `gh secret list --repo <owner/name>`.
- `.git/hooks/pre-commit` and `.git/hooks/commit-msg` are lefthook-managed (their contents reference lefthook), and `lefthook validate` succeeds.
- Release tooling sanity-checks: `release-it-git-cliff` is a direct devDependency and resolves from the repo root (`bun --print 'import.meta.resolve("release-it-git-cliff")'`), `git-cliff --config node_modules/@adamhl8/configs/dist/configs/cliff.base.toml --bumped-version` prints a version, and all three workflow files exist. Do NOT run `just release` / `just release-run` / `gh workflow run release.yml`.

### Report and stop

Leave every change uncommitted. The report covers:

- What changed.
- What was skipped (e.g. no tests).
- Decisions made (suppressions and overrides, manual fixes).
- What `tofu plan` wants to change, pending my apply, and whether `tofu-check` (and so `just build` and the pre-commit hook) fails until then.
- Stale Actions secrets, each with its `gh secret delete <NAME> --repo <owner/name>` command (except one the plan already destroys).
- The `bun.lock` `lockfileVersion` change, if any.
- Anything that couldn't be resolved (a knip configuration hint, a lint rule that needed judgment).
- Anything the spec doesn't cover, or where the repo seems to diverge on purpose (e.g. an archetype or package shape the tables don't list). Report it rather than forcing it into the nearest fit.

The lefthook `commit-msg` (commitlint) and `pre-commit` (`just build`) hooks are now installed, so a later commit needs a conventional message (`just commit-types` lists the allowed types), files `just build` leaves unchanged, and a drift-free tofu state.
