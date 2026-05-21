# Session 3 — pfSense install and firewall rules

> **Goal:** install pfSense as the first VM in the lab, give it five network interfaces (one per segment plus WAN), configure DHCP on each internal segment, and build the inter-segment firewall rules from the Session 1 design.
>
> **Time:** ~2.5 hours. This is the longest single session in the lab — most of it is firewall rule configuration.

## What you'll have at the end

- A pfSense VM running and reachable from your Mac at `https://10.10.10.254`
- Five interfaces configured: WAN, MGMT, CORP, SERVERS, DMZ
- DHCP servers active on the four internal segments
- Default-deny inter-segment policy with allow rules from the network design
- A backup of the pfSense config saved locally

By the end of Session 3 the firewall is "open for business" — every subsequent session attaches new VMs into the segments behind it.

## 1. Create the pfSense VM in Fusion

Open VMware Fusion and start a new VM.

| Setting              | Value                                                       |
| -------------------- | ----------------------------------------------------------- |
| Installation method  | Install from disc or image — pick the pfSense ISO           |
| Operating system     | Other → FreeBSD 14 64-bit                                   |
| Memory               | 2 GB                                                        |
| Processors           | 2                                                           |
| Disk                 | 20 GB                                                       |
| Display              | Defaults                                                    |
| VM name              | `pfSense-FW01`                                              |

After the VM is created but **before** you boot it, edit the VM settings to configure the network adapters. By default Fusion gives a new VM one network adapter — you need five.

In **Settings → Network Adapter**, configure the first adapter, then click **Add Device → Network Adapter** four more times. Set each one in this exact order — pfSense will detect them as `vmx0` through `vmx4` based on this order:

| Adapter | Connect to                              | Becomes in pfSense    |
| ------- | --------------------------------------- | --------------------- |
| 1       | Share with my Mac (Internet Sharing)    | WAN (em0)             |
| 2       | MGMT (Custom)                           | LAN (em1) → MGMT      |
| 3       | CORP (Custom)                           | OPT1 (em2) → CORP     |
| 4       | SERVERS (Custom)                        | OPT2 (em3) → SERVERS  |
| 5       | DMZ (Custom)                            | OPT3 (em4) → DMZ      |

Order matters. If the adapters are added out of sequence, the segment-to-interface mapping inside pfSense will be wrong and the firewall rules will end up applied to the wrong networks.

> **Interface naming:** by default Fusion presents Intel emulated NICs and pfSense names them `em0`–`em4`. If you switch the adapter type to VMware paravirtualised, pfSense will use `vmx0`–`vmx4` instead. This guide uses `em` throughout because that's the out-of-the-box behaviour; if you see `vmx` instead, the mapping is identical — just substitute one for the other.

## 2. Install pfSense

Boot the VM. The pfSense installer will load automatically. Accept the defaults at every screen unless noted:

1. **Accept** the copyright notice.
2. **Install pfSense.**
3. Keymap: default (or pick your layout).
4. Partitioning: **Auto (ZFS)** is fine for a single-disk lab VM.
5. Pool type: **stripe** (single disk).
6. Disk: pick the only disk Fusion presented.
7. Wait for the install to complete (~5 minutes).
8. **Reboot.** When prompted to "Manual Configuration", choose **No**.
9. After reboot, eject the ISO via Fusion → Virtual Machine → CD/DVD → Disconnect, so the VM boots from disk next time.

When the system finishes booting you'll see the pfSense console menu.

## 3. Assign interfaces

If you used the newer Netgate installer, WAN and LAN are already assigned (to em0 and em1). You still need to add the three optional interfaces. From the console menu:

1. Choose **1) Assign Interfaces.**
2. When asked "Should VLANs be set up now?" answer `n` — this lab uses one NIC per segment, not VLAN tagging.
3. Confirm WAN = `em0`, LAN = `em1` when prompted.
4. When asked for "Optional 1", enter `em2`. For "Optional 2", `em3`. For "Optional 3", `em4`. Press Enter on an empty input when it asks for further optionals.
5. Confirm the assignments with `y`. pfSense reloads its network configuration.

