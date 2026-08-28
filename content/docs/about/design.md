---
title: Design
description: Farrow's simplified one-deployment architecture, networking, state, safety boundaries, and review conclusion.
weight: 10
icon: fa-solid fa-compass-drafting
aliases: [/docs/concepts/, /docs/concepts/networking/, /docs/concepts/storage/, /docs/concepts/safety/, /docs/architecture/, /docs/architecture/overview/, /docs/architecture/networking/, /docs/architecture/security/]
---

## One useful abstraction

Farrow boots one Pigsty Inventory as one local QEMU deployment. It deliberately
has no project marker, project registry, lease model, provider layer, or
second configuration format.

State lives under `~/.farrow` for one Unix user. The product assumes one active
Pigsty deployment per computer; this is not a root-enforced cross-user
singleton.

## Node-level convergence

Farrow extracts only the documented VM and Pigsty-native fields, computes
per-node hashes, and keeps applied state plus process identity. Additions are
incremental. Changes require an
explicit per-node recreate. `up` also starts selected existing stopped nodes;
already-running peers remain untouched. Absence never authorizes deletion.

## Runtime selection

Guest architecture is deployment-wide desired state. Omitted/`native` follows
the host; explicit `amd64` or `arm64` selects that Catalog artifact exactly.
Native HVF/KVM remains the default. A foreign architecture or one catalogued
image/host incompatibility selects a fixed TCG profile; there is no user
accelerator argument and no arbitrary failure fallback.

The effective architecture and accelerator are persisted in each QEMU
invocation and exposed by `status`. Before destructive recreate, Farrow proves
the selected QEMU binary and version, network backend, image bytes, boot mode,
and firmware. A later binary changing runtime policy cannot mix new nodes with
old invocations: runtime drift requires whole-deployment recreation.

## Two NICs, one fixed subnet

The management NIC supplies DHCP, DNS, egress, and loopback SSH. The fixed-IP
NIC supplies host/peer/Ansible traffic. macOS uses socket_vmnet. Linux follows
active NetworkManager; otherwise it uses systemd-networkd and connects through
the distribution bridge helper. Inactive networkd is started only after an
activation-safety scan proves existing units cannot claim a real host link.

On Debian, the helper is temporarily and reversibly scoped to a group the
caller actually belongs to. A real unprivileged QEMU bridge smoke must pass
before setup accepts the network; failure triggers manifest-bounded rollback.

## Safety boundary

QEMU and all guest artifacts run as the caller. Root is limited to host
network setup and the optional hosts publisher. Destruction requires matching
ownership, containment, node identity, QMP/process identity, and an allowlist
of artifacts. Ambiguity stops the operation.

## Review conclusion

The pivot away from projects and leases is correct for a one-lab product and
greatly reduces cognitive and state complexity. Native review was still
essential: the first implementation had cross-layer gaps in control-node
keys, sudo policy, Debian helper access, NetworkManager verification, runtime
preflight, result messages, and failed-install cleanup. Those paths were fixed
and replayed on macOS and Linux before this documentation was simplified. A
later adversarial review also caught destructive preflight ordering and signed
Catalog-baseline upgrade bugs before EL7/EL8 compatibility was committed.
