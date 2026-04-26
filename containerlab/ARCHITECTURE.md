# Stretched Layer 2 over BGP EVPN with OpenStack

## Goal

Provide a stretched Layer 2 domain across two compute nodes using a BGP EVPN underlay
built with FRR. OpenStack (OVN) handles the overlay; the spine-leaf fabric handles the
underlay. VMs on separate compute nodes appear on the same L2 segment regardless of which
leaf they are connected to.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                            Linux Server (bare metal)                         │
│                                                                              │
│  ┌─────────────────────────── containerlab ──────────────────────────────┐  │
│  │                                                                        │  │
│  │        ┌─────────────┐              ┌─────────────┐                   │  │
│  │        │   spine1    │              │   spine2    │                   │  │
│  │        │  AS 65100   │              │  AS 65101   │                   │  │
│  │        │ 10.255.0.1  │              │ 10.255.0.2  │                   │  │
│  │        └──────┬──────┘              └──────┬──────┘                   │  │
│  │         eBGP  │ eBGP EVPN            eBGP  │ eBGP EVPN                │  │
│  │        ┌──────┴──────────────────────┴──────┐                         │  │
│  │        │                                    │                         │  │
│  │  ┌─────┴───────┐                    ┌───────┴─────┐                   │  │
│  │  │    leaf1    │                    │    leaf2    │                   │  │
│  │  │   AS 65001  │                    │   AS 65002  │                   │  │
│  │  │ 10.255.0.11 │                    │ 10.255.0.12 │                   │  │
│  │  └─────┬───────┘                    └───────┬─────┘                   │  │
│  │        │ eth3                          eth3 │                         │  │
│  └────────┼──────────────────────────────────  ┼─────────────────────────┘  │
│           │ veth-compute1        veth-compute2 │                            │
│    ┌──────┴──────┐                    ┌────────┴────┐                       │
│    │ br-compute1 │                    │ br-compute2 │  (Linux bridges)      │
│    └──────┬──────┘                    └────────┬────┘                       │
│           │                                    │                            │
│  ┌────────┴────────┐                  ┌────────┴────────┐                   │
│  │   compute1 VM   │                  │   compute2 VM   │  (KVM)            │
│  │   AS 65201      │                  │   AS 65202      │                   │
│  │   10.0.1.1/31   │                  │   10.0.1.3/31   │                   │
│  │                 │                  │                 │                   │
│  │  OVN + OVS      │                  │  OVN + OVS      │                   │
│  │  OVN BGP Agent  │                  │  OVN BGP Agent  │                   │
│  │                 │                  │                 │                   │
│  │  ┌───────────┐  │  VXLAN (VNI 100) │  ┌───────────┐  │                   │
│  │  │   VM A    ├──┼──────────────────┼──┤   VM B    │  │                   │
│  │  │10.100.0.10│  │  stretched L2    │  │10.100.0.11│  │                   │
│  │  └───────────┘  │                  │  └───────────┘  │                   │
│  └─────────────────┘                  └─────────────────┘                   │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Traffic flow for VM A → VM B

1. VM A sends an Ethernet frame destined for VM B's MAC
2. OVS on compute1 looks up the MAC in the OVN Southbound DB
3. OVN BGP Agent has already populated the VTEP IP for VM B (10.0.1.3) via EVPN type-2 routes
4. OVS encapsulates the frame in VXLAN (VNI 100) and sends it to 10.0.1.3
5. Underlay routing: packet leaves compute1 → leaf1 → spine → leaf2 → compute2
6. OVS on compute2 decapsulates and delivers to VM B

---

## IP and AS Plan

### Underlay point-to-point links

| Link              | Subnet       | Side A IP  | Side B IP  |
|-------------------|--------------|------------|------------|
| leaf1 ↔ spine1    | 10.0.0.0/31  | 10.0.0.1   | 10.0.0.0   |
| leaf1 ↔ spine2    | 10.0.0.4/31  | 10.0.0.5   | 10.0.0.4   |
| leaf2 ↔ spine1    | 10.0.0.2/31  | 10.0.0.3   | 10.0.0.2   |
| leaf2 ↔ spine2    | 10.0.0.6/31  | 10.0.0.7   | 10.0.0.6   |
| leaf1 ↔ compute1  | 10.0.1.0/31  | 10.0.1.0   | 10.0.1.1   |
| leaf2 ↔ compute2  | 10.0.1.2/31  | 10.0.1.2   | 10.0.1.3   |

