# Workflows Reference

`./pw` dispatches to named workflows defined in `workflows.json`. This
page documents what each named group does, which builds/analyzers it
invokes, and the mechanics of the `./pw` wrapper itself.

## How `./pw` works

`./pw` is a shell wrapper (see the `pw` script at the repo root). It
invokes Pigweed's workflow launcher under its own Bazel output base:

```bash
bazelisk \
  --output_base=${_PW_SCRIPT_DIR}/out/workflows_launcher \
  run \
  --symlink_prefix=${_PW_SCRIPT_DIR}/out/workflows_launcher/bazel- \
  --show_result=0 \
  //:pw -- "$@"
```

(`pw:35-41`.) The dedicated output base prevents the workflow-launcher
build from thrashing your primary `bazel-bin/` cache. `//:pw` resolves to
a `native_binary` pointing at `@pigweed//pw_build/py:workflows_launcher`
(`BUILD.bazel:6-9`).

The `--` separator before `"$@"` is load-bearing: without it, any
negative target pattern (`-//third_party/caliptra/...`) that `pw` passes
through to Bazel would be parsed as a Bazel flag. That's why
`workflows.json` also includes a trailing `--` in every
`bazel_default_notest` / clippy / ci_tests `args` list.

## Named groups

`workflows.json` defines four top-level groups under `"groups"`. Each
group is selected by its name as the single argument to `./pw`.

### `presubmit`

```
./pw presubmit
```

Quick check run locally before every commit and mirrored on CI. Invokes
two analyzers and one build:

- `format` — runs `bazel run //:format -- --check`. The `//:format`
  target is an alias to `@pigweed//pw_presubmit/py:format`
  (`BUILD.bazel:13-16`). Drops `--check` if you run `./pw format`
  directly and the launcher applies the fixes in place.
- `presubmit_checks` — runs `//presubmit:presubmit` (license headers,
  C/C++ include guards, and any other checks wired into
  `presubmit/presubmit.py`).
- `clippy` build — invokes Bazel with the clippy aspect:

      bazelisk build \
        --aspects=@rules_rust//rust:defs.bzl%rust_clippy_aspect \
        --output_groups=clippy_checks \
        -- //... -//third_party/caliptra/...

  (`workflows.json:38-57`.) The clippy aspect attaches to every
  `rust_library`/`rust_binary` in the given targets and emits a
  `clippy_checks` output group that Bazel requests explicitly; if any
  lint fires, the build fails.

### `default`

```
./pw default
```

The lightest wildcard build: compiles `//...` minus
`//third_party/caliptra/...`, applying `--build_tag_filters=-hardware,-disabled`.
No tests are executed (`no_test: true` in the driver options).

This is the fastest way to find out if your change compiles across the
tree without waiting for the CI test matrix. See `workflows.json:30-37`
for the builds and `:13-26` for the `bazel_default_notest` config.

### `ci`

```
./pw ci
```

Runs the test suite that CI runs on every pull request. Builds and tests
`//... -//third_party/caliptra/...` with
`--build_tag_filters=-hardware,-disabled,-verilator` and
`--test_tag_filters=-hardware,-disabled,-verilator`
(`workflows.json:59-80`). Uses `--test_output=streamed` so test output
appears as tests run.

`ci` is equivalent to `default` plus test execution plus the
`-verilator` filter. Hardware-gated and verilator-gated tests must run
via dedicated workflows (`upstream_pigweed`) on machines where the
simulator is available.

### `upstream_pigweed`

```
./pw upstream_pigweed
```

The bigger check used to validate that an upstream Pigweed bump didn't
break the Earlgrey verilator path. Runs both:

- `ci_tests` (same as `./pw ci`).
- `earlgrey_verilator_tests` — Bazel with `--build_tag_filters=+verilator`
  and `--test_tag_filters=+verilator` against `//target/earlgrey/...`
  (`workflows.json:82-101`). Requires a working verilator install; most
  developer machines will want to skip this group.

## Configs, builds, tools, analyzers

`workflows.json` has four top-level sections:

- **`configs`** — reusable Bazel argument sets. `bazel_default`
  adds `--test_output=all`; `bazel_default_notest` adds
  `--keep_going --build_tag_filters=-hardware,-disabled --` with
  `no_test: true`. Referenced by name from `builds` and `tools` via
  `use_config: "…"`.
- **`builds`** — named `bazel build`/`bazel test` invocations with a
  target list. Each build either picks a config by name (`use_config`)
  or defines its own inline `build_config`. Builds: `wildcard_build`,
  `clippy`, `ci_tests`, `earlgrey_verilator_tests`.
- **`tools`** — named Bazel run targets with extra analyzer args.
  `format` runs `//:format`; `presubmit_checks` runs
  `//presubmit:presubmit` as an `ANALYZER` (the launcher treats
  analyzers specially — they gate a group on exit code without
  contributing build artefacts). See `workflows.json:103-120`.
- **`groups`** — the user-facing entry points. Each group lists some
  combination of `analyzers` and `builds`, run in order. Adding a new
  group is a matter of adding a `groups` entry and pointing it at
  existing `builds`/`tools` or new ones.

## Raw `bazel` equivalents

If you prefer invoking Bazel directly, here are the commands each group
runs (minus the workflow launcher's progress reporting):

| Group | Equivalent |
|-------|-----------|
| `./pw default` | `bazelisk build --keep_going --build_tag_filters=-hardware,-disabled -- //... -//third_party/caliptra/...` |
| `./pw ci` | `bazelisk test --keep_going --build_tag_filters=-hardware,-disabled,-verilator --test_tag_filters=-hardware,-disabled,-verilator --test_output=streamed -- //... -//third_party/caliptra/...` |
| `./pw presubmit` clippy build | `bazelisk build --aspects=@rules_rust//rust:defs.bzl%rust_clippy_aspect --output_groups=clippy_checks -- //... -//third_party/caliptra/...` |
| `./pw presubmit` format check | `bazelisk run //:format -- --check` |
| `./pw upstream_pigweed` (extra) | `bazelisk test --keep_going --build_tag_filters=+verilator --test_tag_filters=+verilator --test_output=streamed //target/earlgrey/...` |

The `--` before target patterns is mandatory for the first three lines
because they contain a negative pattern (`-//third_party/caliptra/...`)
that Bazel would otherwise try to parse as a flag.

## Adding a new workflow

1. Add a new `builds` entry in `workflows.json` with the Bazel args
   and target list you want, or reuse an existing `build_config`.
2. If the workflow is analyzer-style (exit-code gate, no artefacts),
   add it to `tools` with `"type": "ANALYZER"` instead.
3. Add a `groups` entry listing the builds/analyzers in order.
4. Run `./pw <new-group-name>` to smoke-test it.

`./pw --help` and `./pw list` (workflow-launcher built-ins) enumerate
the available groups at runtime.
