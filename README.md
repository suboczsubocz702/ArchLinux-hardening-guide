# Complete Arch Linux Hardening Guide

## Enterprise-Style Workstation Security Hardening

### Target Platform

- Distribution: Arch Linux
- Boot Mode: UEFI
- Filesystem: Btrfs
- Encryption: LUKS2
- Boot Security: Secure Boot + UKI
- Kernel: linux-hardened
- MAC Framework: AppArmor
- Auditing: auditd
- Recovery: Timeshift + Btrfs snapshots

---

# Table of Contents

0. Threat Model
1 Important takes
2. Installation Philosophy
3. Disk Layout
4. Installing Arch Linux
5. LUKS2 Encryption
6. Btrfs Layout
7. Secure Boot
8. Unified Kernel Images (UKI)
9. linux-hardened
10. AppArmor
11. auditd
12. Firewall
13. Fail2ban
14. Filesystem Hardening
15. Sysctl Hardening
16. USB Security
17. Logging
18. Package Auditing
19. Malware / Rootkit Detection
20. Snapshot and Recovery Strategy
21. User and Authentication Hardening
22. Service Hardening
23. Network Hardening
24. Operational Security Practices
25. Recommended Maintenance Workflow
26. Final Security Checklist

---

# 0. Threat Model

This hardening guide targets:

- personal workstation security,
- developer workstations,
- advanced Linux desktop users,
- security-conscious laptop deployments,
- mobile workstation environments.

The guide prioritizes:

- strong real-world security,
- operational stability,
- reasonable desktop usability,
- maintainability,
- modern Linux security primitives.

This guide intentionally avoids:

- unrealistic compliance-only hardening,
- excessive usability degradation,
- enterprise-only complexity,
- impractical immutable workflows.

---

# 1. Important takes
- For a more secure system search for QubesOS, Whonix, TailsOS
- Most imporant is proper PERSEC, that is securing your personal information from people whom you don't want to share it
- Avoid ecosystems like google, proton ecosystem, which in itself is pretty good (proton), but I'll redirect you to The Hated One yt video about Proton Ecosystems
- Use Tor Browser if in need to do something sketchy or embarrasing
- Use Monero when possible, the guide to convering money or other crypto was covered by Mental Outlaw in his yt video
- If possible, self host
- Try not using your root user to do anything

---

# 2. Installation Philosophy

The security architecture is based on layered defense:

| Layer | Technology |
|---|---|
| Firmware security | UEFI + Secure Boot |
| Boot chain integrity | UKI + sbctl |
| Storage protection | LUKS2 |
| Kernel hardening | linux-hardened |
| Access control | AppArmor |
| Auditing | auditd |
| Network protection | nftables/UFW |
| Intrusion prevention | Fail2ban |
| Integrity monitoring | AIDE/auditd |
| Recovery | Btrfs + Timeshift |
| Vulnerability management | arch-audit |

---

# 3. Disk Layout

Recommended layout:

| Partition | Purpose |
|---|---|
| EFI System Partition | /boot |
| LUKS2 Container | encrypted root |
| Btrfs subvolumes | snapshots and recovery |

Recommended Btrfs subvolumes:

```text
@
@home
@log
@cache
@snapshots
```

---

# 4. Installing Arch Linux

Boot Arch ISO in UEFI mode.

Verify:

```bash
ls /sys/firmware/efi
```

If directory exists, UEFI mode is active.

---

# 5. LUKS2 Encryption

## Create encrypted partition

```bash
cryptsetup luksFormat --type luks2 /dev/nvme0n1p2
```

Open encrypted container:

```bash
cryptsetup open /dev/nvme0n1p2 cryptroot
```

---

# 6. Btrfs Layout

## Create filesystem

```bash
mkfs.btrfs /dev/mapper/cryptroot
```

Mount:

```bash
mount /dev/mapper/cryptroot /mnt
```

Create subvolumes:

