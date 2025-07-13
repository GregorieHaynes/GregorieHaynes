
# Lab 01 – DNS, Subnetwork, OPNsense Firewall

## 🧩 Lab Overview

This lab demonstrates the creation of a segmented virtual network using VirtualBox. I configured:
- A Windows Server 2022 as a DNS server
- An OPNsense router with firewall rules
- A Windows 11 client on a separate subnet
- AD policy control

## 🖥️ Environment Setup

| Machine | OS | IP Address | Role |
|--------|----|------------|------|
| Server | Windows Server 2022 | 192.168.0.2/26 | DNS / Directory Services |
| Client | Windows 11          | 192.168.0.66/26 | Workstation             |
| Router | OPNsense            | 192.168.0.1 / 192.168.0.65 | Firewall/Router |

## 🔧 Key Steps (Brief Descriptions)

<details>
<summary>1. Created Subnet Network in VirtualBox</summary>
- Created two internal network adapters: `Int_Admin` and `Int_User`
- Connected Server to `Int_Admin`, Client to `Int_User`, Router with two adapters
</details>

<details>
<summary>2. Installed and Configured OPNsense</summary>
- Set up interface assignments
- Configured static IPs for WAN and LAN
- Enabled DNS forwarder and firewall rules
</details>

<details>
<summary>3. Installed Windows Server 2022 and DNS Role</summary>
- Promoted to domain controller
- Configured DNS zone and records
- Verified with `nslookup` and `ping`
</details>

<details>
<summary>4. Firewall Rule Testing</summary>
- Created rules to allow:
  - TCP/UDP 53 for DNS
  - TCP 389 for LDAP
- Tested from Win11 client using `nslookup`
</details>

## 🖼️ Screenshots

<details>
### Firewall Rule in OPNsense
![Firewall Rules](./assets/firewall-rule-dns.png)
> Rule allowing TCP/UDP on port 53 for internal LAN subnet.

### Win11 Client Test
![NSLookup Test](./assets/nslookup-success.png)
> Client successfully resolves server domain, confirming DNS routing works.

### DNS Zone Configuration
![DNS Zone](./assets/dns-zone-setup.png)
> Shows the `Portfolio.Lab` zone and A record for `server01.Portfolio.local`.
> </details>
---

## 🧠 Takeaways

- I built an isolated lab simulating a small enterprise environment.
- I applied core sysadmin tasks: DNS setup, subnetting, routing, firewall config, user policies.
- Demonstrated end-to-end connectivity and testing.
