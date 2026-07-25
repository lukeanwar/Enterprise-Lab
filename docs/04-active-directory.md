# Session 4 — Active Directory build

> **Goal:** install Windows Server 2022 in the CORP segment, promote it to a Domain Controller for a new forest (`corp.lab.local`), populate a realistic OU structure with fake users, groups, and service accounts, and update pfSense's DHCP scope so future CORP clients can find the domain via the DC's DNS.
>
> **Time:** ~1.5–2 hours.

## What you'll have at the end

- A Windows Server 2022 VM (`WIN-DC01`) running in the CORP segment with a static IP of `10.10.20.10`
- A new Active Directory forest, `corp.lab.local`, with NetBIOS name `CORP`
- DNS hosted on the DC, integrated with AD
- An OU structure mirroring a small business (departments, service accounts, admin accounts, computers)
- ~10 fake users across departments, one Domain Admin account for yourself, and two service accounts
- pfSense's CORP DHCP scope updated so domain-joined clients use the DC for DNS resolution

## Why we're building AD before anything else in CORP

Active Directory is the identity backbone of any Windows-based enterprise environment, and it's the single most common attack target in real-world incidents. Building it now means every subsequent session — domain clients, the Wazuh SIEM with agents, and detection scenarios — has something to authenticate against and something realistic to defend.

Building AD also means the lab will be useful for studying topics that appear in Security+ and most SOC analyst job descriptions: Kerberos and NTLM authentication, Group Policy, privilege escalation paths, and indicators of compromise such as anomalous logons and Kerberoasting.

## 1. Create the VM in Fusion

In VMware Fusion, create a new VM.

| Setting              | Value                                                       |
| -------------------- | ----------------------------------------------------------- |
| Installation method  | Install from disc or image — pick the Windows Server 2022 ISO |
| Operating system     | Microsoft Windows → Windows Server 2022                     |
| Firmware             | UEFI (default for Windows Server 2022)                      |
| Memory               | 4 GB                                                        |
| Processors           | 2                                                           |
| Disk                 | 60 GB                                                       |
| VM name              | `WIN-DC01`                                                  |

Before booting, edit **Settings → Network Adapter** and set the adapter to use the **CORP** custom network. The default of "Share with my Mac" must be changed — the DC belongs on the CORP segment, not bridged directly to the internet.

## 2. Install Windows Server 2022

Boot the VM. The Windows installer will load.

1. Language, time, and keyboard: defaults.
2. **Install now.**
3. Operating system: **Windows Server 2022 Standard Evaluation (Desktop Experience)** — the GUI edition. The non-GUI Core edition is closer to production but harder to learn on.
4. License terms: accept.
5. Installation type: **Custom: Install Microsoft Server Operating System only (advanced).**
6. Drive: select the only disk, click Next, let it install. Takes ~10–15 minutes including reboots.
7. Set the local Administrator password. Use something memorable but not trivial.
8. Log in.

## 3. Initial server configuration

Once logged in as Administrator, do the following before installing any roles.

### Rename the computer

The default name is something like `WIN-XXXXXX`. Rename it to `DC01`.

- **Server Manager → Local Server → Computer name** (click the current name)
- **Change** → enter `DC01` → OK
- Restart when prompted.

### Set a static IP

DHCP will not do — a domain controller needs a static address.

- **Settings → Network & Internet → Ethernet → Edit IP settings → Manual**
- IPv4 on, then:
  - **IP address:** `10.10.20.10`
  - **Subnet mask / prefix:** `24` (or `255.255.255.0` depending on UI)
  - **Gateway:** `10.10.20.254`
  - **Preferred DNS:** `127.0.0.1` — the DC will resolve DNS for itself once it's promoted. Listing localhost avoids a chicken-and-egg problem during the AD install.
- Save.

Test internet egress from the server. ICMP to the internet is intentionally not allowed by your CORP firewall rules (mirroring how a real corporate network would limit ping out to the internet), so use a test that exercises the rules you did write:

```
nslookup google.com 1.1.1.1
```

