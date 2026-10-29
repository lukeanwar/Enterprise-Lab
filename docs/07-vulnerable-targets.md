# Session 7 — Vulnerable targets in the DMZ

> **Goal:** stand up two deliberately insecure hosts in the DMZ segment — Metasploitable2 and a Docker host running DVWA — confirm the firewall genuinely contains them, and bring the DMZ under SIEM visibility where it makes sense to do so.
>
> **Time:** ~1.5 hours. Lighter than Session 6.

## What you'll have at the end

- `VULN-MS2` — Metasploitable2 at `10.10.40.10`, a Linux host riddled with known vulnerable services
- `VULN-WEB01` — Ubuntu Server at `10.10.40.20` running DVWA in Docker
- Both confirmed reachable from MGMT and confirmed *unable* to reach anything else
- Firewall logs showing blocked DMZ egress attempts
- A documented decision about what you do and don't monitor in an untrusted segment

## Why a DMZ, and why deliberately broken machines

Every previous session has built something that should work. This one builds things that are supposed to be broken, which is a different exercise and an important one: you cannot practise detection without something worth detecting.

The DMZ is the segment for hosts you expect to be attacked, or that you don't fully trust. In a real network that's your public-facing web servers and mail gateways. The defining property isn't that they're exposed — it's that **compromise of a DMZ host must not give an attacker anything else.** That's why your Session 3 design blocks DMZ-initiated traffic entirely, in every direction, with logging on.

That rule is about to get tested properly for the first time.

## 1. Build the Metasploitable2 VM

Metasploitable2 ships as a pre-built VMware image rather than an ISO — there's no installation step.

1. Unzip the download from Session 1. You'll get a folder containing `.vmdk` and `.vmx` files.
2. In Fusion: **File → Open**, select the `.vmx` file.
3. When prompted about copying or moving, choose **I Copied It**.
4. Before powering on, open **Settings → Network Adapter** and set it to the **DMZ** custom network.
5. Rename the VM to `VULN-MS2` in the library for consistency.

> **Do not put this machine on a network that can reach the internet, or on "Share with my Mac".** Metasploitable2 is intentionally vulnerable in ways that are trivially exploitable by automated scanning. On DMZ, behind your default-deny rules, it's contained. Anywhere else, it's a liability. This is the single most important step in the session.

Power on and log in with the documented default credentials: `msfadmin` / `msfadmin`.

### Give it a static address

Metasploitable2 is old enough to use `/etc/network/interfaces` rather than netplan:

```bash
sudo nano /etc/network/interfaces
```

Replace the `eth0` stanza with:

```
auto eth0
iface eth0 inet static
    address 10.10.40.10
    netmask 255.255.255.0
    gateway 10.10.40.254
```

Then:

```bash
sudo /etc/init.d/networking restart
ip addr show eth0
```

No DNS is configured deliberately — this host has no internet egress, so name resolution would only fail slowly rather than fail fast.

### See what it exposes

```bash
netstat -tulpn | grep LISTEN
```

You'll see a long list: FTP on 21, SSH on 22, Telnet on 23, SMTP on 25, HTTP on 80, SMB on 139/445, MySQL on 3306, PostgreSQL on 5432, VNC on 5900, and more. Several of these are running versions with public exploits and some have backdoors built in.

Worth pausing on: this is roughly what an unpatched server left running for a decade looks like. The vulnerabilities aren't exotic — they're old software nobody updated.

## 2. Build the DVWA host

DVWA (Damn Vulnerable Web Application) is a PHP application for practising web attacks — SQL injection, XSS, command injection, file upload abuse. Running it in Docker keeps it isolated from the host OS and makes it trivial to reset after you break it.

Create a second Ubuntu Server 24.04 VM:

| Setting | Value |
|---|---|
| Memory | 2 GB |
| Processors | 2 |
| Disk | 25 GB |
| Network Adapter | **DMZ** |
| VM name | `VULN-WEB01` |

During installation, set a static address — DMZ does have DHCP from pfSense, but a target you'll be scanning repeatedly should sit at a known address:

- Subnet: `10.10.40.0/24`
- Address: `10.10.40.20`
- Gateway: `10.10.40.254`
- Name servers: `10.10.40.254`

Install OpenSSH server when offered.

### The egress problem

Here's a genuine tension. Your DMZ rules block all outbound traffic, which means this VM **cannot download Docker or anything else**. That's the containment design working exactly as intended, and it's the same problem a real DMZ host has during provisioning.

Three honest options:

1. **Temporarily allow egress, install, then remove the rule.** Add a Pass rule on the DMZ tab for TCP/443 and TCP/80 to `!RFC1918_Internal`, build the host, then delete the rule. This mirrors how real environments handle it — a maintenance window with a documented, time-boxed exception.
2. **Install Docker from the Ubuntu ISO's packaged version** before cutting egress, using `docker.io` from the archive rather than Docker's own repository.
3. **Build the VM on another segment, then move it.** Install everything on SERVERS, shut down, change the network adapter to DMZ.

