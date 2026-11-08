# Off-site disaster recovery for Proxmox VE + Ceph

You run a hyper-converged **Proxmox VE + Ceph** cluster and want a copy of your workloads **somewhere else**: another building, a second small cluster, or a single node in the cloud. If the whole primary site is lost, the guests should start again from that copy, with data loss measured in minutes, not days.

Proxmox has no built-in, GUI-driven answer for this between two Ceph clusters. This guide walks through the options, explains why **Ceph RBD snapshot mirroring** fits best, and documents a **tested** two-way setup between a home cluster and a single Proxmox node in Azure over a site-to-site VPN, including the things that went wrong.

> Status: draft. Everything described has been built and tested; a final review of the text is pending.

## Who this is for

- You already run Proxmox VE with Ceph (RBD) storage
- You want off-site DR, not just backups, and can accept an RPO of a few minutes
- You are comfortable on the command line: most of this is not in the Proxmox GUI

## Contents

| | |
|---|---|
| [1. Options](docs/01-options.md) | Stretched cluster, ZFS replication, Proxmox Backup Server sync, RBD mirroring: what each one gives you and costs |
| [2. Concepts](docs/02-concepts.md) | How RBD snapshot mirroring works: primary and non-primary images, mirror snapshots, peers and directions |
| [3. Architecture](docs/03-architecture.md) | Network requirements, a single-node DR site, guest networking on the DR side |
| [4. Setup](docs/04-setup.md) | Step by step on Proxmox VE 9 / Ceph 20 (Tentacle), one-way then two-way, with a non-root `rbd-mirror` daemon |
| [5. Failover and failback](docs/05-failover-failback.md) | Planned and disaster scenarios, with commands that cannot start a guest on the wrong copy |
| [6. Pitfalls](docs/06-pitfalls.md) | Problems met in practice, with symptoms, root cause, evidence and fix |
| [7. Results](docs/07-results.md) | What was tested and what happened, including a filesystem corruption and how to avoid it |

## Tested with

- Proxmox VE 9.2, Ceph 20.2 (Tentacle)
- Primary: 3-node hyper-converged cluster; DR: 1 Proxmox VE node on an Azure VM (Debian 13)
- Site-to-site IPsec VPN (OPNsense ↔ Azure VPN Gateway)
- LXC containers on RBD (`krbd`)

The lab this guide comes from, with its infrastructure code and an agentic operations layer: [hybrid-dr-lab](https://github.com/JhonnyJGarcia/hybrid-dr-lab).

## References

- Proxmox VE wiki: [Ceph RBD Mirroring](https://pve.proxmox.com/wiki/Ceph_RBD_Mirroring)
- Ceph documentation: [RBD Mirroring](https://docs.ceph.com/en/latest/rbd/rbd-mirroring/)

## License

Text and examples: [CC BY 4.0](LICENSE).
