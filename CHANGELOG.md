# Changelog

All notable changes to this role are documented here. This role is consumed via
`ansible-galaxy` from git; releases are git tags (e.g. `v1.1.0`).

## [Unreleased]

### Changed
- **Image selection is now intent-based and resolved from the rescue, with no
  silent fallback.** `hetzner_bootstrap_image` now defaults to `""`; when empty,
  new `resolve-image.yml` resolves the concrete tarball IN-RESCUE (the only
  authoritative source — Hetzner exposes no API for installimage filenames) from
  `hetzner_bootstrap_image_{codename,arch,variant}` (default trixie/amd64/base),
  picking the newest `Debian-<NNNN>-<codename>-<arch>-<variant>.tar.*` and
  **ignoring the `-latest-` symlink** (which was observed installing bookworm
  when trixie was intended). If nothing matches — or a pinned image is absent —
  the role ABORTS and lists what is available; it never substitutes another
  version. `await-installed` additionally asserts the booted release equals the
  requested codename (`hetzner_bootstrap_verify_installed_codename`, default
  true), so a name-matched-but-wrong-release image aborts before any downstream
  role runs. Pin `hetzner_bootstrap_image` to an exact filename to opt out of
  resolution. See README "Image selection".

### Added
- **Set + verify root credentials on the installed system.** installimage only
  copies the single rescue SSH key and sets no root password, so root access
  hung on one key with no fallback (a removed key = full lock-out). New phases
  `set-root-credentials` and `verify-root-credentials` (gated on
  `hetzner_bootstrap_manage_root_credentials`, default true) now ensure BOTH
  root `authorized_keys` (`hetzner_bootstrap_root_authorized_keys`, additive or
  `…_exclusive`) and a root password (`hetzner_bootstrap_root_password`, vaulted;
  hashed on the target via `chpasswd`), write an sshd drop-in permitting root
  password login (`hetzner_bootstrap_permit_root_password_login`), and then
  PROVE each method from the controller with single-method logins (key-only,
  then password-only via `sshpass`). See README "Root credentials".
- **Free the OS disks of stale software RAID before installimage.** New phase
  `prepare-os-disks` (`hetzner_bootstrap_wipe_os_disks_before_install`, default
  true) stops any mdadm array the rescue auto-assembled on the SELECTED OS disks,
  zeroes their mdadm superblocks, and clears fs/partition-table signatures —
  fixing reinstalls that failed because the target disks were held open by an
  assembled array or a new array re-absorbed a stale member. Data disks are
  never touched (enforced by an OS/data overlap assert); guarded to run only in
  the rescue; skipped in `--check`. See README "Reinstall hygiene (stale RAID)".

## [1.2.1] - 2026-06-05

### Fixed
- **validation tasks now run for multiple hosts.** The validation tasks previously
  did not validate more than the first host in a run, this has now changed so that
  every host is validated.

## [1.2.0] - 2026-06-02

### Fixed
- **installimage no longer self-cancels.** The config was rendered to `/autosetup`
  *and* passed as `installimage -a -c /autosetup`. installimage's `-c` handler
  (`get_options.sh`) copies the config file to `/autosetup` itself, so this became
  `cp /autosetup /autosetup` → `cp: ... are the same file`; the installer then
  failed config validation and printed `Cancelled.` The config is now rendered to
  a dedicated path (`hetzner_bootstrap_installimage_config_path`, default
  `/root/installimage.conf`) and `-c` points there.
- **Default image name corrected.** `Debian-1300-trixie-64-minimal.tar.gz` does
  not exist on current rescue systems → install fails. The role default now
  tracks `Debian-trixie-latest-amd64-base.tar.zst` (modern
  `<release>-<arch>-<variant>.tar.<zst|gz>` naming; a `-latest-` symlink, so it
  does not 404 as point releases roll). Consumers needing reproducibility pin a
  specific point release in their own inventory — the validation below works for
  both (`find -L` resolves the symlink).
- **Default partition layout now UEFI-correct.** Added a `/boot/efi esp 256M`
  partition as the first entry. Without it, installimage aborts on UEFI hosts
  with `ERROR: ESP missing or multiple ESP found`. installimage mirrors the ESP
  across SWRAID members itself, so one esp line is correct even with software
  RAID. Harmless (warning only) on legacy-BIOS hosts.
- **Installer failures now surface the real error.** The `async_status` wait
  hard-failed on a non-zero installer rc, which skipped the role's own
  "Tail debug log" / "Fail with debug context" block — so the operator saw only
  `Module failed: non-zero return code`, never the `/root/debug.txt` line that
  explains *why*. Added `failed_when: false` to the wait so the explicit
  debug-surfacing block runs.

### Added
- **New phase `validate-rescue` (tag `preflight`/`validate`), run after disk
  discovery and before installimage — fail fast, not late.** installimage only
  validates its config for internal consistency, *after* it has been invoked
  against the target; a wrong image name or a missing ESP otherwise fails late
  (mid-install) or opaquely (`Cancelled.` in `/root/debug.txt`). The new step,
  read-only and `--check`-safe, asserts up front:
  - the configured `hetzner_bootstrap_image` actually exists in the rescue
    (`find -L` so a `-latest-` symlink resolves), listing what *is* available on
    mismatch;
  - on a UEFI host (`/sys/firmware/efi`), exactly one `esp` partition is
    configured, with a message telling the operator exactly what to add.
- `hetzner_bootstrap_images_dir` (default `/root/.oldroot/nfs/install/../images/`)
  — shared by the template's `IMAGE` line and the existence check so they cannot
  drift.
- `hetzner_bootstrap_installimage_config_path` (default `/root/installimage.conf`).

### Changed
- All gathered-fact references use `ansible_facts.<name>` instead of the
  top-level `ansible_<name>` vars (e.g. `ansible_facts.kernel`,
  `ansible_facts.distribution`). Silences the `INJECT_FACTS_AS_VARS` deprecation
  (top-level fact injection is removed in ansible-core 2.24). Connection/magic
  vars (`ansible_host`, `ansible_check_mode`, …) are unaffected and unchanged.

### Notes
- **Future:** custom/operator-supplied image upload (for non-Proxmox install
  types) is a planned development path; not implemented yet.

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
