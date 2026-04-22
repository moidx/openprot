# VeeR-EL2 Target Bringup Notes

`target/veer/` is openprot's Caliptra VeeR-EL2 target — `rv32imc`,
`pw_kernel`-based, exercised primarily on the Caliptra emulator. This
page collects the facts that are specific to VeeR and the Caliptra
integration around it.

For general target-bringup mechanics see `adding-a-target.md`; for
`pw_kernel` app authoring see `pw-kernel-guide.md`.

## Platform constraints

The veer platform declares (`target/veer/BUILD.bazel:10-32`):

- `@platforms//cpu:riscv32`, `@platforms//os:none`.
- RISC-V extensions `I`, `M`, `C` — and explicitly **`A.not`**. The VeeR
  core in Caliptra does not implement the Atomics extension, so the
  kernel and any userspace code targeting veer must avoid LL/SC and
  atomic RMW ops.
- `@pigweed//pw_kernel/arch/riscv:timer_clint` as the timer backend
  (earlgrey uses `timer_mtime`).
- `@pigweed//pw_build/constraints/rust:no_std`.

Caliptra firmware (ROM, FMC, Runtime) is built against a separate
platform at `//third_party/caliptra/platforms:caliptra` (see
`caliptra_build.bzl:243`), not the veer platform. The veer platform
only covers the MCU side.

## Kernel config

`target/veer/config.rs` sets:

- `SYSTEM_CLOCK_HZ` — `100_000_000` on silicon, `1_000_000` on
  emulator, `10_000_000` on FPGA (marked FIXME).
- `Timer` — CLINT-compatible timer at `TIMER_BASE = 0x2100_0000`, with
  `MTIME_REGISTER = 0x2100_00e4` and `MTIMECMP_REGISTER = 0x2100_00ec`.
  Driven off `MTIME_HZ = SYSTEM_CLOCK_HZ`.
- `PMP_ENTRIES = 16`, with all 16 available to userspace.
- `PIC_BASE_ADDRESS = 0x6000_0000`, `MAX_IRQS = 256`.
- `MTVEC_TABLE` — vectored exception mode, entries provided by a
  `global_asm!` table at `target/veer/entry.rs:40-82`.

## VeeR PIC

The interrupt controller is Caliptra's VeeR Programmable Interrupt
Controller (PIC). The driver lives upstream at
`@pigweed//pw_kernel/arch/riscv/veer_pic.rs` and is pulled in by the
`veer_pic` constraint in the platform declaration.

Key behaviors to be aware of:

- `early_init` disables every IRQ (source 0 is reserved), sets per-IRQ
  priority to `IRQ_PRIORITY = 1` (lowest non-zero), sets global
  `GLOBAL_PRIORITY = 0` so any enabled IRQ can fire, and enables
  `mie::mext`. (`veer_pic.rs:256-285`.)
- `pub fn interrupt()` is the top-level M-mode handler. It writes
  `MEICPCT` to latch the claim, reads the IRQ number out of
  `MeiHAP.claimid`, disables that source, clears the gateway, and
  dispatches through the static `PW_KERNEL_INTERRUPT_TABLE`. The
  handler is expected to re-enable its source via `interrupt_ack`
  (`veer_pic.rs:214-246`).

### Known gaps in the VeeR PIC driver

The driver is intentionally minimal. Things worth knowing when deciding
what to put on an interrupt vs polling:

- **No MEIVT fast-dispatch.** VeeR can offload first-level dispatch to
  hardware via the MEIVT pointer table; the pw_kernel driver uses a
  software dispatch through `PW_KERNEL_INTERRUPT_TABLE` instead
  (`veer_pic.rs:210-246`). This costs a few extra cycles per IRQ and
  means the table is compile-time static.
- **No userspace-facing per-IRQ priority API.** `set_interrupt_priority`
  exists (`veer_pic.rs:395-398`) but is only called from `early_init`,
  and no syscall surfaces it. Every IRQ runs at the same priority.
  Userspace that cares about ordering must serialize through a single
  interrupt object or a helper process.
- **No shared-IRQ fan-out.** Enforced one level up in
  `system_generator` — see `pw-kernel-guide.md` for the details. On
  VeeR specifically this is extra relevant because the PIC driver
  disables the source on entry and re-enables it only via
  `interrupt_ack`; two consumers cannot share one line.

## Running on the Caliptra emulator

Each `system_image` target in `target/veer/` gets a pair of runner rules
via `caliptra_runner` / `caliptra_test`:

```starlark
caliptra_runner(
    name = "measure_syscall_latency_emulator",
    interface = "emulator",
    tags = ["manual"],
    target = ":measure_syscall_latency",
)

caliptra_test(
    name = "measure_syscall_latency_emulator_test",
    timeout = "long",
    interface = "emulator",
    tags = ["exclusive"],
    target = ":measure_syscall_latency",
)
```

(`target/veer/syscall_latency/BUILD.bazel:97-112`.) The rules live in
`target/veer/tooling/caliptra_runner.bzl`; the Python driver that
actually spawns the emulator is `caliptra_runner.py`.

The emulator command line (`caliptra_runner.py:121-148`) pins the
memory map that the kernel config assumes:

| Region | Offset | Size |
|--------|--------|------|
| MCU ROM | `0x8000_0000` | `0x8000` |
| DCCM | `0x5000_0000` | `0x4000` |
| SRAM (flash+ram from system.json5) | `0x4000_0000` | `0x8_0000` |
| PIC | `0x6000_0000` | — |
| I3C | `0x2000_4000` | `0x1000` |
| MCI | `0x2100_0000` | `0xe0_0000` |
| Mailbox | `0x3002_0000` | `0x28` |
| SOC regs | `0x3003_0000` | `0x5e0` |
| OTP | `0x7000_0000` | `0x140` |
| Lifecycle controller | `0x7000_0400` | `0x8c` |

These align with what `target/veer/config.rs` and the various
`system.json5` files use as `memory_mappings` addresses.

### Emulator tests must run exclusively

`caliptra_runner.py` hardcodes `--i3c-port=65534`
(`caliptra_runner.py:128`). Two emulator instances would collide on
that port, so every `caliptra_test` invocation that uses `interface =
"emulator"` must carry `tags = ["exclusive"]` to force Bazel to run it
serially. The `measure_syscall_latency_emulator_test` rule above is an
example; there is a comment in that BUILD file documenting the reason.

## Build/run recipes

- Build all veer kernel targets:

      bazel build --//target/veer:target_type=emulator \
          //target/veer/...

- Run one under the emulator:

      bazel run //target/veer/syscall_latency:measure_syscall_latency_emulator

- Run the full emulator test suite:

      bazel test //target/veer/...

  Exclusive tagging serializes the Caliptra-emulator-based tests
  automatically.

## When the map changes

If the Caliptra emulator bumps any of the offsets above (via an
`uprev`), three places have to agree:

- `caliptra_runner.py` command line.
- `target/veer/config.rs` (`PIC_BASE`, `TIMER_BASE`, any new MMIO).
- The `memory_mappings` block of every `system.json5` under
  `target/veer/` that references the changed region.

There is currently no generated source of truth tying these together;
the uprev workflow (`crate-universes.md`) validates the Cargo side but
not the emulator memory map.
