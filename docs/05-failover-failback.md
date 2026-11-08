# 5. Failover and failback

Four procedures, built from the operations in [2.5](02-concepts.md#25-moving-the-primary-role). Example: container `101`, disk `rbd-dr/vm-101-disk-0`, SDN vnet `drnet` at the DR site.

**The one rule:** never start a guest on a non-primary image. Proxmox will start it and the filesystem will be corrupted ([7. Results](07-results.md#the-incident)). Every start below is chained to a successful promote and a primary check, so it cannot run by itself.

## Preparation

RBD mirroring copies disks, not guest configuration. Keep a copy of each protected guest's config at the DR site, **outside** `/etc/pve/lxc/`: placed there, the guest would appear as startable at the DR site while it still runs at the primary site.

```bash
# DR node
mkdir -p /root/dr-configs
# same content as the primary config (pct config 101), but:
#   net0: ...,bridge=drnet,gw=10.50.2.1,ip=10.50.2.101/24,...
#   no HA entry
vi /root/dr-configs/101.conf
```

Keep it in sync when the guest's config changes (cores, memory, mount points). A script or configuration management can do this; this guide leaves it manual.

## Planned failover (primary site reachable)

Primary site:

```bash
ha-manager set ct:101 --state stopped
ha-manager status | grep ct:101                # repeat until "stopped"
ha-manager remove ct:101                       # so HA does not restart it here
rbd mirror image demote rbd-dr/vm-101-disk-0   # final mirror snapshot; no site is primary now
```

DR node:

```bash
rbd mirror image status rbd-dr/vm-101-disk-0   # wait: local_snapshot_timestamp = remote_snapshot_timestamp

rbd mirror image promote rbd-dr/vm-101-disk-0 \
  && rbd info rbd-dr/vm-101-disk-0 | grep -q 'mirroring primary: true' \
  && cp /root/dr-configs/101.conf /etc/pve/lxc/101.conf \
  && pct start 101 && echo "FAILOVER OK"
```

If the promote fails because the demotion snapshot has not been applied yet, wait and run the same block again. Data lost: none.

## Disaster failover (primary site unreachable)

The primary site cannot demote, so the DR copy is promoted by force. Writes made at the primary site after the last applied snapshot are lost.

```bash
rbd mirror image status rbd-dr/vm-101-disk-0   # local_snapshot_timestamp = your recovery point

# ⚠️ point of no return: both sites now consider themselves primary (split-brain)
rbd mirror image promote --force rbd-dr/vm-101-disk-0 \
  && rbd info rbd-dr/vm-101-disk-0 | grep -q 'mirroring primary: true' \
  && cp /root/dr-configs/101.conf /etc/pve/lxc/101.conf \
  && pct start 101 && echo "FAILOVER OK"
```

When the primary site comes back, its copy still believes it is primary. **First action there:** stop its HA from starting the guest (`ha-manager remove ct:101`, or `ha-manager set ct:101 --state stopped` if it already started). Then follow *Failback after a disaster*.

## Planned failback (DR → primary, keeping DR-site writes)

Requires two-way mirroring. DR node:

```bash
pct shutdown 101 \
  && mv /etc/pve/lxc/101.conf /root/dr-configs/101.conf \
  && rbd mirror image demote rbd-dr/vm-101-disk-0 && echo "DEMOTED"
```

Primary site:

```bash
rbd mirror image status rbd-dr/vm-101-disk-0   # wait: local = remote

rbd mirror image promote rbd-dr/vm-101-disk-0 \
  && rbd info rbd-dr/vm-101-disk-0 | grep -q 'mirroring primary: true' \
  && ha-manager add ct:101 --state started && echo "FAILBACK OK"
```

Data lost: none. Check afterwards: the guest runs at the primary site, the DR site shows `up+replaying`.

## Failback after a disaster

Both copies are primary. The DR copy has the current data; the primary site's copy is the one to discard.

Primary site:

```bash
# ⚠️ point of no return: this copy's writes after the last snapshot that reached the DR site are discarded
rbd mirror image demote rbd-dr/vm-101-disk-0
rbd mirror image resync rbd-dr/vm-101-disk-0
rbd mirror image status rbd-dr/vm-101-disk-0   # wait until up+replaying and caught up
```

Then run the planned failback.

**With one-way mirroring** there is no way to bring DR-site writes back: failback means stopping the guest at the DR site, then `demote` and `resync` **at the DR site**, so the primary site's copy wins, and restarting the guest at the primary site. Everything written at the DR site during the failover is lost.

## After any procedure

```bash
rbd mirror pool status rbd-dr --verbose     # on both sites: health OK, image up+stopped / up+replaying
```

---

Next: [6. Pitfalls](06-pitfalls.md).
