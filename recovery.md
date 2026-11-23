# Recovery
 
What to do when something goes wrong. **No secrets in this file**: they all live in the vault.
 
## Where the secrets are
 
All server secrets are in a **KeePassXC** database, `server-vault.kdbx`:
 
- Copy 1: dedicated USB stick labeled `VAULT` (kept at home, not plugged into the server)
- Copy 2: a second USB stick (or an everyday one), kept somewhere else
- Master password: on paper, at home
After changing anything in the vault, copy the file to both sticks.
 
| Vault entry | What it's for |
|---|---|
| `super-server: disk (SSD)` | Unlocks the system disk at every boot |
| `super-server: user <username>` | sudo / local login |
| `super-server: HDD backup` | Second key for the data disk (normally unlocked by a keyfile) |
| `SSH key (main PC)` | Private key as an attachment + its passphrase |
| LUKS header backups | Attachments (see install.md 3.2) |
| GitHub, email accounts | Account passwords |
 
## Normal reboot
 
1. `nmap -p 22 --open 192.168.1.0/24` → find the server (by its MAC)
2. `ssh root@<ip>` → **type** the disk password from the vault (pasting doesn't work at this prompt;
   KeePassXC Auto-Type `Ctrl+Shift+V` does)
3. `ssh <username>@<ip>`
## Your main PC died / the SSH key is lost
 
The server and its data are fine. Only the key to reach it is missing.
 
1. New PC: install KeePassXC, open the vault from a USB stick.
2. Save the `SSH key (main PC)` attachment as `C:\Users\<user>\.ssh\id_ed25519`.
3. Recreate the public half: `ssh-keygen -y -f $env:USERPROFILE\.ssh\id_ed25519 > $env:USERPROFILE\.ssh\id_ed25519.pub`
4. Start the ssh-agent service (admin PowerShell, see install.md 0.1), `ssh-add` the key with the
   passphrase from the vault, then connect as usual.
No vault at all? Plug a USB keyboard into the server and reboot. The disk password prompt also
appears on the local console, so it can be typed blind. Then a monitor is needed to log in
locally and add a new key to `~/.ssh/authorized_keys` and `/etc/tinyssh/root_key`
(+ `sudo mkinitcpio -P`).
 
## Server won't come up after a change
 
- **Still reachable:** before a risky change, `sudo bootctl set-oneshot archlive.conf`, reboot,
  and work from the rescue system: the live ISO on the HDD's 4 GB `rescue` partition (`sdb1`).
  Find it with nmap as `archiso.lan`, `ssh root@<ip>`.
- **Unlock SSH doesn't come up:** see install.md 2.2.
- **Broken update:** `sudo snapper list` → `sudo snapper rollback <number>` → reboot.
## Forgot the disk password
 
There is no recovery: that's the point of encryption. It's in the vault, on two sticks.
A LUKS header backup does **not** help without the password.
 
## To do
 
- [ ] 2FA on the email account GitHub uses, then on GitHub and the other email account
      (backup codes into the vault)
- [ ] Second SSH key on another device (phone/laptop) as a backup way in
- [ ] Automatic fallback to the rescue system if a boot fails (systemd-boot boot counting)
 