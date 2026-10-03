# riscv-trust-monitor

A minimal RISC-V trusted monitor written in Rust.

The project explores how RISC-V privilege levels and Physical Memory Protection (PMP) can establish a small, hardware-enforced trust boundary between a trusted M-mode monitor and an untrusted U-mode payload.

## Goal

Build the smallest system that can demonstrate the following security property:

> An untrusted U-mode payload must not be able to read, modify, or execute memory owned by the trusted M-mode monitor.

The project intentionally grows in small milestones so that each mechanism can be understood at the ISA and hardware boundary rather than hidden behind a framework.

## Current Status

### M0 — Bare-metal RISC-V execution ✅

The first milestone is complete.

- [x] Build without an operating system using `#![no_std]` and `#![no_main]`
- [x] Target `riscv64imac-unknown-none-elf`
- [x] Define a custom `_start` entry point
- [x] Place the program at `0x80200000` with a linker script
- [x] Load the ELF directly into QEMU `virt`
- [x] Set the virtual CPU PC to the ELF entry point
- [x] Verify execution at `_start` with GDB
- [x] Inspect the generated RISC-V instructions

Observed with GDB:

```text
pc = 0x80200000 <riscv_trust_monitor::_start>
```

After continuing execution and interrupting the CPU inside the spin loop:

```text
pc = 0x80200006 <riscv_trust_monitor::_start+6>
```

This verifies the complete path:

```text
Rust source
    ↓
RISC-V machine code
    ↓
ELF entry point
    ↓
QEMU virtual CPU
    ↓
_start
```

## Project Structure

```text
riscv-trust-monitor/
├── .cargo/
│   └── config.toml
├── src/
│   └── main.rs
├── Cargo.toml
├── linker.ld
└── README.md
```

## Build

Install the RISC-V bare-metal target:

```bash
rustup target add riscv64imac-unknown-none-elf
```

Build the project:

```bash
cargo build
```

The resulting ELF is located at:

```text
target/riscv64imac-unknown-none-elf/debug/riscv-trust-monitor
```

You can inspect the ELF header with:

```bash
readelf -h target/riscv64imac-unknown-none-elf/debug/riscv-trust-monitor
```

The important fields are:

```text
Machine: RISC-V
Entry point address: 0x80200000
```

## Run with QEMU

The project currently uses QEMU's generic loader so that the ELF is loaded directly into memory and CPU 0 starts from the ELF entry point.

```bash
qemu-system-riscv64 \
  -machine virt \
  -bios none \
  -nographic \
  -device loader,file=target/riscv64imac-unknown-none-elf/debug/riscv-trust-monitor,cpu-num=0 \
  -S \
  -s
```

`-S` starts the virtual CPU in a paused state.

`-s` opens QEMU's GDB server on TCP port `1234`.

## Debug with GDB

In another terminal:

```bash
gdb-multiarch \
  target/riscv64imac-unknown-none-elf/debug/riscv-trust-monitor
```

Connect to QEMU:

```gdb
target remote localhost:1234
```

Inspect the current program counter:

```gdb
info registers pc
```

Expected result:

```text
pc = 0x80200000 <riscv_trust_monitor::_start>
```

Inspect the instructions at the current PC:

```gdb
x/5i $pc
```

Example output:

```text
0x80200000 <_start>:    j       0x80200002
0x80200002 <_start+2>:  fence   ...
0x80200006 <_start+6>:  j       0x80200002
```

The Rust code:

```rust
loop {
    core::hint::spin_loop();
}
```

has therefore been compiled into real RISC-V instructions executed by the virtual CPU.

## Memory Layout

The linker script currently starts the image at `0x80200000`.

```text
0x80200000
┌─────────────────────────┐
│ .text.entry             │
│ _start                  │
├─────────────────────────┤
│ .text                   │
│ executable code         │
├─────────────────────────┤
│ .rodata                 │
│ read-only data          │
├─────────────────────────┤
│ .data                   │
│ initialized data        │
├─────────────────────────┤
│ .bss                    │
│ zero-initialized data   │
└─────────────────────────┘
```

`0x80200000` is a memory-layout choice for this QEMU `virt` environment.

It is **not** a fixed RISC-V ISA entry address.

The RISC-V ISA defines processor behavior, instructions, privilege levels, and control registers. The platform determines where RAM, ROM, peripherals, and executable images live in the physical address space.

## Roadmap

- [x] **M0 — Bare-metal execution**

  - Rust `no_std`
  - custom `_start`
  - linker script
  - RV64 ELF
  - QEMU execution
  - GDB verification

- [ ] **M1 — UART MMIO**

  - write directly to the QEMU `virt` UART
  - understand volatile MMIO access
  - print the first monitor message

- [ ] **M2 — Machine-mode traps**

  - configure `mtvec`
  - handle exceptions
  - inspect `mcause`, `mepc`, and `mtval`

- [ ] **M3 — M-mode → U-mode transition**

  - configure `mstatus`
  - configure `mepc`
  - execute `mret`
  - run an untrusted U-mode payload

- [ ] **M4 — PMP isolation**

  - configure `pmpcfg`
  - configure `pmpaddr`
  - define trusted and untrusted memory regions

- [ ] **M5 — Deliberate isolation violation**

  - make the U-mode payload access monitor memory
  - trigger an access fault
  - inspect the resulting machine-mode trap

- [ ] **M6 — Threat model**

  - document trusted components
  - document untrusted components
  - define the security property
  - define what is explicitly out of scope

## Intended Trust Boundary

Later milestones will work toward this structure:

```text
                Trusted

        ┌──────────────────────┐
        │ RISC-V CPU / QEMU    │
        │                      │
        │ M-mode monitor       │
        └──────────┬───────────┘
                   │
                   │ PMP
                   │
        ───────────┼───────────
                   │
                   ▼

               Untrusted

        ┌──────────────────────┐
        │ U-mode payload       │
        └──────────────────────┘
```

The intended security property is:

```text
U-mode payload
      │
      │ attempts access
      ▼
monitor memory
      │
      ▼
PMP check
      │
      ├── allowed → continue
      │
      └── denied
            ↓
      access fault
            ↓
      M-mode trap handler
```

## Threat Model

The final trust model has not yet been implemented.

The intended model is:

### Trusted

- the RISC-V CPU architecture
- the QEMU CPU/device implementation used for the experiment
- the M-mode monitor

### Untrusted

- the U-mode payload

### Initially Out of Scope

- malicious hardware
- compromised QEMU
- physical attacks
- side-channel attacks
- debug interface attacks
- secure boot
- cryptographic firmware verification
- remote attestation
- multi-core isolation

These may become separate experiments later, but they are intentionally excluded from the first version.

## Learning Goal

The primary goal is not to build a production TEE or secure operating system.

The goal is to understand, from the bottom up:

```text
Rust abstraction
      ↓
unsafe / inline assembly
      ↓
RISC-V CSR operations
      ↓
privilege state
      ↓
PMP
      ↓
hardware-enforced isolation
```

The central question of the project is:

> What exactly must be trusted, and what mechanism enforces that trust boundary?
