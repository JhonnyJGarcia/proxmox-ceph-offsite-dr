# 2. Concepts: how RBD snapshot mirroring works

Before any command, the model. Every step in the setup and every rule in the failover procedures follows from five ideas. Each section ends with what it means in practice.

## 2.1 One image, two copies, one owner

A Proxmox container disk on Ceph is an **RBD image** (for example `vm-101-disk-0` in pool `rbd-dr`). With mirroring enabled, that image exists in **two clusters**, but it is still **one image**: both copies carry the same `global_id`.

At any moment exactly **one copy is primary**:

| | Primary | Non-primary |
|---|---|---|
| Who can write to it | The guest | **Only** the `rbd-mirror` daemon, applying changes from the primary |
| For the guest | A normal disk | A read-only disk |

Why only one: RBD works with **blocks** and knows nothing about files. If guests wrote to both copies, the copies would diverge block by block, and there is no way to merge two diverged block devices. The only safe rule is a single writer.

The role is **per image**, not per site. With two-way mirroring, container 101 can be primary at site A while container 102 is primary at site B, each replicating to the other site. "Site A is the main site" is your decision, not Ceph's.

`rbd info <pool>/<image>` shows the role (`mirroring primary: true`). `rbd mirror pool status <pool> --verbose` shows it per site:

- `up+stopped`, *local image is primary*: this site owns the image; its daemon has nothing to pull. Not an error.
- `up+replaying`: this site holds the replica and is applying changes.

**In practice:** a guest started on a non-primary copy gets `EROFS` (`write result -30` in the kernel log) on every write, while the local daemon keeps writing replicated blocks under its mounted filesystem. The result is a corrupted filesystem ([results](07-results.md)). Proxmox does not prevent this start.

## 2.2 Asynchronous replication with mirror snapshots

| | Synchronous | Asynchronous |
|---|---|---|
| The guest's write is acknowledged | When **every** copy has it | As soon as the local site has it |
| Cost | Every write waits for the network round trip | None on the write path |
| Lost if the site dies | Nothing | Writes not yet copied: the **RPO** |
| Example | Ceph replicas inside one cluster | RBD mirroring between clusters |

Ceph inside a cluster is synchronous, which is why it wants a fast, low-latency network, and why a cluster stretched over a WAN or VPN is not an option ([options](01-options.md)). RBD mirroring is asynchronous.

In **snapshot mode**:

1. The site where the image is **primary** takes a **mirror snapshot** on a schedule (for example every 15 minutes), or on demand with `rbd mirror image snapshot`. They show up in `rbd snap ls --all` as `.mirror.primary.<id>.<id>`.
2. The daemon on the other site compares the last snapshot it has applied with the new one and transfers **only the blocks that changed**.
3. When the whole delta is applied, the replica matches that snapshot: in the status, `local_snapshot_timestamp` equals `remote_snapshot_timestamp`.

A snapshot is **not a copy**. Ceph snapshots are copy-on-write: taking one only marks "from now on, keep the old version of any block that changes". It is instant and initially takes no space. The first sync transfers the data actually written (thin provisioning: an 8 GB disk with 80 MB used sends about 80 MB); later syncs send only the changed blocks, found quickly thanks to the `object-map` and `fast-diff` image features.

The site that is primary takes the snapshots, because it owns the data. In a two-way setup **both** sites need a schedule, so that the DR site keeps taking snapshots after a failover.

A snapshot of a running guest is **crash-consistent**: like the state after a power cut, which journaling filesystems recover from. Databases that need application consistency need extra care (flushing or freezing before the snapshot).

**In practice:**
- Worst-case RPO = schedule interval **plus** transfer time. If the last snapshot had not finished transferring when the site died, the recovery point is the one before.
- A shorter schedule lowers the RPO but needs more upload bandwidth; if the interval is shorter than the transfer time, snapshots queue up and the lag grows instead of shrinking. By default Ceph keeps at most 5 mirror snapshots per image (`rbd_mirroring_max_mirroring_snapshots`).
- Inbound traffic to most public clouds is free; the real limit is your upload bandwidth.

## 2.3 The rbd-mirror daemon pulls

The `rbd-mirror` daemon runs on the **receiving** site. It acts as a **client of the other cluster**: it asks the remote MONs for the cluster map, then reads blocks **directly from the remote OSDs**, and writes them into the local copy.

It uses two identities:

| Identity | Lives in | Used for |
|---|---|---|
| `client.rbd-mirror.<site>` | The **local** cluster | Writing the local replica |
| `client.rbd-mirror-peer` | The **remote** cluster | Reading the remote primary |

Pulling means the receiver decides what to fetch, based on what it already has, and the primary site does no extra work: in a one-way setup it runs no daemon at all, and does not notice if the DR site is off.

