# Changelog

All notable changes to this role are documented here. This role is consumed via
`ansible-galaxy` from git; releases are git tags (e.g. `v1.1.0`).

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
