# Build System

openprot is a Bazel module driven through Pigweed's workflow launcher.
There is no Cargo workspace — everything under `//...` is built by
`rules_rust` targets declared in `BUILD.bazel` files. This page explains
the pieces that make that possible.

For the everyday command surface see `usage.md`. For the three Rust
dependency workspaces in particular see `crate-universes.md`.

## Bazel module

`MODULE.bazel` is the root of the build. The module declaration and its
direct dependencies:

```starlark
module(name = "openprot", version = "0.0.1")

bazel_dep(name = "bazel_skylib", version = "1.8.2")
bazel_dep(name = "pigweed")
bazel_dep(name = "platforms", version = "1.0.0")
bazel_dep(name = "rules_rust", version = "0.68.1")
bazel_dep(name = "rules_rust_mdbook", version = "0.68.1")
bazel_dep(name = "rules_platform", version = "0.1.0")
bazel_dep(name = "rules_python", version = "1.8.3")
bazel_dep(name = "ureg")
bazel_dep(name = "caliptra_deps", version = "0.0.0")
```

(`MODULE.bazel:4-17`.) `caliptra_deps` is a nested module at
`third_party/caliptra/` consumed via `local_path_override`; `pigweed` and
`ureg` are pinned to specific upstream commits via `git_override`
(`MODULE.bazel:18-33`).

## Bazel + `./pw`

The required Bazel version is pinned in `.bazelversion` (currently `8.5.1`).
Invoke Bazel through `bazelisk`, which auto-selects that version.

`./pw` is a shell wrapper around `bazelisk run //:pw -- "$@"` — it delegates
to Pigweed's workflow launcher. `workflows.json` defines the named groups
(`presubmit`, `default`, `ci`, `upstream_pigweed`). `workflows-reference.md`
(pending) documents each group; the short version is in `usage.md`.

## Toolchains

All toolchains come from Pigweed:

- Host C/C++: `@pigweed//pw_toolchain/host_clang:host_cc_toolchain_{linux,macos}`.
- RISC-V C/C++ for the veer target:
  `@pigweed//pw_toolchain/riscv_clang:riscv_clang_cc_toolchain_rv32imc`.
- Rust: `@pw_rust_toolchains//:all`, registered via the `pw_rust` module
  extension (`MODULE.bazel:35-37`) with the Rust toolchain revision pinned
  by `cipd_tag`.

`.bazelrc` sets `BAZEL_DO_NOT_DETECT_CPP_TOOLCHAIN=1` so no host toolchain
is auto-picked — the vendored Pigweed toolchains are authoritative on both
CI and developer machines.

## Crate universes

Three `crate_universe` workspaces govern Rust crate dependencies:

| Repo | Scope | Cargo workspace |
|------|-------|-----------------|
| `@rust_crates` | Cross-platform openprot crates | `third_party/crates_io/Cargo.toml` |
| `@rust_caliptra_crates` | Embedded Caliptra (rv32imc, no std/alloc) | `third_party/caliptra/crates_io/embedded/Cargo.toml` |
| `@rust_caliptra_crates_host` | Caliptra host tooling (needs std) | `third_party/caliptra/crates_io/host/Cargo.toml` |

`@rust_crates` is declared directly in the root `MODULE.bazel:46-63`; the
two Caliptra workspaces come from the nested `caliptra_deps` module at
`third_party/caliptra/MODULE.bazel:16-65`. The root `use_repo(crate,
"rust_caliptra_crates", "rust_caliptra_crates_host", "rust_crates")` call
lifts all three into the root namespace.

A separate override at `MODULE.bazel:66-78` reroutes Pigweed's own internal
`rust_crates` extension at `pw_rust_crates_ext` onto `@rust_crates`.
Upstream's default set does not currently support
`riscv32imc-unknown-none-elf`; the override is a workaround tracked by the
`TODO(cfrantz)` there, removable once Pigweed picks up a `rules_rust`
release that lifts the limitation.

See `crate-universes.md` for detail on when to edit each `Cargo.toml` and
the platform-switching alias layer used by Caliptra integration code.

