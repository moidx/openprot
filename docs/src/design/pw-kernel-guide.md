# pw_kernel App Author's Guide

This page is a practical walkthrough for writing a `pw_kernel` userspace
app in openprot. For the broader list of what Pigweed provides, see
`pigweed-overview.md`; for the cross-process IPC pattern in particular,
see `pw-kernel-ipc.md`.

Worked examples: `target/veer/syscall_latency/`, `target/veer/ipc/user/`,
and `target/veer/threads/kernel/`.

## The system image model

A `pw_kernel` binary is a **system image** — a single firmware artefact
containing the kernel, one or more apps, and the codegen that wires them
together. The layout is declared in a JSON5 file and assembled by the
`system_image` Bazel macro:

```starlark
system_image(
    name = "measure_syscall_latency",
    apps = [":syscall_latency"],
    kernel = ":target",
    platform = "//target/veer",
    system_config = ":system_config",
    tags = ["kernel"],
)
```

(`target/veer/syscall_latency/BUILD.bazel:40-49`.) The image pulls the
kernel from `:target`, each app from the `apps` list, and the layout
from `:system_config` (a `filegroup` pointing at `system.json5`).

## `system.json5`

This file is the declarative source of truth for memory layout,
processes, threads, objects, and interrupts. A minimal single-app image
looks like:

```json5
{
  arch: { type: "riscv" },
  kernel: {
    flash_start_address: 0x40000000,
    flash_size_bytes: 65536,
    ram_start_address: 0x40040000,
    ram_size_bytes: 32768,
    interrupt_table: { table: {} },
  },
  apps: [
    {
      name: "syscall_latency",
      flash_size_bytes: 32768,
      ram_size_bytes: 4096,
      processes: [
        {
          name: "syscall latency",
          memory_mappings: [
            { name: "MCI", type: "device",
              start_address: 0x21000000, size_bytes: 0x100 },
          ],
          threads: [
            { name: "syscall_latency", stack_size_bytes: 4096 },
          ],
        },
      ],
    },
  ],
}
```

(`target/veer/syscall_latency/system.json5`.) Key points:

- `arch.type` picks which `pw_kernel` arch backend to codegen against.
- `kernel.flash_*` and `kernel.ram_*` set the linker-visible regions
  used by `target_linker_script` to render `target.ld.jinja`.
- Each `process` owns a PMP-protected window per `memory_mappings`
  entry; anything outside those mappings is inaccessible to userspace.
- Thread stacks are carved out of the owning process's `ram_size_bytes`.

See `target/veer/ipc/user/system.json5` for a two-process layout with
channel endpoints (covered in `pw-kernel-ipc.md`).

## `rust_app` and codegen

Apps are built with `rust_app` (from `@pigweed//pw_kernel/tooling:rust_app.bzl`):

```starlark
rust_app(
    name = "syscall_latency",
    srcs = ["main.rs"],
    codegen_crate_name = "app_syscall_latency",
    crate_name = "syscall_latency",
    edition = "2024",
    system_config = "@pigweed//pw_kernel/target:system_config_file",
    tags = ["kernel"],
    target_compatible_with = TARGET_COMPATIBLE_WITH,
    deps = [
        "//target/veer:config",
        "@pigweed//pw_kernel/syscall:syscall_user",
        "@pigweed//pw_kernel/userspace",
        "@pigweed//pw_log/rust:pw_log",
        "@pigweed//pw_status/rust:pw_status",
    ],
)
```

(`target/veer/syscall_latency/BUILD.bazel:13-38`.) The macro triggers
two code-generation steps:

1. A per-app codegen crate (`app_syscall_latency` here) is emitted
   containing `handle::<ObjectName>` constants for every object the
   app's processes declare in `system.json5`.
2. The app's entry points are rewritten by `#[entry]` /
   `#[process_entry]` attribute macros into the specific symbols the
   kernel expects to find at link time.

`codegen_crate_name` is what you `use` from the app source to pick up
the handle constants.

## `#[entry]`, `#[process_entry]`, `#[panic_handler]`

A single-process app uses `#[entry]` on its main function:

```rust
#![no_main]
#![no_std]
use userspace::entry;

#[entry]
fn entry() -> ! {
    // ...
    loop {}
}

#[panic_handler]
fn panic(_info: &core::panic::PanicInfo) -> ! {
    pw_log::error!("PANIC: syscall latency");
    loop {}
}
```

(`target/veer/syscall_latency/main.rs:45-56`.) The macro validates the
signature is `fn() -> !` and renames the function to the kernel-expected
entry symbol (`userspace_macro.rs:20-90`).

