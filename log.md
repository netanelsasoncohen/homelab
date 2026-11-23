# Log
 
Problems, fixes, and decisions, newest at the top.
 
Template:
 
```
## YYYY-MM-DD: short title
**Step:** install.md section (e.g. 1.9)
**Problem:** what happened / exact error message
**Fix:** what I did
**Updated install.md:** yes / no
```
 
---
 
## 2026-11-23: ✅ Phase 3 done: NVIDIA driver, snapshots, header backups
**Step:** install.md 3.2–3.4
**Result:** yay → `nvidia-580xx-dkms` built for both kernels; after reboot `nvidia-smi` shows the GTX 1060 on driver 580.196.07 (37°C, 5 W idle). snapper with snap-pac makes pre/post snapshots on every pacman run. Header backups for sda2 + sdb2 checked with `luksDump` (right UUIDs and slot counts) and stored in the vault; hashes matched (Windows shows them in uppercase, same value).
**Small snag:** `grep` on `/etc/snapper/configs/root` needs sudo (root-only file).
**Updated install.md:** yes (3.2, 3.3, 3.4)
 
## 2026-11-23: Pasting the disk password into the unlock prompt doesn't work
**Step:** install.md 2.1 (first reboot after changing the disk password)
**Problem:** `No key available with this passphrase`, again and again, even though the password had been tested with `--test-passphrase` before the reboot and the clipboard definitely had it. Also mixed up the two prompts: "Enter passphrase for key ...id_ed25519" (SSH key) vs "Enter passphrase for /dev/sda2" (disk).
**Fix:** Typing the password by hand worked first time. KeePassXC Auto-Type (`{PASSWORD}{ENTER}`, `Ctrl+Shift+V`) sends real keystrokes, so it works too.
**Updated install.md:** yes (2.1, routine maintenance)
 
## 2026-11-23: ssh-agent wasn't running; VS Code Sync hung
**Step:** install.md 0.1
**Problem:** After adding a passphrase to the SSH key, every connection asked for it; `ssh-add` said `Error connecting to agent: No such file or directory`. VS Code "Sync" hung because git's own ssh was waiting on a hidden passphrase prompt.
**Fix:** Admin PowerShell: `Set-Service ssh-agent -StartupType Automatic` + `Start-Service ssh-agent`, then `ssh-add`. `git config --global core.sshCommand "C:/Windows/System32/OpenSSH/ssh.exe"` so git uses the agent.
**Updated install.md:** yes (0.1)
 
## 2026-11-23: HDD split into rescue + encrypted data; sgdisk missing
**Step:** install.md 3.1
**Decision:** Keep the live ISO as a permanent rescue system instead of wiping the whole HDD: `sdb1` 4 GB ext4 (unencrypted, only public packages + my public key), `sdb2` the rest, LUKS2 with a backup password + keyfile. Costs 4 GB, adds no real risk (anyone with physical access could boot a USB stick anyway).
**Problem:** `sgdisk` and `partprobe` weren't installed on the new system (only on the live ISO). `wipefs` still ran and erased the old ext4 signature; harmless since the ISO had been copied off first (sha256 verified).
**Fix:** `sudo pacman -S gptfdisk parted`, re-ran the partitioning. Added both to the pacstrap list. Reboot test: SSD unlocked with the new password, HDD unlocked by keyfile, `/srv/data` mounted on its own.
**Updated install.md:** yes (layout, 1.5, 3.1 rewritten)
 
## 2026-11-23: Moved to a password manager, changed the server passwords
**Step:** after 2.3
**Problem:** I'd used one password for everything (disk, sudo, accounts). If it leaked anywhere, the disk encryption would be worthless. The SSH private key existed only on my PC, with no passphrase.
**Fix:** Set up a KeePassXC vault on two USB sticks. Changed the disk password (`cryptsetup luksChangeKey`, verified with `--test-passphrase`, one key slot left) and the user password. Added a passphrase to the SSH key and backed it up in the vault. New unique passwords for GitHub and both email accounts. KeePassXC clears the clipboard after ~10 s, which caused one empty paste; raised the timeout while changing passwords.
**Updated install.md:** no (new recovery.md)
 
## 2026-11-23: ✅ Install done: first boot + remote unlock worked
**Step:** install.md 1.13 → 2.3
**Result:** After reboot, the unlock SSH (tinyssh) came up on the old IP (.165) within a minute. `ssh-keygen -R` for the old host key, typed the disk password, connection closed, system booted. `ssh root@` afterwards gave "Permission denied (publickey)" (the real sshd refusing root), then logged in as my user. Default boot entry set to `arch-lts.conf`; firewall, zram (8G) and all subvolumes OK.
**Updated install.md:** yes (2.1, 2.3)
 
## 2026-11-23: bootctl in the chroot couldn't register with the motherboard
**Step:** install.md 1.10 / 1.11
**Problem:** `bootctl install` inside arch-chroot printed "Not booted with EFI or running in a container, skipping EFI variable modifications". The motherboard's boot list still pointed at the old (wiped) GRUB.
**Fix:** From outside the chroot: `bootctl --esp-path=/mnt/boot install` → created "Linux Boot Manager", first in BootOrder. Removed the dead GRUB entry with `efibootmgr -b 0002 -B`. Did `set-oneshot` with `--esp-path` before unmounting.
**Updated install.md:** yes (1.10 note, 1.11 rewritten, 1.13)
 
