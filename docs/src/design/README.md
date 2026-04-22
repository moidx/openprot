# Design

This section contains design documents that provide a detailed overview of the
design and implementation of the OpenProt project. These documents are intended
to provide guidance to developers and anyone interested in the internal workings
of the project.

## Documents

-   [**Pigweed Integration Overview**](./pigweed-overview.md): What openprot
    consumes from Pigweed (`pw_kernel`, `pw_log`, `pw_status`, toolchains,
    crate universe, the `./pw` launcher) and where each piece is pinned.
-   [**pw_kernel App Guide**](./pw-kernel-guide.md): Practical
    walkthrough for writing a `pw_kernel` userspace app — `system.json5`
    schema, `rust_app` codegen, entry-point macros, key syscalls,
    interrupt objects, and the limitations worth planning around.
-   [**pw_kernel IPC**](./pw-kernel-ipc.md): How to declare and use channel
    objects to communicate between two `pw_kernel` userspace processes.
    Worked example lives at `target/veer/ipc/`.
-   [**VeeR Target Bringup**](./veer-target.md): VeeR-EL2 specific facts
    — RISC-V extension set, kernel config, the VeeR PIC driver's known
    gaps, and the Caliptra-emulator runner.
