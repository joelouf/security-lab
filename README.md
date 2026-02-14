<p id="top">
  <img src="./docs/assets/images/security_lab_banner.svg" alt="Security Lab Banner" width="100%">
</p>

<p>A comprehensive cybersecurity environment built on Proxmox VE to simulate, detect, and respond to real-world security threats. It features VLAN-segmented networks controlled by a virtualized pfSense firewall, a distributed Wazuh SIEM stack, dual-layer network intrusion detection via Suricata and Zeek, and secure remote access through WireGuard and Apache Guacamole.</p>

<p>This lab is built to evolve and supports the addition of new capabilities, making it a long-term platform for hands-on security engineering.</p>

<details open>
  <summary style="overflow: hidden; cursor: pointer;">
    <h2 style="display: inline; float: right; width: calc(100% - 25px); margin: 0; position: relative; top: -0.085em;" id="table-of-contents">Table of Contents</h2>
  </summary>

  <nav style="clear: both; margin-top: 1rem">
    <ol type="I">
      <li><a href="#overview">Overview</a></li>
      <li><a href="#architecture">Architecture</a></li>
        <ol type="A">
          <li><a href="#network-topology">Network Topology</a></li>
          <li><a href="#hardware">Hardware</a></li>
          <li><a href="#network-segmentation">Network Segmentation</a></li>
          <li><a href="#routing--traffic">Routing & Traffic</a></li>
        </ol>
      <li><a href="#tools-technologies">Tools & Technologies</a></li>
      <li><a href="#environment">Environment</a>
        <ol type="A">
          <li><a href="#hypervisor--virtualization">Hypervisor & Virtualization</a></li>
          <li><a href="#firewall--network-security">Firewall & Network Security</a></li>
          <li><a href="#siem--log-management">SIEM & Log Management</a></li>
          <li><a href="#network-intrusion-detection-nids">Network Intrusion Detection (NIDS)</a></li>
          <li><a href="#remote-access">Remote Access</a></li>
          <li><a href="#offensive-security">Offensive Security</a></li>
          <li><a href="#active-directory-environment">Active Directory Environment</a></li>
          <li><a href="#vulnerable-targets">Vulnerable Targets</a></li>
          <li><a href="#backup--storage">Backup & Storage</a></li>
        </ol>
      </li>
      <li><a href="#roadmap">Roadmap</a></li>
      <li><a href="#references">References</a></li>
      <li><a href="#contact">Contact</a></li>
    </ol>
  </nav>
</details>

<br />

## Overview

This project delivers a self-hosted cybersecurity lab that mirrors enterprise network architecture on a single physical server. The primary goals are to:

- **Engineer and maintain** a segmented, monitored network environment using industry-standard security tools.
- **Simulate realistic attack surfaces** through an Active Directory domain, vulnerable web applications, and intentionally exploitable machines.
- **Detect, analyze, and respond** to threats using a distributed SIEM, network intrusion detection, and centralized logging.
- **Develop offensive security skills** in a controlled, isolated environment with defined trust boundaries.
- **Build an extensible platform** that can expand into new cybersecurity domains as learning objectives evolve.

The architecture reflects deliberate tradeoffs between realism, security, and resource efficiency on a single-node deployment, including network segmentation enforced by firewall policy, endpoint visibility through Wazuh agents, and full-packet network monitoring via SPAN ports on both virtual switches.

## Architecture

### Network Topology

![Network Topology](./docs/assets/diagrams/security_lab_network_topology.svg)
*The diagram above illustrates the full lab topology, from the ISP demarcation point through the dual-router setup, into the Proxmox hypervisor with its two Open vSwitch bridges, the virtualized pfSense firewall, and all VLAN-segmented internal networks.*

### Hardware

The entire lab runs on a single dedicated server connected to an isolated personal network.

| Component | Specification |
|-|-|
| **Server** | Minisforum MS-01-S1390 Mini PC |
| **Processor** | Intel Core i9-13900H |
| **Memory** | 96 GB DDR5-5600 SODIMM (2 × 48 GB Crucial) |
| **Primary Storage** | 2 TB Crucial P3 Plus PCIe 4.0 NVMe M.2 SSD |
| **Secondary Storage** | 500 GB Crucial P3 Plus PCIe 4.0 NVMe M.2 SSD |
| **Backup Storage** | 2 TB External SSD (network-attached via personal router) |
| **Personal Router** | Wi-Fi 6E router (dual-router isolation from ISP gateway) |
| **Powerline Adapter** | Powerline ethernet bridge to server |
| **Home Router** | ISP-provided gateway (demarcation point) |

