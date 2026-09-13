---
name: droidspaces
description: "Manage Droidspaces Linux containers on Android via CLI."
---

# Droidspaces — complete CLI management (no app needed)

Droidspaces (ravindu644/Droidspaces-OSS) runs full Linux distros (Alpine, Debian, Ubuntu...) natively on Android via namespaces — like LXC/Docker, built for Android + KernelSU. This skill is the complete reference for operating it **entirely from a root shell/Termux**, without the Droidspaces app UI.

## Access & environment

- Binary: `/data/local/Droidspaces/bin/droidspaces` — root required. From Termux: `su -c "/data/local/Droidspaces/bin/droidspaces ..."` or enter a root shell.
- Daemon (`droidspacesd`) auto-starts at boot via the KSU module's post-fs-data; it manages sockets/network and keeps running while any container is active. PID at `/data/local/Droidspaces/droidspacesd.pid`, log at `/data/local/Droidspaces/Logs/droidspacesd.log`, boot summary in `Logs/boot-module.log`.
- Data layout under `/data/local/Droidspaces/`:
  - `Containers/<Name>/container.config` — per-container settings (persisted at start)
  - `Containers/<Name>/rootfs.img` — sparse ext4 image (default 8G, `sparse_image_size_gb` in config)
  - `Logs/` — droidspacesd.log, dmesg.log, boot-module.log, per-container logs
  - `Pids/`, `Net/` — runtime state (PIDs, network bridges)
- Kernel requirement: PID_NS + IPC_NS + MNT_NS + UTS_NS + pivot_root + seccomp (run `droidspaces check` to verify). Stock GKI kernels often lack PID/IPC ns → containers cannot start; custom kernel (e.g. FleurX on garnet) is required. If missing, only a kernel rebuild fixes it.
- On this device: `alpine` (Alpine 3.23, host net, sshd:22, cloudflared tunnel, Hermes Agent + gh CLI) and `hermes` (Debian 13, NAT 172.28.12.10). `check` output on garnet: all MUST/REQUIRED/OPTIONAL pass.

## Essential CLI reference

```sh
DS=/data/local/Droidspaces/bin/droidspaces   # then run via su -c '...'

# --- lifecycle ---
$DS --name=<NAME> start [FLAGS]    # create+start container (flags persisted to container.config)
$DS --name=<NAME> stop             # stop one
$DS --name=c1,c2,c3 stop           # multi-stop (comma-separated)
$DS --name=<NAME> restart          # restart
$DS --name=<NAME> enter [user]     # interactive — REQUIRES a TTY (fails in plain su -c)
$DS --name=<NAME> run <cmd>        # run command inside container (non-interactive)
$DS --name=<NAME> run --user=<U> <cmd>   # run as user (e.g. run --user=myuser whoami)

# --- inspect ---
$DS --name=<NAME> info             # OS, hostname, net mode, hw access, android-storage, GPU/X11 flags
$DS --name=<NAME> info --format    # machine-parseable KEY=VALUE
$DS --name=<NAME> usage            # uptime, CPU permill, RAM used/total (KEY=VALUE)
$DS --name=<NAME> pid              # init PID
$DS show                           # table of running containers
$DS scan                           # detect untracked containers (config/PIDs lost)
$DS check                          # full requirement diagnostic (MUST/REQUIRED/OPTIONAL)
$DS version                        # version
$DS docs                           # interactive documentation
```

