# MPLS Network Automation with Ansible — Juniper SRX240

Automated deployment and verification of a multi-area OSPF + MPLS/LDP network across 6 Juniper SRX240 routers using Ansible, Jinja2 templates, and NETCONF.

---

## Topology

```
              ┌─────────────────────────────────────────────────┐
              │              MPLS Domain  (OSPF Area 0)         │
              │                                                   │
Area 1        │      R3 (1.1.1.3)──────R4 (1.1.1.4)            │      Area 2
              │     /    7.1.1.0/30    /    \                    │
R1 ──8.1.1.0/30── R2 (1.1.1.2)      /      9.1.1.0/30 ── R6   │
(1.1.1.1)    │     \                /                  (1.1.1.6)│
              │      R5 (1.1.1.5)──/                             │
              │          7.1.1.8/30  7.1.1.12/30                │
              └─────────────────────────────────────────────────┘
```

| Router | Management IP  | Loopback    | Role               | OSPF Areas |
|--------|---------------|-------------|--------------------|-----------:|
| R1     | 192.168.1.29  | 1.1.1.1/32  | Edge (Area 1)      | Area 1     |
| R2     | 192.168.1.30  | 1.1.1.2/32  | Core / ABR         | Area 0 + 1 |
| R3     | 192.168.1.31  | 1.1.1.3/32  | Core               | Area 0     |
| R4     | 192.168.1.32  | 1.1.1.4/32  | Core / ABR         | Area 0 + 2 |
| R5     | 192.168.1.33  | 1.1.1.5/32  | Core               | Area 0     |
| R6     | 192.168.1.34  | 1.1.1.6/32  | Edge (Area 2)      | Area 2     |

MPLS is enabled on R2–R5 (the Area 0 core). R1 and R6 are pure IP edge routers.

---

## What gets automated

| Task | Template / Module | Applied to |
|------|-------------------|-----------|
| Interface IP assignment | `templates/interfaces.conf` | All 6 routers |
| OSPF multi-area config + security zones | `templates/ospf_template.conf` | All 6 routers |
| MPLS family + LDP on core interfaces | `templates/mpls_template.conf` | R2–R5 |
| Routing table collection | `juniper_junos_rpc` | All 6 routers |
| LDP neighbor verification | `juniper_junos_rpc` | R2–R5 |
| inet.3 (LSP) route collection | `juniper_junos_rpc` | R2–R5 |
| MPLS forwarding table collection | `juniper_junos_rpc` | R2–R5 |
| End-to-end ping (Area 1 → Area 2) | `juniper_junos_rpc` | R1 → R6 |

---

## Project structure

```
mpls-ansible-junos/
├── project01.yaml              # Main Ansible playbook (3 plays)
├── inventory.yaml              # Router inventory with interface definitions
├── templates/
│   ├── interfaces.conf         # Jinja2 — interface IP configuration
│   ├── ospf_template.conf      # Jinja2 — OSPF areas + security zones
│   └── mpls_template.conf      # Jinja2 — MPLS family + LDP
├── verification_outputs/       # Real device output captured after deployment
│   ├── *-routing-information.txt   # Full routing tables (all routers)
│   ├── *-ldp-neighbors.txt         # LDP adjacency state (core routers)
│   ├── *-inet3-routes.txt          # LSP routes via LDP (core routers)
│   ├── *-mpls-routes.txt           # MPLS forwarding table (core routers)
│   └── ansible-playbook-output.txt # Full Ansible run output incl. ping result
└── docs/
    └── project-report.pdf      # Full project report with design diagrams
```

---

## Prerequisites

- Ansible with the `Juniper.junos` role installed:
  ```bash
  ansible-galaxy install Juniper.junos
  ```
- Python 3 with `junos-eznc` and `jxmlease`:
  ```bash
  pip install junos-eznc jxmlease
  ```
- NETCONF enabled on all Juniper devices:
  ```
  set system services netconf ssh
  ```

---

## Setup

**1. Clone the repo**
```bash
git clone https://github.com/ascjreddy/mpls-ansible-junos.git
cd mpls-ansible-junos
```

**2. Set credentials via environment variables** (never hardcode passwords)
```bash
export JUNOS_USER=labuser
export JUNOS_PASSWORD=your_password_here
```

**3. Update `inventory.yaml`** with your devices' management IP addresses.

**4. Run the playbook**
```bash
ansible-playbook project01.yaml -i inventory.yaml
```

---

## How it works

### Play 1 — OSPF and interfaces (all routers)
Ansible loops through each router's `interfaces` list in `inventory.yaml`, renders the Jinja2 templates with per-router variables, and pushes config via NETCONF using `juniper_junos_config`. OSPF is configured with loopback interfaces set as passive (required for LDP loopback reachability).

### Play 2 — MPLS + LDP (core routers R2–R5 only)
`mpls_template.conf` enables `family mpls` on `ge-0/0/1` and `ge-0/0/2`, and enables LDP on those interfaces plus the loopback. The template uses Jinja2 conditionals so only the correct interfaces get MPLS/LDP — no hardcoding.

### Play 3 — Connectivity test
R1 pings `9.1.1.2` (R6's interface in Area 2). Traffic traverses the MPLS backbone via label-switched paths established by LDP. A successful ping with TTL=61 confirms end-to-end forwarding across the full topology.

---

## Verification results

All verification output files are in `verification_outputs/`. Key things to look for:

**LDP neighbors formed correctly (R2 example):**
```
Address            Interface          Label space ID         Hold time
7.1.1.2            ge-0/0/1.0         1.1.1.3:0                14
7.1.1.13           ge-0/0/2.0         1.1.1.5:0                12
```

**LSPs visible in inet.3 (R2 example):**
```
1.1.1.3/32   *[LDP/9]  > to 7.1.1.2 via ge-0/0/1.0
1.1.1.4/32   *[LDP/9]  > to 7.1.1.13 via ge-0/0/2.0, Push 299920
1.1.1.5/32   *[LDP/9]  > to 7.1.1.13 via ge-0/0/2.0
```

**End-to-end ping (R1 → R6, 0% packet loss):**
```
probes-sent: 1  |  responses-received: 1  |  packet-loss: 0  |  rtt: 5777µs
```

---

## Skills demonstrated

- Network automation with **Ansible** and **Jinja2** templates
- **NETCONF** configuration push to Juniper JunOS devices
- **OSPF** multi-area design (Areas 0, 1, 2) with ABRs
- **MPLS** data plane + **LDP** control plane configuration
- Structured verification: routing tables, MPLS forwarding tables, LDP adjacency state
- Secure credential handling via environment variables

---

## License

MIT
