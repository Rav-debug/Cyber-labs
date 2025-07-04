# Active Directory Lab Environment

## Overview

This project demonstrates a full Active Directory lab setup using a pfSense firewall, a Windows Server 2022 Domain Controller, and a Windows 10 client. The purpose of this lab is to simulate enterprise network conditions, practice Windows domain management, and implement network segmentation using pfSense.

---

## Lab Environment

- **Host System**: [ThinkPad, 13th Gen Intel Core, 1TB Storage, 32GB RAM, Windows 11 Pro]
- **Virtualization Platform**: [VirtualBox]
- **Number of VMs**: 3
  - pfSense (Firewall)
  - Windows Server 2022 (Domain Controller)
  - Windows 10 Client Workstation

---

## VM Specs

| VM            | OS                 | RAM   | CPU | Disk   | IP Address     |
|---------------|--------------------|-------|-----|--------|----------------|
| pfSense       | pfSense 2.7.2      | 4 GB  | 2   | 10 GB  | 192.168.1.1    |
| DC01          | Windows Server 2022| 4 GB  | 2   | 40 GB  | 192.168.1.2    |
| Client01      | Windows 10 Pro     | 7 GB  | 2   | 30 GB  | 192.168.1.3    |

> All machines are connected through the pfSense LAN interface with the subnet: `192.168.1.0/24`

---

## Network Configuration

- **pfSense LAN IP**: `192.168.1.1/24`
- **pfSense WAN IP**: DHCP (connected to host NAT network)
- **AD Domain Name**: `homelab.local`
- **DNS Forwarders**: `8.8.8.8`, `1.1.1.1`
- **Firewall Rules**: See below
  
WAN RULES
| # | Action | Protocol   | Source                        | Destination | Port | Description                       |
| - | ------ | ---------- | ----------------------------- | ----------- | ---- | --------------------------------- |
| 1 | Block  | Any        | RFC 1918 networks             | Any         | Any  | Blocks private (non-routable) IPs |
| 2 | Block  | Any        | Reserved (unassigned by IANA) | Any         | Any  | Blocks bogon networks             |
| 3 | Allow  | ICMP       | Any                           | Any         | Any  | Allow pings                       |
| 4 | Allow  | UDP (IPv4) | Any                           | WAN address | 1194 | OpenVPN remote access             |


LAN RULES
| # | Action | Protocol     | Source      | Destination | Port    | Description                               |
| - | ------ | ------------ | ----------- | ----------- | ------- | ----------------------------------------- |
| 1 | Allow  | Any          | Any         | LAN Address | 443, 80 | Anti-lockout rule (web access to pfSense) |
| 2 | Allow  | IPv4         | LAN subnets | Any         | Any     | Allow LAN to WAN                          |
| 3 | Allow  | IPv4 TCP     | 192.168.1.3 | 192.168.1.2 | 53      | Client to AD DNS                          |
| 4 | Allow  | IPv4 TCP/UDP | 192.168.1.2 | Any         | 53      | AD to forward DNS queries                 |
| 5 | Allow  | IPv4         | LAN subnets | Any         | 53      | General DNS rule                          |
| 6 | Allow  | IPv6         | LAN subnets | Any         | 53      | IPv6 DNS allowance                        |
| 7 | Allow  | IPv4         | Any         | 192.168.1.2 | Any     | Allow all to AD server                    |
| 8 | Allow  | IPv4 TCP/UDP | 192.168.1.2 | Any         | 80      | AD HTTP traffic (for updates or browsing) |
| 9 | Allow  | IPv4 TCP/UDP | 192.168.1.2 | Any         | 443     | AD HTTPS traffic                          |

OpenVPN RULES
| # | Action | Protocol | Source | Destination | Port | Description                  |
| - | ------ | -------- | ------ | ----------- | ---- | ---------------------------- |
| 1 | Allow  | IPv4     | Any    | Any         | Any  | Allow full VPN client access |

## Active Directory Domain Services (AD DS)

The domain controller (`DC01`) was configured with the following:

- **Domain Name**: `lab.local`
- **Forest Functional Level**: Windows Server 2022
- **Roles Installed**:
  - Active Directory Domain Services
  - DNS Server

The AD DNS server is configured with forwarders:
- `8.8.8.8`
- `1.1.1.1`

The domain controller uses its own IP (`127.0.0.1`) as its primary DNS server.

---

### Current Status

- Active Directory has been initialized with the default configuration.
- No Organizational Units (OUs) or user accounts have been created yet.
- No systems have been joined to the domain.

The Windows 10 client in this lab is running **Windows Home**, which does not support domain joining. A Pro or Education edition ISO will be used in a future update to complete domain integration with multiple computers and test Group Policy Objects (GPOs).

## Windows Client Configuration

The Windows 10 client is currently running the **Home edition** and is not domain-joined.

### Network Settings

- **IP Address**: `192.168.1.3`
- **Subnet Mask**: `255.255.255.0`
- **Default Gateway**: `192.168.1.1` (pfSense)
- **Preferred DNS Server**: `192.168.1.2` (AD server)
- **Alternative DNS Server**: `8.8.8.8`

### Purpose

While the client cannot join the domain in its current state, it has full network access, can resolve DNS via the AD server, and can reach external sites through the pfSense firewall.

Future testing will include domain join and GPO enforcement after replacing the OS with Windows 10 Pro or Education.

## Connectivity Testing

After full configuration, the following tests were conducted to verify functionality:

### From Windows Client (192.168.1.3)

| Test                          | Command                    | Result                        |
|-------------------------------|----------------------------|-------------------------------|
| Ping pfSense gateway          | `ping 192.168.1.1`         | Successful                    |
| Ping AD server                | `ping 192.168.1.2`         | Successful                    |
| Ping Google (DNS resolution)  | `ping google.com`          | Successful                    |
| Raw internet connectivity     | `ping 8.8.8.8`              | Successful                    |
| DNS resolution via AD         | `nslookup google.com`      | Successful (forwarded to 8.8.8.8) |

### From AD Server (192.168.1.2)

| Test                          | Command                    | Result                        |
|-------------------------------|----------------------------|-------------------------------|
| Ping pfSense gateway          | `ping 192.168.1.1`         | Successful                    |
| Ping client                   | `ping 192.168.1.3`         | Successful                    |
| Ping Google (DNS resolution)  | `ping google.com`          | Successful                    |
| Raw internet connectivity     | `ping 8.8.8.8`              | Successful                    |
| DNS resolution via self       | `nslookup google.com`      | Successful (forwarded through AD) |

---

All results confirm:
- Proper routing through pfSense
- Functional DNS forwarders
- Bi-directional communication on the LAN