This sends DNS over UDP/53 directly to a public resolver — covered by your `Internet_Egress_Ports` allow rule. You should get an answer back with Google's IP addresses.

> Don't bother trying to browse the internet in Edge at this stage — the DC's system DNS is set to `127.0.0.1`, which has no DNS service running until AD DS is installed in the next section. Browser-based name resolution will fail until then. `nslookup` works because it can be pointed at a specific external resolver, bypassing the system DNS setting.

### Set the time zone and clock

`Settings → Time & Language → Date & time`.

1. Set the **Time zone** to your local zone.
2. Toggle **Set time automatically** to **Off**.
3. Click **Change** under "Set the date and time manually" and enter the actual current date and time. Match it to your host machine's clock for accuracy.

Why this matters: AD's Kerberos authentication is sensitive to clock skew. If the DC's clock is more than 5 minutes off from a domain client's clock, tickets will fail to validate and authentication breaks.

Why we can't rely on automatic NTP sync at this stage: the DC's DNS server is currently set to `127.0.0.1`, ready for the AD install in the next section. But until AD DS is installed, no DNS service is actually running on `127.0.0.1`. That means Windows cannot resolve `time.windows.com` (or any other external NTP hostname) and the automatic time sync silently fails. Setting the time manually is the simplest way through this chicken-and-egg problem.

Once the DC is promoted in Section 5 below, its own DNS service will be running and `time.windows.com` will resolve through it.

> **Follow-up — how time sync was ultimately handled in this lab.** Leaving the DC on manual time caused problems later. Because the host is a laptop that sleeps regularly, the guest clocks drifted badly — the DC ended up over an hour behind real time, and the drift was still growing. Windows time sync could not correct it: `w32tm /resync` returned "the computer did not resync because no time data was available", because the CORP egress rule permits TCP but NTP is UDP/123.
>
> Two options at that point. The production-correct one is to make the PDC emulator authoritative, point it at an internal NTP source (pfSense), allow UDP/123 to the firewall itself, and let domain members inherit time from the domain hierarchy. The pragmatic one for a laptop-hosted lab is to enable **Synchronize guest time with host** in the VM's Fusion settings and disable pfSense's NTP server, letting the hypervisor keep every guest honest.
>
> This lab uses the second. In a real environment you would not do this — host time sync is explicitly discouraged on domain controllers, because the domain should be authoritative for its own time rather than a downstream consumer of the hypervisor's clock. But on hardware that suspends several times a day, NTP cannot discipline a clock that keeps losing large chunks of time, and the resulting skew breaks Kerberos and corrupts log timestamps.
>
> Time synchronisation is security infrastructure, not housekeeping. Kerberos rejects tickets beyond five minutes of skew, and a SIEM correlating events across hosts is only as trustworthy as the clocks feeding it.

## 4. Install the AD DS and DNS roles

In **Server Manager**:

1. **Manage → Add Roles and Features**
2. Installation Type: **Role-based or feature-based installation.**
3. Server selection: the default (DC01) is fine.
4. Server Roles: tick **Active Directory Domain Services**. A pop-up will appear to add the required management features — click **Add Features**.
5. Also tick **DNS Server**. Same pop-up — click **Add Features**.
6. Click Next through Features, AD DS, DNS Server confirmation pages.
7. Tick **Restart the destination server automatically if required.**
8. Click **Install**.

This takes a few minutes. When it completes, do **not** close the wizard yet — there's a yellow notification flag at the top of Server Manager.

## 5. Promote the server to a Domain Controller

In Server Manager, click the yellow flag at the top right → **Promote this server to a domain controller**.

1. Deployment configuration: **Add a new forest.**
2. **Root domain name:** `corp.lab.local`
3. Domain Controller Options:
   - Forest functional level: Windows Server 2016 or higher (the default is fine)
   - Domain functional level: same
   - Tick **Domain Name System (DNS) server** and **Global Catalog (GC)** (both should already be on)
   - Set the **Directory Services Restore Mode (DSRM) password**. This is separate from the Administrator password and is used to recover the directory if it breaks. Use something memorable but not the same as your Admin password.
