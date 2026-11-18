# AntOS Technical Documentation - B: Build & Operations

**Project:** AntOS - AI-Native Transport Operating System
**Document type:** Build Pipeline & Engineering Operations Reference
**Source of truth:** `AntOS/tooling/`, `AntOS/BUILD_STATUS.md`, `AntOS/docs/operations/`, `AntOS/platform/image/Containerfile`
**Generated:** 2026-08-25

---

## 1. Build Overview

AntOS is built as a **bootable container image** that is converted to a **qcow2 disk image** via `bootc-image-builder` (BIB). The pipeline is QEMU-first: every phase must be bootable and testable in QEMU before it is considered passed.

### Build flow

```mermaid
%%{init: {"flowchart": {"useMaxWidth": true}}}%%
flowchart LR
    classDef terminal fill:#1a1a2e,stroke:#0f0f1e,color:#fff
    classDef process fill:#4361ee,stroke:#3a56d4,color:#fff
    classDef artifact fill:#00b4d8,stroke:#0096c7,color:#fff
    classDef success fill:#2ecc71,stroke:#27ae60,color:#fff

    cf([Containerfile]) -->|podman build| ci(Container Image)
    ci -->|bootc-image-builder| qcow2(qcow2 Disk Image)
    qcow2 -->|run-qemu.sh| qemu(QEMU Boot)
    qemu --> gates([Phase Gate Scripts])
    class cf terminal
    class ci,qemu process
    class qcow2 artifact
    class gates success
```

### Key artifacts

| Artifact | Path | Purpose |
|---|---|---|
| Container image | `docker.io/antos/image:phase0` (x86_64) / `:phase0-aarch64` | Bootable container, input to BIB |
| qcow2 disk | `build/antos-phase1.qcow2` (x86_64) / `build/antos-phase1-aarch64.qcow2` | Bootable VM image |
| SHA256 checksum | `build/antos-phase1.qcow2.sha256` | Reproducibility verification |
| Image manifest | `build/image-manifest.json` | `podman inspect` output |
| SSH key | `build/ssh/antos` (ed25519) | Throwaway key baked into image |
| ISO image | via `build-iso.sh` | Installable ISO (x86_64/aarch64) |

---

## 2. Prerequisites

### Build host requirements

| Tool | Purpose | Fedora install |
|---|---|---|
| `podman` | Container build + BIB runner | `dnf install -y podman` |
| `qemu-img` | Disk image utilities | `dnf install -y qemu-img` |
| `qemu-system-x86_64` | x86_64 VM gate testing | `dnf install -y qemu-system-x86` |
| `qemu-system-aarch64` | aarch64 VM gate testing | `dnf install -y qemu-system-aarch64` |
| `edk2-aarch64` / `AAVMF` | UEFI firmware for aarch64 QEMU | `dnf install -y edk2-aarch64` |
| `ssh-keygen` | Throwaway SSH key generation | `dnf install -y openssh` |
| `python3` | Parse `CurrentVesion.json` | `dnf install -y python3` |
| `qemu-user-static` / binfmt | Cross-arch builds (x86_64 -> aarch64) | `dnf install -y qemu-user-static` |

### Source files

| File | Role |
|---|---|
| `platform/image/Containerfile` | Multi-stage build: Rust builder + Fedora bootc final image |
| `CurrentVesion.json` | Version metadata source of truth (Version, OSName, BuildVersion, ReleaseDate) |
| `tooling/image-builder/build.sh` | Main build entry point |
| `tooling/image-builder/bib-common.sh` | Shared BIB helpers (distro defs) |
| `tooling/image-builder/build-iso.sh` | ISO build entry point |
| `tooling/vm-runner/run-qemu.sh` | QEMU boot runner (x86_64 + aarch64) |

---

## 3. The Containerfile

`platform/image/Containerfile` is a multi-stage build based on ADR-0001 (Fedora/rpm-ostree), ADR-0002 (OSTree), ADR-0003 (Qt/QML), ADR-0004 (Smithay).

### Stage 1 - Builder (`quay.io/fedora/fedora:41 AS builder`)