### Loopbacks (router IDs / VTEP IPs)

| Node     | Loopback      |
|----------|---------------|
| spine1   | 10.255.0.1/32 |
| spine2   | 10.255.0.2/32 |
| leaf1    | 10.255.0.11/32|
| leaf2    | 10.255.0.12/32|

### BGP AS numbers

| Node      | AS    | Role                        |
|-----------|-------|-----------------------------|
| spine1    | 65100 | EVPN transit, next-hop-unchanged |
| spine2    | 65101 | EVPN transit, next-hop-unchanged |
| leaf1     | 65001 | VTEP, EVPN originator       |
| leaf2     | 65002 | VTEP, EVPN originator       |
| compute1  | 65201 | VTEP, OVN BGP Agent peer    |
| compute2  | 65202 | VTEP, OVN BGP Agent peer    |

### Overlay

| Network         | Subnet         | VNI |
|-----------------|----------------|-----|
| stretched-l2    | 10.100.0.0/24  | 100 |

---

## Setup Steps

### Prerequisites

- Linux server with KVM/QEMU and libvirt installed
- Docker installed (for containerlab)
- containerlab installed: `bash -c "$(curl -sL https://get.containerlab.tools)"`
- Two Ubuntu 22.04 KVM VMs created (compute1, compute2) — do not boot them yet

---

### Step 1 — Deploy the containerlab BGP underlay

```bash
cd containerlab
sudo clab deploy -t bgp-underlay.clab.yaml
```

Verify BGP sessions are up on all nodes:

```bash
sudo docker exec -it clab-bgp-underlay-leaf1 vtysh -c "show bgp summary"
sudo docker exec -it clab-bgp-underlay-leaf2 vtysh -c "show bgp summary"
```

All spine neighbors should show `Estab` state.

---

### Step 2 — Verify EVPN is running on the fabric

```bash
sudo docker exec -it clab-bgp-underlay-leaf1 vtysh -c "show bgp l2vpn evpn summary"
sudo docker exec -it clab-bgp-underlay-spine1 vtysh -c "show bgp l2vpn evpn summary"
```

Spines should show sessions to both leaves. No EVPN routes yet — those come once
compute nodes are up and VNIs are configured.

---

### Step 3 — Connect compute VMs to the fabric

After `clab deploy`, two host interfaces exist: `veth-compute1` and `veth-compute2`.
Create Linux bridges and attach the veths to them:

```bash
# compute1 side (connects to leaf1)
ip link add br-compute1 type bridge
ip link set veth-compute1 master br-compute1
ip link set veth-compute1 up
ip link set br-compute1 up

# compute2 side (connects to leaf2)
ip link add br-compute2 type bridge
ip link set veth-compute2 master br-compute2
ip link set veth-compute2 up
ip link set br-compute2 up
```

Edit each VM's libvirt XML to attach a NIC to the corresponding bridge:

```xml
<!-- compute1: attach to br-compute1 -->
<interface type='bridge'>
  <source bridge='br-compute1'/>
  <model type='virtio'/>
</interface>

<!-- compute2: attach to br-compute2 -->
<interface type='bridge'>
  <source bridge='br-compute2'/>
  <model type='virtio'/>
</interface>
```

Inside each VM, configure the underlay interface:

```bash
# compute1
ip addr add 10.0.1.1/31 dev eth1
ip link set eth1 up

# compute2
ip addr add 10.0.1.3/31 dev eth1
ip link set eth1 up
```

Verify reachability to the leaf:

```bash
# From compute1
ping 10.0.1.0   # should reach leaf1 eth3

# From compute2
ping 10.0.1.2   # should reach leaf2 eth3
```

---

### Step 4 — Install OpenStack (DevStack) on compute1 (controller + compute)

On compute1:

```bash
git clone https://opendev.org/openstack/devstack
cd devstack
cp samples/local.conf local.conf
```

Edit `local.conf`:

```ini
[[local|localrc]]
HOST_IP=10.0.1.1
ADMIN_PASSWORD=secret
DATABASE_PASSWORD=secret
RABBIT_PASSWORD=secret
SERVICE_PASSWORD=secret

# Use OVN as the Neutron backend
Q_AGENT=ovn
Q_ML2_PLUGIN_MECHANISM_DRIVERS=ovn
Q_ML2_TENANT_NETWORK_TYPE=vxlan
OVN_BUILD_FROM_SOURCE=False

# Enable the BGPVPN and OVN BGP agent plugins
enable_plugin networking-bgpvpn https://opendev.org/openstack/networking-bgpvpn
```

