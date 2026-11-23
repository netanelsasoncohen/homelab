# super-server — Arch Linux install (headless, encrypted)
 
A guide to installing Arch Linux on a headless home server with full-disk encryption, unlocked
remotely over SSH at boot. You never need a monitor or keyboard on the server.
 
The example hardware and values below are from the server this guide was written on (`super-server`).
Replace them with your own where they differ, especially device names, IPs and the network driver.
 
 > Written with the help of an AI assistant and tested step by step on real hardware.
> See [README](README.md#how-this-was-made) and [log.md](log.md) for what actually happened.

## Hardware
 
| Part | Model |
|---|---|
| CPU | Intel i7-9700KF (8 cores, AES-NI, **no iGPU**) |
| GPU | NVIDIA GTX 1060 6GB (Pascal → needs `nvidia-580xx-dkms`) |
| RAM | 32 GB |
| SSD | ADATA SU650 256 GB → system |
| HDD | WD Green 2 TB (WD20EZRZ) → data (media, Ollama models) |
| Network | Realtek ethernet `enp4s0` (driver `r8169`), home LAN `192.168.1.0/24` |
 
## Final layout
 
```
SSD /dev/sda
├─ sda1  1 GB    EFI (FAT32, unencrypted)  → /boot
└─ sda2  rest    LUKS2 "cryptroot" → btrfs
                 ├─ @            → /
                 ├─ @home        → /home
                 ├─ @snapshots   → /.snapshots
                 ├─ @var_log     → /var/log
                 ├─ @pkg         → /var/cache/pacman/pkg
                 └─ @docker      → /var/lib/docker
 
HDD /dev/sdb
├─ sdb1  4 GB    ext4 "rescue" (unencrypted): archlinux.iso = rescue system
└─ sdb2  rest    LUKS2 "cryptdata" → ext4 → /srv/data
                 (unlocked automatically by a keyfile stored inside cryptroot)
```
 
Swap: zram (compressed RAM), no swap partition.
 
## How booting works
 
```
power on → systemd-boot → initramfs starts a tiny SSH server (tinyssh)
        → you SSH in as root from your PC and type the disk password
        → cryptroot unlocks → system boots → cryptdata unlocks via keyfile
```
 
## How the headless install works
 
1. On the **old** system: build a custom Arch ISO with your SSH key baked in, put it on the HDD,
   and add a boot menu entry for it.
2. Reboot into the live ISO once, SSH in, and install onto the SSD.
3. **Safety net:** the live ISO stays the *default* boot entry until the new system has
   booted successfully once. If something goes wrong, hold the power button, power on again,
   and it boots back into the live ISO, so you can SSH in and fix things.
4. Only after everything works: make the new system the default, then split the HDD into a
   small rescue partition (the ISO stays as a permanent rescue system) and an encrypted data partition.
> ⚠️ Device names can change between boots. **Always check with `lsblk -o NAME,SIZE,MODEL`**
> before any command that wipes something. SSD = `ADATA SU650`, HDD = `WDC WD20EZRZ`.
 
---
 
## Phase 0 — Prepare (on the old system)
 
### 0.1 SSH key on your main PC
 
The initramfs SSH server (tinyssh) only accepts **ed25519** keys.
 
On the server, make sure SSH is running first:
 
```bash
sudo systemctl enable --now sshd
```
 
On the main PC (Windows PowerShell):
 
```powershell
# check whether you already have an SSH key
dir $env:USERPROFILE\.ssh\id_ed25519.pub
 
# if missing, create one
ssh-keygen -t ed25519
 
# copy your public key from your PC to the server
scp $env:USERPROFILE\.ssh\id_ed25519.pub <username>@<server-ip>:mykey.pub   # lands in ~/mykey.pub
```
 
Protect the private key with a passphrase (store it in the password manager, see recovery.md):
 
```powershell
ssh-keygen -p -f $env:USERPROFILE\.ssh\id_ed25519
ssh-keygen -y -f $env:USERPROFILE\.ssh\id_ed25519     # test: asks for it, then prints the public key
```
 
Let Windows remember it, so SSH and git don't ask every time. In **PowerShell as Administrator**:
 
```powershell
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent
```
 
Then in a normal PowerShell:
 
```powershell
ssh-add $env:USERPROFILE\.ssh\id_ed25519     # must say "Identity added"
ssh-add -l                                     # one line ending in (ED25519)
 
# make git use Windows' ssh (which talks to the agent), otherwise VS Code "Sync" hangs on a hidden prompt
git config --global core.sshCommand "C:/Windows/System32/OpenSSH/ssh.exe"
```
 
`Error connecting to agent` means the ssh-agent service isn't running (do the admin step).
 
### 0.2 Note the network driver
 
```bash
lspci -k | grep -iA3 ethernet
```
 
Write down the `Kernel driver in use:` value (e.g. `r8169`, `e1000e`, `igb`).
It goes into the initramfs later. On the example server it's **`r8169`** (Realtek).
 
### 0.3 Check the HDD health + stop head parking
 
```bash
sudo pacman -S --needed smartmontools
sudo smartctl -a /dev/sdb | grep -E "Reallocated|Pending|Power_On|Load_Cycle"
```
 
`Reallocated_Sector_Ct` and `Current_Pending_Sector` should be `0`.
 
WD Green drives park their heads after 8 seconds idle. On a 24/7 server that happens all day
and wears the drive out (`Load_Cycle_Count`; WD rates Greens for ~300,000). Disable it:
 
```bash
sudo pacman -S idle3-tools
 
sudo idle3ctl -g /dev/sdb     # show the current timer
sudo idle3ctl -d /dev/sdb     # disable it
```
 
This only takes effect after a **full power-off** (not a reboot): `sudo poweroff`, wait
10 seconds, press the power button. Then `sudo idle3ctl -g /dev/sdb` should say disabled.
The setting is stored on the drive itself, so it survives the reinstall.
 
### 0.4 Build a custom live ISO with your SSH key
 
```bash
sudo pacman -S --needed archiso
cp -r /usr/share/archiso/configs/releng ~/archlive
mkdir -p ~/archlive/airootfs/root/.ssh
cp ~/mykey.pub ~/archlive/airootfs/root/.ssh/authorized_keys
 
# broadcom-wl is no longer in the repos; the build fails on it ("target not found")
sed -i '/^broadcom-wl$/d' ~/archlive/packages.x86_64
```
 
Edit `~/archlive/profiledef.sh` and add these two lines inside the `file_permissions=(` block:
 
```bash
  ["/root/.ssh"]="0:0:700"
  ["/root/.ssh/authorized_keys"]="0:0:600"
```
 
Check the key is in place (the live ISO is useless without it):
 
```bash
cat ~/archlive/airootfs/root/.ssh/authorized_keys     # should print ssh-ed25519 ...
```
 
Build it (10–20 minutes; the SquashFS step at the end is the slow part). It finishes with
`Done!` and the ISO path (~1.6 GB). If it fails with `target not found: <package>`,
remove that package from `packages.x86_64` the same way, run `sudo rm -rf /tmp/archiso-work`, and retry.
 
```bash
sudo mkarchiso -v -w /tmp/archiso-work -o ~/archiso-out ~/archlive
sudo cp ~/archiso-out/archlinux-*.iso /mnt/sdb1/archlinux.iso
sudo rm -rf /tmp/archiso-work
```
 
Verify the ISO before booting it:
 
```bash
sudo mkdir -p /mnt/iso
sudo mount -o loop,ro /mnt/sdb1/archlinux.iso /mnt/iso
sudo find /mnt/iso -name 'vmlinuz*' -o -name 'initramfs*' -o -name '*.sfs'
sudo unsquashfs -l /mnt/iso/arch/x86_64/airootfs.sfs | grep -i '\.ssh'
sudo umount /mnt/iso
```
 
Expected: `arch/boot/x86_64/vmlinuz-linux`, `arch/boot/x86_64/initramfs-linux.img`,
`arch/x86_64/airootfs.sfs`, and `/root/.ssh/authorized_keys`.
There is **no separate `intel-ucode.img`**: microcode is built into the initramfs now.
 
### 0.5 Add a boot entry for the live ISO
 
Check which bootloader the old system uses (`bootctl` without `sudo` always fails and looks like GRUB):
 
```bash
sudo ls /boot /boot/loader /boot/grub 2>/dev/null
```
 
A `grub/` folder means GRUB; `loader/` means systemd-boot.
 
**If GRUB** (what the example server had):
 
```bash
HDD_UUID=$(sudo blkid -s UUID -o value /dev/sdb1)
echo "HDD UUID: $HDD_UUID"        # must not be empty
 
sudo tee -a /etc/grub.d/40_custom <<EOF
 
menuentry "Arch Live (install / rescue)" {
    insmod ext2
    insmod loopback
    insmod iso9660
    search --no-floppy --fs-uuid --set=root $HDD_UUID
    loopback loop /archlinux.iso
    linux (loop)/arch/boot/x86_64/vmlinuz-linux img_dev=/dev/disk/by-uuid/$HDD_UUID img_loop=/archlinux.iso
    initrd (loop)/arch/boot/x86_64/initramfs-linux.img
}
EOF
 
# grub-reboot (one-time boot) only works with GRUB_DEFAULT=saved
sudo sed -i 's/^GRUB_DEFAULT=.*/GRUB_DEFAULT=saved/' /etc/default/grub
sudo grub-set-default 0           # normal default stays the old Arch install
sudo grub-mkconfig -o /boot/grub/grub.cfg
sudo grep -c "Arch Live" /boot/grub/grub.cfg     # should print 1
```
 
**If systemd-boot:**
 
```bash
HDD_UUID=$(sudo blkid -s UUID -o value /dev/sdb1)
sudo mkdir -p /mnt/iso /boot/archlive
sudo mount -o loop,ro /mnt/sdb1/archlinux.iso /mnt/iso
sudo cp /mnt/iso/arch/boot/x86_64/vmlinuz-linux /mnt/iso/arch/boot/x86_64/initramfs-linux.img /boot/archlive/
sudo umount /mnt/iso
 
sudo tee /boot/loader/entries/archlive.conf <<EOF
title   Arch Live (install / rescue)
linux   /archlive/vmlinuz-linux
initrd  /archlive/initramfs-linux.img
options img_dev=/dev/disk/by-uuid/$HDD_UUID img_loop=/archlinux.iso
EOF
```
 
### 0.6 Boot into the live ISO, once
 
Install **nmap** on the main PC first (`winget install Insecure.Nmap` in PowerShell, then open a
new window). The live ISO usually gets a **different IP** than the old system.
 
```bash
sudo grub-reboot "Arch Live (install / rescue)" && sudo reboot      # GRUB
# systemd-boot: sudo systemctl reboot --boot-loader-entry=archlive.conf
```
 
This is a **one-time** boot. If nothing answers on SSH after ~5 minutes, hold the power
button, power on again, and the old system boots.
 
---
 
## Phase 1 — Install (from the live ISO, over SSH)
 
### 1.1 Connect
 
Wait 1–3 minutes (it loads a 1 GB image from the HDD), then from PowerShell on the main PC:
 
```powershell
nmap -p 22 --open 192.168.1.0/24     # look for "archiso.lan" with 22/tcp open
ssh root@<that ip>                   # or: ssh root@archiso.lan
```
 
Check that the MAC address nmap shows matches the server's ethernet port.
 
If SSH complains `REMOTE HOST IDENTIFICATION HAS CHANGED`, that's expected (a different system
on the same IP). Fix with `ssh-keygen -R <ip or name>`.
 
