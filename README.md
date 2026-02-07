# arceos-readblk

A standalone block-device reader application running on [ArceOS](https://github.com/arceos-org/arceos) unikernel, with all dependencies sourced from [crates.io](https://crates.io). Demonstrates **VirtIO block device discovery, driver initialization, and disk I/O** across multiple architectures.

## What It Does

This application demonstrates device driver discovery and block device I/O:

1. **Driver discovery**: `axdriver::init_drivers()` probes PCI bus devices and initializes VirtIO-blk driver.
2. **Block device validation**: Asserts device type, name (`"virtio-blk"`), block size (512 bytes), and total disk size (64MB).
3. **Disk read**: Reads the first block (512 bytes) from the VirtIO disk.
4. **Child task**: Spawns a worker thread that parses bytes 3..11 of the boot sector as UTF-8, verifying the FAT OEM ID.
5. **CFS scheduling**: Uses preemptive CFS scheduler (`sched-cfs` feature) with timer interrupts.

### Key Concepts

| Concept | Description |
|---|---|
| VirtIO-blk | Paravirtualized block device over PCI bus |
| Device probing | PCI ECAM scan + VirtIO device negotiation |
| `axdriver` | ArceOS device driver framework |
| `BlockDriverOps` | Trait for read/write block operations |
| CFS scheduler | Timer-interrupt-driven preemptive scheduling |

## Supported Architectures

| Architecture | Rust Target | QEMU Machine | Platform |
|---|---|---|---|
| riscv64 | `riscv64gc-unknown-none-elf` | `qemu-system-riscv64 -machine virt` | riscv64-qemu-virt |
| aarch64 | `aarch64-unknown-none-softfloat` | `qemu-system-aarch64 -machine virt` | aarch64-qemu-virt |
| x86_64 | `x86_64-unknown-none` | `qemu-system-x86_64 -machine q35` | x86-pc |
| loongarch64 | `loongarch64-unknown-none` | `qemu-system-loongarch64 -machine virt` | loongarch64-qemu-virt |

## Prerequisites

- **Rust nightly toolchain** (edition 2024)

  ```bash
  rustup install nightly
  rustup default nightly
  ```

- **Bare-metal targets** (install the ones you need)

  ```bash
  rustup target add riscv64gc-unknown-none-elf
  rustup target add aarch64-unknown-none-softfloat
  rustup target add x86_64-unknown-none
  rustup target add loongarch64-unknown-none
  ```

- **QEMU** (install the emulators for your target architectures)

  ```bash
  # Ubuntu/Debian
  sudo apt install qemu-system-riscv64 qemu-system-aarch64 \
                   qemu-system-x86 qemu-system-loongarch64

  # macOS (Homebrew)
  brew install qemu
  ```

- **rust-objcopy** (from `cargo-binutils`, required for non-x86_64 targets)

  ```bash
  cargo install cargo-binutils
  rustup component add llvm-tools
  ```

## Quick Start

```bash
# install cargo-clone sub-command
cargo install cargo-clone
# get source code of arceos-readblk crate from crates.io
cargo clone arceos-readblk
# into crate dir
cd arceos-readblk
# Build and run on RISC-V 64 QEMU (default)
cargo xtask run

# Build and run on other architectures
cargo xtask run --arch aarch64
cargo xtask run --arch x86_64
cargo xtask run --arch loongarch64

# Build only (no QEMU)
cargo xtask build --arch riscv64
```

Expected output:

```
Load app from disk ...
Wait for workers to exit ...
worker1 checks head:
[mkfs.fat]

worker1 ok!
Load app from disk ok!
```

QEMU will automatically exit after printing the message.

## Project Structure

```
app-readblk/
├── .cargo/
│   └── config.toml       # cargo xtask alias & AX_CONFIG_PATH
├── xtask/
│   └── src/
│       └── main.rs       # build/run tool (disk image + QEMU with VirtIO-blk)
├── configs/
│   ├── riscv64.toml      # Platform config
│   ├── aarch64.toml
│   ├── x86_64.toml
│   └── loongarch64.toml
├── src/
│   └── main.rs           # Block device read + worker thread
├── build.rs              # Linker script path setup (auto-detects arch)
├── Cargo.toml            # Dependencies (axstd + axdriver with virtio-blk)
└── README.md
```

## Key Components

| Component | Role |
|---|---|
| `axstd` | ArceOS standard library (replaces Rust's `std` in `no_std` environment) |
| `axdriver` | Device driver framework — probes PCI bus, initializes VirtIO-blk driver |
| `axdriver_virtio` | VirtIO device abstraction for block devices |
| `axtask` | Task scheduler with CFS algorithm and preemption support |
| `paging` feature | Enables page table management for MMIO region mapping |
| `sched-cfs` feature | CFS scheduler with timer-interrupt preemption |

## How the Disk Image Works

The `xtask` tool creates a 64MB raw disk image (`target/disk.img`) with a FAT-like boot sector:

- **Bytes 0..3**: x86 jump instruction (`EB 3C 90`)
- **Bytes 3..11**: OEM ID `"mkfs.fat"` (8 bytes, valid UTF-8)
- **Bytes 11..12**: Bytes per sector (512, little-endian)
- **Remaining**: Zero-filled (sparse file)

This image is attached to QEMU as a VirtIO PCI block device (`-device virtio-blk-pci`).

## License

GPL-3.0-or-later OR Apache-2.0 OR MulanPSL-2.0