## Global `embed-bitcode=yes`

`.bazelrc:44-51` forces `-C embed-bitcode=yes` on every Rust compile:

```
common --@rules_rust//:extra_rustc_flags=-Cembed-bitcode=yes
common --@rules_rust//:extra_exec_rustc_flags=-Cembed-bitcode=yes
```

`rules_rust` defaults to `embed-bitcode=no` for build speed, but
`//third_party/caliptra/caliptra-sw:caliptra_rom` uses `-C lto=fat` to match
upstream caliptra-sw's release profile so the linked ROM fits in its
96 KiB region, and `lto` is incompatible with `embed-bitcode=no`. Applying
the flag globally keeps the transitive `caliptra_*` rlibs consistent
without annotating every dep individually.

`caliptra_build.bzl`'s `CALIPTRA_FIRMWARE_RUSTC_FLAGS`
(`third_party/caliptra/caliptra_build.bzl:21-34`) reasserts
`embed-bitcode=yes` alongside `panic=abort`, `opt-level=s`,
`codegen-units=1`, `lto=fat`, and `-Wl,--relax` for the same reason.

## Target platforms

Each silicon/SoC target declares a Bazel `platform()` with constraint
values and pipeline flags. `target/veer/BUILD.bazel:10-32` is the canonical
example:

```starlark
platform(
    name = "veer",
    constraint_values = [
        ":target_veer",
        "@pigweed//pw_kernel/arch/riscv:timer_clint",
        "@pigweed//pw_build/constraints/riscv/extensions:I",
        "@pigweed//pw_build/constraints/riscv/extensions:M",
        "@pigweed//pw_build/constraints/riscv/extensions:C",
        "@pigweed//pw_build/constraints/riscv/extensions:A.not",
        "@pigweed//pw_build/constraints/rust:no_std",
        "@platforms//cpu:riscv32",
        "@platforms//os:none",
    ],
    flags = flags_from_dict(
        KERNEL_DEVICE_COMMON_FLAGS | { ... },
    ),
)
```

Per-target helpers live in `target/<name>/defs.bzl`. They export a
`TARGET_COMPATIBLE_WITH` constant that target-only `rust_library`/`rust_binary`
rules must set via `target_compatible_with = TARGET_COMPATIBLE_WITH`. Without
it, wildcard host builds attempt to compile target-only crates on the host
and fail (or silently pull in the wrong crate-universe workspace). See
`adding-a-target.md` (pending) for the full bringup checklist.

## Tag reference

Targets can carry tags that gate which environment builds/tests them. The
set used in this tree:

- `hardware` — requires real silicon; never built locally.
- `verilator` — requires a verilator simulator install.
- `disabled` — temporarily excluded.
- `manual` — never built by `//...`; only when explicitly named.
- `no-sandbox` — genrule steps that call toolchain binaries not exposed as
  proper Bazel tool targets (e.g. `llvm-objcopy`; see
  `caliptra_build.bzl`).
- `exclusive` — forces serial execution; used by Caliptra emulator tests.
- `kernel` — `pw_kernel`-bound system images and supporting targets.

Combine via `--build_tag_filters` (build-time) and `--test_tag_filters`
(test-time). `workflows.json` is the canonical source of which filter each
named workflow applies.

## Practical wildcard build

`bazel build //...` on its own attempts hardware-only, verilator-only, and
Caliptra host-tree targets that will fail locally. The practical invocation
mirrors the `ci_tests` workflow:

```bash
bazel build //... -//third_party/caliptra/... \
    --build_tag_filters=-hardware,-disabled,-verilator
```

`//third_party/caliptra/...` must be excluded on wildcard host builds
because `@rust_caliptra_crates` and `@rust_caliptra_crates_host` resolve
shared crates to different rlib identities; building into both on one host
configuration triggers cross-workspace TypeId mismatches. See
`crate-universes.md` for the full story.

Prefer `./pw ci` or `./pw default` — they apply the filters and the
exclusion automatically.