## 4. Set static IPs on each interface

Still in the console menu, choose **2) Set interface(s) IP address.**

Repeat the following for each interface in turn:

| Interface | IP address       | Subnet bits | Upstream gateway | DHCP on this interface |
| --------- | ---------------- | ----------- | ---------------- | ---------------------- |
| WAN       | leave on DHCP    | —           | —                | n/a (it's WAN)         |
| LAN       | `10.10.10.254`   | `24`        | none             | **Yes**                |
| OPT1      | `10.10.20.254`   | `24`        | none             | **Yes**                |
| OPT2      | `10.10.30.254`   | `24`        | none             | **Yes**                |
| OPT3      | `10.10.40.254`   | `24`        | none             | **Yes**                |

When prompted "Do you want to enable the DHCP server on \[interface\]?" answer **yes** for the four internal segments and use the `.100`–`.200` range from the IP plan.

When asked "Do you want to revert to HTTP as the webConfigurator protocol?" answer **no** (keep HTTPS).

> **Why `.254` and not `.1`?** VMware Fusion auto-assigns `.1` to the Mac host on any custom vmnet you've enabled host access for (MGMT in our case). If pfSense also takes `.1` on that segment, the Mac silently routes traffic to itself instead of the firewall and the web UI becomes unreachable. Putting all pfSense interfaces at `.254` — a common firewall convention anyway — avoids the conflict without disconnecting the Mac from MGMT.

## 5. Reach the web UI from your Mac

Your Mac is connected to MGMT (set up in Session 2), so it can reach pfSense directly at `https://10.10.10.254`. Open that URL in a browser.

- **Username:** `admin`
- **Password:** `pfsense`

You'll get a browser certificate warning — accept it. It's a self-signed cert and we're on a private lab network.

## 6. Run the initial setup wizard

The web UI launches a setup wizard on first login. It's nine steps; the values to set are below.

**General Information**

- **Hostname:** `pfsense-fw01`
- **Domain:** `lab.local`
- **Primary DNS:** `1.1.1.1`
- **Secondary DNS:** `9.9.9.9`
- **Override DNS:** **untick this.** When this is ticked, pfSense uses whatever DNS servers your WAN's DHCP lease provides instead of the ones you just set. For predictable name resolution under your control, leave it unticked.

**Time Server Information**

- **Time server:** `pool.ntp.org`
- **Timezone:** your local timezone

**Configure WAN Interface**

- **Configuration Type:** `DHCP`
- All other fields blank
- **Block RFC1918 private networks:** **untick this.** Your WAN sits behind your Mac's NAT, which gives it a `192.168.x.x` address. With this option enabled, pfSense would block its own WAN traffic.
- **Block bogon networks:** leave ticked.

**Configure LAN Interface**

- **LAN IP:** confirm `10.10.10.254/24`.

**Set Admin Password**

- Pick something memorable but not trivial. The default `admin / pfsense` is now retired.

After the final step, you'll land on the pfSense dashboard.

## 7. Rename the optional interfaces

Default names of OPT1/OPT2/OPT3 are unhelpful when you're writing firewall rules. Rename them:

**Interfaces → Assignments** lists the network ports. For each one, click into it and:

- LAN → rename to **MGMT** (Description field at the top of the page)
- OPT1 → rename to **CORP**, tick **Enable**
- OPT2 → rename to **SERVERS**, tick **Enable**
- OPT3 → rename to **DMZ**, tick **Enable**

Save and apply changes. The interface tabs everywhere in the UI (firewall, status, services) now show MGMT/CORP/SERVERS/DMZ instead of LAN/OPT1/OPT2/OPT3.

## 8. Confirm DHCP on each internal segment

Go to **Services → DHCP Server** and step through each of the four tabs:

| Tab     | Range start     | Range end       | DNS servers                |
| ------- | --------------- | --------------- | -------------------------- |
| MGMT    | 10.10.10.100    | 10.10.10.200    | 10.10.10.254, 1.1.1.1      |
| CORP    | 10.10.20.100    | 10.10.20.200    | 10.10.20.254, 1.1.1.1      |
| SERVERS | 10.10.30.100    | 10.10.30.200    | 10.10.30.254, 1.1.1.1      |
| DMZ     | 10.10.40.100    | 10.10.40.200    | 10.10.40.254               |

Each segment uses its own pfSense interface IP as the primary DNS (pfSense acts as the DNS resolver) and falls back to a public resolver. Two reasons for this pattern:

- **Single chokepoint for name resolution.** Every DNS lookup from a client passes through pfSense first, which means later we can log DNS queries, block known-malicious domains, and detect things like DNS tunnelling — all from one place.
- **Resilience.** If pfSense's resolver service ever stops, clients still get external name resolution via `1.1.1.1` and the lab doesn't grind to a halt.

DMZ deliberately has no public DNS fallback — there's no internet egress allowed from there anyway. If a host in the DMZ ever tries to resolve an external domain, we want that to fail (and be logged) rather than silently succeed.

**WINS Servers:** leave blank on every tab. WINS is legacy NetBIOS name resolution and isn't used in this lab. Once the Domain Controller is built in Session 4, it will handle Windows name resolution via DNS.

## 9. Build the firewall rules

This is the meaty part. Each interface has its own rules tab under **Firewall → Rules**.

**Important pfSense behaviour:** rules are evaluated on the interface where traffic **enters** the firewall. So a rule on the MGMT tab governs what MGMT-originated traffic is allowed to do. Each interface defaults to "deny all" — only the rules you add allow anything through.

> **Setting custom ports:** pfSense's Destination Port Range dropdown only lists a few dozen well-known protocols (HTTP, HTTPS, SSH, MS RDP, etc.). For anything else — WinRM (5985–5986), Wazuh agent (1514–1515), Wazuh dashboard (5601) — leave the From/To dropdowns on **`(other)`** and type the port number into the **Custom** field next to it. For a single port, only fill in the "From" field. For a range, fill in both "From" and "To".
>
> **One port per rule (unless you use an alias):** pfSense rules accept either a single port or a contiguous range — they cannot match multiple non-contiguous ports in one rule. The cleanest answer is to define **aliases** that group ports by purpose, then reference the alias in your rules. We do that next, before building any rules.

### Step 1 — Create the port aliases

Before writing any firewall rules, create four port aliases that group the ports we'll need by purpose. This collapses what would otherwise be ~25 individual rules into ~13, and makes the rule list read like English.

Go to **Firewall → Aliases → Ports → Add** for each alias below. For each one: set **Type** to `Port(s)`, set the **Name** and top-level **Description**, then add every port from the table on its own row using **Add Port** (the per-port Description column in pfSense's alias editor is where the descriptions below go). Save, then click **Apply Changes** at the top of the aliases list.

**`Win_Admin_Ports`** — Description: *Windows admin protocols (RDP, SMB, WinRM)*

| Port | Description                |
| ---- | -------------------------- |
| 3389 | RDP                        |
| 445  | SMB / CIFS                 |
| 5985 | WinRM over HTTP            |
| 5986 | WinRM over HTTPS           |

**`Wazuh_Dashboard_Ports`** — Description: *Wazuh dashboard (HTTPS + legacy Kibana port)*

| Port | Description                |
| ---- | -------------------------- |
| 443  | Wazuh dashboard (HTTPS)    |
| 5601 | Wazuh dashboard (legacy)   |

**`Wazuh_Agent_Ports`** — Description: *Wazuh agent log forwarding and enrollment*

| Port | Description                          |
| ---- | ------------------------------------ |
| 1514 | Wazuh agent data ingestion           |
| 1515 | Wazuh agent enrollment / registration |

**`Internet_Egress_Ports`** — Description: *Standard internet egress (DNS, HTTP, NTP, HTTPS)*

| Port | Description                |
| ---- | -------------------------- |
| 53   | DNS                        |
| 80   | HTTP                       |
| 123  | NTP                        |
| 443  | HTTPS                      |

### Step 1b — Create the internal-networks alias

The egress rules below say "to the internet but not to any internal segment." Express that by creating a network alias listing every internal subnet, then using an inverted destination match. This is how production firewalls scope egress — "destination `any`" is too permissive because it includes internal traffic on the same ports.

Go to **Firewall → Aliases → IP → Add**:

**`RFC1918_Internal`** — Type: `Network(s)` — Description: *All lab segments + WAN-side NAT, used to scope internet egress rules*

| Network          | Description           |
| ---------------- | --------------------- |
| `10.10.10.0/24`  | MGMT segment          |
| `10.10.20.0/24`  | CORP segment          |
| `10.10.30.0/24`  | SERVERS segment       |
| `10.10.40.0/24`  | DMZ segment           |
| `192.168.2.0/24` | WAN-side (Mac NAT)    |

Save, then **Apply Changes**. In rules below, where the destination is shown as `!RFC1918_Internal`, this means "tick the **Invert match** box in the Destination section and enter `RFC1918_Internal` as the destination alias." The resulting rule matches everything **except** the internal subnets — i.e. the actual internet.

In the firewall rule editor later, you'll reference an alias by leaving the **Destination Port Range From** dropdown on `(other)` and typing the alias name into the **Custom** field. pfSense auto-suggests as you type.

### Step 2 — Build the rules

Now the rules. Each table below corresponds to a tab under **Firewall → Rules**.

### MGMT tab

Leave the auto-generated "Anti-Lockout Rule" alone — it stops you accidentally locking yourself out of the web UI.

> **Delete the default-allow rules.** When pfSense first sets up the LAN interface, it auto-creates two permissive rules at the bottom of the LAN/MGMT rules list: "Default allow LAN to any rule" (IPv4) and "Default allow LAN IPv6 to any rule" (IPv6). These let MGMT reach *any* destination on *any* port and silently override the specific allow rules you're about to write. Delete both before continuing. If you forget, your segmentation is just for show — anything from MGMT will still be permitted everywhere by the catch-all at the bottom.
>
> **Side effect:** removing the default-allow rules also blocks ICMP to the firewall's own MGMT IP, because the Anti-Lockout Rule only covers TCP 80/443. The first rule in the table below explicitly re-allows ICMP from MGMT to "This Firewall (self)" so you can still `ping` the firewall from the management network — a basic diagnostic capability you don't want to lose.

| Action | Protocol | Source     | Destination     | Dest. ports             | Description                        |
| ------ | -------- | ---------- | --------------- | ----------------------- | ---------------------------------- |
| Pass   | ICMP     | MGMT net   | This Firewall (self) | any                | ICMP to firewall (diagnostics)     |
| Pass   | Any      | MGMT net   | DMZ net         | any                     | Admin/attacker full DMZ access     |
| Pass   | TCP      | MGMT net   | CORP net        | `Win_Admin_Ports`       | Windows admin (RDP/SMB/WinRM)      |
| Pass   | ICMP     | MGMT net   | CORP net        | any                     | Ping/troubleshooting               |
| Pass   | TCP      | MGMT net   | SERVERS net     | 22                      | SSH to Ubuntu/SIEM                 |
| Pass   | TCP      | MGMT net   | SERVERS net     | `Wazuh_Dashboard_Ports` | Wazuh dashboard                    |
| Pass   | TCP/UDP  | MGMT net   | `!RFC1918_Internal` | `Internet_Egress_Ports` | Internet egress                |

> **Why destination is `any` on the egress rules:** in pfSense, "WAN net" (or "WAN subnets") only matches traffic destined for IPs inside the directly-attached WAN subnet — i.e. the gateway range like `192.168.2.0/24`. It does **not** match traffic destined for the actual public internet (`1.1.1.1`, `8.8.8.8`, etc.). The destination for an "allow out to the internet" rule must be `any`. pfSense's routing table and outbound NAT then take care of sending the matched traffic out through the WAN interface.

### CORP tab

| Action | Protocol | Source     | Destination     | Dest. ports             | Description                        |
| ------ | -------- | ---------- | --------------- | ----------------------- | ---------------------------------- |
| Block  | Any      | CORP net   | DMZ net         | any                     | Explicit deny CORP → DMZ (log)     |
| Pass   | TCP      | CORP net   | SERVERS net     | `Wazuh_Agent_Ports`     | Wazuh agent to SIEM                |
| Pass   | TCP/UDP  | CORP net   | SERVERS net     | 53                      | Internal DNS                       |
| Pass   | UDP      | CORP net   | SERVERS net     | 123                     | Internal NTP                       |
| Pass   | TCP/UDP  | CORP net   | `!RFC1918_Internal` | `Internet_Egress_Ports` | Internet egress                |

Enable logging on the explicit deny rule so you can see attempts to break the policy.

### SERVERS tab

| Action | Protocol | Source       | Destination | Dest. ports             | Description                |
| ------ | -------- | ------------ | ----------- | ----------------------- | -------------------------- |
| Pass   | TCP/UDP  | SERVERS net  | `!RFC1918_Internal` | `Internet_Egress_Ports` | Patching/updates       |

Return traffic from SERVERS to CORP (Wazuh agent responses) is handled automatically by pfSense's state tracking — no explicit allow rule needed. SERVERS should not initiate connections to MGMT, CORP, or DMZ, so there are no further pass rules.

### DMZ tab — Firewall → Rules → DMZ

| Action | Protocol | Source     | Destination | Dest. ports | Description                                |
| ------ | -------- | ---------- | ----------- | ----------- | ------------------------------------------ |
| Block  | Any      | DMZ net    | any         | any         | Block all DMZ-initiated traffic (log)      |

Enable logging on this rule. The DMZ is meant to be a one-way zone — traffic comes in from MGMT (for testing) but nothing initiates outward. Anything in the logs here is either a misconfiguration or a sign that a vulnerable host has been compromised and is trying to phone out.

## 10. Save the configuration

**Diagnostics → Backup & Restore → Backup Configuration → Download configuration as XML.**

Save the file as `configs/pfsense/pfsense-fw01-session3.xml` in the repo. Going forward, snapshot the pfSense config at the end of each session so you always have a working baseline to revert to.

## 11. Verification

From the pfSense console (Diagnostics → Ping in the web UI, or `8) Shell` from the console menu):

- Ping `1.1.1.1` from the WAN interface — should succeed (pfSense has internet).
- Ping `10.10.20.254` from the MGMT interface — should succeed (firewall is on every segment).

From your Mac terminal:

- `ping 10.10.10.254` — should succeed.
- Open `https://10.10.10.254` — should load the pfSense dashboard.

## 12. Verification checklist

- [ ] pfSense VM running, five interfaces configured (WAN + MGMT + CORP + SERVERS + DMZ)
- [ ] WAN gets an address via DHCP from the Mac NAT
- [ ] Each internal segment has DHCP enabled with the `.100`–`.200` range
- [ ] Interfaces renamed from OPT1/2/3 to CORP/SERVERS/DMZ
- [ ] Firewall rules in place on all four internal tabs as described above
- [ ] Explicit deny rules on CORP→DMZ and DMZ→any have logging enabled
- [ ] pfSense itself can reach the internet (ping 1.1.1.1 from console works)
- [ ] Web UI reachable from the Mac at `https://10.10.10.254`
- [ ] Config XML backed up to `configs/pfsense/`

## Lessons learned

_Filled in at the end of the session._

---

**Next:** [Session 4 — Active Directory build](04-active-directory.md)
