# elembio-nextflow-configs

Custom Elembio-specific Nextflow configs, served over raw HTTP to pipelines at
launch time. Each pipeline resolves this repo via:

```groovy
custom_config_version = "main"
custom_config_base    = "https://raw.githubusercontent.com/Elembio/configs/${params.custom_config_version}"
```

Editing `main` is immediately live for every pipeline. To try a change first,
set `custom_config_version` to a branch or SHA in the launch params
(`custom_config_version: "feat-ebc-nfcore-layout"`), which reroutes every include
to that ref.

## Layout (nf-core institution + overlay)

Follows the [nf-core/configs](https://github.com/nf-core/configs) model:
institution profiles are shared across workflows; pipeline overlays add
per-workflow settings under the **same** profile name.

| Path | Role |
|---|---|
| `nfcore_custom.config` | Registers institution + optional pipeline-global stubs |
| `conf/ElembioCloud.config` | Shared EBC settings for all workflows (`cleanup`, `cliPath`, …) |
| `conf/<pipeline>_nf.config` | Optional pipeline-global stub (non-EBC). Empty today |
| `pipeline/<pipeline>_nf.config` | Auto-included by that pipeline; router only |
| `conf/pipeline/<repo>/ElembioCloud.config` | Per-workflow EBC overlay (queues + process resources) |

```text
-profile ElembioCloud
    │
    ├─ nfcore_custom → conf/ElembioCloud.config              # shared
    └─ pipeline/<name>.config → conf/pipeline/<repo>/…       # overlay
```

EBC launch string (current and after this change):

```text
ElembioCloud,notaskdir,bases2fastq_nf
```

`bases2fastq_nf` / `cells2stats_nf` are no-op stubs kept for back-compat and for
future non-EBC pipeline-global options. They are not required for queues or
resources. `ElembioCloud,notaskdir` alone is sufficient.

Keep `pipeline/<name>.config` a router — environment names only:

```groovy
profiles {
    ElembioCloud {
        includeConfig "${params.custom_config_base}/conf/pipeline/bases2fastq-nf/ElembioCloud.config"
    }
}
```

Leaf files under `conf/pipeline/<repo>/` are **plain content** (no nested
`profiles{}`). The wrapper lives in the router / `nfcore_custom`.

## File names do not define profile names

Three separate identifiers, easily confused:

- **Profile name** — comes only from the `profiles { <name> { ... } }` block.
- **`conf/` file name** — cosmetic; referenced from `nfcore_custom.config`.
- **`pipeline/` file name** — a path contract. The consuming pipeline hardcodes
  the full URL, so the name must match what that repo asks for.

By convention `conf/<pipeline>_nf.config` names match the stub profile, with
`-` mapped to `_`. Do not assume the file name is doing the work.

## Guardrails

**Profile names must be `snake_case` (or quoted).** An unquoted hyphen does not
raise an error — Groovy parses it as subtraction and Nextflow silently
registers the wrong name:

```
profiles { bad-name { ... } }   ->   registers a profile called 'name'
```

**`ElembioCloud` in two places is intentional.** Same institution name in
`nfcore_custom` (shared) and `pipeline/<name>.config` (overlay) is the nf-core
pattern. Selecting `-profile ElembioCloud` once applies both. Do not invent a
second EBC profile name for queues.

**Shared `conf/ElembioCloud.config` must not set `process { withName: ... }` or
`resourceLimits`.** Those belong in the pipeline overlay. Same-selector
directives from both layers compete (last wins); keeping process blocks out of
shared avoids overwrite surprises.

**Adding a `pipeline/` file is safe; renaming or deleting one is not.** Pipelines
pin their revision per launch but fetch config from `main` at launch time.
Pipelines that use the strict-syntax ternary (Nextflow >= 26.04) abort on a
missing file instead of warning.

**Profiles replace, they do not merge** with selectors from configs included
before `profiles{}` (including the pipeline's `conf/base.config`). A profile
`withName` block that sets only `cpus` silently drops that process's `memory`,
`scratch`, and everything else. Pipeline ElembioCloud leaves must restate every
directive they need.

**`max_cpus` / `max_memory` / `max_time` are pipeline-wide.** They feed
`process.resourceLimits` (Nextflow >= 24.04). Setting `max_memory` below a
process's request silently clamps it.