Run the install:

```bash
./stack.sh
```

This installs: Keystone, Neutron (OVN), Nova, OVS, and the OVN control plane.

On compute2, install only the compute role. Create a `local.conf` pointing at compute1
as the controller:

```ini
[[local|localrc]]
HOST_IP=10.0.1.3
SERVICE_HOST=10.0.1.1
MYSQL_HOST=10.0.1.1
RABBIT_HOST=10.0.1.1
GLANCE_HOSTPORT=10.0.1.1:9292

ENABLED_SERVICES=n-cpu,q-ovn-metadata-agent,ovn-controller
Q_AGENT=ovn
```

---

### Step 5 — Install and configure OVN BGP Agent

On **both** compute nodes:

```bash
pip install ovn-bgp-agent
```

Create `/etc/ovn-bgp-agent/bgp-agent.conf`:

```ini
# compute1
[DEFAULT]
debug = False
driver = ovn_bgp_driver
bgp_as = 65201            # 65202 on compute2
bgp_router_id = 10.0.1.1  # 10.0.1.3 on compute2

[OVN]
ovn_sb_connection = tcp:10.0.1.1:6642
```

Configure FRR on each compute node to peer with its leaf. Install FRR on the VM:

```bash
apt install frr
```

Add to `/etc/frr/frr.conf` on compute1:

```
router bgp 65201
 bgp router-id 10.0.1.1
 no bgp ebgp-requires-policy
 neighbor 10.0.1.0 remote-as 65001
 neighbor 10.0.1.0 description leaf1
 !
 address-family ipv4 unicast
  neighbor 10.0.1.0 activate
 exit-address-family
 !
 address-family l2vpn evpn
  neighbor 10.0.1.0 activate
  advertise-all-vni
 exit-address-family
!
```

Add to `/etc/frr/frr.conf` on compute2 (mirror with AS 65202, peer 10.0.1.2 / leaf2 AS 65002):

```
router bgp 65202
 bgp router-id 10.0.1.3
 no bgp ebgp-requires-policy
 neighbor 10.0.1.2 remote-as 65002
 neighbor 10.0.1.2 description leaf2
 !
 address-family ipv4 unicast
  neighbor 10.0.1.2 activate
 exit-address-family
 !
 address-family l2vpn evpn
  neighbor 10.0.1.2 activate
  advertise-all-vni
 exit-address-family
!
```

Start both services on each compute node:

```bash
systemctl enable --now frr
systemctl enable --now ovn-bgp-agent
```

---

### Step 6 — Create the stretched Neutron network

From the controller (compute1), source the OpenStack credentials and create the network:

```bash
source devstack/openrc admin admin

openstack network create stretched-l2 \
  --provider-network-type vxlan \
  --provider-segment 100

openstack subnet create stretched-subnet \
  --network stretched-l2 \
  --subnet-range 10.100.0.0/24 \
  --gateway 10.100.0.1
```

Boot one VM on each compute node:

```bash
openstack server create vm-a \
  --image cirros \
  --flavor m1.tiny \
  --network stretched-l2 \
  --availability-zone nova:compute1

openstack server create vm-b \
  --image cirros \
  --flavor m1.tiny \
  --network stretched-l2 \
  --availability-zone nova:compute2
```

---

### Step 7 — Verify end-to-end

**Check EVPN routes are being distributed across the fabric:**

```bash
# On leaf1 — should see MAC/IP routes (type-2) for VM A and VM B
sudo docker exec -it clab-bgp-underlay-leaf1 vtysh -c "show bgp l2vpn evpn route"

# On spine1 — should be passing routes between both leaves
sudo docker exec -it clab-bgp-underlay-spine1 vtysh -c "show bgp l2vpn evpn route"
```

**Check OVN BGP Agent is advertising VM MACs:**

```bash
# On compute1
journalctl -u ovn-bgp-agent -f
```

**Ping between VMs:**

```bash
# Console into vm-a and ping vm-b's IP
openstack console log show vm-a

# From vm-a's console:
ping 10.100.0.11
```

A successful ping confirms:
- EVPN type-2 routes are distributing VM MACs across the fabric
- VXLAN encapsulation is working between OVS instances
- The BGP underlay is routing VTEP-to-VTEP traffic correctly
