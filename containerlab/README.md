# BGP Underlay Lab — Containerlab

A 4-node spine-leaf fabric using FRRouting (FRR) containers, demonstrating an eBGP underlay with BFD.

## Topology

```
        spine1 (AS 65100)       spine2 (AS 65101)
        10.255.0.1/32           10.255.0.2/32
           |   \                   /   |
     eth1  |    \ eth2       eth1 /    | eth2
           |     \               /     |
        leaf1 (AS 65001)       leaf2 (AS 65002)
        10.255.0.11/32         10.255.0.12/32
```

| Link         | Subnet       | spine/leaf IP  | leaf IP        |
|--------------|--------------|----------------|----------------|
| spine1-leaf1 | 10.0.0.0/31  | 10.0.0.0       | 10.0.0.1       |
| spine1-leaf2 | 10.0.0.2/31  | 10.0.0.2       | 10.0.0.3       |
| spine2-leaf1 | 10.0.0.4/31  | 10.0.0.4       | 10.0.0.5       |
| spine2-leaf2 | 10.0.0.6/31  | 10.0.0.6       | 10.0.0.7       |

Each node advertises its loopback (/32) via BGP. Leaves use ECMP (`maximum-paths 2`) across both spines.

---

## Prerequisites

- Linux host (bare metal or WSL2)
- Docker installed and running
- `sudo` access

---

## Install Containerlab

### Option 1 — install script (recommended)

```bash
bash -c "$(curl -sL https://get.containerlab.dev)"
```

### Option 2 — package manager

**Debian / Ubuntu:**
```bash
echo "deb [trusted=yes] https://apt.fury.io/netdevops/ /" \
  | sudo tee /etc/apt/sources.list.d/netdevops.list
sudo apt update && sudo apt install containerlab
```

**RHEL / CentOS / Fedora:**
```bash
sudo yum-config-manager --add-repo https://yum.fury.io/netdevops/
sudo yum install containerlab
```

### Verify installation

```bash
containerlab version
```

---

## Deploy the Lab

1. **Clone the repository** (if you haven't already):

```bash
git clone <repo-url>
cd containerlab
```

2. **Pull the FRR image:**

```bash
docker pull frrouting/frr:latest
```

3. **Deploy the topology:**

```bash
sudo containerlab deploy -t bgp-underlay.clab.yaml
```

Containerlab will create the four containers, wire up the links, and bind-mount the FRR configs automatically.

---

## Verify BGP Sessions

Connect to any node and check BGP:

```bash
# Connect to leaf1
sudo docker exec -it clab-bgp-underlay-leaf1 vtysh

# Inside vtysh
show bgp summary
show ip bgp
show ip route
```

Expected: both spines show as `Established` neighbours on each leaf, and each node's loopback prefix is visible in the routing table.

---

## Destroy the Lab

```bash
sudo containerlab destroy -t bgp-underlay.clab.yaml
```

Add `--cleanup` to also remove the generated `clab-bgp-underlay/` directory:

```bash
sudo containerlab destroy -t bgp-underlay.clab.yaml --cleanup
```

---

## Repository Structure

```
.
├── bgp-underlay.clab.yaml      # Containerlab topology definition
├── configs/
│   ├── spine1/
│   │   ├── frr.conf            # FRR config for spine1
│   │   └── daemons             # FRR daemon enable flags
│   ├── spine2/
│   ├── leaf1/
│   └── leaf2/
└── README.md
```
