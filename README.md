# Droidspaces-Control-Skill

A complete **Hermes Agent skill** for managing [Droidspaces](https://github.com/ravindu644/Droidspaces-OSS) Linux containers on Android **entirely from the CLI** — no app UI needed.

[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/Platform-Android-green.svg)](https://www.android.com/)
[![Requires](https://img.shields.io/badge/Requires-Root-orange.svg)](https://kernelsu.org/)

## What is this?

Droidspaces runs full Linux distros (Alpine, Debian, Ubuntu...) natively on Android via Linux namespaces — like LXC/Docker, built for Android + KernelSU.

This skill is the complete operational reference for driving it **from a root shell / Termux**, covering:

- **Full CLI reference** — lifecycle (`start/stop/restart/enter/run`), inspect (`info/usage/pid/show/scan/check`), and every start flag (`--net`, `--port`, `--upstream`, `--gpu`, `--memory`, `--privileged`, ...)
- **Container config & lifecycle** — `container.config`, `run_at_boot` auto-start behavior, rootfs images
- **Common workflows** — create containers, non-interactive inspection, provisioning files, fixing OpenRC services in host-net mode (cloudflared example)
- **Run Docker images as native Droidspaces containers** — convert OCI images with `crane` and run them directly (no Docker), including the 3 hard-won gotchas:
  1. **Architecture** — always `--platform linux/arm64` (crane defaults to x86-64 → `Exec format error`)
  2. **DNS** — Go binaries need `/etc/resolv.conf`, use the [DNSResolve-Termux](https://github.com/GegeDevs/DNSResolve-Termux) KSU module so `crane` works directly on Termux
  3. **Networking** — NAT port-forward silently fails on cellular networks; use `--net=host`
- **Troubleshooting table** — 11 common symptoms & fixes
- **Pitfalls** — hard-learned lessons (quoting, TTY, auto-start, Alpine zsh, Hermes-in-container)

## Installation

### Option A: Download the release tarball (recommended)

The [Releases](../../releases) page has auto-built `droidspaces-control-skill-<ver>.tar.gz` (built & validated by GitHub Actions):

```bash
# Download & extract
tar -xzf droidspaces-control-skill-v1.0.0.tar.gz
# Run installer (installs to ~/.hermes/skills/software-development/droidspaces/SKILL.md)
./install.sh
```

### Option B: Into Hermes Agent (manual)

```bash
# From your Hermes skills directory:
mkdir -p ~/.hermes/skills/software-development/droidspaces
cp droidspaces.md ~/.hermes/skills/software-development/droidspaces/SKILL.md
```

Then ask your Hermes agent to manage Droidspaces — it will auto-load the skill.

### Option C: Standalone reference

Just read `droidspaces.md` — it's a self-contained markdown reference.

## CI/CD

GitHub Actions (`.github/workflows/validate-build.yml`) runs on every push / PR / tag:

- **validate** — checks `droidspaces.md` has valid YAML frontmatter (`name`, `description`) and structure
- **build** — assembles the Hermes directory layout (`skills/software-development/droidspaces/SKILL.md`) + `install.sh`, tars it up
- **publish** — on `v*` tags, uploads the tarball to GitHub Releases

## Prerequisites

- **Root** (KernelSU / Magisk / APatch)
- **Droidspaces** installed — [ravindu644/Droidspaces-OSS](https://github.com/ravindu644/Droidspaces-OSS)
- **Kernel with namespace support** (PID_NS, IPC_NS, MNT_NS, UTS_NS, pivot_root, seccomp) — stock GKI kernels often lack PID/IPC namespaces; a custom kernel (e.g. FleurX on garnet) is required. Verify with `droidspaces check`
- **Termux** (or any root shell)

## Companion

The **DNSResolve-Termux** module ([GegeDevs/DNSResolve-Termux](https://github.com/GegeDevs/DNSResolve-Termux)) pairs with this skill: it provides `/etc/resolv.conf` on Android so Go binaries (`crane`, `gh`, docker CLI) resolve DNS directly in Termux — required for the Docker-image conversion workflow.

## License

MIT