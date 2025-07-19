
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
  
  ![Subnet Network](https://raw.githubusercontent.com/GregorieHaynes/GregorieHaynes/main/Portfolio/asset-project01/AdminConfigurationNetworkVB.png)
  
- Connected Server to `Int_Admin`, Client to `Int_User`, Router with two adapters

  ![Subnet Network](https://raw.githubusercontent.com/GregorieHaynes/GregorieHaynes/main/Portfolio/asset-project01/SettingUpRouter.png)
  
  
</details>

<details>
<summary>2. Installed and Configured OPNsense</summary>
  
- Set up interface assignments

  ![Subnet Network](https://raw.githubusercontent.com/GregorieHaynes/GregorieHaynes/main/Portfolio/asset-project01/OPNSenseInterfaceIPChanges.png)

- Configured static IPs for LAN1 and LAN2
  
  ![Subnet Network](https://raw.githubusercontent.com/GregorieHaynes/GregorieHaynes/main/Portfolio/asset-project01/OPNSenseInterfaceIPChanges2.png)

- Setup firewall rules for LAN1 & LAN2

  ![Subnet Network](https://raw.githubusercontent.com/GregorieHaynes/GregorieHaynes/main/Portfolio/asset-project01/opnsensefirewallconfigforfirewall.png)

  
</details>

<details>
<summary>3. Installed Windows Server 2022 and DNS Role</summary>
- Promoted to domain controller

  ![Subnet Network](https://raw.githubusercontent.com/GregorieHaynes/GregorieHaynes/main/Portfolio/asset-project01/settingupDomainonAD.png)
  
- Configured DNS zone and records

  ![Subnet Network](https://raw.githubusercontent.com/GregorieHaynes/GregorieHaynes/Portfolio/asset-project01/DNSisSetupfor%20A%20Record.png)
  
- Verified with `ping`
  
  ![Subnet Network](https://raw.githubusercontent.com/GregorieHaynes/GregorieHaynes/Portfolio/asset-project01/PingWorksonsame%20subnet.png)
  
</details>

<details>
<summary>4. Setting up User11 and AD Connecting</summary>
  
- Created Win11 User in AD
  
  ![Subnet Network](https://raw.githubusercontent.com/GregorieHaynes/GregorieHaynes/main/Portfolio/asset-project01/creating_User.png)
  
- Connected Win11 to AD

  ![Subnet Network](https://raw.githubusercontent.com/GregorieHaynes/GregorieHaynes/main/Portfolio/asset-project01/loggingintoaccount.png)
  
</details>

## 🖼️ Issues and Resolution

<details>
  
### Logging Into AD Issues
![AD Error](https://raw.githubusercontent.com/GregorieHaynes/GregorieHaynes/main/Portfolio/asset-project01/IssuesLoggingIntoAD.png)
> Error occured logging into AD server.
![Log File](https://raw.githubusercontent.com/GregorieHaynes/GregorieHaynes/main/Portfolio/asset-project01/IssuesLoggingIntoAD.png)
> The cause was the firewall blocking LDAP, I went into root and grep'ed the filter logs. I used cat latest.log | grep 192.168.0.66 | grep 389
![Solution](https://raw.githubusercontent.com/GregorieHaynes/GregorieHaynes/main/Portfolio/asset-project01/opnsensefirewallconfigforfirewall.png)
> I setup an exception for 192.168.0.66 to talk to 192.168.0.2 and specified IPv4.
![Correction](https://raw.githubusercontent.com/GregorieHaynes/GregorieHaynes/main/Portfolio/asset-project01/SuccessLoggingIntoADWithUser.png)
> I was able to successfully login to AD server. 
 

### Win11 Client Test
![work in progress](./assets/nslookup-success.png)
> Client successfully resolves server domain, confirming DNS routing works.

### DNS Zone Configuration
![work in progress](./assets/dns-zone-setup.png)
> Shows the `Portfolio.Lab` zone and A record for `server01.Portfolio.local`.
> </details>
---

## 🧠 Takeaways

- I built an isolated lab simulating a small enterprise environment.
- I applied core sysadmin tasks: DNS setup, subnetting, routing, firewall config, user policies.
- Demonstrated end-to-end connectivity and testing.