A multi-process app uses one `#[process_entry("<process_name>")]` per
process; the argument must match the `processes[*].name` in
`system.json5`. The name-to-symbol rewriting is at
`userspace_macro.rs:120-180`.

Every app must provide its own `#[panic_handler]` — there is no default
in `userspace`.

## Key syscalls

From `@pigweed//pw_kernel/userspace:userspace` (`syscall.rs`):

- `syscall::debug_nop()` / `syscall::debug_shutdown(result)` — trivial
  syscalls used for wiring and end-of-test signalling. `debug_shutdown`
  communicates an exit status back to the emulator / host runner.
- `syscall::object_wait(handle, signal_mask, deadline) -> Result<WaitReturn>` —
  blocks until one of the bits in `signal_mask` is set on the named
  object, or the deadline expires. `Instant::MAX` means wait forever.
- `syscall::interrupt_ack(handle, signal_mask)` — re-arm an interrupt
  object after its handler has observed the signal; without this the
  IRQ stays masked.
- `syscall::channel_transact(handle, send, recv, deadline)` — single-shot
  client RPC across an IPC channel (covered in `pw-kernel-ipc.md`).
- `syscall::channel_read` / `syscall::channel_respond` — the server half
  of the same IPC pattern.
- `syscall::debug_putc(c)` / `syscall::debug_log(bytes)` — console I/O,
  typically wrapped by `pw_log::info!` through the log backend.

All syscalls return `pw_status::Result<T>`. Errors map 1:1 onto
`pw_status::Error` variants (`InvalidArgument`, `Internal`,
`DeadlineExceeded`, …).

## Interrupt objects

Interrupts are declared per-process as `objects` of `type: "interrupt"`
in `system.json5`. Each object owns up to 15 IRQs, identified by name.
At codegen time the `system_generator` walks each interrupt object and:

- Assigns each named IRQ a `Signals::INTERRUPT_<letter>` bit (A at
  bit 16 through O at bit 30; see
  `@pigweed//pw_kernel/syscall/syscall_defs.rs:352-366`).
- Synthesizes a `handle::<OBJECT_NAME>` constant on the generated
  codegen crate, plus per-IRQ handler shim names (`interrupt_handler_<process>_<object>_<irq>`).
- Fills in `interrupt_table.ordered_table` for the target's linker
  script.

Userspace pattern for one interrupt object with two named IRQs:

```rust
use app_codegen::handle;
use userspace::syscall::{self, Signals};
use userspace::time::Instant;

loop {
    let wait = syscall::object_wait(
        handle::UART_INTERRUPTS,
        Signals::INTERRUPT_A | Signals::INTERRUPT_B,
        Instant::MAX,
    )?;

    if wait.pending_signals.contains(Signals::INTERRUPT_A) {
        // handle TX-empty IRQ
    }
    if wait.pending_signals.contains(Signals::INTERRUPT_B) {
        // handle RX-full IRQ
    }
    syscall::interrupt_ack(handle::UART_INTERRUPTS,
        Signals::INTERRUPT_A | Signals::INTERRUPT_B)?;
}
```

The interrupt bits are coalesced, not queued. If both IRQs fire between
two `object_wait` calls, a single wait returns with both bits set and
no count — you have to drive that from device state.

## Known limitations worth planning around

- **Signal coalescing, no per-event queuing.** If the same IRQ fires
  repeatedly before userspace `interrupt_ack`s it, those events merge
  into one pending signal. Drivers that need edge counts must read the
  underlying device register.
- **No shared-IRQ fan-out.** `system_generator` rejects any
  `system.json5` that assigns the same IRQ number to two interrupt
  objects (or appears twice in one object). The check is at
  `@pigweed//pw_kernel/tooling/system_generator/lib.rs:650-658` —
  `"IRQ {}={} in app {} object {} already handled."` — and is a hard
  codegen error, not a runtime one.
- **Fixed 15-IRQ ceiling per interrupt object.** Driven by the 15
  `INTERRUPT_A..INTERRUPT_O` signal bits. (`system_generator` currently
  checks `> 16` at `lib.rs:638-644`, which admits a 16-IRQ case that
  would overflow the signal space; keep interrupt objects at ≤15 IRQs
  to be safe.)
- **No DeferredCall equivalent.** Unlike Tock, there's no built-in way
  to re-enter the scheduler with a soft-interrupt handle later. Deferred
  work has to be re-triggered from an `object_wait` loop, typically by
  having the driver set a `Signals::USER` bit on an object the app
  already holds.

See `veer-target.md` for target-specific gotchas around the VeeR PIC
that drives all of this on `target/veer/`.
