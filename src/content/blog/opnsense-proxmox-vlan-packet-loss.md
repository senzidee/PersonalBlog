---
title: 'OPNsense in Proxmox'
description: 'OPNsense in Proxmox: chasing a phantom packet loss bug (and the VLAN fix that finally worked)'
pubDatetime: 2026-09-26
ogogImage: '/assets/opnsense-in-proxmox.jpg'
tags: ['OPNsense', 'Proxmox', 'VLAN', 'Packet-Loss']
---

# OPNsense in Proxmox: chasing a phantom packet loss bug (and the VLAN fix that finally worked)

I run OPNsense as a VM inside Proxmox, connected to a VLAN-tagged network via a trunk (`Trk1`) to an HP 2530 switch. Recently I started seeing something odd: **6-10% packet loss** on traffic forwarded through OPT1 to LAN/WAN, but **0% loss** when pinging directly from the Proxmox host to OPT1's address. Here's how I tracked it down — including the dead ends.

## First suspect: VirtIO hardware offload

The classic villain in Proxmox + OPNsense setups is hardware checksum/TSO/LRO offload on VirtIO NICs. It's notoriously buggy specifically on *forwarded* traffic, which matched my symptom perfectly — a direct ping to an interface doesn't trigger the same code path as traffic actually routed between interfaces.

I disabled it in **System → Advanced → Networking**:

- Disable hardware checksum offload
- Disable hardware TCP segmentation offload (TSO)
- Disable hardware large receive offload (LRO)

I verified with `ifconfig` that the flags were actually gone from the interface:

```
options=880008<VLAN_MTU,LINKSTATE,HWSTATES>
```

No RXCSUM, TXCSUM, TSO, or LRO left. Rebooted the VM to be sure the settings fully applied.

**Result: no change.** Whatever this was, it wasn't the guest-side offload.

## Digging deeper: size-dependent loss

I tested with a don't-fragment, near-MTU-size ping:

```
ping -D -s 1472 8.8.8.8
```

and got roughly 50% loss on large packets going out through OPT1 to the internet, while small pings were mostly fine. That pointed at something size-sensitive happening specifically on forwarded traffic — consistent with an offload-style bug, but now ruled out at the guest level. That meant the problem was more likely sitting *below* OPNsense, in the Proxmox bridge itself.

## The VLAN-aware bridge theory

Proxmox's Linux bridge can operate in two modes:

- A **simple mode**, where VLAN tagging is handled per-vNIC outside the bridge (Proxmox tags/untags at the tap interface level; the bridge itself just forwards opaque Ethernet frames).
- A **VLAN-aware mode**, where the bridge itself understands 802.1Q tags — similar to a real switch, with per-port allowed VLANs (`bridge-vids`).

VLAN-aware bridges combined with VirtIO multiqueue have a history of odd bugs around tagged-frame handling, especially at larger packet sizes.

My fix attempt: stop tagging at the Proxmox/vNIC level entirely, give OPNsense a raw trunk, and let OPNsense create its own VLAN sub-interface — the "textbook correct" way to do it, in theory.

## When the "correct" fix broke everything

After the migration, the new VLAN interface was **completely dead**:

- RX counters stuck at zero
- ARP requests going out but never getting a reply
- Packet captures on the interface showing nothing at all

Tracing the path hop by hop with `tcpdump` across `vtnet0 → vmbr0 → bond0 → Trk1` eventually showed the ARP requests *were* leaving OPNsense and reaching the Proxmox bridge — and even the physical bond showed full ARP request/reply pairs plus live DNS and ICMP traffic from other hosts on the VLAN. The VLAN was healthy at the switch and bridge level. But reachability from OPNsense to devices on that VLAN still failed.

## The actual fix: dedicated Linux VLAN + bridge per VLAN

Instead of a shared VLAN-aware bridge, or pushing VLAN handling into OPNsense's own OS, I created a fully separate path for this VLAN in Proxmox:

- A **Linux VLAN** interface: `bond0.22`
- A **dedicated bridge**: `vmbr22`
- OPNsense's OPT interface vNIC attached directly to `vmbr22`

This way, VLAN tagging and untagging happens exactly once, cleanly, in the kernel's VLAN driver — before traffic ever reaches a shared bridge or the VM. OPNsense's vNIC just sees plain untagged Ethernet on its own dedicated bridge, with zero ambiguity about tagging or bridge-level VLAN filtering.

**This fixed it immediately.**

## Takeaways

- Don't assume VirtIO offload is guilty just because it's the usual suspect — verify with `ifconfig`, not just the GUI checkbox. The setting isn't always fully applied until the interface is reinitialized or the VM rebooted.
- Size-dependent packet loss is a strong diagnostic signal, but it can point at multiple layers (guest offload, host offload, bridge handling) — test each layer in isolation rather than assuming the first suspect is the right one.
- When juggling VLANs in Proxmox, a **dedicated Linux VLAN interface + bridge per VLAN** is simpler and more reliable than a shared VLAN-aware bridge or delegating VLAN logic to the VM's own OS. One clear tagging point beats two systems that might disagree about VLAN handling.
