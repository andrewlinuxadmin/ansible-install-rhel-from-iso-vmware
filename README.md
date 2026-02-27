# install-rhel-from-iso-vmware

Ansible role that provisions a VM on VMware vSphere and
performs a RHEL installation using the OS ISO with Anaconda
Kickstart. The Kickstart file is delivered to the VM via a
**second ISO image** (labeled `OEMDRV`), eliminating the
need for an external HTTP server.

## Role structure

```text
.
├── defaults/main.yml
├── meta/
│   ├── main.yml
│   └── requirements.yml
├── tasks/main.yml
├── templates/
│   └── ks.cfg.j2
└── examples/
    ├── ansible.cfg
    └── vmware-install-vm.yml
```

## Prerequisites

| Requirement | Details |
| --- | --- |
| Ansible | >= 2.14 |
| Python | >= 3.9 with `pyvmomi` and `requests` |
| Collection | `community.vmware` >= 4.0 |
| xorriso | `xorriso` package on the controller |
| vCenter/ESXi | Access with permissions to create VMs |
| RHEL ISO | Uploaded to an accessible datastore |

## Installation

### Install the role

From a git repository:

```bash
# Using ansible-galaxy
ansible-galaxy role install \
  git+https://github.com/andrewlinuxadmin/ansible-install-rhel-from-iso-vmware.git,main

# Or from a requirements.yml file
ansible-galaxy role install -r requirements.yml
```

Example `requirements.yml`:

```yaml
roles:
  - src: https://github.com/andrewlinuxadmin/ansible-install-rhel-from-iso-vmware.git
    scm: git
    version: main
    name: install_rhel_from_iso_vmware
```

### Install dependencies

```bash
# System dependencies (kickstart ISO creation)
# RHEL/Fedora
sudo dnf install xorriso

# Debian/Ubuntu
sudo apt install xorriso

# Python dependencies
pip install pyvmomi requests

# Ansible collection dependencies
ansible-galaxy collection install -r \
  ~/.ansible/roles/install_rhel_from_iso_vmware/meta/requirements.yml
```

## Configuration

### 1. Playbook

Create a playbook that includes the role:

```yaml
---
- name: Provision VMware VM and install RHEL via Kickstart
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: Verify xorrisofs is installed
      ansible.builtin.command:
        cmd: which xorrisofs
      changed_when: false

    - name: Install RHEL from ISO on VMware
      ansible.builtin.include_role:
        name: install_rhel_from_iso_vmware
      vars:
        vcenter_hostname: "vcenter.example.com"
        vcenter_username: "administrator@vsphere.local"
        vcenter_password: "{{ vault_vcenter_password }}"
        vcenter_datacenter: "Datacenter"
        vcenter_cluster: "Cluster01"
        vcenter_datastore: "datastore1"
        vcenter_network: "VM Network"

        vm_name: "rhel9-server"
        vm_network_ip: "192.168.1.100"
        vm_network_mask: "255.255.255.0"
        vm_network_gateway: "192.168.1.1"
        vm_network_dns: "192.168.1.1"
        vm_hostname: "rhel9-server.example.com"

        rhel_iso_path: "[datastore1] iso/rhel-9.4-x86_64-dvd.iso"
        ks_root_password: "{{ vault_ks_root_password }}"
        ks_timezone: "America/Sao_Paulo"
```

A complete example is available in
[examples/vmware-install-vm.yml](examples/vmware-install-vm.yml).

### 2. Vault (passwords)

Create a vault to securely store passwords:

```bash
ansible-vault create vault.yml
```

Vault contents:

```yaml
vault_vcenter_password: "your-vcenter-password"
vault_ks_root_password: "$6$rounds=4096$salt$hash..."
vault_ks_user_password: "$6$rounds=4096$salt$hash..."
```

To generate password hashes compatible with Kickstart:

```bash
python3 -c "import crypt; print(crypt.crypt(\
'YourPassword', crypt.mksalt(crypt.METHOD_SHA512)))"
```

### 3. ISO upload

Upload the RHEL ISO to the VMware datastore and set
the path in `rhel_iso_path`:

```yaml
rhel_iso_path: "[datastore1] iso/rhel-9.4-x86_64-dvd.iso"
```

