# super-server

My personal home server: Arch Linux, fully encrypted, headless, unlocked over SSH at boot.

## What it runs

- Jellyfin (video streaming, GTX 1060 transcoding)
- Game servers (Minecraft, …) in Docker
- Ollama (local AI models)
- Samba file share (LAN only)
- Cockpit web admin (LAN only)

## Hardware

| Part | Model |
|---|---|
| CPU | Intel i7-9700KF |
| GPU | NVIDIA GTX 1060 6GB |
| RAM | 32 GB |
| SSD | ADATA SU650 256 GB, system (LUKS2 + btrfs) |
| HDD | WD Green 2 TB, data (LUKS2 + ext4) |
| Network | Ethernet, `192.168.1.0/24` |

## How this was made

I planned and built this server with the help of an AI assistant (Claude, by Anthropic).
The AI helped me design the setup, explained the concepts, and drafted the guides.

Everything in this repo was done by me on real hardware: I ran every command, hit the
problems in `log.md` (bad cable, a missing package in the ISO build, the unlock prompt
rejecting pasted passwords, and more), worked through them, and updated the guide to match
what actually worked. I can explain every step and why it's there.
## Repo layout

| Path | What |
|---|---|
| `install.md` | Step-by-step install guide (kept up to date with what actually worked) |
| `log.md` | Problems I hit and how I fixed them |
| `services/` | One folder per service: compose files, configs, notes |
| `scripts/` | Backup and maintenance scripts |
| `dotfiles/` | Shell and editor configs |
| `recovery.md` | What to do when something breaks; where the secrets are kept |
## Status

- [ ] Base install (encrypted, remote unlock)
- [ ] HDD encrypted, snapshots, NVIDIA driver
- [ ] Docker + Jellyfin
- [ ] Game servers
- [ ] Ollama
- [ ] Samba + Cockpit
- [ ] Tailscale
- [ ] Backups
- [ ] OPNsense router machine

## Reboot / unlock

1. `nmap -p 22 --open 192.168.1.0/24` to find the server
2. `ssh root@<ip>`, then type the disk password
3. `ssh <username>@super-server.local`