Check the basics:
 
```bash
ping -c 2 archlinux.org
timedatectl set-ntp true
lsblk -o NAME,SIZE,MODEL        # confirm which disk is the ADATA SSD!
```
 
> The rest of this phase assumes the SSD is `/dev/sda`. **Do not wipe the HDD**:
> the ISO file on it is the rescue system until Phase 2 is done. (With 32 GB of RAM the live
> system runs from RAM (`copytoram`), so the HDD isn't mounted while you work.)
 
**Before wiping anything:** from step 1.2 until 1.11 there is **no safety net** (the old
bootloader is gone, the new one isn't set up yet). Do 1.2–1.13 in one go, and work inside
`tmux` so a dropped SSH connection doesn't kill a running command:
 
```bash
tmux                 # after a disconnect: ssh back in, then `tmux attach`
```
 
> Tip: paste `exit` and `arch-chroot /mnt` **on their own line** and wait for the new prompt.
> Lines pasted after them in the same block get lost while the shell switches.
 
### 1.2 Wipe and partition the SSD
 
```bash
blkdiscard -f /dev/sda          # SSD erase: instant, old data gone
sgdisk -n1:0:+1G -t1:ef00 -c1:EFI \
       -n2:0:0   -t2:8309 -c2:cryptroot /dev/sda
sgdisk -p /dev/sda                 # partition table as written on disk
partprobe /dev/sda                 # make the kernel re-read it
lsblk -o NAME,SIZE,PARTLABEL /dev/sda
```
 
Expected: `sda1` 1G `EFI` and `sda2` ~237.5G `cryptroot`. (Right after `sgdisk`, `lsblk` can show
only `sda1` until `partprobe` runs.)
 
### 1.3 Encrypt
 
```bash
cryptsetup luksFormat --type luks2 /dev/sda2     # type YES, then the disk password
cryptsetup open --allow-discards --persistent /dev/sda2 cryptroot
```
 
**Save the disk password in a password manager now.** Without it the data is gone forever.
Tip: 4–5 random words is long and still easy to type at every boot. Unlocking takes ~10
seconds; that's deliberate (it makes password guessing slow).
 
`ls /dev/mapper` should now show `cryptroot`.
 
### 1.4 Filesystems and subvolumes
 
```bash
mkfs.fat -F32 -n EFI /dev/sda1
mkfs.btrfs -L arch /dev/mapper/cryptroot
 
mount /dev/mapper/cryptroot /mnt
for sv in @ @home @snapshots @var_log @pkg @docker; do btrfs subvolume create /mnt/$sv; done
umount /mnt
 
O=noatime,compress=zstd:1
mount -o $O,subvol=@ /dev/mapper/cryptroot /mnt
mount --mkdir -o $O,subvol=@home      /dev/mapper/cryptroot /mnt/home
mount --mkdir -o $O,subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots
mount --mkdir -o $O,subvol=@var_log   /dev/mapper/cryptroot /mnt/var/log
mount --mkdir -o $O,subvol=@pkg       /dev/mapper/cryptroot /mnt/var/cache/pacman/pkg
mount --mkdir -o $O,subvol=@docker    /dev/mapper/cryptroot /mnt/var/lib/docker
mount --mkdir -o fmask=0077,dmask=0077 /dev/sda1 /mnt/boot
findmnt -R /mnt -o TARGET,SOURCE     # should list 7 mounts
```
 
Why separate subvolumes: snapshots of `/` then don't include logs, the package cache
or Docker data, so a rollback won't wipe those.
 
### 1.5 Install packages
 
Check the remote-unlock packages exist first (if one is missing, pacstrap fails):
 
```bash
pacman -Sy
pacman -Si mkinitcpio-netconf mkinitcpio-tinyssh mkinitcpio-utils | grep -E "^(Repository|Name)"
```
 
```bash
pacstrap -K /mnt base base-devel linux-lts linux-lts-headers linux linux-headers \
  linux-firmware intel-ucode btrfs-progs cryptsetup dosfstools e2fsprogs \
  networkmanager openssh sudo nano vim git man-db man-pages \
  mkinitcpio-netconf mkinitcpio-tinyssh mkinitcpio-utils \
  snapper smartmontools nftables zram-generator avahi nss-mdns gptfdisk parted
```
 
- `linux-lts` = main kernel (stable). `linux` = fallback if an LTS update ever breaks.
- `gptfdisk` (`sgdisk`) and `parted` (`partprobe`) are needed later for partitioning the HDD (3.1).
- `mkinitcpio-netconf/tinyssh/utils` = network + SSH + password prompt inside the initramfs.
### 1.6 fstab
 
```bash
genfstab -U /mnt >> /mnt/etc/fstab
sed -i 's/subvolid=[0-9]*,//' /mnt/etc/fstab     # mount by name only (needed for snapper rollbacks)
cat /mnt/etc/fstab                               # sanity check
```
 
Copy your SSH key into the new system before chrooting:
 
```bash
cp /root/.ssh/authorized_keys /mnt/root/mykey.pub
```
 
### 1.7 Configure the system
 
```bash
arch-chroot /mnt
```
 
Everything below runs **inside the chroot** (prompt: `[root@archiso /]#`).
 
```bash
# time, locale, hostname
ln -sf /usr/share/zoneinfo/<Region/City> /etc/localtime   # e.g. Europe/London
hwclock --systohc
sed -i 's/^#en_US.UTF-8/en_US.UTF-8/' /etc/locale.gen && locale-gen
echo 'LANG=en_US.UTF-8' > /etc/locale.conf
echo 'KEYMAP=us'        > /etc/vconsole.conf
echo 'super-server'     > /etc/hostname
 
# user (root login is disabled)
useradd -m -G wheel <username>
passwd <username>                                   # used for sudo
echo '%wheel ALL=(ALL:ALL) ALL' > /etc/sudoers.d/wheel && chmod 440 /etc/sudoers.d/wheel
passwd -l root
 
# SSH keys: for <username> (normal login) and for the initramfs unlock
install -d -m 700 -o <username> -g <username> /home/<username>/.ssh
install -m 600 -o <username> -g <username> /root/mykey.pub /home/<username>/.ssh/authorized_keys
install -d /etc/tinyssh
cp /root/mykey.pub /etc/tinyssh/root_key
rm /root/mykey.pub
ssh-keygen -A          # create host keys now, so the unlock SSH and normal SSH share the same fingerprint
```
 
Check: `ls -la /home/<username>/.ssh /etc/tinyssh` shows `authorized_keys` owned by the user
(`-rw-------`) and `root_key` in `/etc/tinyssh`.
 
### 1.8 SSH hardening
 
```bash
cat > /etc/ssh/sshd_config.d/10-hardening.conf <<'EOF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
AuthenticationMethods publickey
EOF
```
 
Keys only, no root. Your key is already installed, so this can't lock you out.
 
### 1.9 initramfs with remote unlock
 
Set these two lines in `/etc/mkinitcpio.conf` (`r8169` = **your driver from 0.2**):
 
```bash
sed -i 's/^MODULES=.*/MODULES=(r8169)/' /etc/mkinitcpio.conf
sed -i 's/^HOOKS=.*/HOOKS=(base udev autodetect microcode modconf keyboard keymap consolefont block netconf tinyssh encryptssh filesystems fsck)/' /etc/mkinitcpio.conf
grep -E '^(MODULES|HOOKS)=' /etc/mkinitcpio.conf
```
 
- `netconf` brings up the network, `tinyssh` starts SSH, `encryptssh` asks for the disk password.
- `kms` is deliberately left out: it would load the open-source `nouveau` driver, which conflicts with NVIDIA's.
```bash
mkinitcpio -P
```
 
For both kernels, check the output shows `[netconf]`, `[tinyssh]` (it prints the host key),
`[encryptssh]`, and ends with `Initcpio image generation successful`.
Warnings about missing firmware (`qla`, `cfg80211`, …) and `consolefont: no font found` are
harmless. `ls -lh /boot` should show both `vmlinuz-*` and `initramfs-*` for `linux` and `linux-lts`.
 
### 1.10 Bootloader
 
```bash
bootctl install
ROOT_UUID=$(blkid -s UUID -o value /dev/sda2)
OPTS="cryptdevice=UUID=$ROOT_UUID:cryptroot:allow-discards root=/dev/mapper/cryptroot rootflags=subvol=@ rw ip=:::::eth0:dhcp"
 
cat > /boot/loader/entries/arch-lts.conf <<EOF
title   Arch Linux (LTS)
linux   /vmlinuz-linux-lts
initrd  /initramfs-linux-lts.img
options $OPTS
EOF
 
cat > /boot/loader/entries/arch.conf <<EOF
title   Arch Linux (fallback kernel)
linux   /vmlinuz-linux
initrd  /initramfs-linux.img
options $OPTS
EOF
 
# safety net: the live ISO stays the default until the new system is proven
cat > /boot/loader/loader.conf <<'EOF'
default archlive.conf
timeout 3
editor  no
EOF
```
 
`editor no` stops anyone at the machine from editing boot options (e.g. to get a root shell).
 
Check: `cat /boot/loader/entries/arch-lts.conf`. The `options` line must contain a real UUID after
`cryptdevice=UUID=`.
 
> `bootctl install` inside the chroot says **"Not booted with EFI or running in a container,
> skipping EFI variable modifications"**. So the motherboard doesn't know about systemd-boot yet;
> that's fixed from outside the chroot in 1.11.
 
### 1.11 Rescue entry + register the bootloader (outside the chroot)
 
The old EFI partition was wiped, so the live entry has to be recreated.
The live system ran `copytoram` (whole system loaded into RAM), so the ISO isn't mounted
anymore: mount it from the HDD read-only and copy the kernel and initramfs out of it.
 
Leave the chroot (on its own line):
 
```bash
exit
```
 
```bash
ls /sys/firmware/efi/efivars > /dev/null && echo "EFI OK"
lsblk -o NAME,SIZE,MODEL                         # double-check sdb is the HDD!
 
mkdir -p /tmp/hdd /tmp/iso /mnt/boot/archlive
mount -o ro /dev/sdb1 /tmp/hdd
mount -o loop,ro /tmp/hdd/archlinux.iso /tmp/iso
cp /tmp/iso/arch/boot/x86_64/vmlinuz-linux /tmp/iso/arch/boot/x86_64/initramfs-linux.img /mnt/boot/archlive/
umount /tmp/iso /tmp/hdd
 
HDD_UUID=$(blkid -s UUID -o value /dev/sdb1)
cat > /mnt/boot/loader/entries/archlive.conf <<EOF
title   Arch Live (install / rescue)
linux   /archlive/vmlinuz-linux
initrd  /archlive/initramfs-linux.img
options img_dev=/dev/disk/by-uuid/$HDD_UUID img_loop=/archlinux.iso
EOF
cat /mnt/boot/loader/entries/archlive.conf
```
 
Register systemd-boot with the motherboard (this is what the chroot couldn't do):
 
```bash
bootctl --esp-path=/mnt/boot install
efibootmgr
```
 
Expected: `Created EFI boot entry "Linux Boot Manager"`, and its number first in `BootOrder`.
`bootctl` prints "Created directory" for folders that already exist; check nothing was overwritten:
 
```bash
ls /mnt/boot/loader/entries          # arch.conf  arch-lts.conf  archlive.conf
cat /mnt/boot/loader/loader.conf     # still: default archlive.conf
```
 
Remove the dead entry for the old GRUB (it points at the wiped partition). Use the number
`efibootmgr` shows for it, e.g. `Boot0002* GRUB`:
 
```bash
efibootmgr -b 0002 -B
```
 
(A `Windows Boot Manager VenHw(...)` entry on Gigabyte boards is a firmware placeholder, not a
real Windows install. Leave it.)
 
Go back into the new system (on its own line):
 
```bash
arch-chroot /mnt
```
 
### 1.12 Services, zram, firewall
 
```bash
cat > /etc/systemd/zram-generator.conf <<'EOF'
[zram0]
zram-size = min(ram / 2, 8192)
compression-algorithm = zstd
EOF
 
cat > /etc/nftables.conf <<'EOF'
#!/usr/bin/nft -f
destroy table inet filter
 
define LAN = 192.168.1.0/24
 
table inet filter {
    chain input {
        type filter hook input priority filter; policy drop;
 
        ct state established,related accept
        ct state invalid drop
        iif lo accept
        meta l4proto { icmp, ipv6-icmp } accept
 
        ip saddr $LAN tcp dport 22   accept    # SSH
        ip saddr $LAN udp dport 5353 accept    # mDNS (super-server.local)
    }
}
EOF
 
nft -c -f /etc/nftables.conf && echo "firewall config OK"     # check syntax without loading it
systemctl enable NetworkManager sshd avahi-daemon nftables systemd-timesyncd
```
 
- Incoming traffic is blocked by default; only SSH and mDNS from the home network are allowed.
- `destroy table` instead of `flush ruleset`, so reloading the firewall won't wipe Docker's rules later.
### 1.13 First boot
 
Leave the chroot (on its own line):
 
```bash
exit
```
 
```bash
bootctl --esp-path=/mnt/boot set-oneshot arch-lts.conf   # boot the new system ONCE; default is still the live ISO
umount -R /mnt
cryptsetup close cryptroot
reboot
```
 
(`set-oneshot` runs **before** unmounting, with `--esp-path`.)
 
---
 
## Phase 2 — Unlock and verify
 
### 2.1 Unlock the disk (this is how every boot works from now on)
 
Wait ~1 minute. From PowerShell on the main PC:
 
```powershell
nmap -p 22 --open 192.168.1.0/24     # find the server by its MAC; it usually keeps its IP
ssh root@<ip>
```
 
Two different prompts can appear. Don't mix them up:
 
| Prompt | Wants |
|---|---|
| `Enter passphrase for key '...id_ed25519':` | the **SSH key passphrase** (only if ssh-agent isn't running, see 0.1) |
| `Enter passphrase for /dev/sda2:` | the **disk password** |
 
> **Type the disk password by hand.** Pasting into the unlock prompt does **not** work
> (`No key available with this passphrase` even with the right password). Either type it, or use
> KeePassXC **Auto-Type**: in the disk entry set a custom sequence `{PASSWORD}{ENTER}`, then at the
> prompt select the entry in KeePassXC and press `Ctrl+Shift+V`.
 
- First time: `REMOTE HOST IDENTIFICATION HAS CHANGED` is expected (new system, same IP).
  Run `ssh-keygen -R <ip>` and connect again.
- After the right password, the connection closes after ~10 seconds **without** an error: the system is booting.
- Each new `ssh root@<ip>` gives 3 more tries; nothing locks.
- `ssh root@<ip>` **after** unlocking gives `Permission denied (publickey)`: that's the real
  system's SSH refusing root, which means it booted.
After ~30 seconds:
 
```bash
ssh <username>@<ip>                  # or super-server.lan / super-server.local
```
 
The unlock SSH and the normal SSH share the same host key, so there's no second warning.
 
### 2.2 🆘 If it doesn't work
 
No SSH on port 22 after ~3 minutes means the initramfs network didn't come up.
 
1. Hold the power button until it turns off, then power on. **It boots the live ISO** (still the default).
2. Find it with `nmap -p 22 --open 192.168.1.0/24` (`archiso.lan`), `ssh root@<ip>`, then remount:
```bash
cryptsetup open /dev/sda2 cryptroot
mount -o subvol=@ /dev/mapper/cryptroot /mnt && mount /dev/sda1 /mnt/boot
arch-chroot /mnt
```
3. Usual fixes:
   - Change `eth0` to `enp4s0` in the `ip=` part of `/boot/loader/entries/*.conf`.
   - Check that `MODULES=()` has the right network driver, then run `mkinitcpio -P`.
4. `exit`, `umount -R /mnt`, `bootctl set-oneshot arch-lts.conf`, `reboot`, and try 2.1 again.
### 2.3 Make the new system the default
 
Only once 2.1 has worked:
 
```bash
sudo bootctl set-default arch-lts.conf
sudo bootctl status 2>/dev/null | grep -iE "default|current"   # both: arch-lts.conf
nmcli device status           # enp4s0 connected
sudo nft list ruleset         # firewall loaded
zramctl                       # zram swap active (8G)
lsblk -o NAME,SIZE,MOUNTPOINTS
```
 
The live ISO stays in the boot menu as a permanent rescue option (see 3.1).
If a USB Wi-Fi adapter is still plugged in, unplug it: ethernet only.
 
---
 
## Phase 3 — Finish the setup
 
### 3.1 HDD: rescue partition + encrypted data (`/srv/data`)
 
Split the HDD: **4 GB unencrypted** keeps the live ISO as a permanent rescue system (it only
contains public packages and your *public* key), **the rest** is encrypted data.
 
**Stage A: save the ISO to the SSD temporarily:**
 
```bash
lsblk -o NAME,SIZE,MODEL,MOUNTPOINTS          # sdb = WD 1.8T!
sudo mkdir -p /mnt/hdd
sudo mount -o ro /dev/sdb1 /mnt/hdd
cp /mnt/hdd/archlinux.iso ~/archlinux.iso
sha256sum /mnt/hdd/archlinux.iso ~/archlinux.iso   # the two codes must be identical
sudo umount /mnt/hdd
```
 
**Stage B: repartition, put the ISO back, update the rescue entry.**
Create the `super-server: HDD backup` passphrase in KeePassXC and save **before** Stage C.
 
```bash
sudo sgdisk --zap-all /dev/sdb
sudo sgdisk -n1:0:+4G -t1:8300 -c1:rescue -n2:0:0 -t2:8309 -c2:cryptdata /dev/sdb
sudo partprobe /dev/sdb
sudo wipefs -a /dev/sdb1 /dev/sdb2
lsblk -o NAME,SIZE,PARTLABEL /dev/sdb         # sdb1 4G rescue, sdb2 1.8T cryptdata
 
sudo mkfs.ext4 -L rescue /dev/sdb1
sudo mount /dev/sdb1 /mnt/hdd
sudo cp ~/archlinux.iso /mnt/hdd/archlinux.iso
sha256sum ~/archlinux.iso /mnt/hdd/archlinux.iso   # must match again
sudo umount /mnt/hdd
 
RESCUE_UUID=$(sudo blkid -s UUID -o value /dev/sdb1)
echo "Rescue UUID: $RESCUE_UUID"
sudo sed -i "s|img_dev=/dev/disk/by-uuid/[^ ]*|img_dev=/dev/disk/by-uuid/$RESCUE_UUID|" /boot/loader/entries/archlive.conf
sudo cat /boot/loader/entries/archlive.conf    # new UUID on the options line
```
 
**Stage C: encrypt `sdb2` with a backup password + a keyfile:**
 
```bash
sudo cryptsetup luksFormat --type luks2 /dev/sdb2              # YES, then the HDD backup password
 
sudo install -d -m 700 /etc/cryptsetup-keys.d
sudo dd if=/dev/urandom of=/etc/cryptsetup-keys.d/cryptdata.key bs=512 count=8 iflag=fullblock
sudo chmod 600 /etc/cryptsetup-keys.d/cryptdata.key
sudo cryptsetup luksAddKey /dev/sdb2 /etc/cryptsetup-keys.d/cryptdata.key   # asks for the backup password
 
# test both keys before formatting
sudo cryptsetup open --test-passphrase /dev/sdb2 && echo "BACKUP PASSWORD WORKS"
sudo cryptsetup open --test-passphrase /dev/sdb2 --key-file /etc/cryptsetup-keys.d/cryptdata.key && echo "KEYFILE WORKS"
sudo cryptsetup luksDump /dev/sdb2 | grep -cE "^\s+[0-9]+: luks2"   # 2
```
 
Format and mount automatically at boot:
 
```bash
sudo cryptsetup open /dev/sdb2 cryptdata --key-file /etc/cryptsetup-keys.d/cryptdata.key
sudo mkfs.ext4 -L data -m 0 /dev/mapper/cryptdata
 
echo "cryptdata UUID=$(sudo blkid -s UUID -o value /dev/sdb2) /etc/cryptsetup-keys.d/cryptdata.key luks,nofail" | sudo tee -a /etc/crypttab
echo "/dev/mapper/cryptdata /srv/data ext4 defaults,noatime,nofail 0 2" | sudo tee -a /etc/fstab
sudo mkdir -p /srv/data
sudo systemctl daemon-reload && sudo mount -a
df -h /srv/data                                 # 1.8T
rm ~/archlinux.iso                              # temporary copy no longer needed
```
 
- `-m 0`: no space reserved for root (pointless on a data disk).
- `nofail`: a dead HDD won't stop the server from booting.
Test with a reboot: after unlocking, `df -h /srv/data` should show it mounted without doing anything.
 
**Using the rescue system:** while the server still runs, `sudo bootctl set-oneshot archlive.conf`
and reboot; find it with nmap as `archiso.lan` and `ssh root@<ip>`.
 
### 3.2 Back up the LUKS headers (do not skip)
 
If a header gets damaged, the partition is unreadable even with the password.
 
```bash
sudo cryptsetup luksHeaderBackup /dev/sda2 --header-backup-file ~/sda2-header.img
sudo cryptsetup luksHeaderBackup /dev/sdb2 --header-backup-file ~/sdb2-header.img
sudo chown <username>: ~/sd*-header.img
 
# check each file really came from the right disk
sudo cryptsetup luksDump ~/sda2-header.img | grep -E "^UUID|[0-9]+: luks2"   # SSD UUID, 1 slot
sudo cryptsetup luksDump ~/sdb2-header.img | grep -E "^UUID|[0-9]+: luks2"   # HDD UUID, 2 slots
sha256sum ~/sd*-header.img                                                     # two different codes
```
 
Copy to the main PC and attach both files to a `super-server: LUKS header backups` entry in KeePassXC:
 
```powershell
scp <username>@<server-ip>:sd*-header.img $env:USERPROFILE\Downloads\
Get-FileHash $env:USERPROFILE\Downloads\sd*-header.img -Algorithm SHA256   # same codes (Windows shows UPPERCASE)
```
 
Then delete the loose copies, **never in git**:
 
- Server: `shred -u ~/sd*-header.img`
- PC: `Remove-Item $env:USERPROFILE\Downloads\sd*-header.img`
**After changing a disk password, make new header backups and replace the old ones**:
an old header still accepts the old password.
 
### 3.3 Snapshots (snapper)
 
Replace snapper's own `.snapshots` with the `@snapshots` subvolume (needed for clean rollbacks):
 
```bash
sudo umount /.snapshots
sudo rmdir /.snapshots
sudo snapper -c root create-config /
sudo btrfs subvolume delete /.snapshots
sudo mkdir /.snapshots
sudo mount -a
sudo chmod 750 /.snapshots
findmnt /.snapshots                              # cryptroot[/@snapshots]
```
 
Keep it small:
 
```bash
sudo sed -i \
  -e 's/^TIMELINE_LIMIT_HOURLY=.*/TIMELINE_LIMIT_HOURLY="5"/' \
  -e 's/^TIMELINE_LIMIT_DAILY=.*/TIMELINE_LIMIT_DAILY="7"/' \
  -e 's/^TIMELINE_LIMIT_WEEKLY=.*/TIMELINE_LIMIT_WEEKLY="2"/' \
  -e 's/^TIMELINE_LIMIT_MONTHLY=.*/TIMELINE_LIMIT_MONTHLY="0"/' \
  -e 's/^TIMELINE_LIMIT_YEARLY=.*/TIMELINE_LIMIT_YEARLY="0"/' \
  -e 's/^NUMBER_LIMIT=.*/NUMBER_LIMIT="10"/' \
  /etc/snapper/configs/root
sudo grep -E "TIMELINE_LIMIT|^NUMBER_LIMIT" /etc/snapper/configs/root   # needs sudo: root-only file
```
 
```bash
sudo pacman -S snap-pac      # automatic snapshot before + after every pacman run
sudo systemctl enable --now snapper-timeline.timer snapper-cleanup.timer
sudo snapper list            # shows a pre/post pair from installing snap-pac
```
 
### 3.4 NVIDIA driver (GTX 1060)
 
NVIDIA's 590+ drivers dropped Pascal cards. **Never install `nvidia` or `nvidia-open`.**
Use the 580 legacy branch from the AUR. First install `yay` (AUR helper):
 
```bash
cd /tmp
git clone https://aur.archlinux.org/yay-bin.git
cd yay-bin
makepkg -si
cd ~
yay -S nvidia-580xx-dkms     # pulls in nvidia-580xx-utils + graphics libraries automatically
```
 
yay prompts: `cleanBuild?` and `Diffs to show?` → Enter (None); providers → Enter (default);
`Proceed?` → Y. The initramfs gets rebuilt; check `[tinyssh]` and `[encryptssh]` are still in it.
 
```bash
dkms status                  # "installed" for both kernels
sudo reboot                  # then unlock as in 2.1
nvidia-smi                   # GeForce GTX 1060 6GB, Driver Version 580.x
```
 
### 3.5 Check the HDD head parking fix
 
The idle3 timer was disabled in 0.3. Confirm `Load_Cycle_Count` has stopped climbing:
 
```bash
sudo smartctl -A /dev/sdb | grep Load_Cycle     # note the number, check again in a week
```
 
---
 
## Routine maintenance
 
**Updates, about weekly:**
 
1. Read <https://archlinux.org/news/> for anything needing manual steps.
2. `yay -Syu` (updates official packages + AUR, including the NVIDIA driver).
3. If the kernel updated: `sudo reboot`, then unlock (2.1).
4. If an update breaks something: `sudo snapper list` → `sudo snapper rollback <number>` → reboot.
**Reboot checklist:** find IP → `ssh root@<ip>` → **type** the disk password (or KeePassXC Auto-Type)
→ wait 30 s → `ssh <username>@<ip>`.
 
**Changing a disk password:**
 
```bash
sudo cryptsetup luksChangeKey /dev/sda2
sudo cryptsetup open --test-passphrase /dev/sda2 && echo "WORKS"     # test it BEFORE rebooting
sudo cryptsetup luksDump /dev/sda2 | grep -cE "^\s+[0-9]+: luks2"   # 1 = only the new one left
```
 
Save the new password in KeePassXC **first**, and raise KeePassXC's clipboard timeout
(Settings → Security) so a paste doesn't arrive empty. Then redo the header backup (3.2).
 
---
 
## Repo hygiene
 
`.gitignore` for this repo:
 
```gitignore
# secrets: never commit
*.key
*.img
*.env
.env
*_key
id_*
!*.pub
secrets/
```
 
Never commit: LUKS header backups, keyfiles, the disk password, `/etc/shadow`,
`/etc/ssh/ssh_host_*`, Wi-Fi passwords.
 
---
 
## Next stages
 
- [ ] Docker + Jellyfin (GTX 1060 NVENC transcoding), media on `/srv/data/media`
- [ ] Ollama, models on `/srv/data/ollama`
- [ ] Minecraft server in Docker (+ playit.gg for players outside the LAN)
- [ ] Samba share, LAN only
- [ ] Cockpit, LAN only
- [ ] Tailscale for remote access
- [ ] Backups (restic)
- [ ] OPNsense router machine → DHCP reservation, port forwards, WireGuard
 