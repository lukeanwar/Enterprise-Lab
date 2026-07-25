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
3. **Network configuration** — this matters. The SERVERS segment has no DHCP server, so configure a static address manually. Select the interface, then **Edit IPv4 → Manual**:
   - Subnet: `10.10.30.0/24`
   - Address: `10.10.30.10`
   - Gateway: `10.10.30.254`
   - Name servers: `10.10.20.10` (the Domain Controller)
   - Search domains: `corp.lab.local`
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

### Verify connectivity before going further

Log in and confirm the basics:

```bash
ip a
ping -c 3 10.10.30.254
nslookup dc01.corp.lab.local
```

The first shows your static IP is set. The second proves you can reach the firewall. The third proves DNS resolution through the DC is working across segments — which also confirms your SERVERS-to-CORP firewall rule for DNS is correct.

If the `nslookup` fails, check the pfSense SERVERS rules allow UDP/53 to `10.10.20.10` before continuing. Wazuh doesn't strictly need it, but a server that can't resolve names will make everything else harder.

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
curl -sO https://packages.wazuh.com/4.12/wazuh-install.sh
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

From a machine on MGMT, browse to:

```
https://10.10.30.10
```

Accept the self-signed certificate warning — the installer generates its own CA, and in a lab that's fine. In production you'd replace these with certificates from a trusted internal CA.

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

## 10. Deploy the agent to the Ubuntu host itself

The Wazuh server can monitor itself, which gives you Linux telemetry alongside the Windows estate.

```bash
sudo /var/ossec/bin/agent-auth -m 10.10.30.10
```

Or simply enrol it through the dashboard wizard using the Linux package option, the same way you did for Windows.

## 11. Take a VM snapshot

Snapshot `SRV-WAZUH01` as `Session 6 complete - Wazuh deployed, agents enrolled`. Take fresh snapshots of `WIN-DC01` and `WIN-CLIENT01` too, since both now have agents installed.

## 12. Verification checklist

- [ ] `SRV-WAZUH01` running on SERVERS at `10.10.30.10` with a static IP
- [ ] All three Wazuh services active (`indexer`, `manager`, `dashboard`)
- [ ] Dashboard reachable over HTTPS from MGMT
- [ ] pfSense aliases `Wazuh_Agent_Ports` and `Wazuh_Manager` created
- [ ] CORP → Wazuh rule in place; dashboard access restricted to MGMT
- [ ] Agents enrolled and Active for `WIN-DC01`, `WIN-CLIENT01`, and the Ubuntu host
- [ ] Event 4688 with a command line located in the dashboard via search
- [ ] A failed-authentication alert deliberately triggered and found
- [ ] VM snapshots taken

## Lessons learned

_Filled in at the end of the session._

---

**Next:** [Session 7 — Vulnerable targets in the DMZ](07-vulnerable-targets.md)
