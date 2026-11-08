# 7. Results

What was tested on the reference setup, what happened, and what is still open. Tests ran in November 2026 on Proxmox VE 9.2 / Ceph 20.2, with LXC containers on RBD (`krbd`).

## Replication

| Measure | Value |
|---|---|
| First full sync of a test container (~80 MB written on an 8 GB disk) | 38 s over the VPN (~2.2 MB/s) |
| Later snapshot deltas of an idle container | 4–330 KB each, synced in under a second |
| Schedule | Every 15 minutes, both sites |

## One-way test (disaster-style)

Failover with `promote --force` while the primary site was still alive (a simulated disaster), failback with `demote` + `resync` at the DR site.

| Check | Result |
|---|---|
| Container starts at the DR site with the replicated data | ✅ |
| DR gateway reachable from the container; container reachable from the primary site through the VPN | ✅ |
| DNS, internet egress over TCP | ✅ |
| `ping` to the internet | ❌ expected: cloud default outbound access does not carry ICMP |
| Failback: container back at the primary site, mirroring `up+replaying` again | ✅, DR-site writes discarded (by design in one-way) |

## Two-way test (planned)

1. A marker file written at the primary site.
2. Planned failover: stop, `demote` at the primary site, `promote` at the DR site (no `--force`), start.
3. A second marker file written at the DR site.
4. Planned failback: stop, `demote` at the DR site, `promote` at the primary site (no `--force`), start under HA.

| Check | Result |
|---|---|
| Primary-site marker present at the DR site | ✅ |
| **DR-site marker present at the primary site after failback** | ✅ |
| Filesystem after the round trip | ✅ `rw`, no errors |
| Mirroring after failback | ✅ primary site `up+stopped`, DR site `up+replaying`, both daemons `OK` |
| HA restarted the container on a different primary-site node than before | ✅ (all nodes share the pool) |

| Metric | Value |
|---|---|
| RPO, planned failover and failback | 0: a non-forced promote waits for the demotion snapshot |
| RPO, disaster | Up to the 15-minute schedule plus transfer time |
| RTO | Not timed; the manual steps take a few minutes |

## The incident

The first two-way attempt corrupted the test container's filesystem. It is the most useful result of the whole exercise.

**What was seen after failback:** `/root/dr-marker` (written at the DR site) contained the primary site's `/etc/resolv.conf`. Both names pointed to the same inode with a link count of 1, and the root filesystem was mounted `emergency_ro`.

**Kernel log on the primary-site node** (times in order):

```
23:27:33  I/O error, dev rbd0, sector 74040 op 0x1:(WRITE)
23:27:33  Buffer I/O error on dev rbd0, logical block 9255, lost sync page write
23:27:38  rbd: rbd0: peer will not release lock
23:27:38  rbd: rbd0: write result -30
          ... repeated every 5 seconds ...
23:28:49  EXT4-fs error (device rbd0): ext4_journal_check_start:86: comm cat: Detected aborted journal
23:28:49  EXT4-fs (rbd0): Remounting filesystem read-only
```

and `rbd snap ls --all` at the primary site showed the promotion snapshot at **23:28:16**, after the container had started.

**What happened:**

1. The promote at the primary site failed: the DR site's demotion snapshot had not been applied yet.
2. The next command, `ha-manager add ct:101 --state started`, ran anyway, and HA started the container on a node of the primary site, on a **non-primary** image.
3. Every write from the container was refused: `write result -30` is `EROFS`. `peer will not release lock`: the primary site's `rbd-mirror` daemon held the image's exclusive lock.
4. Meanwhile that daemon kept applying the DR site's changes **under the mounted filesystem**. ext4 held its own view of the metadata in memory; the blocks underneath changed. The DR site's `/root` directory block arrived, pointing the new file at an inode number that, in the primary site's still-cached inode table, belonged to `resolv.conf`.
5. The promote, re-run a minute later, succeeded; ext4 found its journal aborted and went read-only.

**Lessons:**

- Proxmox and `krbd` **do not refuse** to map and start a guest on a non-primary image. The only protection is procedure.
- Promote, check and start are now one chained command ([5](05-failover-failback.md)). If the promote fails, nothing starts.
- The corrupted image was mirrored on to the DR site too, as replication does with anything. Without separate backups, that container would have been lost.

The test container was rebuilt and the two-way test repeated with the chained commands. It passed.

## Open questions

- **DR site switched off for longer than the snapshot retention.** Ceph keeps up to 5 mirror snapshots per image by default; after two days off with a 15-minute schedule, is the catch-up incremental or a full sync? To be measured.
- **Narrower caps for the peer user.** Can `client.rbd-mirror-peer` work with less than `osd 'profile rbd'` on the pulling direction? Untested.
- **High availability of the primary site's daemon.** A second daemon on another node, and a test of the leader handover.
- **Timed RTO** for each procedure.

---

Back to the [README](../README.md).
