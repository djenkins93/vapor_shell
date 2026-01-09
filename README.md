# VaporShell (`vapor`)

VaporShell is a **Bash security wrapper** that launches an **ephemeral, isolated shell session**. It’s intended as both a practical local sandbox and a security engineering proof‑of‑concept you can defend in a resume or interview.

It combines:

- **Bubblewrap** (`bwrap`) for Linux namespace isolation and filesystem containment
- **Quixand** for a per‑session encrypted workspace and teardown via **key destruction (crypto‑erase)**
- Optional **Tor‑oriented networking**, with enforcement only when a network namespace can be created

## Features

- **Ephemeral by default**: each run creates a fresh session workspace and removes it on exit.
- **Isolation by default**:
  - namespaces: user, pid, uts, ipc, cgroup (and network depending on mode)
  - hardening: `--cap-drop ALL`, `--new-session`, `--die-with-parent`, `--clearenv`
- **Two modes**:
  - `default`: Tor‑oriented workflow (enforced only when netns is available; otherwise best‑effort proxy mode)
  - `paranoid`: maximum isolation (airgapped) and hides `/proc`
- **Deterministic teardown**:
  - attempts **Quixand purge/key destruction** on exit in *both* modes
  - removes per‑session directories
  - best‑effort cleanup of plaintext artifacts under the session directory

## Threat model

Good at:
- reducing host contamination: no writes to your real `$HOME` unless you explicitly bind something writable
- limiting file/process visibility from inside the session
- making **post‑exit recovery** of encrypted workspace contents impractical via **crypto‑erase**

Not designed to protect against:
- a compromised kernel/host or privileged malware
- RAM acquisition / cold boot / DMA forensics
- “provable secure deletion” of plaintext on SSDs/NVMe or CoW filesystems if data ever hits host disk

## Security notes (read this)

### “Secure wipe” semantics

VaporShell uses **crypto‑erase** (key destruction) as the primary destruction mechanism when Quixand is in use.

It may also overwrite certain plaintext artifacts prior to removal, but that is **best‑effort** and **not a guarantee** on modern storage/filesystems (SSD/NVMe wear‑leveling, CoW, journaling, snapshots, backups).

If you need the strongest “no disk residue” behavior, run sessions in RAM‑backed locations, disable or encrypt swap, and keep sensitive work inside the encrypted workspace.

### Tor routing semantics

- **Enforced Tor**: only when the tool can create a network namespace and apply redirects (requires the right privileges/capabilities).
- **Best‑effort Tor**: proxy environment variable fallback (`ALL_PROXY`) can be bypassed by applications that ignore proxy env vars.

Use `--tor-required` if you want fail‑closed behavior (do not run if enforcement is not possible).

## Requirements

- `bash`
- `bwrap` (bubblewrap)
- `quixand`
- `ip` (iproute2)
- optional: a running Tor instance (if using Tor modes)

## Install

```sh
chmod +x vapor_260107_hardened2
sudo install -m 0755 vapor_260107_hardened2 /usr/local/bin/vapor
```

Man page:

```sh
sudo install -m 0644 vapor.1 /usr/local/share/man/man1/vapor.1
sudo mandb 2>/dev/null || true
man vapor
```

## Usage

Start default mode:

```sh
vapor
```

Start paranoid mode:

```sh
vapor paranoid
```

Interactive menu:

```sh
vapor -i
```

Debug logs:

```sh
vapor -d
```

Fail‑closed Tor (do not fall back to proxy mode):

```sh
vapor --tor-required
```

Leak check (best‑effort sanity check inside the sandbox):

```sh
vapor --leak-check
```

## Files and paths

- Session base directory: prefers `/dev/shm/vaporshell-<id>/` (RAM‑backed on most systems), falls back to `/tmp/vaporshell-<id>/`.
- Per‑session mounts:
  - `/home/user` (session home)
  - `/work` (scratch/work directory)
  - `/tmp` (session temp directory)

## Quick demo checklist (PoC)

Inside the session:

```sh
id
ps -o pid,ppid,comm | head
mount | head -n 40
echo "$HISTFILE"
```

After exit (host):

```sh
ip netns list
ls -la /dev/shm | grep -i vaporshell || true
ls -la /tmp | grep -i vaporshell || true
```

## License

MIT
# vapor_shell
Encrypted Single Use Ephemeral Shells