Compiles all AntOS Rust services with Cargo dependency caching:

1. For each crate, copy only `Cargo.toml` first and build a dummy binary -> downloads/compiles all external dependencies once.
2. Remove dummy source, copy real source, rebuild -> only the local crate recompiles.

Crates built:
- `antos-shell-compositor` (Smithay Wayland compositor)
- `antos-shell-api-dbus` (Shell D-Bus API)
- `antos-policy-gateway` (Secure Action Gateway)
- `antos-command-center` (Search service)
- `antos-intelligence-dbus` (AI provider D-Bus)

Builder dependencies: `cargo`, `rust`, `libxkbcommon-devel`, `wayland-devel`, `libdrm-devel`, `libgbm-devel`, `pixman-devel`, `pkgconf`, `gcc`.

### Stage 2 - Final bootable image (`quay.io/fedora/fedora-bootc:41`)

Installs runtime packages, rebrands as AntOS, installs services and UI.

**Runtime packages:** NetworkManager, systemd-resolved, qemu-guest-agent, openssh-server/clients, sudo, libxkbcommon, wayland-utils, libdrm, libgbm, pixman, weston, weston-demo, dbus, dbus-tools, qt6-qtbase-gui, qt6-qtdeclarative, qt6-qtdeclarative-devel, qt6-qtwayland, python3-jsonschema, dialog.

**Branding steps:**
- Generates `/usr/lib/os-release` with `ID=fedora` / `ID_LIKE=fedora` (required by BIB/dnf parsing) but `NAME`/`PRETTY_NAME` = AntOS.
- `VERSION_ID` = integer Fedora base version (41) for BIB compatibility; product version stays in `VERSION`/`PRETTY_NAME`.
- Sets `/etc/dnf/vars/releasever` to base version.
- Replaces `/etc/issue`, `/etc/issue.net`, `/etc/fedora-release`, `/etc/redhat-release`, `/etc/hostname`, GRUB distributor, profile prompt.
- Installs branding assets (icons, wallpapers, logos) to hicolor icon theme, pixmaps, plymouth spinner watermark.

**User setup:**
- Creates `antos` user (UID 1000, groups `wheel,systemd-journal`).
- First-boot wizard (`antos-first-boot.service`) collects password interactively unless `CONSOLE_PASSWORD` build-arg is supplied.
- Optional SSH pubkey injection via `SSH_PUBKEY` build-arg.

**Service enablement:**
- `NetworkManager`, `systemd-resolved`, `qemu-guest-agent`, `sshd`, `antos-recovery-shell.service`, `antos-hardware-profile.service`.
- User session lingering for `antos` (compositor starts at boot without login).
- `antos-wayland-session.target` wires compositor, API, intelligence, command-center, shell-ui.
- Default target: `multi-user.target` (headless boot; compositor via user session).

---

## 4. Build Commands

### 4.1 Standard x86_64 build

```bash
cd AntOS
tooling/image-builder/build.sh
```

Output: `build/antos-phase1.qcow2` + `build/antos-phase1.qcow2.sha256`.

### 4.2 Architecture-specific build

```bash
# x86_64 (default)
tooling/image-builder/build.sh --arch x86_64

# aarch64 (requires qemu-user-static/binfmt on x86_64 host, or native ARM host)
tooling/image-builder/build.sh --arch aarch64
```

Output: `build/antos-phase1-aarch64.qcow2`.

### 4.3 Container-only build (skip qcow2)

```bash
tooling/image-builder/build.sh --skip-qcow2
```

Useful for iterating on the Containerfile without waiting for BIB.

### 4.4 ISO build

```bash
tooling/image-builder/build-iso.sh --arch <arch> [--no-test]
```

Produces an installable ISO with AntOS branding. See `docs/operations/feature-and-iso-rebuild-playbook.md` for the full branding checklist.

### 4.5 Build arguments (environment variables)

