# Session 1 — Prep & planning

> **Goal:** lock the design on paper, download every ISO, create the GitHub repo, and push the initial structure. No VMs are created in this session.
>
> **Time:** ~1–2 hours, most of which is ISO download time running in the background.

## What you'll have at the end

- A finished network diagram and IP plan committed to the repo
- All ISOs downloaded and stored locally in one organised folder
- The `Enterprise-Lab` repo created on GitHub with an initial commit pushed
- A clear picture of exactly what gets built in each future session

## 1. Lock the IP plan

Every VM gets a static IP. DHCP is enabled on each segment for guest devices, but the lab VMs themselves use static assignments so nothing moves around between reboots.

| VM             | Role                       | Segment    | IP             | Hostname       |
| -------------- | -------------------------- | ---------- | -------------- | -------------- |
| pfSense-FW01   | Firewall                   | All        | .254 on each   | pfsense-fw01   |
| KALI01         | Attacker / jump box        | MGMT       | 10.10.10.10    | kali01         |
| WIN-DC01       | Active Directory DC + DNS  | CORP       | 10.10.20.10   | dc01           |
| WIN-CLI01      | Windows 11 client          | CORP       | 10.10.20.20   | cli01          |
| WIN-CLI02      | Windows 11 client (opt.)   | CORP       | 10.10.20.21   | cli02          |
| LIN-SIEM01     | Ubuntu + Wazuh manager     | SERVERS    | 10.10.30.10   | siem01         |
| LIN-WEB01      | Ubuntu + DVWA              | DMZ        | 10.10.40.10   | web01          |
| META01         | Metasploitable2            | DMZ        | 10.10.40.20   | meta01         |

pfSense itself uses `.254` on every segment — putting the firewall at the top of the address space avoids a conflict with VMware Fusion's automatic `.1` assignment for the host on the MGMT vmnet. DHCP ranges (handed out by pfSense) reserve `.100–.200` per segment for ad-hoc VMs you might spin up later.

## 2. Firewall rule design

You'll implement these rules in Session 3. Documenting them now means you build with intent, not by trial and error.

**Default policy:** deny all inter-segment traffic. The allow rules below override the default.

| From      | To        | Allowed                                                  | Why                                                           |
| --------- | --------- | -------------------------------------------------------- | ------------------------------------------------------------- |
| MGMT      | CORP      | RDP (3389), SMB (445), WinRM (5985–6), ICMP              | Kali admins the Windows hosts                                 |
| MGMT      | SERVERS   | SSH (22), HTTPS (443), Wazuh dashboard (5601)            | Kali admins the SIEM                                          |
| MGMT      | DMZ       | All                                                      | The attacker box needs full reach into the DMZ                |
| CORP      | SERVERS   | 1514/1515 (Wazuh agent), 53 (DNS), 123 (NTP)             | Agents send logs; clients use internal DNS/NTP                |
| SERVERS   | CORP      | ESTABLISHED only                                         | The SIEM doesn't initiate to clients, only responds           |
| CORP      | DMZ       | DENY                                                     | No legitimate reason for users to reach vulnerable hosts      |
| DMZ       | ANY       | DENY (egress)                                            | Vulnerable hosts must not call out — contains compromises     |
| ANY       | WAN       | HTTP/HTTPS/DNS/NTP for MGMT, CORP, SERVERS only          | DMZ has no internet                                           |

## 3. ISO and image downloads

Start these in the background while you finish the rest of Session 1. Save them all under `~/VMs/ISOs/`.

| What                       | Where                                                                                       | Approx size |
| -------------------------- | ------------------------------------------------------------------------------------------- | ----------- |
| pfSense CE 2.7.x           | https://www.pfsense.org/download/ — pick AMD64, ISO Installer                               | ~800 MB     |
| Windows Server 2022 Eval   | https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022                     | ~5 GB       |
| Windows 11 Enterprise Eval | https://www.microsoft.com/en-us/evalcenter/evaluate-windows-11-enterprise                   | ~6 GB       |
| Ubuntu Server 24.04 LTS    | https://ubuntu.com/download/server                                                          | ~2.5 GB     |
| Kali Linux (VMware image)  | https://www.kali.org/get-kali/#kali-virtual-machines — pick the VMware 64-bit prebuilt      | ~3.5 GB     |
| Metasploitable2            | https://sourceforge.net/projects/metasploitable/                                            | ~870 MB     |
| DVWA                       | https://github.com/digininja/DVWA — clone repo, deploy onto Ubuntu later                    | <50 MB      |

Microsoft evaluation downloads require a free Microsoft account. The 180-day Server eval and 90-day Win 11 eval are sufficient — they can be reset a few times if you need longer.

## 4. Naming conventions

A small thing that adds polish. Used consistently across VMs, hostnames, AD users, and screenshots:

- **VM names:** `ROLE-NAME##` — e.g. `WIN-DC01`, `LIN-SIEM01`, `KALI01`
- **Hostnames:** lowercase short form — `dc01`, `siem01`, `kali01`
- **AD domain:** `corp.lab.local`
- **AD NetBIOS:** `CORP`
- **AD users:** `firstname.lastname` (e.g. `alice.morgan`)
- **Service accounts:** `svc-<purpose>` (e.g. `svc-wazuh`)
- **Admin accounts:** `adm-<name>` (e.g. `adm-luke`)

## 5. Create the GitHub repo

1. Sign in at github.com.
2. New repository → name `Enterprise-Lab`, public, **do not** add a README or .gitignore (we have ours already).
3. On your Mac, in Terminal:

   ```bash
   cd ~/Documents/Claude/Projects/Career/Enterprise-Lab
   git init
   git branch -M main
   git add .
   git commit -m "Session 1: initial repo skeleton, network design, ISO plan"
   git remote add origin https://github.com/<your-username>/Enterprise-Lab.git
   git push -u origin main
   ```

   If you haven't authenticated git with GitHub before, the easiest path on macOS is to install [GitHub CLI](https://cli.github.com/) (`brew install gh`) and run `gh auth login` once. It handles credentials for HTTPS pushes automatically via the macOS Keychain.

## 6. Verify before closing the session

- [ ] All seven ISOs/images downloaded and stored together
- [ ] `Enterprise-Lab` repo exists publicly on GitHub
- [ ] First commit visible on the repo page
- [ ] README renders correctly on GitHub (network diagram displays)
- [ ] You can explain why MGMT can reach the DMZ but CORP cannot

## Lessons learned

_Filled in at the end of the session._

---

**Next:** [Session 2 — VMware Fusion custom networking](02-vmware-networking.md)
