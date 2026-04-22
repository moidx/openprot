# Adding a Target

A "target" in openprot is the collection of Bazel rules, constraints, and
glue code that teaches the build how to produce firmware for a specific
SoC/board. This page walks through the pieces — grounded in the two
targets that already exist, `target/earlgrey/` and `target/veer/`.

For the broader build-system overview see `build-system.md`; for the
crate-universe split that target code interacts with, see
`crate-universes.md`.

## What a target needs

A target directory is expected to provide, at minimum:

1. A Bazel `platform()` describing the CPU, ISA extensions, OS, and
   Pigweed kernel backends.
2. A `constraint_value()` identifying the target, plus a `TARGET_COMPATIBLE_WITH`
   helper in `defs.bzl` that gates target-only rules off the host build.
3. A `string_flag()` + per-interface `config_setting()`s for the
   silicon / fpga / emulator (or verilator) split.
4. Entry point, console backend, kernel config, and linker script template
   crates wired in through the platform's `flags_from_dict(...)` call.
5. Test/runner tooling (a `.bzl` rule + Python runner) for each interface
   that can be exercised locally.

## Platform declaration

The canonical example is `target/veer/BUILD.bazel:10-32`:

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
        KERNEL_DEVICE_COMMON_FLAGS | {
            "@pigweed//pw_kernel/arch/riscv:veer_pic": True,
            "@pigweed//pw_kernel/config:kernel_config": ":config",
            "@pigweed//pw_kernel/subsys/console:console_backend": ":console",
            "@pigweed//pw_log/rust:pw_log_backend": "@pigweed//pw_kernel:log_backend_basic",
        },
    ),
    visibility = [":__subpackages__"],
)
```

The earlgrey target (`target/earlgrey/BUILD.bazel:10-33`) is very similar
but adds `@pigweed//pw_build/constraints/riscv/extensions:Smepmp` and
swaps `timer_clint` for `timer_mtime`.

`KERNEL_DEVICE_COMMON_FLAGS` comes from `@pigweed//pw_kernel:flags.bzl`
and bundles the defaults every `pw_kernel`-hosted platform needs. Merge
your target-specific overrides on top via the `dict | { … }` operator.

## `defs.bzl` and `TARGET_COMPATIBLE_WITH`

`target/veer/defs.bzl`:

```starlark
TARGET_COMPATIBLE_WITH = select({
    "//target/veer:target_veer": [],
    "//conditions:default": ["@platforms//:incompatible"],
})
```

`target/earlgrey/defs.bzl` is the same shape with `target_earlgrey`
substituted. Every `rust_library`/`rust_binary` that only makes sense on
this target must set `target_compatible_with = TARGET_COMPATIBLE_WITH`.

Without it, a wildcard host build (`bazel build //...`) will try to
compile target-only code on the host. In the Caliptra integration path
specifically, this also silently pulls `@rust_caliptra_crates_host` into
a position where `@rust_caliptra_crates` was expected, producing TypeId
collisions. Setting `target_compatible_with` causes wildcard host builds
to skip the rule instead.

## `string_flag` + `config_setting` per interface

Each target exposes a `target_type` string flag with the interfaces it
supports:

```starlark
string_flag(
    name = "target_type",
    build_setting_default = "silicon",
    values = ["silicon", "fpga", "emulator"],  # veer
    # values = ["silicon", "fpga", "verilator"],  # earlgrey
)

config_setting(name = "silicon", flag_values = {":target_type": "silicon"})
config_setting(name = "fpga",    flag_values = {":target_type": "fpga"})
config_setting(name = "emulator", flag_values = {":target_type": "emulator"})
```

(`target/veer/BUILD.bazel:34-57`, `target/earlgrey/BUILD.bazel:35-58`.)

Kernel-config and entry-point `rust_library` rules then `select()` on
these `config_setting`s to pass `crate_features = [...]` that switch
address maps, timer frequencies, etc. — see `target/veer/BUILD.bazel:65-87`
for the pattern.

## The `target_veer` / `target_earlgrey` constraint value

```starlark
constraint_value(
    name = "target_veer",
    constraint_setting = "@pigweed//pw_kernel/target:target",
    visibility = [":__subpackages__"],
)
```