## Variable reference

### vCenter / ESXi

| Variable | Description | Req | Default |
| --- | --- | --- | --- |
| `vcenter_hostname` | vCenter address | Yes | — |
| `vcenter_username` | vCenter user | Yes | — |
| `vcenter_password` | vCenter password | Yes | — |
| `vcenter_validate_certs` | Validate SSL | No | `false` |
| `vcenter_datacenter` | Datacenter name | Yes | — |
| `vcenter_cluster` | Cluster name | Yes | — |
| `vcenter_datastore` | Datastore | Yes | — |
| `vcenter_network` | VM network | Yes | — |
| `vcenter_folder` | VM folder | No | (root) |

### VM — Hardware

| Variable | Description | Req | Default |
| --- | --- | --- | --- |
| `vm_name` | VM name | Yes | — |
| `vm_guest_id` | Guest OS ID | No | `"rhel9_64Guest"` |
| `vm_num_cpus` | vCPUs | No | `2` |
| `vm_memory_mb` | RAM (MB) | No | `4096` |
| `vm_disk_size_in_gb` | OS disk (GB) | No | `60` |
| `vm_disk_type` | Disk type | No | `"thin"` |
| `vm_wait_for_ip_timeout` | IP timeout (s) | No | `900` |
| `vm_force_recreate` | Delete and recreate | No | `false` |

### Network

| Variable | Description | Req | Default |
| --- | --- | --- | --- |
| `vm_network_ip` | VM static IP | Yes | — |
| `vm_network_mask` | Network mask | Yes | — |
| `vm_network_gateway` | Default gateway | Yes | — |
| `vm_network_dns` | DNS server | Yes | — |
| `vm_hostname` | VM FQDN | Yes | — |

### ISO / Kickstart

| Variable | Description | Req | Default |
| --- | --- | --- | --- |
| `rhel_iso_path` | ISO path on datastore | Yes | — |
| `ks_filename` | Kickstart filename | No | `"ks.cfg"` |

### OS — Localization and security

| Variable | Description | Req | Default |
| --- | --- | --- | --- |
| `ks_root_password` | Root password (SHA-512) | Yes | — |
| `ks_timezone` | System timezone | Yes | — |
| `ks_locale` | System locale | No | `"en_US.UTF-8"` |
| `ks_keyboard` | Keyboard layout | No | `"us"` |

### Additional user (optional)

If `ks_user_name` is not defined, only the root user will
be created. If defined, `ks_user_password` is required.

| Variable | Description | Req | Default |
| --- | --- | --- | --- |
| `ks_user_name` | Username | No | — |
| `ks_user_password` | Password (SHA-512) | Cond | — |
| `ks_user_gecos` | Full name (GECOS) | No | `""` |
| `ks_user_is_admin` | Add to wheel group | No | `false` |

### Partitioning — OS disk (LVM)

| Variable | Description | Default |
| --- | --- | --- |
| `ks_disk_device` | OS disk device | `"sda"` |
| `ks_vg_name` | Volume Group name | `"vg_rhel"` |

The `ks_lvs` variable defines the logical volumes on the
OS disk. Each entry requires `name`, `mount`, `size` (MB)
and optionally `fstype` (default: `xfs`). Use `swap` for
both `mount` and `fstype` for the swap partition.

The role validates that the total LV size does not exceed
the disk capacity (minus 1624 MB for `/boot` + `/boot/efi`)
and that a root partition (`/`) is present.

Default:

```yaml
ks_lvs:
  - name: lv_root
    mount: /
    size: 20480
    fstype: xfs
  - name: lv_var
    mount: /var
    size: 10240
    fstype: xfs
  - name: lv_var_log
    mount: /var/log
    size: 10240
    fstype: xfs
  - name: lv_home
    mount: /home
    size: 5120
    fstype: xfs
  - name: lv_swap
    mount: swap
    size: 4096
    fstype: swap
```

### Extra disk (optional)

The `vm_extra_disk` variable adds a second disk to the VM
with a custom VG and LVs. If not defined, the VM will only
have the OS disk.

