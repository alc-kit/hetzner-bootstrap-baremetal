# Changelog

All notable changes to this role are documented here. This role is consumed via
`ansible-galaxy` from git; releases are git tags (e.g. `v1.1.0`).

## [1.1.5] - 2026-06-02

### Added
- **`--check` is now a meaningful provisioning dry-run.** Everything except the one
  un-simulatable step — the actual `installimage` wipe — runs in check mode: inventory
  validation, SSH-key handling, rescue detection, disk enumeration + serial→device
  resolution, and rendering `/autosetup`. The installer run, its async wait, and the
  reboot are gated `when: not ansible_check_mode` (with a dry-run notice naming the
  disks that *would* be wiped); the post-reboot verification (`await-installed`) is
  skipped in check. Read-only steps Ansible would otherwise skip in check carry
  `check_mode: false` (`lsblk` enumeration, the `/autosetup` render + dump). This
  **supersedes the v1.1.4 "not runnable under --check" note.**
- A check-mode run assumes the host is **already in the rescue** (the normal
  pre-provision state, e.g. after `discover-disk-layout.yml`); `--check` does not
  simulate the boot-into-rescue hardware reset.

## [1.1.4] - 2026-06-02

### Fixed
- `discover-disks`: the human-readable disk table used `| ljust(N)`, which is **not**
  a Jinja filter → `No filter named 'ljust'`. Switched to the `.ljust(N)` string method
  (`(name | regex_replace(...)).ljust(10)`, `size.ljust(8)`). This breaks a **real**
  provision run as soon as the table renders (and the `--tags discover` path) — it is
  not check-mode specific.

### Note
- `provision-hetzner-bare-metal.yml` is **not runnable under `--check`**: it is an
  imperative wipe → reboot → wait sequence, and Ansible skips `command` tasks in check
  mode (so `lsblk` yields `from_json('')`, then the async installer var is undefined,
  then the post-reboot assert fails). Use the read-only `discover-disk-layout.yml`
  (consumer side) for pre-wipe disk validation; the real provision is gated by the
  tier-3 policy check + double confirmation.

## [1.1.3] - 2026-06-01

### Fixed
- `rescue-boot`: detect the current Hetzner rescue by **hostname** (`rescue`), not
  only by kernel/distro. The rescue is now Debian-based with a custom kernel (e.g.
  `kernel=6.12.67`, `distro=Debian`), so the old `'rescue' in kernel` / `archlinux`
  heuristic mis-classified a genuine rescue as an installed OS and aborted ("does not
  look like the Hetzner rescue system"). Both the `_in_rescue` reuse check and the
  final sanity-assert now also accept `ansible_hostname` / `ansible_nodename ==
  'rescue'` (gathered from the box via the `setup` already run there), keeping the
  kernel/distro terms as fallback. The change is purely additive (`OR`), so the
  wipe-gate can only gain *positive* rescue signals, never newly match an installed
  host; if the fact gather fails the terms are undefined → `''` → safe abort.
  (`installimage` was evaluated as a marker but is absent from a non-interactive
  shell's PATH, so it is not used.) Supersedes the v1.1.1 "Arch-based rescue"
  known-limitation note.

## [1.1.2] - 2026-06-01

### Fixed
- Localhost-delegated tasks no longer inherit `become`. Tasks that
  `delegate_to: localhost` — reading the public key (`prepare-ssh-key`), calling
  the Hetzner Robot API (`prepare-ssh-key`, `rescue-boot`), probing port 22
  (`rescue-boot`), and stat-ing the key file (`validate`) — inherited the
  consuming play's `become: true` and were wrapped in `sudo` on the **control
  node**, failing with `sudo: a password is required` whenever the operator's
  local sudo is password-protected. None of these tasks need root on the
  controller, so each now sets `become: false` (matching the `known_hosts` tasks
  that already did). No effect on the privileged work done over SSH on the target.

## [1.1.1] - 2026-06-01

### Fixed
- `rescue-boot`: the rescue-vs-installed identify step now sets
  `ignore_unreachable: true`, so a host whose port 22 is open but does not accept
  our SSH key degrades to a clean "installed → bail" instead of aborting the play
  with an unreachable error.

### Known limitation
- `rescue-boot` classifies "already in rescue" from the kernel/distro heuristic
  (`'rescue'` in kernel, or distro `archlinux`/`rescue` — Hetzner's rescue is
  Arch-based). A host already running a **non-Debian** OS that matches that
  heuristic (e.g. an installed Arch box) would be misread as rescue and the
  boot+reset skipped. This role targets Debian installs only, so an installed
  host here is Debian (correctly classified "installed" → bail). If you reuse the
  role against non-Debian installed hosts, pass `hetzner_bootstrap_force_reinstall`
  deliberately or extend the check with a positive rescue marker.

## [1.1.0] - 2026-06-01

### Added
- **`installimage_os_disk_serials`** — select the OS disks by stable serial (or
  WWN) instead of device names. The role resolves each serial to the *current*
  device name inside the rescue, in the same session that runs installimage, so
  the selection survives disk reordering across reboots. Matches on `SERIAL` or
  `WWN` (WWN covers disks that report a blank/duplicate serial); order is
  preserved (drives `DRIVE1`, `DRIVE2`, …); fails clearly if a serial matches
  zero or more than one disk. Mutually exclusive with `installimage_os_disks`.
- `lsblk` enumeration now also captures `WWN` (shown in the disk table).

### Changed
- **`rescue-boot`** now distinguishes a host already in the rescue from an
  installed OS. A host already sitting in rescue is re-used (no bail, and the
  redundant boot+reset is skipped); only a live *installed* system is refused
  without `hetzner_bootstrap_force_reinstall`. This makes the role idempotent on
  re-entry and lets a discovery run (which leaves the host in rescue) compose
  cleanly with a later provisioning run.

### Compatibility
- Backward compatible: `installimage_os_disks` (raw device names) still works as
  a direct/legacy selector.

## [1.0.0] - initial

- Initial scaffold: rescue boot, installimage, await-installed; OS disks selected
  by device name (`installimage_os_disks`).