| Variable | Default | Purpose |
|---|---|---|
| `ANTOS_BUILD_LABEL` | `AntOS` | Build label for MOTD |
| `ANTOS_CONSOLE_PASSWORD` | (empty) | If set, pre-sets antos user password and skips first-boot wizard |
| `ANTOS_BASE_VERSION` | `41` | Fedora base version (for dnf vars / BIB distro def) |

Version metadata is read from `CurrentVesion.json`:
```json
{
  "Version": "1.0.0",
  "OSName": "AntOS",
  "BuildVersion": 1,
  "ReleaseDate": "2024-08-25"
}
```

---

## 5. QEMU Validation

### 5.1 Run a built image

```bash
# x86_64
tooling/vm-runner/run-qemu.sh build/antos-phase1.qcow2

# aarch64
tooling/vm-runner/run-qemu.sh --arch aarch64 build/antos-phase1-aarch64.qcow2
```

### 5.2 aarch64 QEMU configuration (ADR-0021)

For `aarch64`, `run-qemu.sh` uses:
- `qemu-system-aarch64`
- `-machine virt`
- `-cpu cortex-a72` (or `host` if KVM on ARM host)
- `-m 2048 -smp 2`
- `-bios /usr/share/AAVMF/AAVMF_CODE.fd` (AAVMF UEFI)
- virtio block and net

### 5.3 SSH access

The build generates a throwaway ed25519 key at `build/ssh/antos`. The public key is baked into the image for the `antos` user. Connect with:

```bash
ssh -i build/ssh/antos antos@<vm-ip-or-localhost-port>
```

---

## 6. Phase Gate System

AntOS uses a phased build where each phase has explicit gate criteria. A phase is not complete until its gate passes. `BUILD_STATUS.md` is the single source of truth for phase progress.

### 6.1 Gate script convention

- Each phase has a consolidated gate: `tooling/vm-runner/phase<N>-gate.sh`
- Sub-slices have individual gates: `tooling/vm-runner/phase<N>-<M>-gate.sh`
- Full regression: `tooling/vm-runner/antos-full-gate.sh` runs all Phase 0-N gates
- CI linting: `tooling/ci/antos-ci.sh`

### 6.2 Phase status (as of v1.2.0 + Phase 21)

| Phase | Status | Gate |
|---|---|---|
| 0 - Architecture & Toolchain | Passed | Reproducible minimal image boots in QEMU |
| 1 - Minimal AntOS Image | Passed | boot/reboot/shutdown stable, rollback works |
| 2 - Wayland Desktop Foundation | Passed | Usable graphical session |
| 3 - Design System & Core Shell | Passed | Consistent reusable UI |
| 4 - AntOS Core APIs | Passed | Contract tests across services |
| 5 - Secure Action Gateway | Passed | AI/automation cannot bypass gateway |
| 6 - Command Center | Passed | Useful without AI, no UI freeze on search |
| 7 - Local AI Foundation | Passed | AI provider removable, privacy gateway enforced |
| 8 - Explainable System Intelligence | Passed | No fabricated telemetry |
| 9 - Health / Infrastructure Intelligence | Passed | Monitoring does not destabilize |
| 10 - Automation | Passed | All privileged actions pass gateway |
| 11 - Smart Workspaces & Knowledge | Passed | Clear privacy controls |
| 12 - Security Guardian | Passed | Independent security review |
| 13 - Developer & Device Ecosystem | Passed | Integrations optional |
| 14 - Installer & Recovery UX | Passed | Install/update/rollback/recovery tested |
| 15 - Beta Hardening | Passed | No new architecture, all tests pass |
| **v1.0 - Baseline** | **Passed** | `antos-full-gate.sh` passes Phase 0-15 |
| 16 - Interaction & Accessibility | Passed | New input without breaking keyboard/mouse |
| 17 - Connectivity, Storage & Migration | Passed | P2P sync offline, migration preserves privacy |
| 18 - Security, Privacy & Trust | Passed | No false positives, AI safety enforced |
| **v1.1 - Release** | **Passed** | `antos-full-gate.sh` passes Phase 0-18 |
| 19 - Productivity, Ecosystem & Sustainability | Passed | Privacy-respecting |
| 20 - Advanced / Futuristic | Passed | Feature flags, architecture/security/ethics review |
| **v1.2 - Release** | **Passed** | `antos-full-gate.sh` passes Phase 0-20 |
| 21 - ARM64 / Small Form Factor | Passed | Multi-arch build, aarch64 QEMU, SFF profile |

