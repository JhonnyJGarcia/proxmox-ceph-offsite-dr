# 6. Pitfalls

Problems met while building and testing the setup, with the evidence that pointed to the cause.

## Ceph on a single-node DR site

| Symptom | Cause | Fix |
|---|---|---|
| Placement groups stuck `undersized` on a new pool | The default CRUSH rule needs replicas on different hosts; there is one | A rule with failure domain `osd` ([3.2](03-architecture.md#32-a-single-node-ceph-cluster-at-the-dr-site)) |
| `HEALTH_WARN OSD count 2 < osd_pool_default_size 3`, although every PG is `active+clean` | The check concerns the **default** size for new pools. `pveceph init` writes `osd_pool_default_size = 3` into `/etc/pve/ceph.conf`, and the file overrides `ceph config set` | Edit the file (`2` and `1`), restart MON and MGR |
| Hours after the cluster was healthy: `HEALTH_WARN`, 1 PG `undersized`, `acting [0]` | The `.mgr` pool, created by the manager itself, uses the default rule | `ceph osd pool set .mgr crush_rule replicated_osd` |

## The rbd-mirror daemon on Proxmox

| Symptom | Cause | Fix |
|---|---|---|
| `ceph-rbd-mirror@….service does not exist` | The daemon is a separate package; `pveceph install` does not install it | `apt install rbd-mirror`, same release as the cluster |
| Daemon exits: `unable to find a keyring … /etc/pve/priv/ceph.client.rbd-mirror.<id>.keyring: (13) Permission denied` | Proxmox's `[client]` section points every client at `/etc/pve/priv/`, readable by root only. The daemon runs as `ceph` | Keyring in `/etc/ceph/`, owned by `ceph`, named in the daemon's own config ([4. Setup, step 2](04-setup.md#2-the-rbd-mirror-daemon-dr-site-primary-site-too-for-two-way)) |
| Daemon running, `daemon health: ERROR`, callout `unable to connect to local cluster`; journal: `init_rados: error connecting to local cluster: (2) No such file or directory`, `unable to get monitor info from DNS SRV` | `/etc/ceph/ceph.conf` is a symlink to `/etc/pve/ceph.conf` (`root:www-data`, mode 640). The daemon starts as root, then switches to `ceph` and cannot read it when opening the local cluster connection; with no config it falls back to DNS SRV lookup for MONs | A config readable by `ceph`, passed with `CEPH_CONF` in a systemd drop-in |
| Adding `ceph` to the `www-data` group seems the obvious fix, and does **not** work | The daemon switches user with `--setuser ceph --setgroup ceph`, which sets the uid and primary gid and **drops supplementary groups**. Evidence: `/proc/<pid>/status` shows an empty `Groups:` line | Do not rely on group membership; remove `ceph` from `www-data` again |

The Proxmox wiki solves the same problem by running the daemon as **root** (editing the unit file). That works, but a flaw in the daemon would then give root on a node that holds keys to both clusters. The dedicated config keeps it unprivileged.

## Peers

| Symptom | Cause | Fix |
|---|---|---|
| After importing a token at the primary site, its peer for the DR site is still `tx-only`, with no MON host or client; the primary site's daemon reports `WARNING` | The primary site already had a peer entry for the DR site (created automatically when the DR site imported the primary's token). The import did not update it | `rbd mirror pool peer set` for `client`, `mon-host`, `key-file`, `direction` ([4. Setup, step 6](04-setup.md#6-two-way)) |
| Two-way configured, but after a failover nothing comes back to the primary site | The DR site's peer is still `rx-only`, so its mirror snapshots are not addressed to the primary site (`peer_uuids:[]` in `rbd snap ls --all`) | `direction rx-tx` on **both** sites |

## Failover

| Symptom | Cause | Fix |
|---|---|---|
| Guest filesystem corrupted after a failback: two file names on one inode, root remounted `emergency_ro` | The guest was started while its image was still **non-primary** (the promote had failed). Its writes were refused (`EROFS`) while the local daemon kept writing replicated blocks under the mounted filesystem | Chain promote, primary check and start in one command ([5](05-failover-failback.md)). Full timeline: [7. Results](07-results.md#the-incident) |
| A guest at the DR site appears startable while it runs at the primary site | Its config was copied into `/etc/pve/lxc/` ahead of time | Stage configs outside `/etc/pve` |

## Networking and the cloud

| Symptom | Cause | Fix |
|---|---|---|
| Mirroring stalls or sessions hang on large transfers, while small checks pass | MTU / fragmentation through IPsec | MSS clamping on the tunnel interface |
| Mirroring breaks after giving DR guests the primary site's subnet | A connected route on the DR node captures traffic to the primary site's MONs | Separate subnet for DR guests, routed through the node ([3.1](03-architecture.md#guest-networking-at-the-dr-site)) |
| `ping` to the internet fails from the DR node and its guests; HTTP works | Cloud default outbound access does not carry ICMP for a VM without a public IP | Test egress with TCP |
| SDN zone in *error* on the DR node | The cloud image's `/etc/network/interfaces` lacks `source /etc/network/interfaces.d/*` | Add the line, apply SDN again |

## Security

| Issue | Detail | Mitigation |
|---|---|---|
| The pulling site holds a write-capable key to the other site | `client.rbd-mirror-peer` gets `mon 'profile rbd-mirror-peer' osd 'profile rbd'` (read **and write** on RBD pools). Root on the DR node can extract it and reach the primary site's disks through the VPN | Protect the DR node like production (no public IP, VPN-only access); keep separate backups. Narrower OSD caps for the peer user are untested |
| Bootstrap tokens and key files are secrets | They hold a cephx key | Delete them after import |

---

Next: [7. Results](07-results.md).
