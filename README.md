# hetzner-bootstrap-baremetal

An Ansible role that takes a freshly-ordered Hetzner dedicated server from its
factory state to a freshly-installed Debian system reachable over SSH, ready
for downstream configuration playbooks (Proxmox, K8s, Postgres, whatever you
run on top).

## What it does

1. **Validates** the inventory locally before touching the Robot API.
2. **Registers** your SSH public key with the Hetzner Robot account if it is
   not already there (idempotent).
3. **Activates rescue mode** for the target server and **hardware-resets**
   it into the rescue.
4. **Resolves the OS image** from the rescue's own image directory by intent
   (codename/arch/variant), picking the newest concrete point-release and
   ignoring the unreliable `-latest-` symlink — **aborting** if nothing matches
   rather than falling back. See [Image selection](#image-selection).
5. **Enumerates disks** in the rescue and validates that the operator-supplied
   list of OS disks actually exists on the host.
6. **Frees the OS disks of stale software RAID** — stops any mdadm array the
   rescue auto-assembled on the selected OS disks and clears their RAID +
   partition signatures, so `installimage` can repartition cleanly (data disks
   are never touched). See [Reinstall hygiene](#reinstall-hygiene-stale-raid).
7. **Runs `installimage`** with a templated config that wipes only the OS
   disks; all other block devices are left pristine.
8. **Reboots** into the installed system, waits for SSH on the real OS, and
   **asserts the installed release matches the requested codename** (aborts on a
   wrong-release image rather than proceeding).
9. **Sets and verifies root credentials** — ensures root's `authorized_keys`
   **and** a root password are both in place, then proves each one works from
   the controller. See [Root credentials](#root-credentials).

After the role finishes, the host is reachable as root over SSH with the
public key you provided **and** (by default) a break-glass root password — both
verified — on the same IP it was ordered with. From there, any downstream role
can pick up.

## Supported targets

- **OS:** Debian (latest stable only — currently **trixie**)
- **Hardware:** Hetzner dedicated (Robot) — *not* Hetzner Cloud
- **Ansible:** `>=2.14`

## Requirements

- Hetzner Robot **Webservice user** credentials. Create one under
  *Settings → Web service and app settings* in the Robot UI.
- The `community.hrobot` collection (`>=2.0.0`):

  ```yaml
  # requirements.yml
  collections:
    - name: community.hrobot
      version: ">=2.0.0"
  ```

- A local SSH keypair. The public key is what the role registers with Robot
  and authorises into both the rescue and the installed system.

## Quick start

1. Add the role to your `requirements.yml`:

   ```yaml
   roles:
     - src: git@github.com:kvalitetsit/hetzner-bootstrap-baremetal.git
       name: hetzner_bootstrap_baremetal
       scm: git
       version: main   # pin to a tag once you start versioning
   ```

   And the collection dep:

   ```yaml
   collections:
     - name: community.hrobot
       version: ">=2.0.0"
   ```

   Install both:

   ```bash
   ansible-galaxy install -r requirements.yml
   ```

2. Define your inventory:

   ```yaml
   # inventory/hosts.yml
   all:
     children:
       hetzner_baremetal:
         hosts:
           my-node-01:
             ansible_host: 1.2.3.4
             hetzner_server_number: 1234567
             installimage_os_disks: [nvme0n1, nvme1n1]
   ```

   And the vault:

   ```yaml
   # group_vars/hetzner_baremetal/vault.yml  (ansible-vault encrypted)
   hetzner_robot_user:     "..."
   hetzner_robot_password: "..."
   ```

3. Run the example play:

   ```bash
   ansible-playbook -i inventory/hosts.yml \
       playbooks/example-bootstrap.yml \
       --ask-vault-pass --limit my-node-01
   ```

## Required variables

| Variable | Where | Description |
|---|---|---|
| `hetzner_robot_user` | vault (group/all) | Robot Webservice user |
| `hetzner_robot_password` | vault (group/all) | Robot Webservice password |
| `hetzner_server_number` | host_vars | Numeric server ID from the Robot UI |
| OS disks (one of) | host_vars | `installimage_os_disk_serials` (recommended — disk serials/WWNs, stable across reboots) **or** `installimage_os_disks` (raw device names, fragile). Mutually exclusive. See [Disk selection](#disk-selection). |

## Optional variables

See `defaults/main.yml` for the full list, but the most useful overrides:

| Variable | Default | Purpose |
|---|---|---|
| `hetzner_bootstrap_image` | `""` (empty → auto-resolve) | Exact image filename to pin. Leave empty (default) to auto-resolve from the intent vars below. A non-empty value skips resolution; `validate-rescue` still asserts it exists. See [Image selection](#image-selection). |
| `hetzner_bootstrap_image_codename` | `trixie` | Debian codename to resolve (used when `hetzner_bootstrap_image` is empty). |
| `hetzner_bootstrap_image_arch` | `amd64` | Architecture to resolve. |
| `hetzner_bootstrap_image_variant` | `base` | Image variant to resolve (`base`, `minimal`, …, as present in the rescue). |
| `hetzner_bootstrap_verify_installed_codename` | `true` | After install, assert the booted release equals the codename — aborts on a wrong-release image. Set `false` for non-Debian images. |
| `hetzner_bootstrap_images_dir` | `/root/.oldroot/nfs/install/../images/` | Directory the rescue serves images from (shared by resolution, the `IMAGE` line, and the existence check) |
| `installimage_swraid` | `1` | `0`/`1`. Auto-disabled if only one OS disk is configured. |
| `installimage_swraid_level` | `1` | `0`, `1`, `5`, `6`, or `10` |
| `installimage_partitions` | `/boot/efi esp 256M, /boot 1G, swap 32G, / all` | List of partition dicts, each `{mount, fs, size}`. The ESP is required on UEFI hosts (pre-flight asserts exactly one); installimage mirrors it across SWRAID members. |
| `hetzner_bootstrap_installimage_config_path` | `/root/installimage.conf` | Where the rendered config is written and passed to `installimage -c`. Must NOT be `/autosetup`. |
| `installimage_hostname` | `{{ inventory_hostname_short }}` | Hostname written by installimage |
| `installimage_bootloader` | `grub` | `grub` or `lilo` |
| `installimage_post_install_script` | *unset* | Optional path to a script installimage runs in the chroot |
| `hetzner_bootstrap_ssh_public_key_path` | `~/.ssh/id_ed25519.pub` | Local pubkey to register with Robot |
| `hetzner_bootstrap_force_reinstall` | `false` | Allow reinstall of a server that is currently up |
| `hetzner_bootstrap_wipe_os_disks_before_install` | `true` | Stop stale mdadm RAID + clear signatures on the OS disks before installimage (see [Reinstall hygiene](#reinstall-hygiene-stale-raid)) |
| `hetzner_bootstrap_reset_type` | `hardware` | `hardware`, `power`, `manual`, `software` |
| `hetzner_bootstrap_reset_type_when_down` | `""` (off) | Reset type to use instead, when the host was not answering SSH before the reset. Set to `power` for a server you know is powered OFF. |
| `hetzner_bootstrap_rescue_wait_seconds` | `600` | How long to wait for rescue SSH |
| `hetzner_bootstrap_installimage_timeout_seconds` | `1800` | Hard ceiling on installer runtime |
| `hetzner_bootstrap_installed_wait_seconds` | `600` | How long to wait for installed SSH |
| `hetzner_bootstrap_manage_root_credentials` | `true` | Master switch for the set+verify credentials phase ([Root credentials](#root-credentials)) |
| `hetzner_bootstrap_root_authorized_keys` | `[<provisioning key>]` | SSH public keys authorised for root on the installed system; append operator/break-glass keys |
| `hetzner_bootstrap_root_authorized_keys_exclusive` | `false` | `true` = replace root's `authorized_keys` with exactly this set; `false` = additive |
| `hetzner_bootstrap_set_root_password` | `true` | Set a break-glass root password (REQUIRES `hetzner_bootstrap_root_password`, vaulted) |
| `hetzner_bootstrap_root_password` | `""` | The root password (vault it). Plaintext is hashed on the target via `chpasswd`; a `$6$…` crypt hash is used as-is |
| `hetzner_bootstrap_permit_root_password_login` | `true` | Write an sshd drop-in enabling `PermitRootLogin`/`PasswordAuthentication` so the password is a real fallback |
| `hetzner_bootstrap_verify_root_credentials` | `true` | After setting, prove BOTH methods from the controller; fail if key auth doesn't work (password probe needs `sshpass` + a plaintext password) |
| `hetzner_bootstrap_ssh_private_key_path` | `<pubkey path minus .pub>` | Private key for the post-install key-auth probe |

## Disk selection

Hetzner bare-metal servers are commonly ordered with mixed storage: e.g.
2x NVMe for OS + 4x rotational HDD for data. **installimage only touches
disks listed explicitly as `DRIVE<N>` in its config.** This role lets you
pick which disks count as "OS" — everything else is left pristine.

### Selecting by serial (recommended) vs by device name

Device names (`sda`, `nvme0n1`) are **not stable across rescue reboots** — the
kernel can enumerate the same physical disks in a different order, so a committed
`installimage_os_disks: [sda, sdb]` can point at the wrong disks on a later run.
Prefer **`installimage_os_disk_serials`**: a list of disk serials (or WWNs). The
role resolves each to the current device name *in the rescue, in the same session
that runs installimage*, matching on `SERIAL` or `WWN` (WWN covers disks that
report a blank/duplicate serial). Order is preserved → `DRIVE1`, `DRIVE2`, ….
It fails clearly if a serial matches zero or more than one disk.

```yaml
# host_vars/my-node-01/disks.yml  (recommended)
installimage_os_disk_serials: ["S6S1NE0R123456", "S6S1NE0R123457"]
```

`installimage_os_disks` (raw names) still works as a direct/legacy override, but
the two are **mutually exclusive** — set exactly one.

### The flow

1. **First deploy**: run only the discovery stage to dump the disk inventory
   of the rescue system:

   ```bash
   ansible-playbook ... --tags discover
   ```

   You will see something like:

   ```
   Disks visible in rescue system:
     nvme0n1   894G    flash       SAMSUNG MZQL2960HCJR-00A07  SN=S6S1...
     nvme1n1   894G    flash       SAMSUNG MZQL2960HCJR-00A07  SN=S6S1...
     sda       18.2T   rotational  ST18000NM004J               SN=ZRX1...
     sdb       18.2T   rotational  ST18000NM004J               SN=ZRX1...
     sdc       18.2T   rotational  ST18000NM004J               SN=ZRX1...
     sdd       18.2T   rotational  ST18000NM004J               SN=ZRX1...
   ```

2. **Populate host_vars** with the OS disks:

   ```yaml
   # host_vars/my-node-01/disks.yml
   installimage_os_disks: [nvme0n1, nvme1n1]
   ```

3. **Re-run** without the tag filter — the role validates that those disks
   exist, computes `hetzner_bootstrap_data_disks` (everything else), and
   proceeds.

### Why explicit and not autodetected

A drive sometimes gets replaced with a different model after RMA, or a server
is reconfigured to a different storage layout. An explicit list catches this
as a validation error instead of silently picking the "wrong" pair of NVMe
devices.

### Downstream consumption

After the role finishes, `hetzner_bootstrap_data_disks` is set as a fact on
each host. Downstream roles (data-pool creation, LUKS provisioning, etc.) can
read it as the authoritative source of available data disks:

```yaml
- name: "Create encrypted data pool from leftover disks"
  ansible.builtin.include_role:
    name: clevis-encryption
  vars:
    clevis_encryption_disks: "{{ hetzner_bootstrap_data_disks }}"
```

## Image selection

There is **no Hetzner API that lists installimage base-image filenames** — the
Robot API only returns human-readable rescue-distro names. The single
authoritative source for what a given rescue can install is that rescue's own
image directory. So instead of pinning a brittle exact filename (which you can
only discover by shelling into a rescue), you declare **intent** and the role
resolves the concrete filename in-rescue at provision time (`resolve-image.yml`):

```yaml
hetzner_bootstrap_image_codename: trixie   # what you want
hetzner_bootstrap_image_arch: amd64
hetzner_bootstrap_image_variant: base
# hetzner_bootstrap_image: ""              # empty → auto-resolve; set to pin exactly
```

It matches `Debian-<NNNN>-<codename>-<arch>-<variant>.tar.*`, picks the **newest
concrete point-release**, and **deliberately ignores the `-latest-` symlink** —
that symlink has been seen resolving to the wrong release in the field
(installing bookworm when trixie was intended).

**No silent fallback — it aborts instead.** If nothing matches the intent (or a
pinned `hetzner_bootstrap_image` is absent), the role fails and lists what *is*
available; it never substitutes a different version. And after install,
`await-installed` asserts the booted release equals the requested codename
(`hetzner_bootstrap_verify_installed_codename`), so an image whose *name* matched
but installed the wrong release still aborts — before any downstream role runs.

To just eyeball the menu without provisioning, the resolution step logs every
base image it finds; run the rescue + image phases only:

```bash
ansible-playbook ... --tags 'rescue,image'
```

## Reinstall hygiene (stale RAID)

Does installimage unmount/stop the assembled boot devices before reinstalling?
**Not reliably.** The Hetzner rescue system auto-assembles any pre-existing
mdadm array on boot. installimage *attempts* to stop RAID, but when an array is
assembled on the very disks it must repartition, this is fragile: the disks stay
held open (partition/format fails partway), or the freshly-created array
silently re-absorbs an old member because a leftover mdadm superblock survived.
On a RAID1 OS pair (the common Hetzner layout) that surfaces as an install that
fails late or a system that boots degraded.

So before installimage runs, `tasks/prepare-os-disks.yml`
(`hetzner_bootstrap_wipe_os_disks_before_install`, default `true`):

1. Finds the mdadm arrays backed by the **selected OS disks** (via `lsblk` — it
   never looks at data disks).
2. `swapoff`s and unmounts anything on those disks / arrays.
3. `mdadm --stop`s the arrays and `--zero-superblock`s every OS disk + partition
   (kills the re-absorb-a-stale-member case).
4. `wipefs -a` + GPT/MBR zap (`sgdisk --zap-all`, or a `dd` fallback).
5. Re-checks and **asserts** no array still references an OS disk before handing
   off to the installer.

Scope is enforced by an assert that the OS-disk selection does **not** overlap
`hetzner_bootstrap_data_disks`, and the whole file is guarded to run only in the
rescue system. It is skipped in `--check`.

## Root credentials

installimage copies only the single rescue SSH key into the new system and sets
**no** root password — so root access hangs on that one key. If it is ever
removed, the host is locked out with no fallback. To prevent that, after the
install the role (`hetzner_bootstrap_manage_root_credentials`, default `true`)
guarantees **both** auth methods and then **proves** each independently:

- **`authorized_keys`** — every key in `hetzner_bootstrap_root_authorized_keys`
  is authorised for root (additive by default; set
  `…_exclusive: true` to replace the set exactly).
- **Root password** — `hetzner_bootstrap_root_password` (vault it) is set via
  `chpasswd` on the target, and an sshd drop-in
  (`/etc/ssh/sshd_config.d/00-hetzner-bootstrap-rootlogin.conf`, sorts first so
  it wins) permits root password login so it is a genuine fallback.
- **Verification** — `verify-root-credentials.yml` opens two logins from the
  controller, each forcing a single method (pubkey-only, then password-only).
  Key auth failing fails the play; the password probe needs `sshpass` on the
  controller and a plaintext password (a pre-hashed value can't be logged in
  with, so it's skipped with a warning).

> **Security note.** `hetzner_bootstrap_permit_root_password_login: true` enables
> `PermitRootLogin yes` + `PasswordAuthentication yes`. These hosts are managed
> over an internal VLAN/VPN; use a strong, vaulted password. The role's contract
> is "both methods work **at hand-off**" — keeping the password fallback alive
> for the host's whole life is a fleet concern: a later hardening role that
> rewrites sshd_config must preserve (or re-create) that drop-in. Set the flag
> `false` to leave sshd untouched (the password is still set, but only usable
> where password auth is already permitted, e.g. the Robot KVM console).

## Tags

| Tag | What it runs |
|---|---|
| `hetzner_bootstrap` | The whole flow (alias for "everything") |
| `validate` | Local validation only — never talks to Robot or the server |
| `ssh_key` | Register the SSH key with Robot |
| `rescue` | Boot config + reset + wait for rescue SSH |
| `image` | Resolve the concrete OS image from the rescue by intent |
| `discover` | Enumerate disks (and, with `image`, list available OS images) in the rescue |
| `preflight` | Fail-fast checks against the live rescue: image exists, UEFI/ESP sane |
| `disks` / `wipe` | Stop stale RAID + clear signatures on the OS disks |
| `installimage` | Render config and run the installer |
| `await` | Wait for the installed system to come up |
| `credentials` | Set root authorized_keys + password on the installed system |
| `verify` | Prove both root auth methods from the controller |

Useful combinations:

```bash
# Just check inventory and key registration without touching the server:
ansible-playbook ... --tags 'validate,ssh_key'

# Dump disks for a host that is already in the rescue (skip the reset):
ansible-playbook ... --tags 'discover' --skip-tags 'rescue'
```

## Example: end-to-end

The included `playbooks/example-bootstrap.yml` is the minimal viable wrapper.
For a real fleet you would typically chain this with downstream playbooks:

```yaml
# site.yml in the consuming project
- import_playbook: provision-bare-metal.yml   # wraps this role
- import_playbook: install-base-os.yml
- import_playbook: install-your-stack.yml
```

`provision-bare-metal.yml` can be a one-liner that includes this role.

## Troubleshooting

**The role times out at "Wait for rescue SSH to respond"**
First question: was the server **powered off**? Hetzner's reset types are distinct
physical actions, not severity levels — `hardware` presses the **Restart** button,
which does nothing at all on a powered-off server. The role warns about this before
the wait when the host was not answering beforehand, and the failure message says so
too. Fix: power it on in Robot, or re-run with
`-e hetzner_bootstrap_reset_type_when_down=power`. Note `power` presses the **Power**
button, so on a *running* server it triggers a graceful shutdown — which is why the
role will not switch for you. The Robot API exposes no power-state query, so this
cannot be detected automatically.

A timeout is **not** a reason to raise `hetzner_bootstrap_rescue_wait_seconds`; that
only helps a server that is slow rather than stuck. Rule out your own egress with a
sibling server on the same public path, then check Robot for whether rescue is still
active/unconsumed (if it is, the server never booted the rescue image), then the
remote console.

**The role fails at "sanity-check we are actually in the rescue"**
The server probably did not actually reboot into rescue — check the Robot UI
for the current boot state and that the reset succeeded. Try
`hetzner_bootstrap_reset_type: power` if `hardware` did not take.

**The role fails at `validate-rescue` (image or ESP)**
This is the fail-fast guard doing its job, *before* any wipe. Either the
configured `hetzner_bootstrap_image` is not present in this rescue (the error
lists what *is* available — pick one, or bump the pinned release) or the host
booted UEFI without an `esp` partition (add `{ mount: "/boot/efi", fs: "esp",
size: "256M" }`). These exist because installimage does not check image
existence up front and reports a missing ESP only as an opaque `Cancelled.`
in `/root/debug.txt` — *after* the installer is already triggered.

**`installimage` exits with rc=1 / prints `Cancelled.`**
The role surfaces the last 200 lines of `/root/debug.txt` (look for the
`ERROR:` line). `Cancelled.` specifically means installimage's own config
validation rejected the layout — e.g. `ESP missing or multiple ESP found`
(see above). Other common causes: unsupported partition syntax, or an image
filetype/arch it cannot parse. A bad image *name* is caught earlier by
`validate-rescue`; a disk that does not exist is caught by `discover`.

**The post-reboot host never comes back**
installimage's grub install can fail on weird disk layouts. Reboot into
rescue manually via the Robot UI and inspect `/dev/mdX` and the EFI
partitions.

**community.hrobot.boot complains about an unknown SSH key fingerprint**
The `prepare-ssh-key` stage normally guarantees the key is in Robot, but if
you manually deleted the key, re-run with `--tags ssh_key` to re-register.

## License

MIT (see `meta/main.yml`).

## Contributing

PRs welcome. Keep changes focused and run `ansible-lint` before submitting.