**This lab uses option 1**, because the exception is explicit, logged and reversed — which is the part worth practising. Add the rule, note the time, do the work, remove the rule, and verify it's gone.

### Install Docker and DVWA

With temporary egress in place:

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo docker run -d -p 80:80 --name dvwa --restart unless-stopped vulnerables/web-dvwa
```

Confirm it's running:

```bash
sudo docker ps
curl -I http://localhost
```

You should get `HTTP/1.1 302 Found` redirecting to the login page.

### Now remove the egress rule

Go back to **Firewall → Rules → DMZ**, delete the temporary Pass rule, and **Apply Changes**. Then verify containment properly in the next section — don't assume.

## 3. Verify the containment

This is the actual security work of the session. Everything above was setup.

### From the DMZ outward — all of this should fail

On `VULN-WEB01`:

```bash
# Internet — should fail
curl -m 5 https://ubuntu.com

# The Domain Controller — should fail
ping -c 2 10.10.20.10
curl -m 5 http://10.10.20.10

# The SIEM — should fail
curl -m 5 -k https://10.10.30.10

# The other DMZ host — this one SUCCEEDS
ping -c 2 10.10.40.10
```

Every test except the last should time out. Note that intra-segment traffic between the two DMZ hosts is *not* blocked — pfSense only sees traffic that crosses between segments, so two hosts on the same subnet talk directly via the virtual switch and never reach the firewall at all.

That's worth understanding properly: **segmentation protects between segments, never within them.** An attacker who compromises `VULN-WEB01` can freely attack `VULN-MS2`. In production this is why sensitive systems get their own segment rather than sharing one, and why east-west traffic inside a segment is a recognised blind spot.

### From MGMT inward — this should work

From your Mac (add the route first):

```bash
sudo route -n add -net 10.10.40.0/24 10.10.10.254
curl -I http://10.10.40.20
```

Then open `http://10.10.40.20` in a browser. Log into DVWA with `admin` / `password`, click **Create / Reset Database**, and set the security level to **Low** under DVWA Security.

### Read the firewall logs

**Status → System Logs → Firewall** in pfSense. Filter on the DMZ interface.

Your blocked attempts from the tests above should be there — source `10.10.40.20`, destinations on CORP, SERVERS and the public internet, all blocked by the DMZ deny rule.

**Take a screenshot of this.** A firewall log showing contained egress attempts is more convincing evidence that segmentation works than any configuration page, because it shows the rule being enforced rather than merely existing.

## 4. Decide what to monitor

A reasonable instinct at this point is to install Wazuh agents on both vulnerable hosts. Think about it before doing it.

**The case against:** an agent is software running with high privileges that maintains an outbound connection from an untrusted host into your SIEM. If an attacker fully compromises a DMZ box, that connection is a path toward the thing holding all your evidence. You've also just punched a hole in a deliberately airtight deny rule.

**The case for:** you cannot detect what happens on a host you have no telemetry from, and DMZ hosts are the ones most likely to be attacked.

**How production generally resolves it:** monitor the DMZ, but shift the collection point away from the host where possible — network-based detection (an IDS watching the traffic) rather than agents on untrusted machines, and where agents are used, the data flows one way into a collector that the host cannot query.

**This lab's decision:** no agents on `VULN-MS2` or `VULN-WEB01`. The DMZ gets monitored at the network layer by Suricata in Session 8, which watches traffic crossing the firewall without requiring anything to run on the targets. The attack scenarios in Session 10 are then detected from the network side and from the attacker's own host — which is a more realistic exercise anyway, since a real attacker wouldn't leave your agent running.

Worth being able to explain this reasoning. "Why didn't you monitor those hosts?" is a good interview question, and "I decided not to, here's the trade-off" is a much stronger answer than having done it without thinking.

## 5. Take VM snapshots

Snapshot both: `Session 7 complete - clean vulnerable baseline`.

These snapshots matter more than the others. You're going to break these machines deliberately in Sessions 9 and 10, and a clean baseline to revert to means you can run the same attack repeatedly and get consistent results — which is what makes a detection scenario reproducible rather than a one-off.

## 6. Verification checklist

- [ ] `VULN-MS2` running on DMZ at `10.10.40.10`, services listening
- [ ] `VULN-WEB01` running on DMZ at `10.10.40.20` with DVWA reachable
- [ ] DVWA database initialised, security level set to Low
- [ ] Temporary egress rule added, used, and **removed** — verified gone from the DMZ rules list
- [ ] DMZ → internet blocked (verified, not assumed)
- [ ] DMZ → CORP blocked
- [ ] DMZ → SERVERS blocked
- [ ] MGMT → DMZ works
- [ ] Firewall log shows the blocked attempts
- [ ] Decision recorded on DMZ monitoring approach
- [ ] Snapshots taken on both hosts

## Lessons learned

_Filled in at the end of the session._

---

**Next:** [Session 8 — Suricata IDS and pfBlockerNG](08-suricata-pfblockerng.md)