```bash
btrfs su cr /mnt/@
btrfs su cr /mnt/@home
btrfs su cr /mnt/@log
btrfs su cr /mnt/@cache
btrfs su cr /mnt/@snapshots
```

Unmount:

```bash
umount /mnt
```

Mount final layout:

```bash
mount -o noatime,compress=zstd,subvol=@ /dev/mapper/cryptroot /mnt
mkdir -p /mnt/{home,var/log,var/cache,.snapshots}
mount -o noatime,compress=zstd,subvol=@home /dev/mapper/cryptroot /mnt/home
mount -o noatime,compress=zstd,subvol=@log /dev/mapper/cryptroot /mnt/var/log
mount -o noatime,compress=zstd,subvol=@cache /dev/mapper/cryptroot /mnt/var/cache
mount -o noatime,compress=zstd,subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots
```

---

# 7. Install Base System

```bash
pacstrap -K /mnt base linux-hardened linux-firmware vim sudo networkmanager apparmor audit sbctl
```

Generate fstab:

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

Chroot:

```bash
arch-chroot /mnt
```

---

# 8. linux-hardened

Install hardened kernel:

```bash
pacman -S linux-hardened linux-hardened-headers
```

Benefits:

- additional exploit mitigations,
- hardened defaults,
- memory corruption resistance,
- stricter kernel behavior.

---

# 9. AppArmor

Install:

```bash
pacman -S apparmor
```

Enable kernel parameters:

Edit:

```text
/etc/default/grub
```

Append:

```text
apparmor=1 security=apparmor
```

Regenerate:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

Enable service:

```bash
systemctl enable apparmor
```

Reboot.

Verify:

```bash
aa-status
```

---

# 10. auditd

Install:

```bash
pacman -S audit
```

Enable:

```bash
systemctl enable --now auditd
```

---

## Audit Rules

Create:

```text
/etc/audit/rules.d/rootkit.rules
```

Content:

```text
-w /usr/bin -p wa -k binaries
-w /usr/sbin -p wa -k binaries
-w /etc/passwd -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/sudoers -p wa -k privilege
-w /etc/systemd/system -p wa -k persistence
-w /etc/ssh/sshd_config -p wa -k ssh
```

Load rules:

```bash
augenrules --load
```

Verify:

```bash
auditctl -l
```

---

# 11. Firewall

Recommended:

- nftables
- or UFW

Install UFW:

```bash
pacman -S ufw
```

Default deny:

```bash
ufw default deny incoming
ufw default allow outgoing
```

Enable:

```bash
ufw enable
```

---

# 12. Fail2ban

Install:

```bash
pacman -S fail2ban
```

Enable:

```bash
systemctl enable --now fail2ban
```

Verify:

```bash
fail2ban-client status
```

---

# 13. Secure Boot

## Install sbctl

```bash
pacman -S sbctl
```

Create keys:

```bash
sbctl create-keys
```

Reboot into firmware settings.

Clear existing Secure Boot keys.

Firmware must enter Setup Mode.

Verify:

```bash
sbctl status
```

Expected:

```text
Setup Mode: Enabled
```

Enroll keys:

```bash
sbctl enroll-keys -m
```

---

# 14. Unified Kernel Images (UKI)

UKI combines:

- kernel,
- initramfs,
- cmdline,
- signature

into one EFI executable.

This is superior to traditional GRUB chainloading for Secure Boot.

Generate initramfs:

```bash
mkinitcpio -P
```

Verify signatures:

```bash
sbctl verify
```

Sign all:

```bash
sbctl sign-all
```

---

# 15. EFI Boot Entry
Because GRUB works worse under secure boot, I reccomend:

Create direct UKI entry:

```bash
efibootmgr --create \
--disk /dev/nvme0n1 \
--part 1 \
--label "Secure-Boot" \
--loader '\EFI\Linux\arch-linux.efi'
```

Set boot order:

```bash
efibootmgr --bootorder 0001
```