## 2026-11-23: Commands pasted after `exit` got lost
**Step:** install.md 1.11
**Problem:** Pasted `exit` plus the next commands as one block; only `exit` ran.
**Fix:** Paste `exit` and `arch-chroot /mnt` on their own line and wait for the prompt. Also used `tmux` for the whole install, so a dropped SSH connection wouldn't kill anything.
**Updated install.md:** yes (tip before 1.2)
 
## 2026-11-23: /run/archiso/bootmnt doesn't exist in the live system
**Step:** install.md 1.11
**Problem:** Checked the path 1.11 copies the rescue kernel from: `ls /run/archiso/bootmnt/...` gave "No such file or directory". `ls /run/archiso` showed `copytoram`: with 32 GB RAM the live system loads itself fully into RAM and unmounts the ISO and the HDD.
**Fix:** 1.11 now mounts the ISO from the HDD read-only (`mount /dev/sdb1` → `mount -o loop` the ISO) and copies `vmlinuz-linux` + `initramfs-linux.img` from there.
**Updated install.md:** yes (1.11, note in 1.1)
 
## 2026-11-23: Live ISO booted, but on a new IP
**Step:** install.md 0.6 / 1.1
**Problem:** After `grub-reboot`, `ssh` to the old IP (.165) just hung. The live ISO had booted fine but got a different address from the router.
**Fix:** Installed nmap on Windows (`winget install Insecure.Nmap`), ran `nmap -p 22 --open 192.168.1.0/24` and found it at .166 as `archiso.lan` (MAC matched the server). The router names devices, so `archiso.lan` works too.
**Updated install.md:** yes (0.6, 1.1, 2.2)
 
## 2026-11-23: Old system uses GRUB; the ISO has no intel-ucode.img
**Step:** install.md 0.5
**Problem:** `bootctl is-installed` said "Permission denied" and fell through to "probably GRUB" (needs sudo). `ls /boot` confirmed GRUB. Checking the built ISO showed no `intel-ucode.img`: new archiso bakes microcode into the initramfs, so the guide's `initrd` line for it would have broken the boot.
**Fix:** Added the GRUB entry without the ucode file, plus `insmod ext2/loopback/iso9660`. Set `GRUB_DEFAULT=saved` and `grub-set-default 0`, which `grub-reboot` needs for one-time boots.
**Updated install.md:** yes (0.4 verify step, 0.5 rewritten, 1.11 ucode removed)
 
## 2026-11-23: ISO build failed on broadcom-wl
**Step:** install.md 0.4
**Problem:** `mkarchiso` stopped with `error: target not found: broadcom-wl`. The releng profile lists a package that's no longer in the repos.
**Fix:** `sed -i '/^broadcom-wl$/d' ~/archlive/packages.x86_64`, cleared `/tmp/archiso-work`, rebuilt. Not needed anyway (Broadcom Wi-Fi driver; the server uses ethernet).
**Updated install.md:** yes
 
## 2026-11-23: SSH key in /tmp was lost after the power-off
**Step:** install.md 0.3 → 0.4
**Problem:** The guide copied my public key to `/tmp/mykey.pub`. The full power-off in 0.3 cleared `/tmp`, so the key was gone before 0.4 needed it.
**Fix:** Copied the key again. Changed the guide to copy it to `~/mykey.pub` (home folder), which survives reboots.
**Updated install.md:** yes (0.1 and 0.4)
 
## 2026-11-23: WD Green HDD already past its rated head-parking cycles
**Step:** install.md 0.3
**Problem:** SMART shows 0 reallocated / 0 pending sectors (surface is healthy), but `Load_Cycle_Count` is 397,849 after 22,267 hours. WD rates Green drives for ~300,000. The heads park every 8 seconds when idle, ~18 times an hour.
**Fix:** Disabled the idle3 timer with `idle3ctl -d /dev/sdb`, then did a full power-off; `idle3ctl -g` now reports it disabled. The guide said idle3-tools was in the AUR, but it's in the official repos: `sudo pacman -S idle3-tools`. Treating this drive as replaceable: media and Ollama models only, nothing irreplaceable. Plan to replace it with a NAS drive (WD Red Plus / IronWolf) eventually.
**Updated install.md:** yes (moved from 3.5 to 0.3, fixed the install command)
 
## 2026-11-23: SSH wasn't running on the old system
**Step:** install.md 0.1
**Problem:** `scp` to the server failed with "Connection refused". `ss -tlnp | grep :22` showed nothing listening; I'd been using Cockpit's web terminal the whole time.
**Fix:** `sudo systemctl enable --now sshd`
**Updated install.md:** yes
 
## 2026-11-23: scp from Windows couldn't find my key
**Step:** install.md 0.1
**Problem:** In PowerShell/cmd, `~/.ssh/...` doesn't work: "No such file or directory".
**Fix:** Use `$env:USERPROFILE\.ssh\id_ed25519.pub` (PowerShell) or run it from Git Bash.
**Updated install.md:** yes
 
## 2026-11-23: Ethernet wouldn't get an IP address
**Step:** before install
**Problem:** `enp4s0` stuck on "connecting (getting IP configuration)"; link lights were on, but DHCP timed out.
**Fix:** Swapped the ethernet cable. The old one was faulty.
**Updated install.md:** no
 
## 2026-11-23: Decisions
- Clean reinstall instead of fixing the old install; nothing kept from the old system.
- Headless install (no monitor/keyboard), ethernet instead of Wi-Fi.
- Disk password + remote unlock over SSH (tinyssh in the initramfs), not TPM auto-unlock.
- HDD encrypted too; it unlocks with a keyfile stored inside the encrypted SSD.
 