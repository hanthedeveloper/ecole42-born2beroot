*This project has been created as part of the 42 curriculum by haincel.*

# Born2beRoot

## Description

Born2beRoot is a system administration project. The goal is to set up a minimal, hardened Debian server inside a VirtualBox virtual machine, following strict rules, without any graphical interface. The project teaches how a Linux server is built, secured and maintained from scratch: partitioning, encryption, user and group management, access control, firewalling, service configuration and basic monitoring.

Overview of what the server provides:

- Debian 13 (trixie) with AppArmor active at startup
- Encrypted LVM partitions
- SSH service on port 4242 with root login disabled
- UFW firewall with only port 4242 open (IPv4 and IPv6)
- Strong password policy (expiration, complexity, difference from the previous password)
- Strict `sudo` configuration (attempt limit, custom message, I/O logging, TTY, restricted path)
- A `monitoring.sh` bash script broadcasting system information to all terminals with `wall`, on startup and every 10 minutes, scheduled by `cron`

### Personal learning goals

My main goal with this project was to move from simply using a Linux system to actually understanding how one is built and maintained.
I wanted to learn:

- **How `sudo` rules work.** I wanted to understand why privileged access must be controlled and how to do it in practice: limiting password attempts, showing a custom error message, logging every command with its input and output, forcing a TTY, and restricting the executable path. Writing these rules in `/etc/sudoers` showed me that security is the sum of many small, deliberate restrictions rather than a single setting.
- **How virtual machines work.** I wanted to understand what a hypervisor really does: how it shares CPU, memory and disk between the host and the guest, how NAT networking isolates the VM, and how port forwarding connects the two. Debugging my own SSH connection problem (a host port that was already taken) taught me how ports, listening sockets and NAT fit together far better than reading about them would have.
- **How to partition a disk manually.** Instead of using a guided installer, I wanted to design the layout myself: an encrypted partition, an LVM volume group, and separate logical volumes for `/`, `/home`, `/var`, `/srv`, `/tmp`, `/var/log` and swap. This helped me understand why separating directories matters for stability and security, and how encryption and LVM work together.
- **How to harden a server.** I wanted to apply a coherent security baseline: a firewall exposing a single port, key services restricted (SSH without root login), a strong password policy enforced through PAM, and mandatory access control with AppArmor.
- **How file permissions and groups work.** I wanted to understand `umask`, user private groups, and how to restrict a directory to a chosen set of users with groups (and ACLs), and to verify that it really works.
- **How to automate and monitor a system.** Writing `monitoring.sh` and scheduling it with `cron` and `wall` taught me how to extract information from system tools with `awk` and `grep`, how the kernel exposes data through `/proc`, and how scheduled tasks interact with the boot process.

## Instructions

### Virtual machine

- Hypervisor: VirtualBox
- Hostname: `haincel42`
- Main user: `haincel` (groups: `sudo`, `user42`)
- RAM / CPU / disk size: [TODO: e.g. RAM in MB, number of CPUs, 30 GB disk]
- Network: NAT with a port forwarding rule: host `127.0.0.1:4343` to guest `4242` (TCP)

### Installing / running

1. Create a new VM in VirtualBox and install Debian (stable) with a minimal system (no desktop environment), using manual partitioning with encrypted LVM (see "Partitioning" below).
2. Configure the system as described in the "Project description" section (sudo, SSH, UFW, password policy, AppArmor, cron).
3. Place `monitoring.sh` in `/usr/local/bin/` and make it executable, then add the cron entries below to root's crontab.
4. Start the VM, enter the disk encryption passphrase, then log in.

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

### User and group management

```
sudo adduser newuser                     # create a user (the password policy applies)
sudo groupadd evaluating                 # create a group
sudo usermod -aG evaluating newuser      # add the user to the group
getent group evaluating                  # check group members
sudo chage -l newuser                    # check password aging
sudo hostnamectl set-hostname <new>      # change the hostname (then edit /etc/hosts)
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
sha1sum "born2beroot.vdi" > signature.txt
```

To verify a signature against a freshly computed hash (no output from `diff` means they are identical):

```
sha1sum "born2beroot.vdi" > current.txt
diff signature.txt current.txt
```

[TODO: paste the real signature in signature.txt only after shutting the VM down. Starting the VM again changes it.]

## Project description

### Choice of operating system

I chose **Debian** (latest stable, 13 "trixie").

**Advantages**

- Very stable: packages are thoroughly tested before a stable release, and a new stable version comes out roughly every two years, with about five years of support including LTS.
- Large documentation and community, so most problems are already documented.
- Simple to administer: `apt`, `adduser`, `ufw`, and AppArmor enabled by default make a hardened baseline quick to reach.
- Free, independent and community-driven, with no commercial subscription model.
- Recommended for beginners in system administration, which fits the goal of understanding each step.

**Disadvantages**

- Package versions in stable are older than in rolling or fast-moving distributions.
- Fewer enterprise-oriented tools and certifications out of the box than the RHEL family.
- Support period per release is shorter than the roughly ten years of Rocky Linux.

**Why not Rocky Linux?** Rocky is an excellent choice for enterprise environments (RHEL compatibility, long lifecycle), but it uses SELinux and firewalld by default, which have a steeper learning curve. For a first server, Debian let me focus on understanding the fundamentals.

### Design choices

#### Partitioning

One encrypted partition (`sda5_crypt`) holding an LVM volume group `LVMGroup`, with separate logical volumes:

| Volume | Mount point | Size |
|---|---|---|
| root | `/` | 10G |
| swap | `[SWAP]` | 2.3G |
| home | `/home` | 5G |
| var | `/var` | 3G |
| srv | `/srv` | 3G |
| tmp | `/tmp` | 3G |
| var--log | `/var/log` | 1.9G |

`/boot` (476M) is a separate unencrypted partition, since the bootloader must read the kernel before the encrypted volume can be unlocked.

Why this layout:

- **Encryption** protects the data at rest: without the passphrase, the disk content is unreadable.
- **LVM** allows logical volumes to be resized later without repartitioning the disk.
- **Separate volumes** limit the impact of a full partition: a runaway log in `/var/log` or a full `/tmp` cannot fill `/` and crash the system. They also allow different mount options per volume.
- `/srv` holds data served by the machine, following the Filesystem Hierarchy Standard.

#### Security policies

- **Password aging** (`/etc/login.defs` and `chage`): maximum 30 days, minimum 2 days between changes, warning 7 days before expiration. `login.defs` only applies to newly created users, so existing users were updated with `chage`.
- **Password quality** (`pam_pwquality` in `/etc/pam.d/common-password`): `retry=3 minlen=10 ucredit=-1 lcredit=-1 dcredit=-1 maxrepeat=3 reject_username difok=7 enforce_for_root`. This means at least 10 characters, with at least one uppercase letter, one lowercase letter and one digit, no more than 3 identical consecutive characters, no username inside the password, at least 7 characters different from the old password, and the same rules for root.
- **sudo** (`/etc/sudoers`): `passwd_tries=3`, custom `badpass_message`, `logfile="/var/log/sudo/sudo_config"`, `log_input`, `log_output`, `iolog_dir="/var/log/sudo"`, `requiretty`, restricted `secure_path`.
- **SSH**: port 4242, `PermitRootLogin no`.
- **Firewall**: UFW active at boot, default deny incoming, only 4242/tcp allowed.
- **Mandatory access control**: AppArmor loaded and enforcing at startup.

#### User management

- Users: `root` and `haincel`.
- `haincel` belongs to the `sudo` and `user42` groups. Each user also has a private primary group (Debian's default user private group scheme).
- All passwords were changed after the password policy was applied, so that every account complies with it.
- Access to sudo is granted only through the `sudo` group, and all sudo activity is logged.

#### Services installed

| Service / package | Role |
|---|---|
| `ssh` (OpenSSH server) | Remote access on port 4242, root login disabled |
| `ufw` | Firewall, only 4242/tcp allowed |
| `sudo` | Controlled privilege escalation with logging |
| `cron` | Scheduling of `monitoring.sh` |
| `libpam-pwquality` | Password complexity enforcement through PAM |
| AppArmor | Mandatory access control, active at boot |

No graphical server is installed.

### Comparisons

**Debian vs Rocky Linux**

| | Debian | Rocky Linux |
|---|---|---|
| Family | Independent, community-driven (deb / apt) | RHEL-compatible (rpm / dnf) |
| Release model | Stable release about every 2 years | Follows RHEL major releases |
| Support length | About 5 years including LTS | Up to about 10 years |
| Default MAC | AppArmor | SELinux |
| Default firewall tool | UFW (optional front-end) | firewalld |
| Typical use | Servers, desktops, general purpose | Enterprise servers, RHEL replacement |
| Learning curve | Easier | Steeper |

**AppArmor vs SELinux**

- AppArmor is path-based: each program has a profile describing which files, capabilities and network access it may use. It is simpler to read and configure.
- SELinux is label-based: every file and process has a security context and policies decide which contexts can interact. It is more fine-grained and powerful, but more complex to write and troubleshoot.
- Both implement mandatory access control on top of the normal Unix permissions, limiting the damage if a service is compromised.

**UFW vs firewalld**

- UFW ("Uncomplicated Firewall") is a simple front-end for iptables/nftables with short commands (`ufw allow 4242`). Rules form a single ordered list, which is easy to read and enough for a single server.
- firewalld organizes rules into zones and services, distinguishes runtime from permanent configuration, and can be changed on the fly through D-Bus. It is more flexible for complex or multi-interface setups, but heavier.

**VirtualBox vs UTM**

- VirtualBox: free and open source, cross-platform (Linux, Windows, Intel macOS), easy snapshots and port forwarding. Recent versions have only limited Apple Silicon support, and then for ARM guests only.
- UTM: based on QEMU, works on Apple Silicon Macs and can emulate other architectures, but is macOS/iOS only and is often slower when emulating a different architecture.

## Resources

- Debian documentation: https://www.debian.org/doc/
- Debian Administrator's Handbook: https://debian-handbook.info/
- Filesystem Hierarchy Standard: https://refspecs.linuxfoundation.org/fhs.shtml
- `man` pages: `sudoers`, `sshd_config`, `ufw`, `pam_pwquality`, `chage`, `crontab`, `wall`, `lvm`, `awk`, `umask`, `chmod`, `setfacl`, `sha1sum`, `diff`
- Ubuntu/Debian UFW guide: https://help.ubuntu.com/community/UFW
- AppArmor wiki: https://gitlab.com/apparmor/apparmor/-/wikis/home
- VirtualBox manual: https://www.virtualbox.org/manual/

### Use of AI

AI was used as a learning and troubleshooting assistant, not to replace my own work.

- **Troubleshooting:** the VirtualBox port forwarding problem (a host port already in use).
- **`monitoring.sh`:** understanding and debugging the script and the cron scheduling aligned on boot time.