### Network Segmentation

The lab uses a dual-router topology to isolate its traffic from the household network. Inside Proxmox, two Open vSwitch bridges and a virtualized pfSense firewall enforce segmentation across five distinct network zones.

| VLAN | Name | Subnet | Purpose | Internet Access |
|-|-|-|-|-|
| - | Production LAN | 192.168.1.0/24 | Infrastructure services (Wazuh, NIDS, WireGuard, Guacamole) | Yes |
| Native | Default | 10.0.0.0/24 | Untagged traffic on vmbr1 | Per pfSense policy |
| 10 | SEC_OPS | 10.10.10.0/24 | Security operations (Kali Linux) | Yes |
| 20 | AD_LAB | 10.20.20.0/24 | Active Directory environment (DC, Windows endpoints) | Yes |
| 30 | SEC_EGRESS | 10.30.30.0/24 | Untrusted hosts with internet access (OWASP Juice Shop) | Yes |
| 40 | SEC_ISOLATED | 10.40.40.0/24 | Fully isolated untrusted hosts (VulnHub machines) | No - Kali access only |

This segmentation enforces the principle of least privilege at the network level. Vulnerable and untrusted hosts in VLANs 30 and 40 are restricted from reaching production infrastructure, while the SEC_ISOLATED zone has no internet access at all, limiting blast radius in the event of an uncontrolled exploit.

### Routing & Traffic

Traffic flows through the lab as follows:

1. **WAN-bound traffic** from internal VMs routes through pfSense (vtnet1 → vtnet0), out to the personal router's LAN (vmbr0), and through the home router to the ISP.
2. **Inter-VLAN traffic** is routed and filtered by pfSense firewall rules. VLAN tags are stamped on frames at vmbr1 and processed by pfSense sub-interfaces (vtnet1.10 through vtnet1.40).
3. **Production LAN traffic** (vmbr0) carries infrastructure services and the pfSense WAN interface. Static routes on the personal router direct 10.10.10.0/24 and 10.20.20.0/24 traffic back to pfSense's WAN IP for return routing.
4. **NIDS visibility** is achieved via SPAN ports configured on both vmbr0 and vmbr1, forwarding copies of all frames to the NIDS node for full-packet capture and analysis across every network segment.
5. **Remote access** enters via WireGuard (UDP 51820, exposed through Dynamic DNS on the home router) or Apache Guacamole on the production LAN.

## Tools & Technologies

| Category | Tools |
|-|-|
| **Hypervisor** | Proxmox VE |
| **Virtual Networking** | Open vSwitch |
| **Firewall & Routing** | pfSense |
| **SIEM & Log Management** | Wazuh (Manager, Indexer, Dashboards), Filebeat |
| **Network Intrusion Detection** | Suricata, Zeek |
| **Remote Access** | WireGuard, Apache Guacamole |
| **Offensive Security** | Kali Linux |
| **Vulnerable Targets** | OWASP Juice Shop, VulnHub machines |
| **Directory Services** | Active Directory, Windows Server 2025 |
| **Endpoints** | Windows 11 Enterprise, Debian |
| **Containerization** | Docker |
| **Storage & Backup** | ZFS, Proxmox Backup |

## Environment

### Hypervisor & Virtualization

Proxmox VE manages all lab workloads on a single physical node, providing KVM virtual machines for resource-intensive systems and LXC containers for lightweight services. It was selected over VMware ESXi and Microsoft Hyper-V primarily for licensing stability - Broadcom's acquisition of VMware introduced uncertainty that disqualified ESXi as a long-term foundation, and Hyper-V required a full Windows Server license after Microsoft discontinued the free standalone product. Proxmox offers open-source licensing with no feature gating, native ZFS snapshots, and a dedicated backup product at no additional cost.

Two Open vSwitch (OVS) bridges enforce Layer 2 segmentation. OVS was chosen over standard Linux bridges to natively support 802.1Q VLAN trunking, port mirroring (SPAN), and strict Layer 2 isolation - capabilities that would otherwise require supplementary tooling such as physical TAPs or iptables TEE targets.

