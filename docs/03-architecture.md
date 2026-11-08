# 3. Architecture

The reference setup: a 3-node Proxmox VE + Ceph cluster at the primary site, a single Proxmox VE node with its own Ceph cluster at the DR site in a public cloud, and a site-to-site VPN between them.

```
 PRIMARY SITE                                          DR SITE (cloud)
 ┌───────────────────────────────────┐                ┌──────────────────────────────────┐
 │ Firewall / VPN endpoint ◄─────────┼── IPsec S2S ───┼─► Cloud VPN gateway              │
 │                                   │                │                                  │
 │ 3 × Proxmox VE + Ceph             │                │ 1 × Proxmox VE + Ceph (VM)       │
 │  MON, MGR, OSD on each node       │  RBD mirroring │  1 MON, 1 MGR, 2 OSD (2 disks)   │
 │  rbd-mirror daemon on one node    │ ◄────────────► │  rbd-mirror daemon               │
 │  pool rbd-dr (size 3, host)       │   both ways    │  pool rbd-dr (size 2, osd)       │
 │  guests on 10.10.10.0/24          │                │  failed-over guests 10.50.2.0/24 │
 └───────────────────────────────────┘                └──────────────────────────────────┘
```

Example addressing used in this guide: primary site LAN and Ceph public network `10.10.10.0/24`; cloud network `10.50.0.0/16` with the DR node at `10.50.1.10` and failed-over guests in `10.50.2.0/24`. Use your own.

## 3.1 Network requirements

### What must reach what

