# Design Decisions

Why the lab is built the way it is. Updated as each phase lands.

## Phase 1 - On-prem network

**Flat Layer 2, no VLANs.**
VLAN segmentation, STP tuning and HSRP were already demonstrated in my first project
([enterprise-network-lab](https://github.com/njegosvasic/enterprise-network-lab)). This lab is about
hybrid connectivity and automation, so the on-prem side is kept deliberately small - every extra
moving part would only dilute that focus.

**SW1 does not route (`no ip routing`).**
SW1 is a pure access switch; R1 is the only Layer 3 device in the LAN. The IOL L2 image ships with
routing enabled, which silently made `ip default-gateway` ignored (Issue #3). Disabling it keeps
the role of each device unambiguous.

**R1 WAN address via DHCP.**
The WAN side sits on the VMware NAT network behind EVE-NG Cloud0. DHCP also installs the default
route (AD 254), so any static route added later automatically wins. The lab reaches the internet
through two NAT layers (VMware NAT, then the home router) - relevant later for the IPsec tunnel,
which will need NAT traversal.

**PAT on R1, defined by a named ACL.**
`ACL-NAT-INTERNET` selects LAN traffic for translation; `overload` lets every LAN host share the
single WAN address. ACL remarks use a plain descriptive style ("Allow LAN to Internet ...") kept
consistent across the whole configuration.

**SSH-only management with local accounts.**
VTY lines accept SSH only (`transport input ssh`, `login local`), SSH version 2, 2048-bit RSA keys
and a 15-minute idle timeout. This is what the Netmiko automation in Phase 5 connects to.

**Strongest password hashing each image supports.**
R1 (IOSv 15.9) uses scrypt (Type 9). SW1 (IOL 15.1) does not support scrypt, so its secrets are
Type 4 - a known weakness of that older software train, accepted here because it is a lab image.

**Unused ports shut down.**
R1 Gi0/2-0/3 and SW1 Et0/3 are administratively down. Host ports use PortFast and BPDU Guard.

**Legacy SSH algorithms allowed only for the lab subnet.**
The older Cisco images need SHA1 key exchange, `ssh-rsa` and CBC ciphers. Instead of weakening
the Ubuntu host globally, `~/.ssh/config` re-enables them only for `192.168.10.*` (Issue #5).

**Automation host disk grown on the node overlay only.**
The node's own `virtioa.qcow2` overlay was resized; the shared base image stays at its original
size for other labs (Issue #4).