4. DNS Options: ignore the warning about delegations. It's expected for an isolated lab.
5. Additional Options:
   - NetBIOS domain name: `CORP` (this is auto-populated from the FQDN)
6. Paths: defaults.
7. Review options and prerequisite check. Some warnings are normal in a lab (e.g. cryptography defaults, DNS delegation). Errors stop the install; warnings don't.
8. **Install.**

The server will reboot itself when promotion is finished. Log back in as `CORP\Administrator` using the same password you set during the Windows install.

## 6. Verify the domain is working

Open a Command Prompt as Administrator and run:

```
nltest /dsgetdc:corp.lab.local
```

You should see DC01 listed as the DC for the domain.

```
dcdiag /q
```

The `/q` flag shows only errors. A healthy DC produces no output here. Some warnings about DNS delegation are normal and can be ignored.

```
nslookup
> set type=SRV
> _ldap._tcp.corp.lab.local
```

You should see `dc01.corp.lab.local` as the LDAP server. This is what domain-joined machines will use to find the DC during authentication.

## 7. Update pfSense's CORP DHCP scope

Now that the DC is up and serving DNS for the domain, future CORP clients need to use it as their primary DNS — otherwise they won't be able to find `corp.lab.local` and will fail to join the domain.

On the pfSense web UI (`https://10.10.10.254`):

1. **Services → DHCP Server → CORP**
2. Change **DNS servers** from `10.10.20.254, 1.1.1.1` to:
   - DNS Server 1: `10.10.20.10` (the DC)
   - DNS Server 2: leave blank
3. Save, Apply Changes.

We deliberately drop the public fallback. Once a host is domain-joined, you want all DNS traffic going through the DC so the domain (and security tooling later) can see and log it. The DC will forward unresolvable queries upstream to pfSense, which has the public resolvers configured.

## 8. Build the OU structure

Open **Active Directory Users and Computers** (`Server Manager → Tools` menu).

Right-click on `corp.lab.local` and create the following Organisational Units. Right-click → **New → Organizational Unit**:

```
corp.lab.local
├── Admin Accounts
├── Departments
│   ├── Finance
│   ├── HR
│   ├── IT
│   ├── Marketing
│   └── Sales
├── Service Accounts
└── Workstations
```

Create `Departments`, `Admin Accounts`, `Service Accounts`, and `Workstations` at the top level, then right-click `Departments` to create the five department sub-OUs inside it.

Leave "Protect container from accidental deletion" ticked on each — it's the default and is good practice.

## 9. Create users and groups

Right-click on the relevant OU → **New → User** for each row in the tables below. For all users, untick "User must change password at next logon" (it's a lab — we don't want password-change prompts breaking later sessions). Set a strong password; the same password across all lab users is fine for the lab but never appropriate in production.

### Admin Accounts OU

| Name           | Username     | Group memberships     |
| -------------- | ------------ | --------------------- |
| Luke Anwar     | adm-luke     | Domain Admins         |

To add `adm-luke` to Domain Admins: right-click the user → **Add to a group...** → type `Domain Admins` → OK.

### Departments → IT OU

| Name           | Username     |
| -------------- | ------------ |
| James Harper   | james.harper |
| Sarah Chen     | sarah.chen   |

### Departments → HR OU

| Name           | Username      |
| -------------- | ------------- |
| Alice Morgan   | alice.morgan  |
| Michael Brown  | michael.brown |

### Departments → Finance OU

| Name           | Username     |
| -------------- | ------------ |
| David Patel    | david.patel  |
| Emma Taylor    | emma.taylor  |

### Departments → Marketing OU

| Name             | Username        |
| ---------------- | --------------- |
| Olivia King      | olivia.king     |
| Thomas Wright    | thomas.wright   |

### Departments → Sales OU

| Name             | Username       |
| ---------------- | -------------- |
| Jennifer Lee     | jennifer.lee   |
| Robert Davis     | robert.davis   |

### Service Accounts OU

| Name             | Username       | Purpose                                              |
| ---------------- | -------------- | ---------------------------------------------------- |
| Wazuh Service    | svc-wazuh      | Account used later by the Wazuh agent for log access |
| Backup Service   | svc-backup     | Placeholder for a future backup tool                 |

