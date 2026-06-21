---
layout: post
title: "AT&T BGW320 IP Passthrough: The Part Every Guide Leaves Out"
date: 2026-06-20
categories: [networking, home-lab]
tags: [att, bgw320, tp-link, ip-passthrough, double-nat, home-network]
excerpt: "Every AT&T IP Passthrough guide tells you to reserve a static DHCP IP for your downstream router. That's exactly what breaks it on the BGW320-500."
---

# AT&T BGW320 IP Passthrough: The Part Every Guide Leaves Out

So you've got AT&T Fiber, a BGW320-500 acting as your gateway, and a perfectly good router sitting downstream getting double-NATted into oblivion. You've read the guides. You've found the IP Passthrough setting. You've done everything right. And your router is still getting a 192.168.1.x address.

Welcome to the club. Here's what actually fixed it.

---

## The Goal

The setup I was going for is pretty common in home labs and serious home networks:

- **AT&T BGW320-500** handles the ISP connection and serves as the gateway for a general-purpose WiFi network (what I'm calling UltraViolet — `192.168.1.0/24`)
- **TP-Link AX1500** acts as a secondary router for a dedicated lab/project network (HyperViolet — `192.168.0.0/24`)
- The TP-Link gets a real public IP via IP Passthrough — no double-NAT, no pain when port forwarding later for things like a Minecraft server or media services

The end state looks like this:

```
Internet
   │
AT&T BGW320-500 (192.168.1.254)
   ├── UltraViolet WiFi  (192.168.1.x devices)
   └── TP-Link WAN port ──> TP-Link AX1500 (192.168.0.1)
                                └── HyperViolet WiFi  (192.168.0.x devices)
                                └── Proxmox home lab  (192.168.0.x static) [coming soon]
```

Pretty clean. Getting there was less clean.

---

## Hardware

| Device | Model | Role |
|---|---|---|
| AT&T Gateway | BGW320-500 | ISP modem/router, UltraViolet network |
| Secondary Router | TP-Link AX1500 (WiFi 6) | HyperViolet network, lab traffic |

---

## What the Guides Tell You To Do

Every AT&T IP Passthrough guide follows the same basic steps:

1. Find the TP-Link's WAN MAC address (check the label on the bottom or find it in the BGW320's connected devices list)
2. Reserve a static LAN IP for the TP-Link in the BGW320's DHCP settings (Home Network > Subnets & DHCP)
3. Enable IP Passthrough: Settings > Firewall > IP Passthrough, set Allocation Mode to `Passthrough`, Passthrough Mode to `DHCPS-fixed`, and point it at the TP-Link's MAC
4. Reboot everything and check the TP-Link's WAN status — it should show your public IP

On paper, solid. In practice, step 2 is a trap.

---

## What Actually Happened

After following the steps above, the TP-Link's WAN interface was showing `192.168.1.253` — the reserved private IP from step 2 — instead of the public IP. IP Passthrough appeared to be configured correctly:

- Allocation Mode: **Passthrough** ✓
- Passthrough Mode: **DHCPS-fixed** ✓
- Passthrough Fixed MAC Address: **confirmed, matched the TP-Link WAN MAC** ✓

The BGW320's broadband IP was a real public address (`xxx.xx.xxx.xxx`), so there was definitely something to pass through. The physical connection was correct — BGW320 LAN port to TP-Link blue WAN/Internet port. The TP-Link WAN was set to Dynamic IP. Everything looked right and nothing worked.

### Attempted fixes that didn't help:

- Software restart of the BGW320 from the admin panel — no change
- Release and renew DHCP on the TP-Link — still got `192.168.1.253`
- Verifying the MAC match in Connected Devices vs. IP Passthrough config — they matched

### What finally worked:

**Delete the static DHCP reservation.**

That's it. The DHCP reservation for the TP-Link's MAC (step 2 in every guide) conflicts directly with IP Passthrough on the BGW320-500. When both exist, the BGW320 applies the reservation first and the TP-Link gets its private address — `192.168.1.253` in this case — instead of the public IP. The IP Passthrough logic never fires.

After removing the reservation:

1. Saved the change
2. Power cycled the BGW320 (unplugged from wall, waited 60 seconds — **not** a software restart, which doesn't fully apply IP Passthrough settings)
3. Once the BGW320 was fully back online, power cycled the TP-Link
4. Checked TP-Link admin > Status/Internet

WAN IP: `xxx.xx.xxx.xxx` — the real public address. Done.

---

## The Actual Working Steps

If you're setting this up from scratch on a BGW320-500, here's what works:

### 1. Find the TP-Link's WAN MAC address
Check the label on the bottom of the TP-Link, or connect it to the BGW320 and look it up in Home Network > Connected Devices.

### 2. Skip the DHCP reservation
Do not create a static reservation for the TP-Link. IP Passthrough with DHCPS-fixed mode handles MAC matching internally — a reservation is redundant and actively breaks it.

### 3. Enable IP Passthrough
- BGW320 admin (`192.168.1.254`) > Settings > Firewall > IP Passthrough
- Allocation Mode: `Passthrough`
- Passthrough Mode: `DHCPS-fixed`
- Passthrough Fixed MAC Address: TP-Link WAN MAC

### 4. Cold reboot the BGW320
Unplug it from the wall. Wait at least 60 seconds. Plug it back in. Wait until it's fully online (broadband light solid). The admin panel's restart button does not fully apply IP Passthrough on the BGW320-500.

### 5. Power cycle the TP-Link
Once the BGW320 is back up, unplug and replug the TP-Link. This forces it to make a fresh DHCP request, which the BGW320 will now answer with the public IP.

### 6. Verify
TP-Link admin (`192.168.0.1`) > Status or Internet — WAN IP should be your real public address, not a 192.168.x.x address.

---

## Why This Matters

If you're running anything behind that secondary router that needs inbound connections — game servers, self-hosted services, remote access — double-NAT will cause you pain. Port forwarding through two NAT layers is possible but annoying to maintain, and some protocols don't cooperate at all. Getting a real public IP on the TP-Link makes all of that straightforward.

The BGW320 is also the gateway for the UltraViolet network, which will need a static route added later so devices on both networks can reach each other. But that's a separate post.

---

## TL;DR

AT&T BGW320-500 IP Passthrough (DHCPS-fixed) does not work if the target device also has a DHCP reservation. Delete the reservation. Cold reboot the BGW320 from the wall (not from the admin panel). Power cycle the downstream router. Done.

---

*Hardware: AT&T BGW320-500, TP-Link AX1500 (WiFi 6 / Archer AX1500)*
