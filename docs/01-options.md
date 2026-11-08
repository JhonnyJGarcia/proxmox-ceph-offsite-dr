# 1. Options for off-site disaster recovery

The question this chapter answers: *my Proxmox VE + Ceph site may be lost entirely (fire, flood, theft, a long power or network outage). How do I get the guests running somewhere else, and how much data and time will I lose?*

Two numbers frame every option:

- **RPO** (recovery point objective): how much recent data you can afford to lose. "15 minutes" means after a disaster you restart from the state of at most 15 minutes ago.
- **RTO** (recovery time objective): how long the service may be down before it runs again at the other site.

## The options

### A. Stretch the cluster over both sites

One Proxmox cluster, one Ceph cluster, nodes in two places.

- **Proxmox:** cluster membership runs on Corosync, which the Proxmox documentation describes as very sensitive to latency and sizes for LAN performance (latencies under 5 ms). Over a WAN or a VPN to the cloud, latency spikes cost quorum, and with HA, nodes get fenced.
- **Ceph:** writes are acknowledged only when every replica has them. Replicas at a remote site put the WAN round trip on **every write**. Ceph has a dedicated *stretch mode*, designed for two nearby data centres with a third site as tie-breaker, still with low latency between them.

**Fits:** two buildings on the same campus, joined by fibre.
**Does not fit:** a remote office, a home lab, or a cloud region over a VPN.

### B. Proxmox storage replication (`pvesr`)

Built into Proxmox and visible in the GUI, it replicates guest disks between nodes on a schedule.

- Works only with **ZFS** local storage (the only supported storage type), replicating to **another node of the same cluster**.
- With Ceph it does not apply: Ceph already replicates inside the cluster, and `pvesr` cannot target another cluster.

The ZFS equivalents for an off-cluster copy (`pve-zsync`, or `zfs send`/`receive` tools such as syncoid) have the same limit: they need ZFS storage, not Ceph.

### C. Backups to an off-site Proxmox Backup Server

Back up guests to a Proxmox Backup Server (PBS), and get those backups off-site: either a PBS at the other site, or a local PBS whose datastore is copied to a remote PBS by a **sync job**.

- **RPO:** the backup interval, typically hours to a day. Incremental, deduplicated backups make frequent runs affordable, but they rarely get down to minutes.
- **RTO:** a full **restore** of every disk at the other site before the guests can start: minutes for small guests, hours for large ones. The other site needs a Proxmox node with storage ready to restore onto.
- **History:** many restore points, retention policies, verification. That protects against what replication cannot: accidental deletion, corruption, ransomware. A replica copies those too, within minutes.
- Simple, well supported, GUI-driven.

### D. Ceph RBD mirroring

A second Ceph cluster at the other site receives a continuous copy of selected images. Explained in detail in [chapter 2](02-concepts.md).

- **RPO:** minutes. Snapshot mode copies the changes at every scheduled mirror snapshot (for example every 15 minutes); journal mode is near-continuous but doubles write latency and is not supported by `krbd`, which Proxmox uses for **containers**.
- **RTO:** minutes. The data is already there: promote the image, register the guest, start it.
- **Failback** without data loss, with two-way mirroring.
- **Requirements:** a Ceph cluster at the DR site (it can be a single node, see [chapter 3](03-architecture.md)), Ceph MONs and OSDs reachable from the other site, and command-line work: none of this is in the Proxmox GUI. **Guest configurations are not replicated** and must be kept at the DR site separately.
- **No history.** A deleted file or an encrypted disk is replicated at the next snapshot.

### E. Do-it-yourself snapshot shipping

`rbd export-diff` / `rbd import-diff` over SSH, driven by a script: snapshot, send the difference since the last snapshot, apply it at the other side. It is what snapshot mirroring does, without the daemon, the peer setup or the status reporting, and with your own script to maintain. Worth knowing; rarely worth choosing over D.

### Not disaster recovery: moving guests between clusters

Proxmox can migrate a guest to **another cluster** (`qm remote-migrate` / `pct remote-migrate`, marked *experimental* in the documentation, and Proxmox Datacenter Manager). It is a migration: it is started from the source cluster and copies the guest's disks from there at that moment. Useful to move workloads ahead of planned maintenance; no help once the source site is gone.

## Side by side

| | RPO | RTO | Needs at the DR site | Survives a deletion / ransomware | Complexity |
|---|---|---|---|---|---|
| A. Stretched cluster | 0 | Automatic (HA) | Full nodes, low-latency link | No | Low on paper; latency rules it out off-site |
| B. `pvesr` / ZFS tools | Minutes | Minutes | ZFS storage | No | Low, but not for Ceph |
| C. Off-site PBS | Hours | Restore time | PBS + a node to restore onto | **Yes** | Low |
| D. RBD mirroring | **Minutes** | **Minutes** | A Ceph cluster (≥ 1 node) | No | **High**, all CLI |
| E. DIY diff shipping | Minutes | Minutes | A Ceph cluster | No | High, your own script |

## What this guide recommends

**Use D and C together.** They cover each other's gaps:

- **RBD mirroring** answers *"the site is gone, restart elsewhere within minutes, losing minutes of data"*.
- **Off-site backups** answer *"something bad was replicated, give me yesterday's version"*.

Mirroring alone is not a backup; backups alone mean a slow recovery from a site loss.

If an RPO of hours and an RTO of a restore are acceptable, C alone is simpler and fully supported: choose it and stop here. Read on if minutes matter.

## Sizing the DR site

A second 3-node cluster, like the primary, needs nothing special: Ceph's default rules work, and the DR site survives the loss of one of its nodes. It also triples the cost.

This guide uses the cheapest working option: a **single Proxmox node with its own small Ceph cluster**, in a public cloud. It costs a few euros a day while running, and can be switched off when not needed, at the price of no protection while it is off ([chapter 2.3](02-concepts.md#23-the-rbd-mirror-daemon-pulls)). It needs a few adjustments, all covered in [chapter 3](03-architecture.md): a CRUSH rule that places replicas on separate disks instead of separate hosts, and default pool sizes that match one node. It survives the loss of a disk, not of the node, which is acceptable for a copy whose authoritative version lives at the primary site.

Two nodes is not a useful middle ground: two Ceph MONs cannot keep quorum if one fails, and a two-node Proxmox cluster needs an external vote (QDevice). For a multi-node DR site, use three.

---

Next: [2. Concepts](02-concepts.md).

References: [Proxmox VE: Cluster Manager](https://pve.proxmox.com/pve-docs/chapter-pvecm.html) · [Proxmox VE: Storage Replication](https://pve.proxmox.com/pve-docs/chapter-pvesr.html) · [Proxmox VE: pct remote-migrate](https://pve.proxmox.com/pve-docs/pct.1.html) · [Proxmox VE wiki: PVE-zsync](https://pve.proxmox.com/wiki/PVE-zsync) · [Proxmox Backup Server documentation](https://pbs.proxmox.com/docs/) · [Ceph: Stretch Clusters](https://docs.ceph.com/en/latest/rados/operations/stretch-mode/) · [Ceph: RBD Mirroring](https://docs.ceph.com/en/latest/rbd/rbd-mirroring/)