### 6.3 Status definitions

- **Not Started**: No work accepted yet.
- **In Progress**: At least one sub-slice is active.
- **Blocked**: Waiting on a dependency or human decision.
- **Passed**: All gate criteria verified and documented.

### 6.4 Updating build status

At the end of every sub-slice, update the `Notes` column in `BUILD_STATUS.md` and mark the phase `Passed` only when the gate passes.

---

## 7. Atomic Updates & Rollback

### 7.1 Mechanism (ADR-0002)

AntOS uses **OSTree via rpm-ostree/bootc** for atomic, image-based system deployments.

- Deployments are content-addressed, immutable trees stored in `/ostree/repo`.
- A/B boot entries allow automatic fallback on failed boot.
- Health verification runs before a deployment is made default.
- Rollback cannot be blocked by a running application - the boot loader owns the previous entry.

### 7.2 Operator commands

```bash
# View current and previous deployments
bootc status

# Manually roll back to the previous deployment
bootc rollback
```

### 7.3 Verification

```bash
tooling/rollback/verify-rollback.sh
```

End-to-end rollback is tested via `tooling/vm-runner/test-rollback.sh` (Phase 1 gate).

### 7.4 Important distinction

| Concept | What it is | What it is NOT |
|---|---|---|
| OS deployment rollback | rpm-ostree A/B boot entry switch | Not a backup |
| Btrfs snapshots | Efficient local filesystem state | Not a backup; not the primary rollback mechanism |
| User backups | Separate user-data backup | Not OS rollback |

---

## 8. Recovery

### 8.1 Recovery target

`platform/image/config/antos-recovery.target` - a dedicated systemd target for recovery mode.

### 8.2 Recovery shell

`antos-recovery-shell.service` + `/usr/local/bin/antos-recovery-shell` - a serial console recovery shell enabled at boot.

### 8.3 Rollback test preparation

`/usr/local/bin/antos-prepare-rollback-test` - prepares the system for rollback testing.

### 8.4 First-boot wizard

If no `CONSOLE_PASSWORD` was supplied at build time, the `antos` account is locked and `antos-first-boot.service` runs on first boot to interactively collect a password. The service only runs when `/etc/antos-first-boot` exists.

---

## 9. Feature & ISO Rebuild Playbook

(`docs/operations/feature-and-iso-rebuild-playbook.md`)

### 9.1 Pre-flight checklist

1. Update `CurrentVesion.json` if the version changed.
2. Update `platform/image/Containerfile` or `platform/image/branding/` for any branding changes.
3. Run `tooling/ci/antos-ci.sh` to lint all scripts.
4. Ensure build host has all prerequisites (sec. 2).

### 9.2 Rebuild flow

```bash
# 1. Build the container + qcow2
tooling/image-builder/build.sh --arch <arch>

# 2. Build the ISO
tooling/image-builder/build-iso.sh --arch <arch> --no-test

# 3. Verify no "Fedora" strings leak
sudo grep -R -i "fedora" /tmp/antos-iso || echo "OK: no Fedora strings in ISO"
```

### 9.3 Known limitation: kernel version string

The running kernel reports `6.17.8-100.fc41.x86_64` via `uname -r`. This is a build-time kernel identifier, not a UI/shell string. Changing it requires rebuilding the kernel with a custom `LOCALVERSION`. It is acceptable to leave as-is.

### 9.4 Fast rebuild tips

- Use `--skip-qcow2` to iterate on the Containerfile without running BIB.
- Cargo dependency caching in the builder stage means source-only changes recompile only the local crate.
- The `docker.io/` prefix on the image tag ensures BIB resolves against local containers/storage instead of trying to pull from a registry.

---

## 10. CI