### Start flags (persisted)
```sh
# Networking
--net=host                  # default — share Android stack (sshd/cloudflared use this)
--net=nat                   # isolated netns, internet via NAT, own 172.28.x.x IP
--net=none                  # air-gapped
--net=gateway --gateway=X   # LAN delegated to another container (e.g. OpenWRT)
--port [H:]C[/P]            # port fwd in nat: --port 22, 80:80/tcp, 1000-2000:1000-2000/udp
--dns=1.1.1.1,8.8.8.8       # custom DNS
--disable-ipv6              # turn off IPv6 inside
--upstream=wlan0,rmnet*     # pin WAN uplink(s), wildcards ok (rmnet* = mobile data)
--upstream=tun0             # VPN killswitch — route only through the phone VPN

# Hardware / integration
--enable-android-storage -S # mount /sdcard into container (/storage/emulated/0)
--hw-access -H              # expose /dev nodes
--gpu                       # GPU acceleration nodes
--termux-x11 -X [--tx11-flags="..."]  # Termux-X11 display
--virgl [--virgl-flags="..."]          # VirGL 3D acceleration
--pulse-audio               # PulseAudio sound server

# Security & limits
--selinux-permissive -P     # host SELinux → permissive
--volatile -V               # OverlayFS — discard changes on exit
--force-cgroupv1            # legacy cgroup v1
--block-nested-namespaces   # deadlock shield (no nested ns)
--memory=512M|2G            # mem limit
--cpus=1.5|2                # CPU limit
--pids-limit=N              # max PIDs
--privileged=nomask,nocaps,noseccomp,shared,unfiltered-dev,full  # relax security
--allow-userns              # allow user namespaces

# Advanced / other
--rootfs=PATH | --rootfs-img=PATH   # dir or .img (image: -i)
--init=PATH                # custom init (default /sbin/init; e.g. --init=/bin/bash)
--config=PATH | --conf=PATH -C      # load a specific config file
--bind=SRC:DEST[:ro] | -B   # bind mount (repeatable or comma-separated)
--env=PATH | -E            # load env vars from file
--user=USER | -u           # (run only) run as user
--reset                    # reset config to defaults (keeps name/rootfs)
--foreground -f            # run in foreground (attach console)
--format                   # machine-parseable output for info/usage
```

## Default lifecycle & config

- `container.config` (auto-generated at start; fields: name, hostname, rootfs_path, net_mode, disable_ipv6, enable_android_storage, enable_hw_access, enable_gpu_mode, enable_termux_x11, enable_virgl, enable_pulseaudio, selinux_permissive, allow_userns, volatile_mode, **run_at_boot**, force_cgroupv1, block_nested_ns, use_sparse_image, sparse_image_size_gb, uuid).
- `run_at_boot` controls start at device boot. Daemon scans containers with `run_at_boot=1` each boot (boot-module.log: "Scanning for containers with run_at_boot=1..."). Default 0 — containers do NOT restart across reboots unless you set it (edit container.config or start with the flag; verify with `info`).
- Rootfs is a sparse ext4 image (default 8 GiB) mounted via loop device. `stop` unmounts; `start` re-mounts.

## Common workflows

### Create & start a new container
```sh
$DS --name=mydistro --rootfs=/path/to/rootfs --net=host --enable-android-storage start
$DS --name=kali --rootfs=/path/to/kali --net=nat --port 22,8080:80 start
$DS --name=openwrt --rootfs=/path/to/openwrt --net=nat start
$DS --name=dev --rootfs=/path --memory=512M --cpus=2 --pids-limit=256 start
```

### Inspect (non-interactive)
```sh
$DS --name=alpine run uname -a          # container sees HOST kernel
$DS --name=alpine run cat /etc/os-release
$DS --name=alpine run sh -c 'ss -tlnp'  # listening ports inside
$DS --name=alpine run sh -c 'ps aux'
$DS --name=alpine usage                 # resource usage KEY=VALUE
$DS --name=alpine info --format | grep net_mode
```

### Provision files into a container
`run` does NOT accept piped stdin — heredocs through it are unreliable. Reliable pattern: stage on host → copy via shared Android storage → move inside:
```sh
su -c "cp /data/data/com.termux/files/home/x.tmp /storage/emulated/0/Download/x.tmp"
su -c "$DS --name=alpine run sh -c 'cp /storage/emulated/0/Download/x.tmp /root/x && rm /storage/emulated/0/Download/x.tmp'"
```
Use `.tmp` suffix for partial writes; clean up after. If android-storage isn't mounted, use `--bind=/host/path:/container/path` at start, or embed the value directly in the command string.

### Host-network OpenRC services (cloudflared example)
Host-net containers share Android's network; dhcpcd refuses there (`Skipping dhcpcd: not in NAT network mode`) and breaks services with `need net`. Fix:
```sh
sed -i 's/need net/use net/' /etc/init.d/<svc>   # network always present in host mode
rc-update del dhcpcd
rc-service <svc> start
```
`service install` may abort before writing config (e.g. cloudflared token file) — check expected files exist, write manually. Verify a tunnel is REALLY connected: `run sh -c 'tail -25 /var/log/cloudflared.log'` — expect `Registered tunnel connection connIndex=0..3 location=sin22/cgk01 protocol=quic`. 4 QUIC conns = live; crash-loops land in .err log while rc status still says started.

