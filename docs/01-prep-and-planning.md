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

**Designing the firewall rules before building anything was the most valuable hour of this session.** Writing the segment table and the allow list up front meant every later decision had a reference to check against. The alternative — standing up the firewall and then working out what should be permitted — leads to rules added reactively whenever something breaks, which is how real networks end up with permissive rulesets nobody can justify.

**Deciding naming conventions early pays off repeatedly.** `WIN-DC01`, `adm-luke`, `svc-wazuh`, `firstname.lastname` — trivial choices in isolation, but they made every subsequent session faster because there was never a question about what to call something. It also makes the repo readable to someone who wasn't there.

**Git authentication on macOS is not obvious.** The first push failed with an SSH permission error because no key was set up. Installing GitHub CLI (`brew install gh`, then `gh auth login`) and using HTTPS was far simpler than generating and registering an SSH key — `gh` stores credentials in the macOS Keychain and Git picks them up automatically.

**A stray apostrophe will hang your shell.** Pasting a command with a comment containing "you're" into zsh opened an unterminated quoted string and left the terminal sitting at a `quote>` prompt. Ctrl+C to escape, and paste commands without prose attached.

**Evaluation licences are a real constraint worth planning around.** Windows Server 2022 gives 180 days and Windows 11 Enterprise gives 90. That's ample for a ten-session build, but it puts a clock on the lab that's worth knowing about before investing time in it.

---

**Next:** [Session 2 — VMware Fusion custom networking](02-vmware-networking.md)
