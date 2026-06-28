---
layout: post
title: "Building a Home Media Server on Proxmox: Simpler Than I Expected"
date: 2026-06-27
categories: [home-lab, self-hosting]
tags: [proxmox, lxc, samba, jellyfin, ntfs, intel-qsv, home-server]
excerpt: "A Dell OptiPlex 7070 repurposed as a Proxmox home server running Samba and Jellyfin in LXC containers. The hardest part was already done."
---

# Building a Home Media Server on Proxmox: Simpler Than I Expected

I picked up a Dell OptiPlex 7070 to repurpose as a home server — 48 GB of RAM, a 1 TB NVMe for the OS, and a 2 TB SATA drive full of media I've been dragging from machine to machine for years. The goal was straightforward: get that media accessible over the network and stream it properly, without paying a monthly fee to Plex or copying files manually.

Spoiler: it worked, and it was surprisingly painless. Here's how it went.

---

## Why Proxmox

I wanted to run multiple services — a file server and a media server at minimum — in a way that kept them isolated and easy to manage. Proxmox VE gives you a web UI, LXC containers, and a solid ecosystem for exactly this kind of thing. Alternatives like plain Ubuntu with Docker are fine, but I like having Proxmox as the foundation: clean separation between the host and workloads, and containers that feel more like lightweight VMs than Docker services.

For Linux-native services (Samba, Jellyfin), LXC containers are the right call over full VMs. Less overhead, faster startup, and you can pass host devices and directories in cleanly.

---

## The Network Was Already Set Up

I won't go into the networking here — I covered that in the [previous post](/posts/2026/06/20/att-bgw320-ip-passthrough/). Short version: the TP-Link AX1500 runs a dedicated lab network (`192.168.0.0/24`), has a real public IP via AT&T passthrough, and has a static route so devices on my main network can reach lab IPs without any extra config. Having that foundation in place made everything below much smoother.

The Proxmox host sits at `192.168.0.10`.

---

## The NTFS Drive

This was the thing I expected to be a headache. The 2 TB drive has years of media on it formatted as NTFS — I wasn't going to reformat it and lose everything, and I didn't want to copy 2 TB somewhere else just to copy it back.

Turns out `ntfs-3g` handles this without drama. A few lines in `/etc/fstab` on the Proxmox host and the drive mounts at boot with read/write access. The key options I used: `uid=1000,gid=1000,umask=022,nofail`. The `nofail` flag means if the drive ever isn't present, the server still boots instead of hanging.

Mount the drive once on the **host**, then bind-mount it into whatever containers need it. One NTFS driver, multiple containers — clean.

---

## Samba File Server

Container 101 (`192.168.0.11`), privileged (required for NTFS bind mounts to work correctly with file permissions), 4 GB rootfs on the NVMe.

Setup inside the container:
1. `apt install samba`
2. Edit `/etc/samba/smb.conf` to add a share pointing at the bind-mounted media directory
3. `useradd -M -s /sbin/nologin <username>` then `smbpasswd -a <username>` to create a Samba user

That's it. Macs connect via `smb://192.168.0.11/media`, Windows boxes via `\\192.168.0.11\media`. Both work across network segments because of the static route on the AT&T modem — no VPN, no extra firewall rules needed.

---

## Jellyfin Media Server

Container 102 (`192.168.0.12`, gateway `192.168.0.1`), unprivileged, 2 CPU cores, 2 GB RAM.

Jellyfin is free, open source, no account required, and the setup wizard is good. Install via the official apt repo, point it at the media directory, run the wizard at `http://192.168.0.12:8096`. Done.

### Hardware transcoding

The OptiPlex 7070 has Intel UHD 630 integrated graphics. Jellyfin supports Intel Quick Sync for hardware transcoding, which means it can transcode video using the GPU instead of burning CPU cycles. To enable it, pass `/dev/dri` into the container via the Proxmox LXC config, then enable Intel QSV in Jellyfin's transcoding settings.

I got it configured but haven't put it under real load yet — streaming a few test files worked fine. I'll know more once I've actually used it day-to-day.

---

## Overall Impression

Honestly? The hardest part of this build was the networking, and I'd already done that. Once the TP-Link had a real public IP and the static route was in place, setting up Proxmox and the containers was just following steps. The NTFS situation — which I expected to be the most painful — was a non-issue with `ntfs-3g`.

The Proxmox web UI is genuinely good. Spinning up an LXC container, setting a static IP, and adding a bind mount is a few minutes of work. Having everything isolated in containers with their own IPs makes the whole thing feel manageable in a way that a single Ubuntu box running everything wouldn't.

---

## What's Next

**Minecraft Bedrock server** — that's the next container. UDP 19132, forwarded on the TP-Link. I'll need DDNS since the public IP can rotate, but the infrastructure for that is already in place.

**Longer term**, I'm thinking about:

- A self-hosted **DNS server** — Unbound or Pi-hole in DNS-only mode. Gives me more control over name resolution on the lab network.
- **Ad blocking** — Pi-hole and AdGuard Home are the obvious options; I need to research which fits better with where the DNS setup goes.
- An **Active Directory lab** — Windows Server and Win11 machines in full VMs (not LXC) for practicing AD and Group Policy. A natural next step once the infrastructure is more settled.

For now, the media server works, the file share works, and I can watch my library from anywhere on the network. That's a good place to be.

---

*Hardware: Dell OptiPlex 7070, TP-Link AX1500 (WiFi 6 / Archer AX1500), AT&T BGW320-500*  
*Stack: Proxmox VE, Debian 13 (Trixie) LXC, Samba, Jellyfin*