| Interface | Type | Ports / Slaves | CIDR | Role |
|-|-|-|-|-|
| `enp89s0` | OVS Port | - | - | Physical uplink to vmbr0 |
| `vmbr0` | OVS Bridge | `enp89s0` `vmbr0_mgmt` | - | Production LAN |
| `vmbr0_mgmt` | OVS IntPort | - | 192.168.1.16/24 | Management interface (gw: 192.168.1.1) |
| `vmbr1` | OVS Bridge | `vmbr1_10` `vmbr1_20` `vmbr1_30` `vmbr1_40` | - | Security switch (no physical uplink) |
| `vmbr1_10` | OVS IntPort | - | - | SEC_OPS (VLAN 10) |
| `vmbr1_20` | OVS IntPort | - | - | AD_LAB (VLAN 20) |
| `vmbr1_30` | OVS IntPort | - | - | SEC_EGRESS (VLAN 30) |
| `vmbr1_40` | OVS IntPort | - | - | SEC_ISOLATED (VLAN 40) |

vmbr0 bridges to the physical NIC for LAN and WAN connectivity. vmbr1 has no physical uplink and is reachable only through pfSense's trunk port. SPAN ports on both bridges feed mirrored frames to the NIDS node, providing passive full-packet visibility across all network segments.

Storage is split across two NVMe drives for fault isolation: the 500 GB drive hosts the Proxmox OS and templates as Directory-type storage, while the 2 TB drive is a ZFS pool for all VM disk images and container volumes, chosen for its copy-on-write snapshots that support rapid rollback after destructive testing. The personal router's LAN uses a dedicated subnet, distinct from the home router's default range, to isolate the lab network from the household network.

### Firewall & Network Security

A virtualized **pfSense** firewall serves as the single policy enforcement point for the entire lab. Its WAN interface (vtnet0) sits on vmbr0 with a static address on the production LAN, while its LAN interface (vtnet1) connects to vmbr1 as a VLAN trunk carrying sub-interfaces for each security zone (vtnet1.10 through vtnet1.40). All inter-VLAN routing and access control is enforced here - no traffic crosses between VLANs without traversing pfSense.

Firewall rules implement least-privilege network access per VLAN. SEC_OPS (VLAN 10) is permitted to reach target VLANs 20, 30, and 40 for offensive operations. AD_LAB (VLAN 20) and SEC_EGRESS (VLAN 30) have internet access but cannot reach production infrastructure. SEC_ISOLATED (VLAN 40) has no internet access and no route to any network other than inbound connections from Kali on VLAN 10, constraining blast radius from intentionally exploitable systems. DHCP is served by pfSense on each VLAN sub-interface, with static reservations for infrastructure hosts. Static routes on the personal router direct 10.x.x.x return traffic back to pfSense's WAN IP.

### SIEM & Log Management

The **Wazuh** stack is deployed as a distributed architecture across three dedicated VMs on the production LAN:

- **Wazuh Manager** (192.168.1.19) - Receives agent events, applies rule matching and correlation, and forwards processed data to the Indexer.
- **Wazuh Indexer** (192.168.1.17) - OpenSearch-based storage and full-text search across alert, archive, and monitoring indices.
- **Wazuh Dashboards** (192.168.1.18) - Web interface for alert visualization, threat hunting, and agent management. Connects to both the Indexer (queries) and the Manager (API for configuration and active response).

The distributed deployment separates ingestion, storage, and presentation concerns, allowing each component to be independently resourced and maintained. Wazuh agents are deployed on monitored endpoints across VLANs 10 and 20 (Kali, DC1, Win10Ent1, Win10Ent2), reporting host-level telemetry - log collection, file integrity monitoring, and security configuration assessment - to the Manager over TCP 1514. Filebeat on the NIDS node ships Suricata and Zeek alerts into the Wazuh pipeline, enabling centralized correlation between network-level detections and endpoint events.

### Network Intrusion Detection (NIDS)

A dedicated **NIDS node** (192.168.1.20) on the production LAN runs both **Suricata** and **Zeek**, providing complementary detection capabilities. Suricata performs signature-based intrusion detection against known threat patterns, while Zeek provides protocol analysis and connection metadata that supports behavioral investigation and threat hunting beyond what signatures alone can identify.

The node receives mirrored traffic from SPAN ports on both vmbr0 and vmbr1, giving it full visibility into production infrastructure and all VLAN-segmented security traffic simultaneously - without inline placement that could introduce latency or become a point of failure. Alerts from both engines are forwarded via Filebeat into the Wazuh SIEM for unified alerting and correlation with endpoint telemetry.

### Remote Access

Two independent access paths provide secure lab management:

- **WireGuard** - Runs in a dedicated LXC container (192.168.1.4) on the production LAN. UDP port 51820 is exposed through the home router via Dynamic DNS, enabling encrypted tunnel access from external networks. Once connected, remote clients have network-level access equivalent to being on the production LAN.

