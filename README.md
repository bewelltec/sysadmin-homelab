# SysAdmin Homelab with Ansible

**21 November 2026 · VMware Workstation · Five virtual machines**

The five-machine lab is operational. The RHEL control node manages three Linux clients over SSH and one Windows server over WinRM. Disk health and alert decisions were tested on Linux and Windows. Ubuntu’s root filesystem was expanded manually by 1 GiB, and its on-demand Ansible resize playbook was tested below the growth threshold.

## Virtual machines and networks

| VM | Operating system | Lab address | Role |
|---|---|---|---|
| rhel-ansible | RHEL 10.2 | 10.50.10.10 | Ansible control node |
| rhel-client01 | RHEL 10.2 | 10.50.10.11 | Linux managed client |
| rhel-client02 | RHEL 9.8 | 10.50.10.12 | Linux managed client |
| win-server01 | Windows Server 2025 Standard Evaluation, Desktop Experience | 10.50.10.13 | Windows managed server |
| ubuntu-client01 | Ubuntu Server | 10.50.10.14 | Linux managed client and LVM growth test host |

| VMware network | Subnet | Purpose |
|---|---|---|
| VMnet2, host-only | 10.50.10.0/24 | SysAdmin management network |
| VMnet8, NAT | 192.168.20.0/24 | Internet access for guests configured with a NAT adapter |
| VMnet3, host-only | 10.50.20.0/24 | Reserved for the SOC lab |

## Connect and verify management

From Windows PowerShell, use `ssh labadmin@10.50.10.10`. Run all Ansible commands below from the `labadmin@rhel-ansible:~$` prompt. The control node ran Ansible Core 2.16.19. Its inventory is `~/ansible-lab/inventory.ini`.

| Inventory group | Hosts | Connection |
|---|---|---|
| managed_linux | rhel-client01, rhel-client02, ubuntu-client01 | SSH as labadmin using the control node’s ED25519 key |
| managed_windows | win-server01 | WinRM on HTTP port 5985 with NTLM and message encryption set to always |

If the SSH agent is absent, start it in the current terminal. Load the key when the agent has no identities. Use the SSH **key passphrase** at the `ssh-add` prompt.

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
ansible-inventory -i ~/ansible-lab/inventory.ini --graph
```

```bash
ANSIBLE_SSH_ARGS='-o ControlMaster=no -o ControlPath=none' \
  ansible managed_linux -i ~/ansible-lab/inventory.ini -m ansible.builtin.ping

ansible win-server01 -i ~/ansible-lab/inventory.ini --ask-pass \
  -m ansible.windows.win_ping
```

Both tests returned `pong`. The Linux command disables SSH connection reuse because earlier runs with reuse enabled returned connection closures. At the Windows test’s `SSH password:` prompt, enter the Windows Administrator password. The inventory selects WinRM for that host. At a Linux `BECOME password:` prompt, use the target VM’s `labadmin` sudo password.

### Troubleshooting observed during setup

| Symptom | Resolution used in the lab |
|---|---|
| Inventory gave an INI parse warning or listed the second RHEL client twice | Fixed the malformed inventory and removed the misspelled duplicate. Verified the group list with `ansible-inventory --graph`. |
| Several SSH passphrase prompts appeared at once | Loaded the ED25519 key into the SSH agent in the control-node terminal before running Ansible. |
| Windows TCP connection returned HTTP 405 to a GET request | The listener was reachable and expected POST. Confirmed authenticated management separately with `win_ping`. |
| WinRM reported `No module named winrm` | Installed `pywinrm` for the control-node Python used by Ansible and verified Windows management with the `ansible.windows` collection. |
| Ubuntu sudo worked interactively while Ansible timed out | The traditional `/usr/bin/sudo.ws` worked as an Ubuntu-specific `ansible_become_exe` override. The resize playbook carries that setting. |
| Ubuntu said `ansible-playbook` was missing | Returned to `rhel-ansible`, where the playbooks are installed. |

## Playbooks on the control node

| File under `~/ansible-lab/` | Purpose and verified result |
|---|---|
| `linux-baseline.yml` | Linux baseline information. Two tasks passed per client with zero changes and failures. |
| `windows-baseline.yml` | Windows baseline information. Two tasks passed with zero changes and failures. |
| `linux-disk-health.yml` | Checks Linux root filesystem usage. Ubuntu produced ALERT at exactly the temporary 44% threshold. |
| `windows-disk-health.yml` | Checks the Windows system drive. C: reported 12% and OK at the saved 98% threshold, then ALERT with a temporary 10% threshold. |
| `ubuntu-growth-preview.yml` | Read-only test of the growth decision. Verified no growth at the saved threshold, insufficient-capacity blocking, and a proposed 1 GiB extension at a temporary threshold. |
| `ubuntu-auto-grow.yml` | On-demand 1 GiB root growth on Ubuntu. Both the default run and the explicit apply run reported no growth at 41%, with zero changes and failures. |

## Last recorded disk readings

| VM | Filesystem | Used | Available inside filesystem | Free LVM capacity |
|---|---|---:|---:|---:|
| rhel-client01 | / | 6% | 41 GiB | 0 GiB |
| rhel-client02 | / | 9% | 32 GiB | 0 GiB |
| ubuntu-client01 | / | 41% | 8.3 GiB | 13 GiB |
| win-server01 | C: | 11.3% raw, 12% rounded by playbook | 87.8 GiB | — |

Ubuntu uses ext4 on `/dev/ubuntu-vg/ubuntu-lv`. The manual 1 GiB test grew the LV from about 14 GiB to 15 GiB, reduced usage from 44% to 41%, and reduced unallocated VG capacity from 14 GiB to 13 GiB. Available filesystem space and unallocated LVM capacity measure different things. The RHEL clients had space for files but no free LVM capacity for growth.

## Run the Ubuntu growth check

```bash
ANSIBLE_SSH_ARGS='-o ControlMaster=no -o ControlPath=none' \
  ansible-playbook -i ~/ansible-lab/inventory.ini \
  ~/ansible-lab/ubuntu-auto-grow.yml --ask-become-pass
```

The default run checks eligibility without growing the disk. The playbook checks the root mount and ext4 device, requires at least 98% usage and at least 1 GiB free in `ubuntu-vg`, and uses a lock to prevent overlapping runs of the same playbook on Ubuntu. Add `-e apply_growth=true` only when you intend to allow one 1 GiB increment if the conditions are met.

After a resize attempt, verify from the Ubuntu shell:

```bash
df -h /
sudo lvs -o lv_path,lv_size /dev/ubuntu-vg/ubuntu-lv
sudo vgs -o vg_name,vg_free ubuntu-vg
```

If the resize command reports an error, compare both the filesystem and logical volume sizes before retrying. The LV may have grown before the filesystem operation failed.

## Resume point

- **SysAdmin setup complete.** Management access, baseline playbooks, disk alerts, manual Ubuntu expansion, and the below-threshold growth guard have been verified.
- **Still unverified.** An Ansible-triggered resize at 98% has not been exercised. The resize playbook runs on demand. No scheduled run was set up in this lab.
- **SOC lab next.** In VMware Workstation, open **Edit → Virtual Network Editor** and confirm VMnet3 is host-only with subnet `10.50.20.0` and mask `255.255.255.0` before creating the first SOC VM.