```yaml
vm_extra_disk:
  size_gb: 100
  type: "thin"
  device: "sdb"
  vg_name: "vg_data"
  lvs:
    - name: lv_data
      mount: /data
      size: 51200
      fstype: xfs
    - name: lv_apps
      mount: /opt/apps
      size: 20480
      fstype: xfs
    - name: lv_backup
      mount: /backup
      size: 20480
      fstype: xfs
```

### SSH — Post-installation (optional)

The `ks_ssh_config` variable is a dictionary of
`sshd_config` directives applied in the Kickstart `%post`
section. If not defined, the default RHEL `sshd_config` is
left unchanged.

Default:

```yaml
ks_ssh_config:
  PermitRootLogin: "no"
  PasswordAuthentication: "yes"
```

Additional examples:

```yaml
ks_ssh_config:
  PermitRootLogin: "no"
  PasswordAuthentication: "yes"
  MaxAuthTries: "3"
  ClientAliveInterval: "300"
```

### Extra packages (optional)

The `ks_extra_packages` variable is a list of additional
packages to install beyond the base
`@^server-product-environment` group. If not defined,
no extra packages are installed.

```yaml
ks_extra_packages:
  - vim-enhanced
  - tmux
  - bash-completion
  - net-tools
  - bind-utils
```

### Kickstart ISO — internal role variables

These variables have defaults in `defaults/main.yml` and
typically do not need to be changed.

| Variable | Description | Default |
| --- | --- | --- |
| `ks_iso_work_dir` | Temp directory | `"/tmp/kickstart_iso"` |
| `ks_iso_image` | Local ISO path | `"...dir.../vm-ks.iso"` |
| `ks_iso_label` | ISO volume label | `"OEMDRV"` |
| `ks_iso_datastore` | Datastore for ISO | `vcenter_datastore` |
| `ks_iso_datastore_dir` | Datastore dir | `"kickstart"` |
| `ks_iso_datastore_path` | Full datastore path | `"[ds] ks/vm-ks.iso"` |

## Usage

```bash
# Default execution
ansible-playbook install-vm.yml --ask-vault-pass

# With verbose output
ansible-playbook install-vm.yml --ask-vault-pass -vvv

# Overriding variables
ansible-playbook install-vm.yml --ask-vault-pass \
  -e "vm_name=my-server vm_network_ip=10.0.0.50"

# Force recreate an existing VM
ansible-playbook install-vm.yml --ask-vault-pass \
  -e vm_force_recreate=true
```

## Execution flow

1. **Check VM** — aborts if VM already exists
   (unless `vm_force_recreate=true`)
2. **Validate variables** — asserts all required
   variables are defined
3. **Generate Kickstart** — renders `ks.cfg.j2` with
   the configured variables
4. **Build kickstart ISO** — creates an ISO image
   (labeled `OEMDRV`) containing the Kickstart file
   using `xorrisofs`
5. **Upload to datastore** — sends the kickstart ISO
   to the VMware datastore
6. **Create VM** — provisions the VM on vSphere with
   CPU, memory, disk(s), network, install ISO (CD-ROM 0)
   and kickstart ISO (CD-ROM 1)
7. **Power on VM** — Anaconda boots from the install
   ISO and auto-detects the `OEMDRV` kickstart ISO
8. **Wait for installation** — monitors until the VM
   obtains an IP and SSH becomes reachable
9. **Cleanup** — shuts down the VM, unmounts both ISOs,
   removes the kickstart CD-ROM device, powers on again,
   removes the kickstart ISO from the datastore and
   cleans up local temp files

## Important notes

- The role **aborts** if a VM with the same name already
  exists. Use `-e vm_force_recreate=true` to delete and
  recreate it
- Kickstart passwords can be SHA-512 hashes (starting
  with `$`) or plain text (auto-detected). SHA-512
  hashes are recommended for production
- The kickstart CD-ROM device is **removed** from the
  VM after installation
- The kickstart ISO is deleted from the datastore after
  installation
- Default installation timeout is 900s (15 min); adjust
  `vm_wait_for_ip_timeout` as needed
- The `xorriso` package (provides `xorrisofs`) is
  required on the Ansible controller to build the
  kickstart ISO
- SELinux is set to `enforcing` and firewall is enabled
  with SSH by default
