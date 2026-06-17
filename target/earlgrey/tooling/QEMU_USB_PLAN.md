# Running the earlgrey USB tests under QEMU (with host enumeration)

Implementation plan for running `target/earlgrey/tests/usbdev` and
`target/earlgrey/tests/usbserial` under the lowRISC QEMU `ot-earlgrey`
machine, including enumerating the emulated USB device from a host.

> **Status:** plan / handoff. No code from this plan has been landed yet.
> Authored on branch `claude/modest-bardeen-2n95om`.

## Why this document exists

The USB tests currently only declare `verilator`, `hyper310`, `hyper340`, and
`teacup` (silicon) variants — there is no `qemu` variant (compare
`tests/ipc/user/BUILD.bazel`, which has `ipc_runner_qemu_test`). The QEMU
launcher (`qemu_start.sh`) explicitly strips USB. This plan adds a QEMU USB
path and a software "host" that performs real enumeration.

It must be executed on a full-network machine (workstation), because:

- The Claude Code web environment's egress allowlist blocks
  `pigweed.googlesource.com`, `bcr.bazel.build`, `static.crates.io`,
  `files.pythonhosted.org`, etc., so Bazel cannot build
  `//third_party/qemu:qemu-system-riscv32`.
- Phase 3 (real host-OS enumeration) needs the `vhci-hcd` / `vudc` kernel
  modules and the `usbip` userspace, which sandboxed containers lack.

## How QEMU models `usbdev` (lowRISC fork, tag `v10.2.0-2026-01-15`, branch `ot-10.2.0`)

From `hw/opentitan/ot_usbdev.c` and `hw/riscv/ot_earlgrey.c`:

- `usbdev` is a real device in `ot-earlgrey` (base `0x40320000`, PLIC IRQ
  135+). It is **not** on a QEMU USB bus and does **not** use libusb/usbredir/
  USB-IP. The upstream doc (`docs/opentitan/ot_usbdev.md`) explicitly says
  USB/IP and usbredir are "too high-level for the purpose of emulating and
  testing a low-level UDC driver."
- It exposes two **chardev** backends plus a vbus property:
  - `chardev-cmd` — newline-terminated text: `vbus_on` / `vbus_off`.
  - `chardev-usb` — a binary "UDCX" protocol (the host side of a UDC).
  - `vbus-override` (bool, default false). Normal mode: vbus = (cmd gate) AND
    (server `VBUS_ON`). Override mode: vbus = cmd gate only.
- The earlgrey machine wires these **by chardev ID** — the IDs are
  load-bearing and must match exactly:
  - `ibex_get_chardev_by_id("usbdev-cmd")`  -> `chardev-cmd`
  - `ibex_get_chardev_by_id("usbdev-host")` -> `chardev-usb`
  If the chardevs are absent, the device is created but left unconnected.
- Note: the firmware's `pinmux UsbdevSense = ConstantOne` only affects OT's
  pinmux selection; it does **not** satisfy QEMU's vbus model. Vbus must come
  from the cmd chardev or `vbus-override=on`.

> ⚠️ Upstream `ot_usbdev.md` opens with: "the USBDEV driver is still in
> development and not expected to work at the moment." Phase 0 must validate
> this empirically before investing in test code.

### UDCX wire protocol (from `ot_usbdev.c`)

12-byte header, little-endian, payload follows when `size != 0`:

```c
struct OtUsbdevServerPktHdr { uint32_t cmd; uint32_t size; uint32_t id; };

enum OtUsbdevServerCmd {
    INVALID, HELLO, VBUS_ON, VBUS_OFF, CONNECT, DISCONNECT, RESET, RESUME,
    SUSPEND, SETUP, TRANSFER, COMPLETE, CANCEL,
};

/* HELLO payload (handshake first, before anything else) */
struct Hello { char magic[4] /* "UDCX" */; uint16_t major /*1*/; uint16_t minor /*0*/; };

/* SETUP payload (control transfers) */
struct Setup { uint8_t address; uint8_t endpoint; uint8_t rsvd[2]; uint8_t setup[8]; };

/* TRANSFER payload header (ep&0x80 = IN; flags&1 = ZLP; OUT data follows) */
struct Transfer { uint8_t address; uint8_t endpoint; uint16_t packet_size;
                  uint8_t flags; uint8_t rsvd[3]; uint32_t transfer_size; };

/* COMPLETE payload (status: 0=SUCCESS,1=STALLED,2=CANCELLED,3=ERROR) */
struct Complete { uint8_t status; uint8_t rsvd[3]; uint32_t transfer_size; };

/* CANCEL payload */
struct Cancel { uint32_t xfer_id; };
```

The external peer plays the **host**: `HELLO -> VBUS_ON -> CONNECT -> RESET`,
then enumerates with `SETUP`/`TRANSFER` and observes `COMPLETE`.

## Phase 0 — Gate: confirm `ot-usbdev` works in the pinned build