For service accounts, also tick **Password never expires** so they don't break when password policy expiry kicks in. In production, service account credentials would be rotated via a privileged access management tool; in a lab, the simpler approach is fine.

### Department groups

Create one security group per department to make later GPO targeting and access-control work cleaner.

In each Department sub-OU, right-click → **New → Group**, scope **Global**, type **Security**, names: `IT-Staff`, `HR-Staff`, `Finance-Staff`, `Marketing-Staff`, `Sales-Staff`.

Add each department's users to their respective group.

## 10. Verification

From the Domain Controller, open **PowerShell as Administrator** (these are PowerShell cmdlets — they won't work in Command Prompt) and run the following to confirm everything is in place:

```
Get-ADUser -Filter * | Measure-Object
```

Should report a count in the 14–16 range — the 13 users you created (10 department + 1 admin + 2 service accounts) plus the built-in accounts Windows auto-creates on every domain (`Administrator`, `Guest`, `krbtgt`, and on newer versions `DefaultAccount`). The exact number isn't important; the point is that the command runs successfully, which proves the AD directory is responsive and the PowerShell module is loaded.

```
Get-ADOrganizationalUnit -Filter * | Select-Object Name
```

Should list all 9 OUs you created.

```
Get-ADGroup -Filter "Name -like '*-Staff'" | Select-Object Name
```

Should list all five department staff groups.

## 11. Take a VM snapshot

In Fusion: **Virtual Machine → Snapshots → Take Snapshot.** Name it `Session 4 complete - DC promoted, OUs and users built`. From here on, every session ends with a snapshot.

## 12. Verification checklist

- [ ] WIN-DC01 VM running on the CORP segment with static IP `10.10.20.10`
- [ ] Promoted to Domain Controller for `corp.lab.local`, NetBIOS `CORP`
- [ ] DNS role installed and integrated with AD
- [ ] `nltest /dsgetdc:corp.lab.local` returns DC01
- [ ] `dcdiag /q` produces no errors
- [ ] OU structure built (Departments + sub-OUs, Admin Accounts, Service Accounts, Workstations)
- [ ] 10 department users + 1 admin + 2 service accounts created in their correct OUs
- [ ] 5 department groups created and populated
- [ ] pfSense CORP DHCP scope updated to use `10.10.20.10` as DNS
- [ ] VM snapshot taken in Fusion

## Lessons learned

**The DC's DNS chicken-and-egg problem is real.** Setting the DC's preferred DNS to `127.0.0.1` before AD DS is installed means there's no DNS service running yet, so anything that relies on name resolution from the server itself — including the automatic Windows time sync — silently fails. The fix is to set the time manually before promotion, then let automatic NTP sync resume once the DC is up and its own DNS is answering queries.

**Kerberos is unforgiving about clock drift.** Five minutes of skew between a DC and a domain client is enough to break authentication entirely. The Windows installer defaulted to a time zone that didn't match my actual location, which silently put the clock out by hours. Worth checking the clock is correct before promotion — fixing it afterwards is harder because joined clients won't be able to authenticate to talk to the DC in the first place.

**Command Prompt and PowerShell are not interchangeable.** The verification cmdlets (`Get-ADUser`, `Get-ADOrganizationalUnit`, `Get-ADGroup`) are PowerShell, not cmd. Running them in Command Prompt produces unhelpful "not recognized" errors. PowerShell is the right tool for AD work and worth getting used to early — it's also how most real administration is scripted in production.

**The default DHCP DNS setting will break domain joins.** pfSense was handing out `1.1.1.1` as a fallback DNS to CORP clients. Once the domain exists, that fallback has to go: clients need to use the DC for DNS so they can find `corp.lab.local`. Leaving public DNS in the scope means clients sometimes resolve through Cloudflare instead of the DC, and any security tool sitting on the DC (Wazuh, in a few sessions' time) won't see those queries.

---

**Next:** [Session 5 — Domain clients and GPOs](05-domain-clients.md)