### NAT networking notes
- NAT containers get an IP in 172.28.*.*; `--port` forwards host→container (e.g. `--port 22` exposes container sshd on host:22).
- Uplink auto-detected and re-tracked in real time as the host switches Wi-Fi ↔ mobile; pin with `--upstream=wlan0,rmnet*` or VPN-killswitch with `--upstream=tun0`.

## Run Docker images as native Droidspaces containers (no Docker)

Docker/Podman inside a container is wasteful (container-in-container); instead convert an OCI image to a plain rootfs and run it directly with droidspaces.

### 1. Export the image (two critical gotchas)
Use **crane** (go-containerregistry) on the HOST — with the DNSResolve module active, Termux DNS works for Go binaries:
```sh
./crane export --platform linux/arm64 nginx:alpine - > /storage/emulated/0/Download/nginx.tar
```
- **GOTCHA 1 (architecture):** crane defaults to the manifest's first platform — often **x86-64**! On an arm64 phone you get `Exec format error` at runtime and the container "fails to boot" while foreground seems to work. ALWAYS pass `--platform linux/arm64` explicitly.
- **GOTCHA 2 (DNS):** Go binaries read `/etc/resolv.conf` (absent on Android). Without the DNSResolve KSU module (overlay `/system/etc/resolv.conf` — `/etc` is a symlink to `/system/etc`), crane fails with `lookup ... on [::1]:53: connection refused`. Install the module from **github.com/GegeDevs/DNSResolve-Termux** (build via its GitHub Actions, flash the release zip, reboot) — then crane works directly on the Termux host, no container needed.

### 2. Extract & prepare the rootfs
```sh
mkdir -p /data/local/tmp/imgwork/nginx && cd /data/local/tmp/imgwork/nginx
tar -xf /storage/emulated/0/Download/nginx.tar
# verify architecture before going further:
file usr/sbin/nginx   # expect: ELF ... arm64 ... ld-musl-aarch64
```
Image rootfs dirs (bin, etc, usr, ...) ARE the container rootfs — point `--rootfs` at the extraction dir directly.

### 3. Provide an init (images often lack one)
`nginx:alpine`'s `/etc/inittab` calls `/sbin/openrc` which is NOT in the image → busybox init exits → background start fails (`failed to boot correctly`). Replace with a minimal inittab + wrapper:
```sh
# /etc/inittab — minimal, no openrc, no getty (no tty devices in container)
cat > etc/inittab <<'EOF'
::sysinit:/usr/local/bin/ds-init
::shutdown:/bin/busybox poweroff
EOF
```
```sh
# /usr/local/bin/ds-init — exec the image's real entrypoint as PID1
cat > usr/local/bin/ds-init <<'EOF'
#!/bin/sh
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
exec /docker-entrypoint.sh nginx -g 'daemon off;'
EOF
chmod +x usr/local/bin/ds-init
```
If the image has no entrypoint script, `exec` the binary directly (e.g. `exec redis-server --daemonize no`).

### 4. Start with --net=host (not NAT)
```sh
$DS --name=nginx --rootfs=/data/local/tmp/imgwork/nginx --net=host start
# verify from host:
curl -s -m 5 -o /dev/null -w 'HTTP %{http_code}\n' http://127.0.0.1:80/   # HTTP 200
```
- **GOTCHA 3 (NAT port fwd):** `--net=nat --port 80:80` installs correct iptables DNAT rules, but host→container forwarding silently fails on cellular (rmnet) networks. Use `--net=host` instead, or access from another NAT container. Verify with `ss -tlnp` (DNAT rules exist even when unreachable).
- If starting with a stale config (old rootfs/custom init persisted), delete `container.config` or run `--reset` before `start`.
- Debug rootfs boot errors quickly with a manual chroot: `mount -t proc proc <rootfs>/proc && mount -t tmpfs tmpfs <rootfs>/dev && chroot <rootfs> /usr/local/bin/ds-init` — it surfaces `Exec format error` etc. (skip the port-forward confusion entirely).