```bash
bazelisk build //third_party/qemu:qemu-system-riscv32
QEMU=$(bazelisk cquery --output=files //third_party/qemu:qemu-system-riscv32 2>/dev/null)
"$QEMU" -M ot-earlgrey -device help 2>&1 | grep -i usbdev      # expect ot-usbdev
# confirm the global parses without error:
"$QEMU" -M ot-earlgrey -global ot-usbdev.vbus-override=on -S -display none -monitor none &
```

If this fails, bump the QEMU pin (`third_party/qemu/extensions.bzl`) or use a
local source override (`--override_repository`, see `third_party/qemu/setup.md`)
before continuing.

## Phase 1 — QEMU boot/init smoke test

Goal: run the existing `:usb` image under a new `qemu` interface and confirm
the firmware reaches `RUNNING` and initializes `usbdev` without faulting (watch
UART; keep `-d guest_errors,unimp` clean). No enumeration yet.

1. `target/earlgrey/tooling/qemu_start.sh` — add alongside the UART/monitor
   chardevs (IDs must be exactly `usbdev-host` / `usbdev-cmd`):

   ```sh
   "-chardev" "socket,id=usbdev-host,path=${QEMU_USB_SOCKET},server=on,wait=off"
   "-chardev" "socket,id=usbdev-cmd,path=${QEMU_USBCMD_SOCKET},server=on,wait=off"
   "-global" "ot-usbdev.vbus-override=on"   # smoke-test only; drop in Phase 2
   ```

   Add `QEMU_USB_SOCKET` / `QEMU_USBCMD_SOCKET` to the mandatory-var checks.

2. `target/earlgrey/tooling/qemu_runner.py` — create the socket paths in the
   tempdir and pass them via env (mirror `uart0.sock` / `monitor.sock` around
   lines 282-305). For the smoke test, optionally connect `usbdev-cmd` and send
   `vbus_on\n`.

3. `target/earlgrey/tests/usbdev/BUILD.bazel` and `.../usbserial/BUILD.bazel` —
   add a `qemu` variant (copy `ipc_runner_qemu_test`):

   ```python
   opentitan_test(
       name = "usb_qemu_test",
       timeout = "moderate",
       interface = "qemu",
       tags = ["qemu"],
       target = ":usb",
   )
   ```

## Phase 2 — Host-side UDCX server -> software enumeration (closes `TODO(cfrantz)`)

Write `target/earlgrey/tooling/usbdev_server.py` that connects to the
`usbdev-host` socket and performs standard enumeration as the host:

```
HELLO -> VBUS_ON -> CONNECT -> RESET
      -> GET_DESCRIPTOR(device) -> SET_ADDRESS
      -> GET_DESCRIPTOR(config + strings) -> SET_CONFIGURATION
```

Then assert VID `0x18d1` / PID `0x503a` and the expected descriptors. For
`usbserial`, additionally exercise the CDC-ACM bulk loopback. Print the
runner's success line on success.

Wire it into `qemu_runner.py`: launch the server (thread or subprocess) after
QEMU is up, and route its verdict into `_Watcher` (around line 368). Add the
server to the qemu runfiles in `opentitan_runner.bzl` (`_QEMU_ATTRS`, ~line
178) and pass a `--usb-server` flag in the generated run script (qemu branch,
~line 45). Drop `vbus-override=on` so the real server drives vbus.

> The test firmware (`test_usb.rs`) loops forever and never prints `PASS` on
> its own, so the success signal must come from the server. Alternatively, add
> an "enumerated" log line to the firmware on `SET_CONFIGURATION` and match it.

## Phase 3 — Real host enumeration (`lsusb`), workstation-only, non-CI

Bridge the Phase-2 server to `usbip`'s virtual UDC so the real OS enumerates
the device:

1. `modprobe vhci-hcd` (and `vudc`).
2. Have the server export the QEMU-backed device over usbip.
3. `usbip attach` it into `vhci-hcd`; the device now appears in `lsusb` / as a
   `/dev/ttyACM*` (for `usbserial`).

This cannot be a hermetic Bazel test — mark any such target
`tags = ["manual", "local"]`. Gate it behind Phase 2 success.

## Key file references

| File | Anchor |
|---|---|
| `target/earlgrey/tooling/qemu_start.sh` | `:10` USB stripped; `:53` args array |
| `target/earlgrey/tooling/qemu_runner.py` | `:264` socket setup; `:368` `_Watcher` |
| `target/earlgrey/tooling/opentitan_runner.bzl` | `:45` qemu branch; `:178` `_QEMU_ATTRS` |
| `target/earlgrey/tests/{usbdev,usbserial}/BUILD.bazel` | `:88` test variants |
| `target/earlgrey/tests/usbserial/test_usb.rs` | CDC-ACM loopback firmware |
| `target/earlgrey/tests/ipc/user/BUILD.bazel` | `:115` working `qemu` test template |

## Upstream references

- lowRISC/qemu (branch `ot-10.2.0`): `hw/opentitan/ot_usbdev.c`,
  `hw/riscv/ot_earlgrey.c`, `docs/opentitan/ot_usbdev.md`.
- Linux usbip tools (`vhci-hcd` / `vudc`) for Phase 3.