### 10.1 CI script

`tooling/ci/antos-ci.sh` - lints all shell scripts and validates structure.

### 10.2 CI workflow

`.github/workflows/ci.yml` - GitHub Actions CI configuration.

### 10.3 Cross-architecture CI

Per ADR-0021, CI needs either:
- An aarch64 runner, or
- `qemu-user-static` / binfmt support for periodic ARM builds.

**Known blocker:** aarch64 cross-build on WSL is blocked by upstream `bootc-image-builder` / `qemu-user-static` issues. The validated path is: x86_64 container build -> native ARM BIB -> ARM VM smoke test.

---

## 11. Hardware Profiles (Phase 21)

### 11.1 Hardware profile service

`antos-hardware-profile` (`services/devices/`) detects:
- Device-tree model on ARM SBCs
- DMI chassis type on x86_64 SFF/NUC systems
- QEMU `virt` machine for testing (with `--sff` override)

Emits `/run/antos/hardware-profile.json`.

### 11.2 SFF tuning profile

`services/devices/config/sff-tuning.json`:

```json
{
  "profile": "sff",
  "compositor_refresh_hz": 30,
  "cpu_governor": "powersave",
  "swapiness": 60,
  "disabled_services": [
    "antos-command-center.service",
    "antos-intelligence-dbus.service"
  ],
  "resource_hints": {
    "max_memory_mb": 4096,
    "max_cpu_cores": 4
  }
}
```

On SFF-class devices, the compositor refresh is capped at 30Hz, non-essential services (Command Center, Intelligence D-Bus) are disabled, and resource hints cap at 4GB RAM / 4 CPU cores.

### 11.3 Profile detection defaults

Default to `standard` unless DMI/DT clearly reports `embedded` or `pico`. This prevents false positives on x86_64 SFF/NUC systems.

---

## 12. Version Management

### 12.1 Source of truth

`CurrentVesion.json` is the single source of truth for version metadata:

```json
{
  "Version": "1.0.0",
  "OSName": "AntOS",
  "BuildVersion": 1,
  "ReleaseDate": "2024-08-25"
}
```

### 12.2 Build-info injection

The Containerfile writes `/usr/share/antos/build-info.json` with: `label`, `version`, `build_version`, `release_date`, `commit`, `timestamp`. Query with:

```bash
antos-build-info
```

### 12.3 os-release

`/usr/lib/os-release` (generated at build time):
- `NAME` / `PRETTY_NAME` = AntOS product name + version
- `ID=fedora` / `ID_LIKE=fedora` (required by BIB/dnf parsing)
- `VERSION_ID` = integer Fedora base version (41)
- `BUILD_ID` = AntOS build version

---

## 13. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `bootc-image-builder` cannot find image | Image tag not resolvable in local storage | Ensure tag uses `docker.io/` prefix (e.g., `docker.io/antos/image:phase0`) |
| aarch64 cross-build fails on WSL | Upstream BIB / qemu-user-static issue | Use a native ARM host for BIB instead |
| "Fedora" strings appear in ISO | Branding not fully patched | Grep ISO contents and patch `isolinux.cfg`, `grub.cfg`, `.discinfo`, `media.repo` |
| Compositor not rendering visible frame | QEMU has no connected DRM connector | Expected in QEMU; headless fallback is used. Use virtio-gpu + VNC/SPICE for visible-pixel testing |
| AAVMF firmware not found | `edk2-aarch64` not installed on host | `dnf install -y edk2-aarch64` |
| `uname -r` shows `fc41` | Kernel version string is build-time | Acceptable; see sec. 9.3 |

---

## 14. Cross-References

- Architecture: see **Document A - Architecture**
- Full technical reference: see **Document C - Full Technical Reference**
- Build status: `AntOS/BUILD_STATUS.md`
- Phase build plan: `AntOS/docs/operations/phase-build-plan.md`
- Feature/ISO rebuild playbook: `AntOS/docs/operations/feature-and-iso-rebuild-playbook.md`
- Rollback docs: `AntOS/docs/operations/rollback.md`
