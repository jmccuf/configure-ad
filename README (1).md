# Active Directory Lab in Microsoft Azure

### Windows administration · Identity management · Remote access

A hands-on lab exploring Active Directory Domain Services (AD DS) with a Windows Server domain controller and a Windows client hosted in Microsoft Azure.

[Portfolio](https://github.com/justinmccuff) · [Repository](https://github.com/justinmccuff/configure-ad)

> **Documentation status:** This revised guide retains the original lab's screenshot links and adds missing setup, security, and validation guidance. The new steps and checks have not yet been rerun or verified against a live environment. Example names and addresses below are illustrative, not recorded deployment values.

## Project Overview

The original project documents AD DS installation, organizational units, user creation, and Remote Desktop access. This expanded walkthrough explains the supporting sequence needed for a working domain: stable private addressing, domain-controller promotion, client DNS configuration, and domain join.

**Learning objectives**

- Understand the relationship between AD DS, DNS, and domain authentication.
- Deploy a domain controller and client in an Azure virtual network.
- Organize accounts using organizational units and security groups.
- Grant selected standard users Remote Desktop access to a member workstation.
- Validate domain membership and document results without exposing credentials.

## Software & Environment

| Component | Role |
| :--- | :--- |
| Microsoft Azure | Resource group, virtual network, VM hosting, and network controls |
| Windows Server 2022 | Operating system for `DC-1` |
| Active Directory Domain Services | Directory and domain authentication |
| DNS Server | AD domain and service-record resolution |
| Windows client | Domain-joined `Client-1`; use a supported Pro or Enterprise edition |
| Remote Desktop | Administrative and standard-user access testing |
| PowerShell | Configuration inspection and validation |

The original lab used Windows 10 21H2. Treat that as historical context, not a recommended image for a new deployment. Use a supported, appropriately licensed client release; Home editions do not support AD domain join or hosting Remote Desktop.

## Example Architecture

```text
Azure resource group: rg-ad-lab
└── Virtual network: vnet-ad-lab (10.0.0.0/16)
    └── Lab subnet: 10.0.0.0/24
        ├── DC-1       10.0.0.4 (static allocation on Azure NIC)
        │   Windows Server 2022 · AD DS · DNS
        └── Client-1   Azure-assigned private address
            Supported Windows client · DNS points to 10.0.0.4

Example forest/domain: ad.example.test
Example NetBIOS name:  LAB
Remote access: private connectivity or Azure Bastion preferred
```

`ad.example.test` is an example for an isolated lab only. For production, plan an AD namespace under a domain your organization owns. This single-domain-controller environment is not a production architecture.

## Before You Begin

- Use an Azure subscription and permissions that allow VM and network creation.
- Set a budget alert; VMs, disks, public IPs, and Bastion can incur charges.
- Use unique passwords stored in a password manager. Never commit passwords or secrets.
- Plan private access through Bastion or VPN. If temporary public RDP is unavoidable, restrict TCP 3389 to your current trusted public IP and remove the rule afterward. Never allow it from the entire internet.
- Keep Windows Firewall enabled. Review effective NSG rules and permit the required AD traffic between lab machines; do not open AD ports to the internet.

## Deployment Walkthrough

### 1. Create the Azure environment

1. Create a dedicated resource group, VNet, and subnet.
2. Deploy `DC-1` with Windows Server 2022 and `Client-1` with a supported Windows client edition.
3. Place both VMs in the same VNet for this lab. A shared resource group simplifies cleanup but is not a networking requirement.
4. On the Azure network interface for `DC-1`, set the private IP allocation to **Static**. Record the assigned private address.
5. Keep the guest NIC configured to obtain its address from Azure DHCP; do not independently hard-code a conflicting address inside Windows.
6. Connect using your chosen restricted remote-access method. Confirm hostnames before promotion or domain join, and restart if you rename either VM.

### 2. Check private connectivity

From `Client-1`, test the domain controller's private address:

```powershell
ping 10.0.0.4
```

If using ICMP for this check, enable only the appropriate inbound ICMPv4 echo rule on `DC-1`, scoped to the client or lab subnet. A blocked ping does not necessarily mean AD connectivity is broken; ICMP is optional and does not validate DNS or domain services.

### 3. Install AD DS and promote DC-1

1. On `DC-1`, open **Server Manager → Add Roles and Features**.
2. Select **Active Directory Domain Services** and include the management tools.
3. After installation, select **Promote this server to a domain controller** in Server Manager.
4. For this isolated lab, select **Add a new forest** and enter your chosen lab domain, such as `ad.example.test`.
5. Keep the DNS Server and Global Catalog options enabled. Set a strong Directory Services Restore Mode password and store it securely.
6. Review the domain/NetBIOS names, paths, and prerequisite checks. Investigate errors before continuing. A DNS delegation warning may be expected for a new isolated test namespace without a parent DNS zone.
7. Complete promotion and allow the server to restart.
8. Sign in with the domain administrator account used for setup. Confirm that **Active Directory Users and Computers** and **DNS Manager** are available.

> Installing the AD DS role alone does not create a domain. Promotion is a separate, required step.

### 4. Configure client DNS

Set `Client-1` to use the domain controller's private address as its DNS server. In Azure, configure the client NIC's **DNS servers → Custom**, or configure custom DNS at the VNet level if it should apply to all relevant VMs. NIC-level settings override VNet settings.

For this single-DC lab, enter `10.0.0.4` (replace with your actual DC address). Do not add a public resolver as a secondary client DNS server: it cannot resolve the private AD records. Configure appropriate DNS forwarding on the domain DNS server if external resolution is needed.

Restart the client after the Azure DNS change so it receives updated settings. Azure clients need private reachability and DNS configured for the AD domain. [1](https://learn.microsoft.com/en-us/answers/questions/1536553/is-it-possible-to-join-a-win-10-or-11-machine-to-a)

Run on `Client-1`:

```powershell
ipconfig /all
Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.ad.example.test
```

Confirm the DNS server address matches `DC-1` and the SRV query identifies your domain controller. Resolve DNS failures before continuing.

### 5. Join Client-1 to the domain

1. Sign in to `Client-1` with its local administrator account.
2. Open `sysdm.cpl`, select **Computer Name → Change**, and select **Domain**.
3. Enter the lab domain, such as `ad.example.test`.
4. Supply an account authorized to join computers. A lab administrator can be used for this setup; production should use appropriately delegated permissions.
5. Restart when prompted.
6. Sign in using an authorized domain account. Keep the local recovery account available using the `CLIENT-1\username` format when necessary.

Verify from elevated PowerShell on the client:

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object Name, Domain, PartOfDomain
Test-ComputerSecureChannel -Verbose
```

Expected, not yet recorded: the chosen domain, `PartOfDomain = True`, and a successful secure-channel result. Run the secure-channel test on the member client, not the domain controller.

### 6. Create organizational units and accounts

On `DC-1`, open **Active Directory Users and Computers** using an authorized administrator account.

1. Create `_EMPLOYEES` and `_ADMINS` organizational units.
2. Create a standard test user in `_EMPLOYEES`.
3. If practicing delegated administration, create a separate administrative identity and grant only the permissions needed for the exercise.
4. Set unique passwords. Complete any required first-logon password change at the client console before testing RDP; Network Level Authentication can prevent an initial RDP login when the password must be changed.

**Important:** An OU organizes objects; placing an account in `_ADMINS` does not grant administrative rights. Do not add ordinary users to Domain Admins, and do not attempt standard-user sign-in to the domain controller. Test standard-user access on `Client-1`.

### 7. Grant limited Remote Desktop access

1. Create a domain security group such as `GG_Client1_RDP`.
2. Add only the intended test users to that group.
3. On `Client-1`, enable Remote Desktop and retain Network Level Authentication.
4. Add `LAB\GG_Client1_RDP` to the client's local **Remote Desktop Users** group through **Select users** or Local Users and Groups.
5. Confirm firewall/NSG rules allow RDP only through the approved access path.
6. Test a member account using `LAB\username` or `username@ad.example.test`.
7. Test that an ordinary nonmember without another access grant cannot connect.

Avoid granting RDP to all **Domain Users**. If a group member is denied, inspect the **Allow log on through Remote Desktop Services** and **Deny log on through Remote Desktop Services** policies. A deny assignment takes precedence.

### 8. Add and test more users

Create two or three additional users through Active Directory Users and Computers. Add only users who require RDP to the dedicated access group. Sign out and back in after membership changes before retesting.

PowerShell can automate future bulk provisioning, but no bulk-creation script is provided or validated in this revision. Review any script before execution; avoid shared passwords, unnecessary privilege, and real personal data.

## Validation Checklist

Leave these unchecked until you rerun the lab and capture the results.

- [ ] `DC-1` has a stable Azure private IP allocation.
- [ ] AD DS promotion completed and the DNS zone exists.
- [ ] Client DNS points to the domain DNS server.
- [ ] The AD SRV lookup returns the domain controller.
- [ ] `Client-1` is domain-joined and its secure-channel check succeeds.
- [ ] `_EMPLOYEES` and `_ADMINS` exist with the intended accounts.
- [ ] A permitted standard user can RDP to `Client-1`.
- [ ] An ordinary user without access cannot RDP to `Client-1`.
- [ ] Standard users do not have administrative access to `DC-1`.
- [ ] Screenshots and command output have been sanitized before publication.

On the client, capture `whoami` and `hostname` after a successful test to demonstrate the account and machine. Do not publish credentials or authentication tokens.

## Troubleshooting

| Symptom | Checks |
| :--- | :--- |
| Domain cannot be contacted | Client DNS address, AD SRV records, private connectivity, NSGs, and Windows Firewall |
| Ping fails | ICMP rules; use DNS and service-specific tests rather than disabling the firewall |
| Domain credentials fail | Domain join, username format, password/account state, DNS, and clock synchronization |
| RDP denied | Target is `Client-1`, group membership, password-change requirement, NLA, and logon-right policies |
| Secure-channel test fails | DNS, reachability, clock synchronization, and computer-account state; diagnose before rejoining |

## Original Lab Screenshots

These links are retained from the original README. They are historical artifacts, not evidence that the added steps or hardening recommendations were performed. Review their contents for sensitive information and accuracy before publishing this revision.

### Original installation screenshot
![Original lab installation screenshot](https://github.com/justinmccuff/configure-ad/assets/143865133/b406af22-80da-4ca0-8539-ec94511dee0f)

### Original directory administration screenshot
![Original lab directory administration screenshot](https://github.com/justinmccuff/configure-ad/assets/143865133/990fbfa3-0d9c-428d-8fbb-458dfaadefbf)

### Original account and remote-access screenshots
![Original lab account and remote-access screenshot 1](https://github.com/justinmccuff/configure-ad/assets/143865133/68b21e1c-8c60-4737-8eb5-658eb0c933c3)

![Original lab account and remote-access screenshot 2](https://github.com/justinmccuff/configure-ad/assets/143865133/7486927a-6341-4e39-85db-0711c1976b50)

## Cleanup & Limitations

This is an isolated learning environment, not a hardened or highly available production deployment. Production AD requires additional planning for redundancy, backup/recovery, monitoring, patching, delegation, and security.

When finished, export sanitized evidence and delete only the dedicated lab resource group after checking its contents. Deallocating VMs stops compute charges, but disks and other resources may continue to incur charges. Remove temporary access rules and verify that no unwanted billable resources remain.
