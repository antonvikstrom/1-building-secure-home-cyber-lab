## Project 1: Building a Secure Home Cyber Lab: pfSense, Active Directory, and Splunk

### System Architecture & End-Goal Topology

<img width="782" height="763" alt="Screenshot 2026-07-06 at 21 13 20" src="https://github.com/user-attachments/assets/ecb6b04c-18bf-40f9-a294-591426c0820e" />

**Note:** This blueprint represents the complete 6-zone enterprise topology strived for across my lab builds. Project 1 establishes the virtual networking, centralized routing, storage policies, and core telemetry ingestion pipeline required to safely support future adversary emulation and analysis phases.

#### Asset Inventory

|**Hostname**|**OS**|**IP Address**|**Subnet / Zone**|**Role**|
|---|---|---|---|---|
|**pfSense**|FreeBSD|`10.1.x.1`|Gateway Core|Central Router & Policy Firewall|
|**oob-gateway**|Alpine Linux|`192.168.50.50`|Out-of-Band|WireGuard LXC remote access|
|**splunk-siem**|Ubuntu Server|`10.1.1.10`|Server Vault (LAN)|Central Log Indexer & Analytics|
|**dc01**|Windows Server 2022|`10.1.1.20`|Server Vault (LAN)|Active Directory Domain Controller|
|**win10-jump**|Windows 10|`10.1.1.50`|Server Vault (LAN)|Hardened Administration Workstation|
|**win10-client01**|Windows 10|`10.1.3.100`|Target Zone (OPT2)|Domain-joined Active Directory client|

#### Network Segmentation Policy

| **Interface**     | **Subnet**    | **Gateway** | **DHCP** | **Outbound Rule Policy**                          |
| ----------------- | ------------- | ----------- | -------- | ------------------------------------------------- |
| **LAN (Server)**  | `10.1.1.0/24` | `10.1.1.1`  | Disabled | Isolated. Custom pass rules for AD/DNS sync only. |
| **OPT1 (DMZ)**    | `10.1.2.0/24` | `10.1.2.1`  | Enabled  | Internet allowed; all local lab zones blocked.    |
| **OPT2 (Target)** | `10.1.3.0/24` | `10.1.3.1`  | Enabled  | Internet allowed; all local lab zones blocked.    |
| **OPT3 (Attack)** | `10.1.4.0/24` | `10.1.4.1`  | Enabled  | Internet allowed; all local lab zones blocked.    |
| **OPT4 (Clean)**  | `10.1.5.0/24` | `10.1.5.1`  | Disabled | No internet access allowed.                       |

## Hypervisor & Centralized Routing

I set up my lab on a small Lenovo ThinkCentre M920q mini-PC. First, I turned on virtualization in the BIOS, installed Proxmox VE, and swapped the setup over to the free community repository to get rid of the license pop-up.

<img width="964" height="345" alt="pve01-repos" src="https://github.com/user-attachments/assets/1ab66090-ac39-4507-bdc3-2277018c1225" />

Next, I used Proxmox's virtual switch settings to create five virtual networks named `vmbr1` through `vmbr5`. I left these networks completely disconnected from my physical home network so no traffic could leak out. In the center, I installed pfSense to act as my central router and firewall, mapping my bridge numbers directly to my subnets. I assigned static IPs to my core servers and set up DHCP for the client zones.

To manage the lab remotely without exposing it to the web, I created an unprivileged Alpine Linux container running WireGuard. Because of the safety locks on unprivileged containers, it couldn't talk to the network cards. I found out I had to manually edit the container's config file on Proxmox to pass through the `/dev/net/tun` device. By keeping this container unprivileged and only passing through this one specific network link, I locked it down. If someone ever hacks my VPN, they will be completely trapped inside that single container and won't be able to touch my main Proxmox host.

<img width="896" height="236" alt="Screenshot 2026-07-14 at 12 27 03" src="https://github.com/user-attachments/assets/2011275f-3b0e-4b23-847a-aafa5bedf688" />