- **Apache Guacamole** - Runs in a dedicated LXC container (192.168.1.3) on the production LAN, providing clientless browser-based RDP and SSH access to lab VMs. This supplements WireGuard by enabling access from environments where a VPN client cannot be installed or where only browser-based connectivity is available.

### Offensive Security

**Kali Linux** is deployed on VLAN 10 (SEC_OPS) as the primary attack platform. Placing it on a dedicated VLAN separates offensive tooling from target environments, ensuring that attack traffic is generated across VLAN boundaries and is visible to both the pfSense firewall and the NIDS - mirroring the network visibility that a SOC would have in an enterprise environment. pfSense rules permit Kali to reach VLANs 20, 30, and 40 for engagements against Active Directory, web application, and isolated targets respectively.

### Active Directory Environment

VLAN 20 (AD_LAB) hosts a Windows Active Directory domain:

- **DC1** - Windows Server 2025 domain controller.
- **Win11Ent1** and **Win11Ent2** - Windows 11 Enterprise domain-joined workstations.

This environment supports attack scenarios including credential harvesting, lateral movement, Group Policy abuse, and domain enumeration. All three hosts run Wazuh agents, providing endpoint-level visibility into authentication events, process execution, and file integrity changes alongside the network-level detection from the NIDS.

### Vulnerable Targets

- **OWASP Juice Shop** (VLAN 30 - SEC_EGRESS) - A deliberately vulnerable web application deployed via Docker for testing web application attacks including injection, XSS, and broken authentication. Placed on SEC_EGRESS because it requires internet access for its intended functionality, but is restricted from reaching production infrastructure.

- **VulnHub Machines** (VLAN 40 - SEC_ISOLATED) - Intentionally exploitable VMs for practicing penetration testing techniques. Placed on SEC_ISOLATED with no internet access and no route to production systems - only Kali on VLAN 10 can reach them, constraining the blast radius of any uncontrolled exploit.

### Backup & Storage

Proxmox's built-in backup scheduler writes VM and container backups to a **2 TB external SSD** connected to the personal router as network-attached storage. This separates backup data from the server's local disks, ensuring that a drive failure on the server does not also destroy the backup set. The network-attached approach also keeps the server's NVMe slots dedicated to production workloads.

## Roadmap

Planned enhancements include:

- [ ] **Network Addressing Cleanup** - Migrate the personal router away from the default 192.168.1.0/24 scheme to a custom and more coherent addressing scheme.
- [ ] **Ansible Automation** - Automate maintenance tasks for Wazuh components, such as upgrades and configuration management.
- [ ] **AWS Integration** - Extend the lab into AWS to gain hands-on experience with cloud security monitoring and hybrid environments.
- [ ] **Malware Analysis Sandbox** - Deploy a dedicated analysis environment on an isolated VLAN for safe dynamic and static malware analysis.
- [ ] **Detection Engineering Pipeline** - Build a CI/CD workflow for developing, testing, and deploying custom Wazuh rules and Suricata signatures.
- [ ] **MITRE ATT&CK Coverage Matrix** - Build and maintain a visual map of which ATT&CK techniques the lab can detect, organized by data source.

## References

- [Proxmox Cybersecurity Lab Project - 0xBEN](https://benheater.com/proxmox-cybersecurity-lab/)
- [Building Blue Team Home Lab - facyber (Marko Andrejic)](https://facyber.me/posts/blue-team-lab-guide-part-1/)
- [Proxmox VE Documentation](https://pve.proxmox.com/wiki/Main_Page)
- [pfSense Documentation](https://docs.netgate.com/pfsense/en/latest/)
- [Wazuh Documentation](https://documentation.wazuh.com/current/)
- [Open vSwitch Documentation](https://docs.openvswitch.org/en/latest/)
- [Suricata Documentation](https://docs.suricata.io/en/latest/)
- [Zeek Documentation](https://docs.zeek.org/en/current/)
- [WireGuard Documentation](https://www.wireguard.com/quickstart/)
- [Apache Guacamole Documentation](https://guacamole.apache.org/doc/gug/)

## Contact

### **Joe Maalouf**
<address style="display: flex; justify-content: flex-start; list-style-type: none;">
  <a href="https://github.com/joelouf" title="GitHub: @joelouf"><img src="https://img.shields.io/badge/GitHub-100000?style=for-the-badge&logo=github&logoColor=white" height="32" alt="github.com/joelouf" />
  </a><a href="https://linkedin.com/in/joelouf" title="LinkedIn: /in/joelouf"><img src="https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge" height="32" alt="linkedin.com/in/joelouf" />
  </a>
</address>

<br />

<p align="right"><a href="#top">Back to top ↑</a></p>