## droidctl — unified CLI wrapper (crane + droidspaces)

`droidctl` (`~/bin/droidctl`, bash, Termux) wraps the Docker-image conversion workflow into 2 commands, handling every gotcha below automatically:

```sh
droidctl pull <image>                 # crane export --platform linux/arm64 -> ~/droidimages/
droidctl convert <image> [--cmd ...]  # pull + extract + auto-init (inittab, ds-init, /sbin/init)
droidctl images                       # list rootfs pool (/data/local/tmp/droidimg)
droidctl ps | start | stop | restart | logs | exec | info | rm | check | version
droidctl start <name> [--net=...] [--port X:Y]   # --rootfs auto-resolved from pool
```

### What it automates (all the gotchas below)
- Always `--platform linux/arm64` on pull (GOTCHA 1)
- Extract as root to `/data/local/tmp/droidimg` (ext4 → hardlinks OK) with GNU tar `--hard-dereference` (FUSE /data/media can't hardlink; busybox images are full of hardlinks)
- Auto-detect entrypoint (nginx/redis/docker-entrypoint.sh) → writes `/usr/local/bin/ds-init`; else idle-loop so `exec` works
- Replaces openrc `inittab` with minimal `::sysinit:/usr/local/bin/ds-init` (GOTCHA: openrc absent in images)
- Creates `/sbin/init` → busybox symlink when image lacks init (droidspaces requires /sbin/init)
- `start` auto-resolves rootfs from pool by container name

### Env overrides
`DROIDSPACES_BIN`, `CRANE_BIN`, `DROIDCTL_IMGDIR` (default `~/droidimages`), `DROIDCTL_ROOTFS_DIR` (default `/data/local/tmp/droidimg`), `DROIDCTL_NET` (default `host`).

## Troubleshooting quick reference

| Symptom | Likely cause / fix |
|---|---|
| Container won't start | Kernel lacks namespaces (PID/IPC) → custom kernel needed; rootfs path wrong/missing init; run `$DS check`. |
| `Interactive terminal is required` | Used `enter` without TTY → use `run`. |
| `No containers are currently running` | All stopped → `start` them (run_at_boot=0 by default). |
| Container not auto-started after reboot | `run_at_boot=0` → set `run_at_boot=1` in container.config or via start flags. |
| Empty file written into container | `run` ate quoting/stdin → stage via /storage/emulated/0 or embed value inline. |
| OpenRC service fails with dhcpcd error | Host-net mode: `need net` → `use net` + `rc-update del dhcpcd`. |
| Service 'started' but not working | Check the real log (e.g. cloudflared.err) — supervise-daemon respawns crash-loops, status lies. |
| `run` not found / command fails right after start | Container init not ready; wait a moment and retry. |
| `Internal REBOOT` detected | `reboot` inside → container re-execs; that's normal (droidspaces handles it). |
| Static NAT IP unparseable | Bad `--nat-ip` value → DHCP offers 0.0.0.0, boot fails; use a valid 172.28.*.* IP. |
| Systemd container fails | cgroup2 bootstrap fails on some kernels; use Alpine/Debian init or `--force-cgroupv1`. |

## Pitfalls (learned on this device)
- Always quote the whole droidspaces invocation inside `su -c "..."` carefully — nested quoting eats host variables (empty values). Embed literal values directly.
- `enter` needs a TTY; use `run` for automation. For an interactive session from Hermes use `terminal(pty=true)`.
- Networking mode is fixed at start; changing requires a restart with the flag (config persists).
- Containers do NOT auto-start after device reboot unless `run_at_boot=1`.
- Alpine: zsh at `/bin/zsh` not `/usr/bin/zsh` (chsh writes wrong path); oh-my-zsh lives in `/usr/share/oh-my-zsh`. See android-device-mgmt ref `references/droidspaces-containers.md`.
- Hermes Agent inside a container: shebang must be the venv python (`No module named 'rich'` on relaunch), nodejs must be musl build. See android-device-mgmt ref `references/hermes-in-container.md`.
- Busybox ships next to the binary at `/data/local/Droidspaces/bin/busybox` (useful inside su scripts).
- Daemon keeps running while any container is active; `stop` last container keeps daemon alive (harmless).