While replaying, the daemon holds the **exclusive lock** on the non-primary image (`peer will not release lock` in a kernel log means something else tried to write).

**In practice:**
- The receiving site must reach the remote MONs (TCP 3300, 6789) **and every remote OSD** (6800–7300), since any object can be on any OSD. The remote Ceph *public* network must be routable through the VPN.
- **A DR site that is switched off protects nothing.** Replication resumes when it comes back, but if the primary site is lost in the meantime, the recovery point is the moment the DR site was switched off.
- On Proxmox the daemon runs as user `ceph`, which cannot read `/etc/pve` ([pitfalls](06-pitfalls.md)).

## 2.4 Peers and directions

Mirroring is enabled per pool, and each site has a **peer** entry describing the other site: its name, its mirror UUID, a **direction** and, when needed, credentials.

| Direction | Meaning | Credentials needed |
|---|---|---|
| `rx` | **My** daemon pulls from that peer | Yes: the peer's MON addresses, a cephx user and key |
| `tx` | **That** peer pulls from me. When I am primary, my mirror snapshots are addressed to it too | No: the peer connects to me |
| `rx-tx` | Both. Required on **both** sites for two-way mirroring | Yes |

Peers are created with a **bootstrap token**: one site runs `rbd mirror pool peer bootstrap create`, the other imports it. The token holds the creating cluster's FSID, its MON addresses and the key of `client.rbd-mirror-peer`. After the import these are stored **inside the importing cluster** (its MONs' config-key store); the token file is no longer needed and should be deleted, because it is a secret.

`rbd snap ls --all <image>` shows which peers a mirror snapshot is addressed to: `mirror (primary peer_uuids:[5f1f…])`. That list is the `tx` side in action. A snapshot with an empty list reaches nobody.

**In practice:**
- One-way: daemon only on the DR site; DR peer `rx-only`, primary peer `tx-only` (created automatically).
- Two-way: daemon on both sites, both peers `rx-tx`. If the DR site's peer stayed `rx-only`, its snapshots after a failover would have an empty `peer_uuids`: the primary site would receive nothing, and failing back would lose every write made at the DR site.
- **The site that pulls stores a key that can write to the other site.** By default `client.rbd-mirror-peer` has `osd 'profile rbd'`, read and write on the pool. Whoever gets root on the DR node can extract it and read, or damage, the primary site's disks through the VPN. Mirroring is not an air gap: protect the DR node like production, and keep separate backups (mirroring also copies deletions and ransomware).

## 2.5 Moving the primary role

| Operation | What it does |
|---|---|
| `demote` | Primary → non-primary. First takes a final **demotion snapshot**: "this is everything I had". Right after, **no site is primary** |
| `promote` | Non-primary → primary. Allowed only when the other site has demoted **and** its demotion snapshot has been applied here. Result: one primary, nothing lost. If the sync is not finished, the promote fails: wait and retry |
| `promote --force` | Promotes even though the other site has not demoted (it is down or unreachable). The image restarts from the **last applied snapshot**; later writes are lost |
| split-brain | After a forced promote, when the old site comes back, both copies consider themselves primary, with different histories. Ceph stops mirroring that image until you decide |
| `resync` | On the copy that loses: `demote`, then `resync` discards its content and copies the image again from the primary. Its diverged writes are gone for good |

The four scenarios built from these:

| Scenario | Sequence | Data lost |
|---|---|---|
| Planned failover | stop guest → demote at primary → wait → promote at DR → start | None |
| Disaster failover | promote `--force` at DR → start | Since the last applied snapshot |
| Planned failback | stop guest → demote at DR → wait → promote at primary → start | None |
| Failback after a disaster | old primary returns: keep its HA from starting the guest → demote it → resync it → planned failback | The old primary's writes after the last snapshot (already lost at the disaster) |

**In practice:**
- Stop the guest **before** demoting. A clean shutdown makes the demotion snapshot fully consistent and complete, so the planned RPO is truly zero; a running guest would have later writes refused.
- After a forced promote, the split-brain exists even while the old site is down. If its HA restarts the guest when power returns, there are two live guests writing two histories. Removing the guest from HA at the old site is the first step of recovery.
- The promote can fail. Never start a guest on the line after a promote: chain them, so the start runs only if the promote succeeded and the copy is primary:

```bash
rbd mirror image promote <pool>/<image> \
  && rbd info <pool>/<image> | grep -q 'mirroring primary: true' \
  && pct start <id>
```

---

Next: [3. Architecture](03-architecture.md), which covers network requirements, a single-node DR site (CRUSH failure domain) and guest networking on the DR side.

References: [Ceph: RBD Mirroring](https://docs.ceph.com/en/latest/rbd/rbd-mirroring/) · [Proxmox VE wiki: Ceph RBD Mirroring](https://pve.proxmox.com/wiki/Ceph_RBD_Mirroring)
