# Architecting a Resilient SOC Ecosystem

### Project Overview
Engineered a segmented, multi-zone SOC infrastructure on Proxmox to host adversary emulations and validate detection integrity via a high-fidelity telemetry pipeline. This project demonstrates the full lifecycle of defensive security engineering: from hardware-level optimization and network architecture to identity management and automated detection validation.

### Architectural Blueprint
The core of this ecosystem is a multi-bridge network design that enforces strict isolation between zones. By utilizing a virtualized "air-gap" strategy, the lab ensures that adversary traffic is contained while telemetry flows securely to the central monitoring vault.

<img width="727" height="749" alt="Screenshot 2026-05-10 at 21 37 23" src="https://github.com/user-attachments/assets/f3877ef0-fc66-41f6-8faf-eca04b3e9f4f" />

### Technology Stack
- **Hypervisor:** Proxmox VE (Type-1)
- **Networking:** pfSense (Firewall/Routing), Linux Bridges (Segmentation)
- **Identity & Endpoints:** Windows Server 2022 (Active Directory), Windows 10/11
- **SIEM & Telemetry:** Splunk Enterprise, Sysmon, Splunk Universal Forwarder
- **Adversary Emulation:** Kali Linux, Atomic Red Team
- **Hardware:** Lenovo ThinkCentre M920q (i5-8500T, 32GB RAM)

### Project Roadmap
[**Part 1: Designing Infrastructure and Network Logic**](https://github.com/antonvikstrom/soc-01-infrastructure-network-logic)
- Focused on establishing the physical and virtual foundation. This section covers BIOS-level optimizations, repository hardening, and the implementation of a multi-zone network architecture to minimize the lab's "blast radius".

**Part 2: Engineering Identity and Telemetry Visibility**
- (In Progress) Deployment of the Windows Active Directory domain and the engineering of the data pipeline. This phase focuses on endpoint hardening via GPO and the ingestion of high-fidelity Sysmon logs into Splunk.

**Part 3: Validating Detection via Adversary Emulation**
- (In Progress) The "Proof of Work" phase. Utilizing Atomic Red Team to simulate specific MITRE ATT&CK techniques and validating that the defensive architecture accurately detects and alerts on malicious behavior.

### Key Takeaways
- Implemented a Zero-Trust gateway approach where internal subnets have no physical route to the internet, preventing C2 "call-backs" and data exfiltration.
- Balanced the resource demands of a full-stack SIEM and Windows domain on a single physical node through efficient resource allocation and VirtIO drivers.
- Moved beyond "basic testing" to professional adversary emulation, ensuring that security controls are not just present, but functional and integrated.
