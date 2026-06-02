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
4. **Enumerates disks** in the rescue and validates that the operator-supplied
   list of OS disks actually exists on the host.
5. **Runs `installimage`** with a templated config that wipes only the OS
   disks; all other block devices are left pristine.
6. **Reboots** into the installed system and waits for SSH to come up on the
   real OS, with its new host key accepted.

After the role finishes, the host is reachable as root over SSH with the
public key you provided, on the same IP it was ordered with. From there, any
downstream role can pick up.

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
| `hetzner_bootstrap_image` | `Debian-1303-trixie-amd64-base.tar.zst` | OS image. Must exist in the rescue (pre-flight `validate-rescue` asserts this and lists what's available on mismatch). Pinned, not `-latest-`, for reproducible Proxmox installs. |
| `hetzner_bootstrap_images_dir` | `/root/.oldroot/nfs/install/../images/` | Directory the rescue serves images from (shared by the `IMAGE` line and the existence check) |
| `installimage_swraid` | `1` | `0`/`1`. Auto-disabled if only one OS disk is configured. |
| `installimage_swraid_level` | `1` | `0`, `1`, `5`, `6`, or `10` |
| `installimage_partitions` | `/boot/efi esp 256M, /boot 1G, swap 32G, / all` | List of partition dicts, each `{mount, fs, size}`. The ESP is required on UEFI hosts (pre-flight asserts exactly one); installimage mirrors it across SWRAID members. |
| `hetzner_bootstrap_installimage_config_path` | `/root/installimage.conf` | Where the rendered config is written and passed to `installimage -c`. Must NOT be `/autosetup`. |
| `installimage_hostname` | `{{ inventory_hostname_short }}` | Hostname written by installimage |
| `installimage_bootloader` | `grub` | `grub` or `lilo` |
| `installimage_post_install_script` | *unset* | Optional path to a script installimage runs in the chroot |
| `hetzner_bootstrap_ssh_public_key_path` | `~/.ssh/id_ed25519.pub` | Local pubkey to register with Robot |
| `hetzner_bootstrap_force_reinstall` | `false` | Allow reinstall of a server that is currently up |
| `hetzner_bootstrap_reset_type` | `hardware` | `hardware`, `power`, `manual`, `software` |
| `hetzner_bootstrap_rescue_wait_seconds` | `600` | How long to wait for rescue SSH |
| `hetzner_bootstrap_installimage_timeout_seconds` | `1800` | Hard ceiling on installer runtime |
| `hetzner_bootstrap_installed_wait_seconds` | `600` | How long to wait for installed SSH |

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

## Tags

| Tag | What it runs |
|---|---|
| `hetzner_bootstrap` | The whole flow (alias for "everything") |
| `validate` | Local validation only — never talks to Robot or the server |
| `ssh_key` | Register the SSH key with Robot |
| `rescue` | Boot config + reset + wait for rescue SSH |
| `discover` | Enumerate and validate disks in the rescue |
| `preflight` | Fail-fast checks against the live rescue: image exists, UEFI/ESP sane |
| `installimage` | Render config and run the installer |
| `await` | Wait for the installed system to come up |

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