The `rbd-mirror` daemon on each site is a client of the **other** cluster ([2.3](02-concepts.md#23-the-rbd-mirror-daemon-pulls)):

| From | To | Ports |
|---|---|---|
| DR daemon | Primary site MONs | TCP 3300, 6789 |
| DR daemon | **Every** primary site OSD | TCP 6800–7300 |
| Primary daemon (two-way) | DR MON and OSDs | Same ports |

So the **Ceph public network must be routable through the VPN**, in both directions. The Ceph *cluster* network (OSD-to-OSD replication inside a cluster) does not need to be.

Test it before configuring anything, from a node of each site:

```bash
for p in 3300 6789 6800; do timeout 4 bash -c "</dev/tcp/<remote-mon-ip>/$p" && echo "$p open" || echo "$p CLOSED"; done
```

### The VPN

- **Route-based** IPsec (a virtual tunnel interface plus a static or BGP route), so that adding networks later is a route, not a new tunnel policy.
- **MSS clamping** on the tunnel interface (for example 1300): Ceph moves large TCP segments; without clamping, IPsec overhead leads to fragmentation and sessions that stall for no visible reason.
- The VPN endpoint at home can sit behind the ISP router: if it initiates the tunnel, NAT traversal (UDP 4500) needs no port forward.
- Throughput: the initial sync of each image and every snapshot delta cross this link. Your **upload bandwidth** decides how short the snapshot schedule can be.

### Guest networking at the DR site

Failed-over guests **do not keep their primary-site addresses**. Two reasons:

1. **Routing conflict.** If the DR node had a connected route to the primary site's subnet, traffic to the primary site's Ceph MONs would stay local and mirroring would break. The two sites must keep distinct address spaces.
2. **Cloud networks drop unknown MAC addresses.** A guest bridged onto the VM's network interface would not get traffic: the cloud only delivers frames for the MAC addresses it assigned. Guests must be **routed** behind the node.

The setup that works:

- A Proxmox **SDN simple zone** on the DR node, with a vnet and subnet `10.50.2.0/24`, gateway `10.50.2.1` on the node, **SNAT** enabled for internet egress.
- **IP forwarding** enabled on the VM's network interface in the cloud (the node routes for its guests).
- A cloud **route table** (UDR): `10.50.2.0/24` → next hop the DR node, attached to the gateway subnet (so traffic from the primary site reaches the guests) and to the DR node's subnet.

At failover the guest's `net0` is changed to the SDN vnet bridge and a `10.50.2.x` address. Clients and DNS records then need to follow; this guide does not automate that part.

Cloud default outbound access usually carries TCP and UDP but **not ICMP**: `ping` to the internet fails from both the node and its guests while HTTP works. Test egress with TCP.

## 3.2 A single-node Ceph cluster at the DR site

### CRUSH and the failure domain

Ceph decides where each replica goes with **CRUSH**, an algorithm working on a tree that describes the cluster (`ceph osd tree`):

```
PRIMARY                           DR
root default                      root default
├── host pve1                     └── host pve-dr
│   └── osd.0                         ├── osd.0
├── host pve2                         └── osd.1
│   └── osd.1
└── host pve3
    └── osd.2
```

A CRUSH rule says "put the N replicas on N **different** items of type X". X is the **failure domain**: the thing that can fail as a whole, which no two replicas may share.

- Primary site, default rule (`replicated_rule`, failure domain `host`): 3 replicas on 3 hosts. Losing a whole node loses no data.
- DR site with the default rule: CRUSH looks for 2 different hosts, finds one, and the second replica has nowhere to go. Placement groups stay `undersized` forever.
- DR site with a rule whose failure domain is `osd`: 2 replicas on 2 different **disks** of the same host. It survives a disk failure, not the loss of the node. That is acceptable for a copy whose authoritative version is at the primary site.

```bash
ceph osd crush rule create-replicated replicated_osd default osd
#                                     └ rule name    └ root  └ failure domain
```

In larger clusters the failure domain is often the **rack** (shared power and switch). A 3-node DR cluster needs none of this: the default rule works ([1. Options](01-options.md#sizing-the-dr-site)).

### Three settings that follow from one node

| Setting | Why |
|---|---|
| Pool `rbd-dr` with `size 2`, `min_size 1`, rule `replicated_osd` | Two copies on two disks; stays writable with one |
| `.mgr` pool on `replicated_osd` too | The manager creates this pool by itself, with the default rule: one PG stays `undersized` |
| `osd_pool_default_size = 2`, `osd_pool_default_min_size = 1` in `/etc/pve/ceph.conf` | Otherwise Ceph warns `OSD count 2 < osd_pool_default_size 3`. `pveceph init` writes the value into the file, and the file overrides `ceph config set` |

The pool **must have the same name** as at the primary site: mirroring pairs pools by name.

### Memory

A small VM cannot give each OSD the 4 GiB default target. With 8 GB of RAM and two OSDs:

```bash
ceph config set osd osd_memory_target 2147483648   # 2 GiB
```

## 3.3 The DR node as a cloud VM

- **No public IP.** It is reached through the VPN only, by SSH with a jump through a primary-site node, and the Proxmox GUI through an SSH port-forward. Remember that the DR node stores a key that can write to the primary site's Ceph ([2.4](02-concepts.md#24-peers-and-directions)): treat it as production.
- **Proxmox VE installed on top of the cloud's Debian image** (the official "install on Debian" procedure). Keep the Debian kernel installed as a boot fallback: there is no install media in the cloud, only the serial console.
- The cloud image's `/etc/network/interfaces` may lack `source /etc/network/interfaces.d/*`; Proxmox SDN writes its configuration there, and zones show *error* until the line is added.
- Two data disks for the OSDs, besides the OS disk.
- Expect an initial setup of the size used here (2 vCPU, 8 GB RAM, two 32 GB disks) to cost a few euros per day while running, VPN gateway included. A deallocated VM stops compute billing, and stops protecting you.

## 3.4 Where the daemons run

| | Primary site | DR site |
|---|---|---|
| One-way | No daemon | `rbd-mirror` |
| Two-way | `rbd-mirror` on **one** node | `rbd-mirror` |

A single daemon at the primary site means that if that node is down, the primary site cannot receive from the DR site. Running a second daemon on another node removes that single point of failure (the daemons elect a leader).

---

Next: [4. Setup](04-setup.md).
