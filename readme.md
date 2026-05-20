# Architecting a Resilient SOC Ecosystem

### Project Overview
This project covers how I built a segmented, 6-zone security lab using a Proxmox hypervisor. The network is designed to safely run cyber attack simulations, contain malicious traffic, and send log data to a central security monitoring system. Building this lab taught me the core steps of enterprise security engineering: from setting up firewalls and network zones to managing user identities and validating detections.

### Architectural Blueprint
The foundation of the lab is a network layout that creates strict, isolated "air-gaps" between zones in the server's memory. A central pfSense firewall acts as the sole gatekeeper, separating the lab into three main traffic paths:

- **The Attack Path:** Blocks direct communication between network zones, forcing all attacker movement to pass through the firewall for inspection.
- **The Telemetry Path:** Uses a secure firewall rule to stream system logs straight down to a central Splunk SIEM on port 9997.
- **The Management Path:** Restricts administrative access by routing my home network connection directly and exclusively through a secure Jumpbox.

<img width="939" height="777" alt="soc-network-diagram" src="https://github.com/user-attachments/assets/8f3c2ec3-80bc-4f48-b23b-0dbcd6f5d8f3" />

### Technology Stack
- **Hypervisor:** Proxmox VE (Type-1 Bare-Metal)
- **Firewall & Routing:** pfSense Appliance
- **SIEM & Logging:** Splunk Enterprise, Windows Sysmon, Splunk Universal Forwarder
- **Identity & Management:** Windows Server 2022 (Active Directory DC), Ubuntu Server (Management Jumpbox)
- **Endpoints & Analysis:** Windows 10/11 Workstations, Ubuntu Desktop, Tsurugi Linux (Forensics & Analysis)
- **Offensive Testing:** Kali Linux, Atomic Red Team
- **Physical Hardware:** Lenovo ThinkCentre M920q (Intel i5-8500T, 32GB RAM, 480GB SSD)

### Project Roadmap
[**Part 1: Designing Infrastructure and Network Logic**](https://github.com/antonvikstrom/soc-01-infrastructure-network-logic)

Establishes the physical and virtual foundation. This section covers basic hardware optimization, software repository setup, network zoning, building an asset inventory, and configuring the pfSense firewall to contain the lab's traffic.

**Part 2: Engineering Identity and Telemetry Visibility (In Progress)**

Deploys a Windows Active Directory domain and configures the log collection pipeline. This phase focuses on hardening endpoints using Group Policies, deploying the Splunk agent, and sending high-quality logs to the SIEM.

**Part 3: Validating Detection via Adversary Emulation (In Progress)**

The final verification phase. This section uses Atomic Red Team to launch realistic attack scripts from the Attack Zone, proving that the monitoring setup accurately captures and alerts on malicious behavior.

### Key Takeaways
Through designing this network, I learned how to isolate user traffic, log collection, and admin access. Setting up this model taught me how to stop attackers from communicating outside their designated zones and prevent them from tampering with core servers.

I gained practical experience in resource management by running a full enterprise network—including Active Directory, an attack platform, and a resource-heavy SIEM system—on a single mini-PC. Building this taught me how to configure hypervisor settings and optimized virtual drivers to keep the entire lab running smoothly without performance lag.

Finally, this project changed how I approach security tools. Instead of just installing software and assuming it works, I learned how to actively test my defenses. Using automated attack scripts taught me how to systematically verify that my logging setup successfully catches real-world threats.
