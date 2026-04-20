# Crate Universes

openprot drives three independent `rules_rust` `crate_universe` workspaces.
This page explains what each one is for, which `Cargo.toml` to edit for a
given dependency, and the platform-switching alias layer that keeps shared
crates from colliding across them.

For the build-system-level overview see `build-system.md`.

## The three workspaces

| Repo label | Cargo workspace | Triples | Purpose |
|------------|-----------------|---------|---------|
| `@rust_crates` | `third_party/crates_io/Cargo.toml` | host + `riscv32imc-unknown-none-elf` | Cross-platform openprot crates. Also referenced by Pigweed's own `rust_crates` extension via the `pw_rust_crates_ext` override. |
| `@rust_caliptra_crates` | `third_party/caliptra/crates_io/embedded/Cargo.toml` | host + `riscv32imc-unknown-none-elf` | Caliptra-domain crates resolved with **embedded** features only (no `std`, no `alloc`, no `ecdsa/pem`). |
| `@rust_caliptra_crates_host` | `third_party/caliptra/crates_io/host/Cargo.toml` | host only | Caliptra host tools, rustcrypto crates that need `ecdsa/pem`, and the `caliptra-emu-*` chain. |

`@rust_crates` is declared in the root `MODULE.bazel:46-63`. The two
Caliptra workspaces are declared in the nested `caliptra_deps` module at
`third_party/caliptra/MODULE.bazel:16-65` and re-exported into the root
namespace by `use_repo(...)` in `MODULE.bazel:64`.

## Which `Cargo.toml` do I edit?

- A new dependency used by openprot application, HAL, or target crates →
  `third_party/crates_io/Cargo.toml`.
- A new dependency on a Caliptra crate (or upstream crate needed only on
  the Caliptra boundary) that is pulled into embedded rlibs (ROM, FMC,
  Runtime, MCU ROM) → `third_party/caliptra/crates_io/embedded/Cargo.toml`.
- A new dependency for Caliptra host tooling (`caliptra_emu_*`,
  `caliptra_rom_packager`, `caliptra_firmware_bundler`, etc.) →
  `third_party/caliptra/crates_io/host/Cargo.toml`.
- A crate needed on **both** sides of the embedded/host boundary → add it
  to both Caliptra `Cargo.toml` files **and** route all BUILD dependencies
  through the platform-switching alias layer (see below).

When in doubt, default to `default-features = false`. Host features like
`std`, `alloc`, `ecdsa/pem`, and `caliptra-lms-types/std` would otherwise
back-propagate through Cargo's global feature unification and push
`caliptra_rom` over the 96 KiB ROM budget.

## Why three and not one

Cargo feature unification is global across a workspace. If the Caliptra
embedded and host crates lived in a single workspace, host features would
leak into embedded rlibs. `rules_rust` 0.68.1's
`crate.annotation(crate_features = [...])` is additive-only — there is no
mechanism to subtract features after Cargo resolution — so a
single-workspace layout has no way to keep host features off embedded
rlibs.

Collapsing the split requires one of:

1. `rules_rust` gaining a subtractive `crate.annotation` field.
2. Upstream Caliptra crates splitting host-only features (`std`, `alloc`,
   `ecdsa/pem`) into separate crates.
3. The emulator `ROM_SIZE` budget being raised.

None are close. See `third_party/caliptra/crates_io/README.md` for the
authoritative write-up.

## Platform-switching alias layer

Shared crates (crates referenced from **both** Caliptra workspaces, which
resolve to distinct rlibs) must be routed through a platform-switching
alias layer in `third_party/caliptra/BUILD.bazel`:

```starlark
config_setting(
    name = "is_riscv32imc",
    constraint_values = [
        "@platforms//cpu:riscv32",
        "@platforms//os:none",
    ],
)

[
    alias(
        name = "crate_" + crate_name,
        actual = select({
            ":is_riscv32imc": "@rust_caliptra_crates//:" + crate_name,
            "//conditions:default": "@rust_caliptra_crates_host//:" + crate_name,
        }),
    )
    for crate_name in [
        "bitfield", "bitflags", "cfg-if", "log", "memoffset",
        "registers-generated", "serde", "serde_derive", "smlang",
        "tock-registers", "zerocopy", "zeroize",
    ]
]
```

(`third_party/caliptra/BUILD.bazel:16-50`.) Caliptra integration code that
depends on one of these shared crates must use the alias
(`//third_party/caliptra:crate_serde`, etc.), never the raw
`@rust_caliptra_crates*//:<name>` label. Pointing at a specific workspace
directly will work on one configuration and fail with a TypeId mismatch on
the other.

The list is maintained by hand. When a shared dependency is added to both
embedded and host `Cargo.toml` files, its crate name must also be added to
the loop above.

## Wildcard-build exclusion

`workflows.json` excludes `//third_party/caliptra/...` from every wildcard
build:

```
"targets": [ "//...", "-//third_party/caliptra/..." ]
```

(`workflows.json:34`, `:55`, `:78`.) This is the same
cross-workspace-collision problem in wildcard form: on a single host
configuration, `//...` includes targets built against both
`@rust_caliptra_crates` and `@rust_caliptra_crates_host`, producing
conflicting rlib identities. Build into `//third_party/caliptra/...`
explicitly (or via a platform-specific build like the veer or Caliptra
emulator tests) when you need those targets.

## The `uprev` tool

`bazel run //third_party/caliptra:uprev -- <subcommand>` manages Caliptra
version bumps. Subcommands (`third_party/caliptra/uprev.py:8-29`):

- `verify` — cross-checks `versions.bzl` against the `rev = "…"` values in
  both Caliptra `Cargo.toml` files and against the upstream
  `caliptra-mcu-sw` `Cargo.lock`.
- `bump <new-mcu-sw-sha>` — pins `caliptra_mcu_sw` to the given SHA,
  derives the matching `caliptra_sw` and `caliptra_cfi` revs from its
  `Cargo.lock`, and rewrites `versions.bzl`, `crates_io/embedded/Cargo.toml`,
  and `crates_io/host/Cargo.toml`.
- `latest` — resolves `caliptra-mcu-sw` HEAD, then bumps.
- `release <tag>` — resolves a `caliptra-mcu-sw` tag, bumps, and records
  `release_tag`.

The current pins travel through `--versions-json` (wired via the
`py_binary`'s `args` attribute) rather than being parsed from
`versions.bzl` by uprev itself. `caliptra_sw` and `caliptra_mcu_sw` git
commits are driven by `versions.bzl` via
`third_party/caliptra/extensions.bzl` — `uprev` no longer writes into
`MODULE.bazel`.

After an uprev, rerun `./pw ci` to regenerate both Caliptra `Cargo.lock`
files and validate the build.

## Troubleshooting

- **TypeId mismatch between `@rust_caliptra_crates` and
  `@rust_caliptra_crates_host`.** The consuming target depends on the raw
  workspace label for a shared crate instead of the alias. Switch to
  `//third_party/caliptra:crate_<name>`.
- **`caliptra_rom` over its 96 KiB budget.** A feature leak pulled `alloc`
  / `std` into an embedded rlib. Check for a new `default-features`
  dependency added to `embedded/Cargo.toml`; rerun with
  `default-features = false`.
- **`//...` wildcard build fails in `//third_party/caliptra/...`.** The
  wildcard exclusion was bypassed. Use `./pw ci` / `./pw default`, or pass
  `-//third_party/caliptra/...` explicitly.