Verify:

```bash
efibootmgr -v
```

---

# 16. GRUB Password Protection

Generate hash:

```bash
grub-mkpasswd-pbkdf2
```

Edit:

```text
/etc/grub.d/40_custom
```

Add:

```text
set superusers="root"
password_pbkdf2 root HASH
```

Regenerate:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

---

# 17. Filesystem Hardening

## hidepid

Edit:

```text
/etc/fstab
```

Add:

```text
proc /proc proc defaults,hidepid=2 0 0
```

---

## Harden /tmp

```text
tmpfs /tmp tmpfs rw,nosuid,nodev,noexec 0 0
```

---

# 18. Sysctl Hardening

Create:

```text
/etc/sysctl.d/99-hardening.conf
```

Content:

```conf
kernel.kptr_restrict = 2
kernel.dmesg_restrict = 1
kernel.unprivileged_bpf_disabled = 1
net.core.bpf_jit_harden = 2
kernel.yama.ptrace_scope = 1
fs.protected_fifos = 2
fs.protected_regular = 2
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
```

Apply:

```bash
sysctl --system
```

---

# 19. USB Security

I highly reccomend to verify devices before enabling, not to cut yourself off of your usb keyboard, mouse etc. 

Install:

```bash
pacman -S usbguard
```

Generate policy:

```bash
usbguard generate-policy > /etc/usbguard/rules.conf
```

Enable:

```bash
systemctl enable --now usbguard
```

Verify devices:

```bash
usbguard list-devices
```

---

# 20. Package Auditing

Install:

```bash
pacman -S arch-audit
```

Check vulnerabilities:

```bash
arch-audit
```

---

# 21. Rootkit Detection

Install:

```bash
pacman -S chkrootkit
```

Run:

```bash
chkrootkit
```

Note:

Treat chkrootkit as heuristic tooling only.

---

# 22. Persistent Logging

Edit:

```text
/etc/systemd/journald.conf
```

Set:

```ini
Storage=persistent
SystemMaxUse=500M
```

Restart:

```bash
systemctl restart systemd-journald
```

---

# 23. Timeshift

Install:

```bash
pacman -S timeshift
```

Configure automatic snapshots.

Recommended:

- daily snapshots,
- pre-update snapshots,
- snapshot retention.

---

# 24. Automatic Updates Workflow

Before update:

```bash
timeshift --create --comments "pre-update"
```

Update system:

```bash
pacman -Syu
```

Regenerate initramfs:

```bash
mkinitcpio -P
```

Verify Secure Boot signatures:

```bash
sbctl verify
```

Sign files if necessary:

```bash
sbctl sign-all
```

---

# 25. Operational Security Recommendations

## Recommended

- regular full updates,
- encrypted backups,
- minimal installed services,
- persistent logging,
- snapshot strategy,
- security auditing.

## Avoid

- random AUR packages,
- unnecessary network services,
- running browsers as root,
- disabling Secure Boot,
- excessive kernel modules.

---

# 26. Security Validation

## Verify Secure Boot

```bash
mokutil --sb-state
```

## Verify lockdown

```bash
cat /sys/kernel/security/lockdown
```

## Verify AppArmor

```bash
aa-status
```

## Verify audit rules

```bash
auditctl -l
```

## Verify firewall

```bash
ufw status verbose
```

## Verify vulnerabilities

```bash
arch-audit
```

---

# 27. Final Recommended Architecture

## Final Security Stack

| Component | Purpose |
|---|---|
| Secure Boot | verified boot |
| UKI | signed kernel flow |
| LUKS2 | storage encryption |
| linux-hardened | kernel hardening |
| AppArmor | mandatory access control |
| auditd | auditing |
| UFW/nftables | firewall |
| Fail2ban | intrusion prevention |
| USBGuard | USB protection |
| arch-audit | vulnerability management |
| Timeshift | rollback and recovery |
| journald persistent logs | forensic visibility |

---