This constraint is what `TARGET_COMPATIBLE_WITH` keys off of, and it's
what `pw_kernel`'s own conditional logic uses to know which target is
being built. The `constraint_setting` is owned upstream, so the only
choice is the `name` — by convention `target_<name>`.

## Caliptra firmware binaries

Targets that build Caliptra firmware ELFs (ROM, FMC, Runtime, MCU ROM)
must go through `caliptra_firmware_binary` from
`third_party/caliptra/caliptra_build.bzl`. That macro wraps `rust_binary`
with `CALIPTRA_FIRMWARE_RUSTC_FLAGS`
(`caliptra_build.bzl:21-34`: `panic=abort`, `opt-level=s`,
`codegen-units=1`, `lto=fat`, `embed-bitcode=yes`, `-Wl,--relax`) plus the
supplied linker scripts, and fixes `target_compatible_with` to
`//third_party/caliptra/platforms:target_caliptra`.

```starlark
caliptra_firmware_binary(
    name = "caliptra_fmc",
    srcs = ["@caliptra_sw//:caliptra_sw_fmc_srcs"],
    crate_root = "@caliptra_sw//:fmc/src/main.rs",
    linker_scripts = [":fmc_memory_gen.x", ":link_gen.x"],
    map_file = "caliptra_fmc.map",
    # ...
)
```

(From `third_party/caliptra/caliptra-sw/BUILD.bazel:622-654`.) Do not
reach for a bare `rust_binary` when you mean a Caliptra firmware
artefact — the macro keeps the size-optimisation flag set and the
target-platform constraint in one place.

## Copy-genrule pattern for linker scripts

Linker scripts live as source files but `caliptra_firmware_binary` takes
labels that must be generated outputs (so they land in
`bazel-bin/`). The standard trick is a small copy `genrule`:

```starlark
genrule(
    name = "fmc_memory_x_gen",
    srcs = ["fmc_memory.x"],
    outs = ["fmc_memory_gen.x"],
    cmd = "cp $(location fmc_memory.x) $@",
)
```

(`third_party/caliptra/caliptra-sw/BUILD.bazel:517-522`.) Do the same for
`link.x`, `runtime_memory.x`, etc., and reference the `*_gen.x` outputs
from `linker_scripts = [...]`.

## Test/runner tooling

Each `target/<name>/tooling/` directory holds one or more rules that wrap
a Python runner and produce `bazel run`- and `bazel test`-friendly
targets. The veer pattern uses `caliptra_runner` and `caliptra_test` from
`target/veer/tooling/caliptra_runner.bzl`:

```starlark
caliptra_runner(
    name = "ipc_runner_emulator",
    interface = "emulator",
    tags = ["manual"],
    target = ":ipc",
)

caliptra_test(
    name = "ipc_runner_emulator_test",
    timeout = "long",
    interface = "emulator",
    target = ":ipc",
)
```

(`target/veer/ipc/user/BUILD.bazel:68-80`.) Both rules share the same
implementation; `caliptra_runner` is `executable = True`, `caliptra_test`
is `test = True`. The rule applies a `_target_type_transition` that pins
`//target/veer:target_type` to match the `interface` attribute
(`caliptra_runner.bzl:8-20`), so a single `:ipc` system-image target can
be exercised under each interface by instantiating the runner once per
interface.

## Checklist for a new target

1. Create `target/<name>/{BUILD.bazel,defs.bzl}`.
2. Declare `constraint_value("target_<name>")`, `TARGET_COMPATIBLE_WITH`,
   `string_flag("target_type")`, and per-interface `config_setting`s.
3. Declare the `platform("<name>")` with
   `flags_from_dict(KERNEL_DEVICE_COMMON_FLAGS | {…})`.
4. Add the entry, console, config, and linker-script `rust_library` rules,
   each with `target_compatible_with = TARGET_COMPATIBLE_WITH`.
5. If building Caliptra firmware, use `caliptra_firmware_binary` and copy
   linker scripts via genrule.
6. Add a tooling subpackage with a runner/test rule for every interface
   you want to exercise.
7. If the target ships verilator tests, tag them `verilator` and plumb
   them through a new workflow entry (see `workflows-reference.md`).
8. Update `workflows.json` or your local invocations as needed — wildcard
   builds will skip target-only rules via `TARGET_COMPATIBLE_WITH`.

A new target does not need its own `SUMMARY.md` entry or design doc to
build; docs live alongside the target only when something target-specific
deserves prose.
