# 4. Setup

Step by step, tested on Proxmox VE 9.2 with Ceph 20.2 (Tentacle). The order: DR-site Ceph, then the daemons, then peering, then images. Steps 1–5 give a one-way setup (primary → DR); step 6 makes it two-way.

Conventions: `<site>` is `primary` or `dr`; the pool is `rbd-dr` on both sites; commands run as root on the node named in each step.

## 0. Before you start

- [ ] Both clusters run the **same Ceph release** (`ceph versions`), and the `rbd-mirror` package candidate matches it (`apt policy rbd-mirror`)
- [ ] Each site reaches the other site's MONs and OSDs through the VPN ([3.1](03-architecture.md#31-network-requirements))
- [ ] Guest disks to protect are on an RBD pool with the features `exclusive-lock`, `object-map`, `fast-diff` (Proxmox's defaults)

## 1. DR site: single-node Ceph

After `pveceph install`, `pveceph init` and one MON/MGR, with two OSDs created:

```bash
# Default pool size for this cluster: one node, two disks
sed -i 's/osd_pool_default_size = 3/osd_pool_default_size = 2/; s/osd_pool_default_min_size = 2/osd_pool_default_min_size = 1/' /etc/pve/ceph.conf
systemctl restart ceph-mon@$(hostname) ceph-mgr@$(hostname)

# Replicas on different disks, not different hosts
ceph osd crush rule create-replicated replicated_osd default osd
pveceph pool create rbd-dr --size 2 --min_size 1 --crush_rule replicated_osd \
  --pg_autoscale_mode on --add_storages
ceph osd pool set .mgr crush_rule replicated_osd

ceph config set osd osd_memory_target 2147483648
ceph -s     # HEALTH_OK, all PGs active+clean
```

Why each line: [3.2](03-architecture.md#32-a-single-node-ceph-cluster-at-the-dr-site).

## 2. The rbd-mirror daemon (DR site; primary site too for two-way)

The daemon runs as user `ceph`, which cannot read Proxmox's `/etc/pve/ceph.conf` nor its keyrings in `/etc/pve/priv/`. Instead of running it as root (as the Proxmox wiki does), give it its own readable config. Background: [pitfalls](06-pitfalls.md).

```bash
SITE=dr        # or primary
FSID=$(ceph fsid)
MONS=$(awk -F' = ' '/mon_host/ {print $2}' /etc/pve/ceph.conf)

apt install -y rbd-mirror

ceph auth get-or-create client.rbd-mirror.$SITE mon 'profile rbd-mirror' osd 'profile rbd' \
  -o /etc/ceph/ceph.client.rbd-mirror.$SITE.keyring
chown ceph:ceph /etc/ceph/ceph.client.rbd-mirror.$SITE.keyring

cat > /etc/ceph/rbd-mirror.conf <<EOF
[global]
	fsid = $FSID
	mon_host = $MONS

[client.rbd-mirror.$SITE]
	keyring = /etc/ceph/ceph.client.rbd-mirror.$SITE.keyring
EOF
chown ceph:ceph /etc/ceph/rbd-mirror.conf && chmod 640 /etc/ceph/rbd-mirror.conf

mkdir -p /etc/systemd/system/ceph-rbd-mirror@.service.d
cat > /etc/systemd/system/ceph-rbd-mirror@.service.d/conf.conf <<'EOF'
[Service]
Environment=CEPH_CONF=/etc/ceph/rbd-mirror.conf
EOF
systemctl daemon-reload
systemctl enable --now ceph-rbd-mirror@rbd-mirror.$SITE
```

Check that the daemon can reach its own cluster, as the user it runs as:

```bash
sudo -u ceph env CEPH_CONF=/etc/ceph/rbd-mirror.conf rbd --id rbd-mirror.$SITE ls rbd-dr && echo LOCAL_OK
```

For the primary site, check that `mon_host` lists all MONs. On a multi-node cluster the config file is local to the node running the daemon.

## 3. Enable mirroring on the pool (both sites)

```bash
rbd mirror pool enable rbd-dr image
```

`image` mode: mirroring is switched on per image, which snapshot mirroring requires.

## 4. Peer the sites

Primary site, create a token:

```bash
rbd mirror pool peer bootstrap create --site-name primary rbd-dr > /root/token-primary
```

Copy it to the DR node (it is a single base64 line), then:

```bash
# one-way:
rbd mirror pool peer bootstrap import --site-name dr --direction rx-only rbd-dr /root/token-primary
# two-way: use --direction rx-tx and continue with step 6
rm /root/token-primary          # on both nodes: it contains a key
```

Check on both sites:

```bash
rbd mirror pool info rbd-dr --all
```

The DR site lists peer `primary` with a client and MON host; the primary site lists peer `dr` (added automatically, `tx-only` for one-way).

## 5. Schedule and enable images

Mirror snapshot schedule, on the site where images are primary (both sites for two-way):

```bash
rbd mirror snapshot schedule add --pool rbd-dr 15m
rbd mirror snapshot schedule ls --pool rbd-dr
```

Per disk to protect, at the primary site:

```bash
rbd mirror image enable rbd-dr/vm-101-disk-0 snapshot
```

At the DR site, after the first sync:

```bash
rbd mirror pool status rbd-dr --verbose
# health: OK · vm-101-disk-0: up+replaying · local_snapshot_timestamp = remote_snapshot_timestamp
```

Then stage each protected guest's config at the DR site ([5. Failover](05-failover-failback.md#preparation)).

**One-way setup complete.** Failback then discards writes made at the DR site; for failback without loss, continue.

## 6. Two-way

1. Run step 2 at the primary site (`SITE=primary`), on one node.
2. Add the snapshot schedule at the DR site (step 5).
3. Make **both** peers `rx-tx`, and give the primary site's peer the credentials to reach the DR cluster. The bootstrap import does not reliably update a peer entry that already exists, so set it explicitly.

DR node:

```bash
rbd mirror pool peer bootstrap create --site-name dr rbd-dr > /dev/null   # ensures client.rbd-mirror-peer exists
ceph auth get-key client.rbd-mirror-peer; echo                            # copy this key
rbd mirror pool info rbd-dr                                                # note the UUID of peer "primary"
rbd mirror pool peer set rbd-dr <uuid-of-primary> direction rx-tx
```

Primary node:

```bash
rbd mirror pool info rbd-dr                       # note the UUID of peer "dr"
P=<uuid-of-dr>
echo -n '<key>' > /root/dr-peer.key && chmod 600 /root/dr-peer.key
rbd mirror pool peer set rbd-dr $P client client.rbd-mirror-peer
rbd mirror pool peer set rbd-dr $P mon-host <dr-mon-ip>
rbd mirror pool peer set rbd-dr $P key-file /root/dr-peer.key
rbd mirror pool peer set rbd-dr $P direction rx-tx
rm /root/dr-peer.key
systemctl restart ceph-rbd-mirror@rbd-mirror.primary
```

Check on both sites: `rbd mirror pool info rbd-dr --all` shows the peer as `rx-tx` with client and MON host; `rbd mirror pool status rbd-dr --verbose` shows daemon health `OK`. At the primary site the image is `up+stopped`, *local image is primary*: the primary site's daemon is idle until a failover. That is correct.

---

Next: [5. Failover and failback](05-failover-failback.md).
