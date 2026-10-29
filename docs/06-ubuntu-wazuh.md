# Session 6 — Ubuntu Server and the Wazuh SIEM

> **Goal:** build an Ubuntu Server 24.04 VM in the SERVERS segment, install Wazuh as a single-node deployment, open the firewall paths the agents need, then deploy agents onto the Domain Controller and the Windows 11 client so their security logs start flowing into a central platform.
>
> **Time:** ~2–2.5 hours. This is the longest session so far.

## What you'll have at the end

- An Ubuntu Server 24.04 LTS VM (`SRV-WAZUH01`) on the SERVERS segment at `10.10.30.10`
- Wazuh 4.x installed single-node: indexer, server, and dashboard on one host
- pfSense rules allowing agent enrolment and log shipping from CORP into SERVERS
- Wazuh agents installed and enrolled on `WIN-DC01` and `WIN-CLIENT01`
- Windows security events — including the Event 4688 process creation events from Session 5 — visible and searchable in the Wazuh dashboard
- At least one alert triggered deliberately and located in the dashboard

## Why this is the session that changes the lab

Everything up to now has been infrastructure. A firewall, a domain, a workstation — necessary, but they produce evidence that sits in isolated logs on individual hosts. Nobody is watching.

A SIEM is what turns scattered host logs into something a security team can actually work with. It collects, normalises, correlates, and alerts. It's the single tool most closely associated with the day-to-day work of a SOC analyst, and "I have used a SIEM" is one of the more common filters on entry-level cybersecurity job descriptions.

Wazuh specifically is a good choice for a lab. It's free and open source, it's genuinely used in production, it bundles file integrity monitoring, vulnerability detection, and MITRE ATT&CK mapping alongside plain log collection, and its agent model means you get real endpoint telemetry rather than just syslog.

## 1. Create the VM in Fusion