Once that was done, I forwarded port UDP 51820 on my home router, set up a split-tunnel VPN, and turned on NAT routing using iptables inside Alpine. Finally, I went into pfSense and added a rule that only allows HTTPS management access from my WireGuard IP at 192.168.50.50.

<img width="1155" height="350" alt="pfsense-wan-rule" src="https://github.com/user-attachments/assets/47e72061-ab5f-42b0-b41c-870417b6d2d6" />

### Network Rules & Storage Architecture

To make sure malware couldn't crawl into other networks, I made a pfSense list called `All_Private_IPs` covering my local subnets. On my main networks, I put a simple two-rule security setup at the top of the list: one rule to pass logs up to my Splunk server (10.1.1.10) on port 9997, and another that allows internet access while blocking everything on my local IP list. This let my VMs fetch updates but stopped them from talking to each other.

<img width="1155" height="250" alt="pfsense-aliases" src="https://github.com/user-attachments/assets/18433d18-ed10-4a4f-990b-ac4e7bf8c804" />

I set up DHCP on OPT1, OPT2, and OPT3, but kept it off on the server zone (LAN) and the OPT4 malware clean room to keep things stable. On the malware clean room (OPT4) interface, I took away the internet rule completely. Leaving only the Splunk port open meant active malware would stay completely offline but still send logs back to my SIEM.

<img width="1160" height="258" alt="pfsense-opt4-ruleset" src="https://github.com/user-attachments/assets/8c172f2a-f1ff-4751-a50a-6a3c476a4d66" />


To get extra space, I plugged in a 1TB Samsung T7 external USB SSD. I formatted it as ext4 because I read it's much better at saving data if the USB cord gets bumped. I grabbed the drive's UUID using `blkid` and added it to my fstab file with the `nofail` flag so Proxmox wouldn't crash on startup if the drive wasn't plugged in.

<img width="900" height="111" alt="samsung-t7" src="https://github.com/user-attachments/assets/2cee58ba-a853-451a-aa31-8125e08ec0a9" />

I split up my storage paths to protect my data: Splunk got a 300GB disk on the external SSD , but I kept pfSense and my Active Directory domain controller on the internal drive. I read that Active Directory databases can corrupt instantly if a USB drive gets disconnected for even a split second. I moved my Windows client and jumpbox to the SSD to free up space.

### Setting Up Telemetry (Splunk & Syslog)

I set up my main Splunk server inside the server zone. At first, I couldn't reach the internal network from my laptop. I found out the WireGuard container was missing a route back to my internal lab subnets. Adding a static route inside Alpine fixed the issue immediately.

<img width="423" height="206" alt="alpine-wg-interfaces" src="https://github.com/user-attachments/assets/584ec069-a7c1-4955-9aad-33964a46cc05" />

I installed Splunk on a fresh Ubuntu server at `10.1.1.10`. Right away, I couldn't log into the console because the Ubuntu installer had defaulted to a US keyboard layout, which scrambled the special characters on my physical Swedish keyboard. I copy-pasted the password over an active SSH session to get in, then fixed the keyboard settings.

<img width="531" height="351" alt="ubuntu-keyboard-setup" src="https://github.com/user-attachments/assets/5e4844d0-fb12-4787-b787-5df71f9e0c72" />

When I tried starting Splunk, it crashed immediately because extracting the files with root privileges left the folders owned by root. A quick `chown` fixed the permissions. I created a firewall index capped at 100 GB and a Linux index capped at 50 GB. I found out that Linux blocks regular accounts from using ports below 1024, so I set up my Splunk listener on port 1514 instead.

<img width="1239" height="658" alt="splunk-udp-input" src="https://github.com/user-attachments/assets/71025472-cabd-4b17-8eb4-b55f6dfd9e56" />

In pfSense, I turned on remote logging and pointed it to `10.1.1.10:1514`, which instantly filled my dashboard with logs.

<img width="1151" height="728" alt="pfsense-remote-logging" src="https://github.com/user-attachments/assets/af551fea-965b-4122-ba02-751e0956f34e" />

