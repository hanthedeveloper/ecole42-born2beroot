*This project has been created as part of the 42 curriculum by haincel.*

# Born2beRoot

## Description

Born2beRoot is a system administration project. The goal is to set up a minimal, hardened Debian server inside a VirtualBox virtual machine, following strict rules, without any graphical interface.

Overview of what the server provides:

- Debian 13 (trixie) with AppArmor active at startup
- Encrypted LVM partitions
- SSH service on port 4242 with root login disabled
- UFW firewall with only port 4242 open (IPv4 and IPv6)
- Strong password policy (expiration, complexity, history difference)
- Strict `sudo` configuration (attempt limit, custom message, I/O logging, TTY, restricted path)
- A `monitoring.sh` bash script broadcasting system information to all terminals with `wall`, on startup and every 10 minutes, scheduled by `cron`

### Personal learning goals

My main goal with this project was to move from simply using a Linux system to actually understanding how one is built and maintained.
I wanted to learn:

- **How `sudo` rules work.** I wanted to understand why privileged access must be controlled and how to do it in practice: limiting password attempts, showing a custom error message, logging every command with its input and output, forcing a TTY, and restricting the executable path. Writing these rules in `/etc/sudoers` showed me that security is the sum of many small, deliberate restrictions rather than a single setting.
- **How virtual machines work.** I wanted to understand what a hypervisor really does: how it shares CPU, memory and disk between the host and the guest, how NAT networking isolates the VM, and how port forwarding connects the two. Debugging my own SSH connection problem (a host port that was already taken) taught me how ports, listening sockets and NAT fit together far better than reading about them would have.
- **How to partition a disk manually.** Instead of using a guided installer, I wanted to design the layout myself: an encrypted partition, an LVM volume group, and separate logical volumes for `/`, `/home`, `/var`, `/srv`, `/tmp`, `/var/log` and swap. This helped me understand why separating directories matters for stability and security, and how encryption and LVM work together.
- **How to harden a server.** I wanted to apply a coherent security baseline: a firewall exposing a single port, key services restricted (SSH without root login), a strong password policy enforced through PAM, and mandatory access control with AppArmor.
- **How to automate and monitor a system.** Writing `monitoring.sh` and scheduling it with `cron` and `wall` taught me how to extract information from system tools with `awk` and `grep`, how the kernel exposes data through `/proc`, and how scheduled tasks interact with the boot process.

## Instructions

### Virtual machine

- Hypervisor: VirtualBox
- Hostname: `haincel42`
- Main user: `haincel` (groups: `sudo`, `user42`)
- RAM / CPU / disk size: [TODO: e.g. RAM in MB, number of CPUs, 30 GB disk]
- Network: NAT with a port forwarding rule: host `127.0.0.1:4343` to guest `4242` (TCP)

### Installing / running

1. Create a new VM in VirtualBox and install Debian (stable) with a minimal system (no desktop environment), using encrypted LVM partitioning.
2. Configure the system as described in the "Project description" section.
3. Start the VM, enter the disk encryption passphrase, then log in.

### Connecting by SSH

From the host:

```
ssh haincel@localhost -p 4343
```

(Root login over SSH is disabled.) The host port `4343` is forwarded to the guest port `4242`. Another host port can be used if it is already taken.

### Useful checks

```
head -n 2 /etc/os-release
aa-status | head -n 3
lsblk
sshd -T | grep -Ei "^(port|permitrootlogin)"
ufw status verbose
chage -l haincel
id haincel
sudo -l
crontab -l
```

### Monitoring script

`monitoring.sh` is stored in `/usr/local/bin/` and is run by root's crontab:

```
@reboot sleep 10 && /usr/local/bin/monitoring.sh
*/10 * * * * sleep $(uptime -s | awk -F: '{print $2\%10*60+$3}') && /usr/local/bin/monitoring.sh
```

- `@reboot` runs the script once at startup, after a short delay so that the network is ready.
- The second line runs every 10 minutes. The `sleep` aligns the broadcast on the boot time instead of the wall clock.
- To stop it without modifying the script: `systemctl stop cron`.

It displays: architecture and kernel, physical and virtual CPUs, RAM usage, disk usage, CPU load, last boot, LVM status, established TCP connections, number of logged users, IPv4 and MAC address, and the number of sudo commands.

### Signature

The `signature.txt` file contains the SHA-1 of the VM's `.vdi` disk:

```
sha1sum "born2beroot.vdi"
```

[TODO: paste the real signature in signature.txt only after shutting the VM down. Starting the VM again changes it.]

## Project description

### Choice of operating system

I chose **Debian** (latest stable, 13 "trixie").

- Advantages: very stable, large documentation and community, simple to administer, AppArmor and UFW are easy to use, recommended for beginners in system administration.
- Disadvantages: older package versions than some distributions, fewer enterprise-oriented tools than Rocky Linux out of the box.

### Design choices

**Partitioning.** One encrypted partition (`sda5_crypt`) holding an LVM volume group `LVMGroup`, with separate logical volumes:

| Volume | Mount point | Size |
|---|---|---|
| root | `/` | 10G |
| swap | `[SWAP]` | 2.3G |
| home | `/home` | 5G |
| var | `/var` | 3G |
| srv | `/srv` | 3G |
| tmp | `/tmp` | 3G |
| var--log | `/var/log` | 1.9G |

`/boot` (476M) is a separate unencrypted partition. Separate volumes limit the impact of a full partition, and the encryption protects the data at rest.

**Security policies.**

- Password aging (`/etc/login.defs` and `chage`): maximum 30 days, minimum 2 days between changes, warning 7 days before expiration.
- Password quality (`pam_pwquality` in `/etc/pam.d/common-password`): `retry=3 minlen=10 ucredit=-1 lcredit=-1 dcredit=-1 maxrepeat=3 reject_username difok=7 enforce_for_root`.
- sudo (`/etc/sudoers`): `passwd_tries=3`, custom `badpass_message`, `logfile="/var/log/sudo/sudo_config"`, `log_input`, `log_output`, `iolog_dir="/var/log/sudo"`, `requiretty`, restricted `secure_path`.
- SSH: port 4242, `PermitRootLogin no`.
- Firewall: UFW active at boot, default deny incoming, only 4242/tcp allowed.
- AppArmor loaded and enforcing at startup.

**User management.** `root` and `haincel`; `haincel` belongs to the `sudo` and `user42` groups. All passwords were changed after the policy was applied.

**Services installed.** `ssh`, `ufw`, `sudo`, `cron`, `libpam-pwquality`, AppArmor. No graphical server is installed.

### Comparisons

**Debian vs Rocky Linux**

| | Debian | Rocky Linux |
|---|---|---|
| Family | Independent, community-driven (deb / apt) | RHEL-compatible (rpm / dnf) |
| Release model | Stable releases, long support | Enterprise-oriented, long support |
| Default MAC | AppArmor | SELinux |
| Default firewall tool | UFW (optional front-end) | firewalld |
| Learning curve | Easier | Steeper |

**AppArmor vs SELinux**

- AppArmor is path-based: each program has a profile describing which files and capabilities it may use. It is simpler to read and configure.
- SELinux is label-based: every file and process has a security context and policies decide which contexts can interact. It is more fine-grained and powerful, but more complex.

**UFW vs firewalld**

- UFW ("Uncomplicated Firewall") is a simple front-end for iptables/nftables with short commands (`ufw allow 4242`). Rules are static and reloaded as a whole.
- firewalld manages rules with zones and services, and supports changes at runtime without dropping connections. It is more flexible but heavier.

**VirtualBox vs UTM**

- VirtualBox: free, cross-platform (Linux, Windows, Intel macOS), easy snapshots and port forwarding, but limited support on Apple Silicon.
- UTM: based on QEMU, works on Apple Silicon Macs (and can emulate other architectures), but is macOS/iOS only and often slower when emulating.

## Resources

- Debian documentation: https://www.debian.org/doc/
- Debian Administrator's Handbook: https://debian-handbook.info/
- `man` pages: `sudoers`, `sshd_config`, `ufw`, `pam_pwquality`, `chage`, `crontab`, `wall`, `lvm`, `awk`
- Ubuntu/Debian UFW guide: https://help.ubuntu.com/community/UFW
- AppArmor wiki: https://gitlab.com/apparmor/apparmor/-/wikis/home
- VirtualBox manual: https://www.virtualbox.org/manual/

### Use of AI

AI was used as a troubleshooting assistant, not to replace my own work:

- Troubleshooting the VirtualBox port forwarding problem (a host port already in use) and understanding how ports work.
- Helping me understand and debug the `monitoring.sh` script and the cron scheduling aligned on boot time.

