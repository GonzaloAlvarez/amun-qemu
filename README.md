# Amun QEMU

**amun-qemu** is a plugin for [amun](https://github.com/GonzaloAlvarez/amun) that installs
the QEMU virtualization toolchain — the native-arch system emulator, `qemu-img`, UEFI
firmware, an ISO builder, and `socat` — on Debian/Ubuntu or Arch.

---

## Usage

```sh
amun qemu

# or push-mode from the amun repo:
cd ~/dev/amun && ./deploy <host> -P qemu -u <user> -i ~/.ssh/<key>
```

## What it does

1. Installs the arch-appropriate package set:

   | OS | x86_64 | aarch64 | common |
   |---|---|---|---|
   | Debian / Ubuntu | `qemu-system-x86`, `ovmf` | `qemu-system-arm` (ships `qemu-system-aarch64`), `qemu-efi-aarch64` (AAVMF) | `qemu-utils`, `genisoimage`, `socat` |
   | Arch | `qemu-system-x86`, `edk2-ovmf` | `qemu-system-aarch64`, `edk2-aarch64` | `qemu-img`, `cdrtools` (mkisofs), `socat` |

   Firmware lands at `/usr/share/OVMF/OVMF_CODE*.fd` / `/usr/share/AAVMF/AAVMF_CODE*.fd`
   (Debian) or `/usr/share/edk2/{x64,aarch64}/` (Arch).
2. Adds the connecting user to the `kvm` group when that group exists, so `/dev/kvm`
   is usable without sudo in a fresh session. The role makes **no assertion** that
   `/dev/kvm` exists — it converges identically on tart VMs, Pis, and containers.
3. Smoke-checks each installed tool (`qemu-system-<arch> --version`, `qemu-img`,
   `genisoimage|mkisofs`, `socat -V`).

## Variables

| Variable | Default | What |
|---|---|---|
| `qemu_manage_kvm_group` | `true` | add the user to the `kvm` group when it exists |
| `qemu_packages_debian_common` / `qemu_packages_debian_arch_map` | see defaults | Debian package lists |
| `qemu_packages_arch_common` / `qemu_packages_arch_arch_map` | see defaults | Arch package lists |

## Supported Platforms

| Platform | Method | Arch |
|---|---|---|
| Debian bookworm/trixie, Ubuntu | apt (`install_recommends: false`) | x86_64, aarch64 |
| Arch Linux | pacman | x86_64, aarch64 |

Linux-only by design — on macOS, install qemu via Homebrew directly.

## Testing — verification is the deliverable

Per homelab CLAUDE.md §14.1, four layers:

- **Layer 1 — molecule**: `./molecule test`. Docker Debian 12 container; on an
  Apple Silicon host this proves the **aarch64 Debian** branch (package set,
  idempotence, smoke checks). No `/dev/kvm` in Docker — KVM is not asserted here.
- **Layer 2 — `./test`**: **defaults to the cloud KVM run** — clones amun main,
  runs `amun/test debian --cloud --kvm -p qemu`, which provisions a **billable**
  EC2 devbox (Debian 13 amd64, m7i.large ≈ $0.10/h, ~15–25 min ≈ $0.05, destroyed
  automatically on exit) with real nested virtualization. Needs the `clouddevbox`
  CLI, the cn-socksnode proxy on `127.0.0.1:1055`, and an AWS profile:
  `./test --profile <p>` (or `CLOUDDEVBOX_PROFILE` / `AWS_PROFILE`).
  `DEBUG=1 ./test --profile <p>` drops into a shell on the box before teardown.
  `./test debian` runs the classic local tart path instead (TCG only — installs
  and smoke checks pass, but there is no `/dev/kvm` under macOS hvf).
- **Layer 3 — rpid11.lan**: `ssh rpid11.lan amun qemu` (real-Pi smoke; aarch64
  Debian branch, no KVM expected).
- **Layer 4 — `./verify [--host <h>] [--user <u>]`**: pure-shell PASS/FAIL
  checklist: binaries, firmware blob (multi-path glob), and a **conditional** KVM
  boot probe — when `/dev/kvm` exists, `timeout 10 qemu-system-<arch> -enable-kvm …`
  must be killed by the timeout (rc 124 = ran accelerated); hosts without
  `/dev/kvm` get a SKIP line, never a FAIL.

## Known quirks

- Arch aarch64 firmware: mainline Arch ships `edk2-aarch64`; older Arch Linux ARM
  snapshots may only have `edk2-armvirt`. Override `qemu_packages_arch_arch_map`
  if pacman can't resolve it (`./verify` globs both firmware paths).
- Debian renamed the OVMF blobs (`OVMF_CODE.fd` → `OVMF_CODE_4M.fd` in bookworm) —
  everything here globs `OVMF_CODE*.fd`, never a pinned filename.
- `socat` has no `--version`; the smoke check uses `socat -V`.

## License

GPLv3

Copyright (c) 2026 Gonzalo Alvarez