To collect operating system logs from my Linux client, I installed the Splunk forwarder, but it wouldn't read the command history file. I realized my sudo command had left the file owned by root. After changing file ownership and granting the forwarder permissions to read the home directory, the logs started populating.

<img width="1105" height="434" alt="splunk-log-breakdown" src="https://github.com/user-attachments/assets/dab1e4fb-8d5c-48fe-b17e-a1897951ae82" />

### Active Directory & Domain Policy

I built a Windows Server 2022 domain controller at 10.1.1.20. I used standard drive settings so Windows could read the disk, mapped the network card to the VirtIO driver, and turned off the hypervisor firewall so pfSense would handle the security. Since Windows doesn't have VirtIO drivers built-in, it booted completely offline. I mounted the guest driver ISO to the virtual CD drive and used Device Manager to install the network card.

<img width="663" height="26" alt="dc01-proxmox-hardware" src="https://github.com/user-attachments/assets/d4b56bab-6665-484d-9d19-65973d2b11c6" />

I changed the server name to `dc01` and promoted it to a domain controller for `corp.vks-labs.com`. I also set up a folder structure inside AD with Endpoints, Servers, and Staff folders.

<img width="188" height="197" alt="active-directory-ou-architectureScreenshot 2026-06-28 at 17 19 18" src="https://github.com/user-attachments/assets/dd7f97f7-c9e2-4064-87a1-1d8ac9d3c5a3" />

I configured a Windows Jump Box at 10.1.1.50 but left it unjoined from the domain. Keeping this workstation out of the domain means that sensitive Domain Admin passwords are never saved in its local memory, which protects them from being stolen if the machine ever gets hacked. I used remote management tools and a custom script to securely manage AD from this unjoined workstation, using the DC's console only for Group Policy edits.

<img width="1274" height="204" alt="admanager-showcase" src="https://github.com/user-attachments/assets/e7b8349c-d235-4e6f-a8d6-c5c6da360a15" />

When I tried to join the Windows client in OPT2 to the domain, the connection timed out. I traced this to my firewall isolation rule blocking local traffic. Dragging an AD authentication pass rule to the top of my OPT2 rule list fixed the timeout instantly.

<img width="1159" height="311" alt="pfsense-ad-pass-rule" src="https://github.com/user-attachments/assets/4f56cb2e-d1a9-4d54-8f7f-133f9d708541" />

To capture command-line activity, I made a domain Group Policy and enabled PowerShell script block logging and module logging.

<img width="1228" height="254" alt="Screenshot 2026-07-06 at 18 49 31" src="https://github.com/user-attachments/assets/2e772c72-303c-4f98-acbf-38f9162a11a6" />

The client initially failed to fetch this policy. I found out there was a 9-hour time drift between my host's motherboard clock and the DC. Because Active Directory rejects any clock difference over 5 minutes, I forced the client to sync with the DC using the `net time` command, then fixed the server's timezone.

Even with the time aligned, test commands wouldn't show up in Splunk. A policy report showed that the client computer object was sitting in the default `Computers` folder, which doesn't support policy inheritance. I dragged the machine object into my `Endpoints` folder, renamed the computer to `win10-client01`, and rebooted.

<img width="454" height="158" alt="Screenshot 2026-07-06 at 19 04 43" src="https://github.com/user-attachments/assets/4a631867-4af6-46ec-b3d0-fa98bb2efb41" />

After rebooting, the GPO applied perfectly, but logs still weren't populating Splunk because the forwarder's configurations lacked a path for the PowerShell log channel. I opened `inputs.conf` in Notepad, manually appended the path, and restarted the forwarder.

<img width="538" height="528" alt="Screenshot 2026-07-06 at 19 21 46" src="https://github.com/user-attachments/assets/5f4b9847-6a23-4753-bc9b-033afca60cc4" />

I opened a shell on the client, typed a test command, and watched it immediately register as EventCode 4104 in Splunk!

<img width="1421" height="321" alt="Screenshot 2026-07-06 at 20 17 53" src="https://github.com/user-attachments/assets/10fa1c82-ad9c-4393-b2c8-a0cd87e15038" />
