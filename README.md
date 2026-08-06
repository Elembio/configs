# elembio-nextflow-configs

Custom Elembio-specific Nextflow configs, served over raw HTTP to pipelines at
launch time. Each pipeline resolves this repo via:

```groovy
custom_config_version = "main"
custom_config_base    = "https://raw.githubusercontent.com/Elembio/configs/${params.custom_config_version}"
```

Editing `main` is immediately live for every pipeline. To try a change first,
set `custom_config_version` to a branch or SHA in the launch params
(`custom_config_version: "SW-27985"`), which reroutes every include to that ref.

## Layout

| Path | Loaded by | Holds |
|---|---|---|
| `nfcore_custom.config` | every pipeline | profile wrappers that pull `conf/*.config` |
| `conf/<name>.config` | every pipeline, via the wrapper above | queue routing, genome maps |
| `pipeline/<name>.config` | one pipeline only | that pipeline's deployment profiles |

`pipeline/<name>.config` is included directly by the pipeline's own
`nextflow.config`, so it is never seen by any other pipeline. That is why
`pipeline/bases2fastq_nf.config` and `pipeline/cells2stats_nf.config` can both
define an `ElembioCloud` profile with different resources without colliding, and
why profiles do not need an `EBC_<workflow>` prefix.

## File names do not define profile names

Three separate identifiers, easily confused:

- **Profile name** — comes only from the `profiles { <name> { ... } }` block. The
  file it lives in is irrelevant. `pipeline/bases2fastq_nf.config` defines a
  profile called `ElembioCloud`, not `bases2fastq_nf`.
- **`conf/` file name** — cosmetic. Referenced from exactly one line in
  `nfcore_custom.config`. Rename freely, update that line.
- **`pipeline/` file name** — a path contract. The consuming pipeline hardcodes
  the full URL, so the name must match what that repo asks for.

By convention `conf/` file names match the profile name they are wrapped in, and
both are the repo name with `-` mapped to `_`. Keep it that way, but do not
assume the file name is doing the work.

## Guardrails

**Profile names must be `snake_case`.** An unquoted hyphen does not raise an
error — Groovy parses it as subtraction and Nextflow silently registers the
wrong name:

```
profiles { bad-name { ... } }   ->   registers a profile called 'name'
```

Quote it (`'bad-name' { ... }`) or use underscores. Underscores preferred.

**Never put an `ElembioCloud` profile in `conf/` or `nfcore_custom.config`.**
That tier loads for every pipeline, so the definitions would collide. Deployment
profiles belong in `pipeline/<name>.config`.

**Adding a `pipeline/` file is safe; renaming or deleting one is not.** Pipelines
pin their revision per launch but fetch config from `main` at launch time, so
released tags keep requesting the old path forever. Pipelines that guard the
include with `try/catch` degrade to a warning; those using the strict-syntax
ternary (required by Nextflow >= 26.04, which forbids `try/catch` around
`includeConfig`) abort the launch outright:

```
ERROR ~ No such file or directory: Config file does not exist: .../pipeline/foo.config
```

Only rename a `pipeline/` file if the consuming pipeline has no released
revisions in use, and land the pipeline-repo change alongside it.

**Profiles replace, they do not merge.** A `withName:` block inside an active
profile replaces the same selector from any config the pipeline included before
its `profiles{}` block — including `conf/base.config`. A profile that sets only
`cpus` silently drops that process's `memory`, `scratch`, and everything else.
Deployment profiles must restate every directive they need.

**`max_cpus` / `max_memory` / `max_time` are pipeline-wide.** They feed every
`check_max()` call, not just the process you had in mind. Setting `max_memory`
below a process's request silently clamps it.