| Setting              | Value                                                    |
| -------------------- | -------------------------------------------------------- |
| Installation method  | Install from disc or image — Ubuntu Server 24.04 LTS ISO |
| Operating system     | Linux → Ubuntu 64-bit                                     |
| Memory               | 8 GB (Wazuh's indexer is Java-based and memory hungry)   |
| Processors           | 4                                                        |
| Disk                 | 80 GB                                                     |
| VM name              | `SRV-WAZUH01`                                            |

Set **Settings → Network Adapter** to the **SERVERS** custom network before booting.

> **On the resource allocation:** 8 GB of RAM and 4 vCPUs is a lot to give one lab VM, but Wazuh's indexer is OpenSearch underneath and it will fail in confusing ways if starved. If your host is tight on memory, shut down `WIN-CLIENT01` while you work on this session and start it again at the agent deployment stage.

## 2. Install Ubuntu Server

Boot the VM and work through the installer.

1. Language and keyboard: your preference.
2. Installation type: **Ubuntu Server** (not minimised).
3. **Network configuration** — this matters. pfSense serves DHCP on SERVERS (the `.100`–`.200` range from Session 3), so the installer will pick up an address automatically. Override it with a static one: a server other machines connect to by IP should not have an address that can change. Select the interface, then **Edit IPv4 → Manual**:
   - Subnet: `10.10.30.0/24`
   - Address: `10.10.30.10`
   - Gateway: `10.10.30.254`
   - Name servers: `10.10.30.254` (pfSense)
   - Search domains: `corp.lab.local`

   `.10` sits below the DHCP pool, so there's no risk of pfSense handing the same address to something else later — the same reasoning that put the Domain Controller at `10.10.20.10`.
4. Proxy: leave blank.
5. Mirror: accept the default.
6. Storage: **Use an entire disk**, no LVM needed for a lab. Confirm the destructive write.
7. Profile setup:
   - Your name: your choice
   - Server name: `srv-wazuh01`
   - Username: `labadmin`
   - Password: something you'll remember
8. **Install OpenSSH server:** tick this. You'll want to reach the box from elsewhere rather than working in the Fusion console window.
9. Skip all the snap suggestions.
10. Let it install, then **Reboot Now**.

### Open the firewall paths this host needs

The SERVERS segment was built in Session 3 with a single rule: internet egress to `!RFC1918_Internal`. Because that destination excludes every internal subnet, this host has no permitted path to pfSense itself — so no DNS, and no ping to its own gateway. The mirror step during install will fail with "Temporary failure resolving" until that's fixed.

Add two rules under **Firewall → Rules → SERVERS**, both above the implicit deny:

| Action | Protocol | Source | Destination | Ports | Description |
|---|---|---|---|---|---|
| Pass | TCP/UDP | SERVERS subnets | This Firewall (self) | 53 | Allow SERVERS hosts to resolve DNS via pfSense |
| Pass | ICMP (any) | SERVERS subnets | This Firewall (self) | — | ICMP to firewall (diagnostics) |

Save and **Apply Changes**.

This is the third time in the build that traffic aimed *at* the firewall has needed its own rule — ICMP on MGMT in Session 3, NTP on CORP in Session 4, DNS here. Rules written to let a segment reach *through* the firewall never cover traffic terminating *on* it. Worth internalising, because it's a recurring source of confusion with any default-deny design.

### Let pfSense answer for the AD zone

pfSense can now resolve public names, but it knows nothing about `corp.lab.local` — that zone is authoritative on the Domain Controller. Rather than pointing every host at the DC directly, configure conditional forwarding once on the resolver.

**Services → DNS Resolver → General Settings → Domain Overrides → Add**:

| Field | Value |
|---|---|
| Domain | `corp.lab.local` |
| IP Address | `10.10.20.10` |
| Description | Forward AD zone queries to the domain controller |

Save and **Apply Changes**. Also confirm **Enable DNSSEC Support** is unticked — internal zones have no public chain of trust, so validation fails against them and the override silently returns nothing.

Every segment now gets working internal *and* external resolution from a single resolver, and no individual host needs to know where the AD zone lives. This is the standard arrangement wherever an internal AD domain sits alongside a separate DNS resolver.

### Verify connectivity before going further

Log in and confirm the basics:

```bash
ip a
ping -c 3 10.10.30.254
nslookup ubuntu.com
nslookup win-dc01.corp.lab.local
```

In order: the static IP is set; the firewall is reachable; external resolution works; and the domain override is forwarding AD queries correctly.

> **Note the hostname.** It is `win-dc01`, not `dc01` — matching the VM name and the `ROLE-NAME##` convention from Session 1. An authoritative `NXDOMAIN` for a name that doesn't exist looks identical to a broken resolver at first glance, so check the name before assuming the configuration is at fault. `drill <name> @10.10.20.10` from pfSense's Diagnostics → Command Prompt is the quickest way to tell: the `aa` flag in the response means the server answered authoritatively, which is to say the configuration is working and the question was wrong.

If `nslookup ubuntu.com` fails, the port 53 rule above is missing or wasn't applied. If only the `win-dc01` lookup fails, the problem is the domain override or the DC being powered off. Also flush the client cache with `sudo resolvectl flush-caches` after any fix — systemd-resolved caches negative answers and will keep serving a stale `NXDOMAIN`.

### Set up remote access to the lab hosts

Working inside Fusion's console windows gets old quickly — no copy-paste, awkward scrolling, and a separate window per machine. From here on it's worth administering each host the way you actually would in a real environment: SSH to Linux, RDP to Windows.

#### First: routes from the host machine

Fusion connects the host to the MGMT segment only. Traffic to any other segment has nowhere to go until you add a route pointing it at the firewall:

```bash
sudo route -n add -net 10.10.20.0/24 10.10.10.254   # CORP
sudo route -n add -net 10.10.30.0/24 10.10.10.254   # SERVERS
sudo route -n add -net 10.10.40.0/24 10.10.10.254   # DMZ
```

macOS discards manually added routes on reboot, so these need repeating each session.

That the host can't reach other segments without this is the segmentation working, not a fault. Every one of those connections is also matched against a firewall rule on the way through — the MGMT rules from Session 3 are what permit SSH on 22, RDP on 3389 and the Wazuh dashboard on 443.

#### SSH to the Ubuntu server

OpenSSH was installed during setup, so this works immediately:

```bash
ssh labadmin@10.10.30.10
```

Confirm the service is running if it refuses the connection:

```bash
sudo systemctl status ssh
```

#### RDP to the Windows hosts

**Windows has no SSH server by default.** OpenSSH Server is an optional feature that isn't installed unless you add it, so `ssh` to a Windows box will simply hang — nothing is listening on port 22. Windows hosts are administered over RDP or WinRM instead.

**Enable RDP** on each Windows machine (it's off by default on Windows Server):

- **Settings → System → Remote Desktop → On**

**Install a client** on the host machine — Microsoft Remote Desktop, free from the App Store.

**Add each machine** with **+ → Add PC**:

| Field | `WIN-DC01` | `WIN-CLIENT01` |
|---|---|---|
| PC name | `10.10.20.10` | `10.10.20.100` |
| Credentials | `CORP\adm-luke` | `CORP\adm-luke` |
| Friendly name | `WIN-DC01` | `WIN-CLIENT01` |

Use the **IP address, not the hostname.** The host machine resolves DNS through whatever your home router provides, which knows nothing about `corp.lab.local` — the domain override on pfSense only helps machines that use pfSense as their resolver, and the host isn't one of them.

Use `adm-luke` rather than the built-in Administrator. A named admin account means the logs show *who* performed an action rather than just "Administrator", which is the difference between an audit trail and a shrug. Note also that only Domain Admins can log on interactively to a domain controller by default — `james.harper` would be refused even with the correct password, which is deliberate.

**Two settings worth changing** once the connection is saved (right-click the tile → Edit):

- **Display → Update the session resolution on resize**, or the session renders at a fixed size and gets clipped by the window
- **Devices & Audio → Clipboard**, which gives you copy-paste between the host machine and the VM — worth having before pasting the agent install commands in the next section

> **The client's address comes from DHCP**, so `10.10.20.100` may differ on your build. Check with `ipconfig` on the machine, or look at **Status → DHCP Leases** in pfSense. If you'd rather it stayed put, add a static mapping in **Services → DHCP Server → CORP** keyed on its MAC address — a DHCP reservation is centrally managed and visible on the firewall, which a host-side static address is not.

### Update the system

```bash
sudo apt update && sudo apt upgrade -y
```

### Check the clock before going any further

```bash
timedatectl
```

Confirm the local time matches reality and the timezone is correct. A SIEM correlates events across hosts by timestamp — if the clocks disagree, the picture it builds is wrong in ways that are hard to spot and easy to act on. This lab keeps every guest in step by enabling **Synchronize guest time with host** in Fusion (see the time synchronisation note in [Session 4](04-active-directory.md)); make sure that setting is enabled on this VM too.

## 3. Install Wazuh

Wazuh provides an assisted installation script that deploys all three components (indexer, server, dashboard) onto a single host. For a lab this is the right choice; a production deployment would separate them.

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

The `-a` flag means "all in one". The script downloads several hundred megabytes, generates certificates, and configures each component. **Expect 10–20 minutes.** It's normal for it to sit apparently idle for long stretches.

> Check the current release before running this — the version in the URL moves. `https://documentation.wazuh.com/current/installation-guide/` has the current quickstart command.

At the end it prints the dashboard credentials:

```
INFO: --- Summary ---
INFO: You can access the web interface https://<wazuh-dashboard-ip>
    User: admin
    Password: <a long generated string>
```

**Copy that password somewhere safe immediately.** You can retrieve it later with the command below, but saving yourself the trouble is easier:

```bash
sudo tar -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt -O
```

### Confirm the services are running

```bash
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-dashboard
```

All three should report `active (running)`.

## 4. Open the firewall paths

Wazuh agents on CORP need to reach the manager on SERVERS. Your default-deny design means that traffic is currently blocked, which is exactly what you'd want until you deliberately allow it.

### Create a port alias

On the pfSense web UI, **Firewall → Aliases → Ports → Add**:

| Field       | Value                                                                  |
| ----------- | ---------------------------------------------------------------------- |
| Name        | `Wazuh_Agent_Ports`                                                    |
| Type        | Port                                                                   |
| Port 1514   | Agent event forwarding (agent → manager, TCP)                          |
| Port 1515   | Agent enrolment and certificate exchange                               |

### Create a host alias

**Firewall → Aliases → IP → Add**:

| Field | Value                     |
| ----- | ------------------------- |
| Name  | `Wazuh_Manager`           |
| Type  | Host(s)                   |
| IP    | `10.10.30.10`             |

### Add the CORP rule

**Firewall → Rules → CORP → Add** (place it above the default deny):

| Field       | Value                                          |
| ----------- | ---------------------------------------------- |
| Action      | Pass                                            |
| Interface   | CORP                                            |
| Protocol    | TCP                                             |
| Source      | CORP subnets                                    |
| Destination | Single host or alias → `Wazuh_Manager`          |
| Dest. port  | `Wazuh_Agent_Ports`                             |
| Description | Allow CORP endpoints to ship logs to Wazuh      |

Save and **Apply Changes**.

### Add the MGMT rule for dashboard access

You'll want to reach the Wazuh web interface from the Kali jump box in Session 9, and from your own browser in the meantime.

| Field       | Value                                    |
| ----------- | ---------------------------------------- |
| Action      | Pass                                      |
| Interface   | MGMT                                      |
| Protocol    | TCP                                       |
| Source      | MGMT subnets                              |
| Destination | `Wazuh_Manager`                           |
| Dest. port  | `443`                                     |
| Description | MGMT access to the Wazuh dashboard        |

**Note what you have not done:** you haven't allowed CORP to reach the dashboard, or SERVERS to initiate connections into CORP. Agents connect outbound to the manager; the manager never needs to connect back. Keeping that direction one-way is the correct design and worth being deliberate about.

## 5. Log into the dashboard

From the host machine — which Fusion connects to the MGMT segment, and which is therefore the only management host on the network until the Kali box arrives in Session 9 — browse to:

```
https://10.10.30.10
```

Accept the self-signed certificate warning — the installer generates its own CA, and in a lab that's fine. In production you'd replace these with certificates from a trusted internal CA.

> **If the page never loads**, the host needs a route to the SERVERS segment. Fusion gives it an address on MGMT only, so traffic to `10.10.30.0/24` has nowhere to go until you tell it to use the firewall:
>
> ```bash
> sudo route -n add -net 10.10.30.0/24 10.10.10.254
> ```
>
> macOS discards manually added routes on reboot, so this needs repeating per session. That the host can only reach MGMT directly, and everything else has to be routed through pfSense and matched against a rule, is the segmentation working as designed.

Log in as `admin` with the generated password. You'll land on the Wazuh overview page with no agents enrolled yet.

## 6. Deploy the agent to the Domain Controller

In the dashboard, go to **Agents → Deploy new agent**. The wizard builds an install command for you.

1. Package: **Windows MSI**
2. Server address: `10.10.30.10`
3. Agent name: `WIN-DC01`
4. Agent group: `default` for now

It generates a PowerShell command. On `WIN-DC01`, open **PowerShell as Administrator** and run it — it will look approximately like:

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.12.0-1.msi -OutFile $env:tmp\wazuh-agent.msi
msiexec.exe /i $env:tmp\wazuh-agent.msi /q WAZUH_MANAGER='10.10.30.10' WAZUH_AGENT_NAME='WIN-DC01'
```

Then start the service:

```powershell
NET START WazuhSvc
```

> The DC will need to reach `packages.wazuh.com` over HTTPS to download the MSI. Your CORP egress rule allows TCP/443 to non-internal destinations, so this should work. If it doesn't, download the MSI on the host machine and transfer it in via a shared folder rather than loosening the firewall.

Back in the dashboard, **Agents** should show `WIN-DC01` as **Active** within a minute or two.

## 7. Deploy the agent to the Windows 11 client

Same process on `WIN-CLIENT01`, with `WAZUH_AGENT_NAME='WIN-CLIENT01'`. Run it from an elevated PowerShell — you'll need `CORP\adm-luke` or `CORP\Administrator` credentials at the UAC prompt, since `james.harper` is a standard user and shouldn't be able to install software. (That's the least privilege model from Session 5 doing its job.)

## 8. Confirm Windows events are arriving

This is the moment the lab becomes a security environment rather than a network.

On `WIN-CLIENT01`, as `james.harper`, run something distinctive:

```
whoami /all
```

In the Wazuh dashboard, go to **Discover** (or **Security events**), set the time range to the last 15 minutes, and search:

```
data.win.eventdata.commandLine:*whoami*
```

You should find the Event 4688 you just generated, complete with the command line, the user, and the parent process — the same data you were reading manually in Event Viewer last session, now centralised, indexed, and searchable across every enrolled host.

**Take a moment on this.** The difference between reading Event Viewer on one machine and querying process execution across an entire estate from one search bar is the difference between IT support and security operations.

### If no events appear

Work through these in order:

1. **Is the agent Active in the dashboard?** If it shows Disconnected, the firewall rule or the manager address is wrong.
2. **Is the audit policy actually enabled on the endpoint?** Run `auditpol /get /subcategory:"Process Creation"` on the client. If it says `No Auditing`, Wazuh is working correctly and there is simply nothing to collect — Windows isn't generating the events. This was an open issue at the end of Session 5 and needs resolving here.
3. **Is the agent configured to read the Security channel?** Check `C:\Program Files (x86)\ossec-agent\ossec.conf` for a `<localfile>` block with `<location>Security</location>`. It's there by default.

## 9. Trigger a deliberate alert

Collecting logs is one thing; alerting on them is the point.

On `WIN-CLIENT01`, at the login screen, attempt to log in as `corp\james.harper` with a deliberately wrong password five times. Your Session 5 lockout policy (5 attempts) will lock the account.

In the dashboard, search for:

```
rule.groups:authentication_failed
```

You should see Event 4625 (failed logon) entries, and a 4740 (account lockout). Wazuh maps these to rules with severity levels and, where applicable, MITRE ATT&CK technique IDs — failed authentication maps to **T1110 Brute Force**.

Unlock the account afterwards in Active Directory Users and Computers: right-click the user → Properties → Account → untick "Unlock account".

## 10. Linux telemetry — nothing to install

The obvious next step is to deploy an agent to the Ubuntu host so the Linux side is monitored too. Don't — it will fail, and the failure explains why:

```
dpkg: regarding wazuh-agent_4.14.9-1_amd64.deb containing wazuh-agent:
 wazuh-agent conflicts with wazuh-manager
  wazuh-manager (version 4.14.9-1) is present and installed.
```

**The manager already monitors itself.** It runs its own log collector and file integrity checking locally and registers as **agent 000**, so the Linux telemetry is there from the moment the server is installed. The two packages conflict deliberately because both want to own `/var/ossec`.

Confirm it:

```bash
sudo /var/ossec/bin/agent_control -l
```

The output lists `ID: 000` with the server's own hostname alongside your enrolled Windows agents.

Worth knowing for a production deployment: because the manager self-monitors, splitting Wazuh across multiple servers means each additional host — a dedicated indexer, say — *does* need its own agent. Only the manager host is covered automatically.

## 11. Review and clean up the firewall rules

Four sessions of adding rules have accumulated some duplication, and now is a good moment to read the whole ruleset back. Reviewing your own firewall is a routine task in security operations — rulesets grow by addition and almost never by subtraction, until nobody can say what half of them are for.

Go through each tab under **Firewall → Rules** and pay attention to the **States** column. It shows how many connections currently match each rule and how much traffic has passed through it. A rule reading `0/0 B` has never matched a single packet.

### What the review turned up

**MGMT — a redundant pair.** Two rules both granted dashboard access:

| Rule | Destination | Ports |
|---|---|---|
| MGMT access to the Wazuh dashboard | `Wazuh_Manager` (host alias) | 443 |
| Wazuh dashboard | SERVERS subnets | `Wazuh_Dashboard_Ports` |

The second is a superset of the first — the whole subnet rather than one host, and both ports rather than one. The first can never match anything the second wouldn't.

Keep the **host-specific** rule and delete the subnet-wide one. It permits access to one machine rather than to anything that ever gets built on that segment, which is the tighter expression of the same intent.

**CORP — the same pattern again.** `CORP → Wazuh_Manager : Wazuh_Agent_Ports` had passed 39 MiB of agent traffic. `CORP → SERVERS subnets : Wazuh_Agent_Ports` had passed nothing, because the first rule matches first. Deleted the broader one.

**CORP — two rules that can never match.** The Session 1 plan assumed internal DNS and NTP would live on the SERVERS segment, so rules were written for `CORP → SERVERS` on 53 and 123. The build went differently: DNS for CORP clients is the Domain Controller, which sits on CORP itself, and time now comes from the hypervisor. Both rules read `0/0 B` and always would. Deleted.

### Why this matters beyond tidiness

Every rule is something a reviewer has to understand and a change has to be tested against. A ruleset containing dead entries is one where nobody can confidently say what is actually enforced, and that uncertainty is what allows a genuinely bad rule to sit unnoticed among the harmless ones.

The States column is the quickest honest answer to "is this rule doing anything?" — and it's worth getting into the habit of checking it before adding yet another rule to work around a problem an existing rule already covers.

## 12. Take a VM snapshot

Snapshot `SRV-WAZUH01` as `Session 6 complete - Wazuh deployed, agents enrolled`. Take fresh snapshots of `WIN-DC01` and `WIN-CLIENT01` too, since both now have agents installed.

## 13. Verification checklist

- [ ] `SRV-WAZUH01` running on SERVERS at `10.10.30.10` with a static IP
- [ ] All three Wazuh services active (`indexer`, `manager`, `dashboard`)
- [ ] Dashboard reachable over HTTPS from MGMT
- [ ] pfSense aliases `Wazuh_Agent_Ports` and `Wazuh_Manager` created
- [ ] CORP → Wazuh rule in place; dashboard access restricted to MGMT
- [ ] Agents enrolled and Active for `WIN-DC01` and `WIN-CLIENT01`
- [ ] Ubuntu host visible as agent 000 (`agent_control -l`) — self-monitored by the manager, no agent install needed
- [ ] Event 4688 with a command line located in the dashboard via search
- [ ] A failed-authentication alert deliberately triggered and found
- [ ] Firewall ruleset reviewed; redundant and unmatched rules removed
- [ ] VM snapshots taken

## Lessons learned

**Traffic destined for the firewall itself always needs its own rule.** This came up for the third time in the build. The SERVERS segment had an internet egress rule using `!RFC1918_Internal`, which by definition excludes pfSense's own address — so the host could reach the internet but couldn't resolve DNS or ping its gateway. Rules written to let a segment pass *through* the firewall never cover traffic terminating *on* it. Once that clicked, the ICMP rule from Session 3 and the NTP rule from Session 4 stopped looking like exceptions and started looking like the same rule.

**Conditional DNS forwarding is how a split environment should be wired.** The Linux host needed to resolve both public names and the internal AD zone. Pointing it at the Domain Controller would have worked but tied one host to one server; pointing it at pfSense lost the AD zone. The answer was a domain override on pfSense: resolve everything normally, except `corp.lab.local`, which forwards to the DC. Configure it once on the resolver and every segment inherits it. DNSSEC has to be off for this — internal zones have no public chain of trust, so validation fails silently against them.

**An authoritative NXDOMAIN is not a broken resolver.** An hour went into debugging DNS that turned out to be working perfectly. The DC's hostname is `WIN-DC01`, and every lookup was for `dc01` — so the server was correctly answering "that name does not exist". The giveaway was in the response all along: `drill` showed the `aa` (authoritative answer) flag and an SOA record naming `win-dc01`. When a lookup fails, read the response rather than assuming the configuration is at fault, and check the name you're asking for before changing anything.

**Windows has no SSH server by default.** Obvious in hindsight, but worth stating: `ssh administrator@10.10.20.10` will hang forever because nothing is listening on port 22. Windows hosts are administered over RDP or WinRM. RDP also needs enabling explicitly — it's off by default on Windows Server.

**The host machine needs static routes to reach anything but MGMT.** Fusion gives the host an address on the management segment only. Reaching SERVERS or CORP from the Mac requires `sudo route -n add -net 10.10.30.0/24 10.10.10.254`, and macOS discards those on reboot. Mildly annoying, but it's the segmentation design working — nothing on another segment is reachable without a route and a matching firewall rule.

**`msiexec /q` hides failures.** The agent install appeared to succeed and then `NET START WazuhSvc` reported the service didn't exist. Quiet mode suppresses errors along with everything else, so a non-elevated install failed silently. Dropping `/q` and adding `/l*v` for a verbose log makes failures visible. The underlying cause was UAC: being a Domain Admin isn't the same as running elevated.

**Agent enrolment lags behind installation.** Both agents took 30–60 seconds to appear in the dashboard after the service started. Easy to start debugging a system that's merely still handshaking.

**Severity is a property of the pattern, not the event.** A single failed logon scores level 5, because people mistype passwords constantly. Severity climbs through composite rules that fire on *N events matching a pattern within a window* — repeated failures, then a lockout, then spraying across accounts. If every failed login were High, nobody would read the alerts. That correlation step is what separates a SIEM from a log archive.

**Don't screenshot credentials.** The installer prints the admin password at the end, and the obvious instinct is to capture the terminal showing "Installation finished". Anything committed to a public repo is permanent — git history keeps it even after the file is deleted.

**Firewall rules accumulate faster than they get removed.** Reading the whole ruleset back at the end of the session turned up four rules that were either redundant or could never match: two subnet-wide rules shadowed by tighter host-specific ones, and two written against an earlier plan for where DNS and NTP would live. None of them were dangerous — but none of them were doing anything either, and a ruleset nobody can account for in full is one where a genuinely bad rule has somewhere to hide. The States column makes this easy to check: a rule showing `0/0 B` has never passed a packet, and is worth either justifying or deleting.

---

**Next:** [Session 7 — Vulnerable targets in the DMZ](07-vulnerable-targets.md)
