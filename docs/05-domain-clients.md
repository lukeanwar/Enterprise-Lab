# Session 5 — Domain clients and Group Policy

> **Goal:** build a Windows 11 client VM in the CORP segment, join it to `corp.lab.local`, log in as a domain user, then create Group Policy Objects that enforce a password policy, harden the workstation, and enable the security auditing that the Wazuh SIEM will rely on in Session 6.
>
> **Time:** ~2 hours.

## What you'll have at the end

- A Windows 11 Enterprise VM (`WIN-CLIENT01`) joined to `corp.lab.local`
- The computer account moved out of the default `Computers` container into the `Workstations` OU
- A domain user (`james.harper`) successfully logged in on the client
- Three Group Policy Objects in place:
  - **Default Domain Policy** — edited to enforce password complexity, length, and account lockout
  - **Lab - Security Auditing Baseline** — advanced audit policies, command-line process creation, PowerShell script-block logging, linked at the domain root
  - **Lab - Workstation Lockdown** — screen-lock timeout and UAC enforcement, linked at the `Workstations` OU
- `gpresult` on the client confirming all three GPOs are applying

## Why GPOs matter for security

Group Policy is how Windows administrators push configuration to thousands of machines at once. It's also where a lot of the visibility a SOC depends on actually gets turned on. By default, Windows logs very little of what happens on a host. No process creation events. No PowerShell script content. Limited logon detail. Without the right audit policies enabled via GPO, a SIEM agent sat on a Windows host has very little of interest to forward.

Building this layer now means Session 6's Wazuh deployment has something worth ingesting, and Session 10's detection scenarios have real telemetry to detect against.

## 1. Create the Windows 11 VM in Fusion

In VMware Fusion, create a new VM.

| Setting              | Value                                                       |
| -------------------- | ----------------------------------------------------------- |
| Installation method  | Install from disc or image — pick the Windows 11 Enterprise ISO |
| Operating system     | Microsoft Windows → Windows 11 64-bit                       |
| Firmware             | UEFI with Secure Boot (default for Windows 11)              |
| TPM                  | Enabled (Fusion adds a virtual TPM 2.0 automatically for Windows 11) |
| Memory               | 4 GB                                                        |
| Processors           | 2                                                           |
| Disk                 | 60 GB                                                       |
| VM name              | `WIN-CLIENT01`                                              |

Before booting, edit **Settings → Network Adapter** and set the adapter to the **CORP** custom network. The default "Share with my Mac" must be changed — the client lives on the CORP segment so it can reach the DC for DNS and authentication.

## 2. Install Windows 11

Boot the VM. The Windows installer will load.

1. Language, time, and keyboard: defaults.
2. **Install now.**
3. Edition: **Windows 11 Enterprise** (Evaluation).
4. License terms: accept.
5. Installation type: **Custom: Install Windows only (advanced).**
6. Drive: select the only disk, click Next, let it install. Takes ~10–15 minutes including reboots.

### Local account workaround during OOBE

Windows 11 22H2 and later push you hard towards signing in with a Microsoft account during the out-of-box setup, even on the Enterprise edition. You don't want that — domain-joined machines authenticate against AD, not Microsoft's cloud. To force a local account creation:

1. On the **"Let's connect you to a network"** screen, press **Shift + F10** to open a command prompt.
2. Type `start ms-cxh:localonly` and press Enter. (On older builds the command is `oobe\bypassnro` followed by a reboot, then "I don't have internet" → "Continue with limited setup".)
3. A "Create local account" dialog appears. Use a simple temporary name — `labadmin` is fine — and a password you'll remember. This local account is just for the initial setup; you'll log in as a domain user once the machine is joined.
4. Decline every privacy and telemetry prompt and finish OOBE.

## 3. Initial client configuration

Once logged in, do the following before joining the domain.

### Verify network and DNS

The client should have picked up DHCP from pfSense. Open a Command Prompt and run:

```
ipconfig /all
```

Confirm:

- IPv4 address in `10.10.20.0/24` (typically `10.10.20.100`–`10.10.20.199`)
- Default Gateway: `10.10.20.254`
- DNS Servers: `10.10.20.10` (the DC, **not** `1.1.1.1`)

If DNS is showing anything other than `10.10.20.10`, the pfSense CORP DHCP scope wasn't updated correctly in Session 4 — go back and check it before continuing. A Windows machine that doesn't use the DC for DNS will not be able to find the domain.

Then check the DC is resolvable:

```
nslookup corp.lab.local
nslookup dc01.corp.lab.local
```

Both should return `10.10.20.10`.

### Rename the computer

Default name is something like `DESKTOP-XXXXXX`. Rename it to `WIN-CLIENT01` to match the VM.

- Right-click **Start → System → About → Rename this PC**
- Enter `WIN-CLIENT01`
- Restart when prompted.

## 4. Join the domain

After the rename reboot, log back in as the local `labadmin` account.

- Right-click **Start → System → About → Domain or workgroup**
- Click **Change** next to "To rename this computer or change its domain or workgroup..."
- Select **Domain** and enter `corp.lab.local`
- Click OK
- When prompted for credentials, use `corp\adm-luke` (the Domain Admin account you created in Session 4) and its password
- A "Welcome to the corp.lab.local domain" dialog confirms the join. Restart.

## 5. Log in as a domain user

On the login screen after reboot, click **Other user** in the bottom left. Sign in as:

- **Username:** `corp\james.harper`
- **Password:** the lab password you set in Session 4

First login takes a few seconds as Windows builds a fresh user profile. When the desktop appears, you're authenticated by Active Directory — Kerberos tickets, group memberships, the lot.

Open a Command Prompt and run:

```
whoami
whoami /groups
```

`whoami` should return `corp\james.harper`. `whoami /groups` should show `CORP\IT-Staff` in the group list, proving group membership flows through correctly from AD.

## 6. Move the computer account to the Workstations OU

By default, a freshly joined machine lands in the built-in `Computers` container at the root of the domain. That container can't have GPOs linked to it, which means workstation-targeted policies wouldn't apply. Move the account into the `Workstations` OU you created in Session 4.

On the DC, open **Active Directory Users and Computers**:

1. Expand `corp.lab.local → Computers`.
2. Right-click `WIN-CLIENT01` → **Move**.
3. Pick `Workstations` and click OK.

This is also a real-world enterprise pattern. The default `Computers` container is treated as a holding pen — joined machines are routinely moved into structured OUs so the right policies apply.

## 7. Group Policy Object 1 — Default Domain Policy (password and lockout)

Open **Group Policy Management** from Server Manager → Tools, on the DC.

Expand `Forest → Domains → corp.lab.local`. The built-in **Default Domain Policy** is already linked at the root.

Right-click **Default Domain Policy → Edit**. The Group Policy Management Editor opens.

Navigate to **Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies**.

### Password Policy

| Setting                                          | Value         |
| ------------------------------------------------ | ------------- |
| Enforce password history                         | 10 passwords  |
| Maximum password age                             | 90 days       |
| Minimum password age                             | 1 day         |
| Minimum password length                          | 12 characters |
| Password must meet complexity requirements       | Enabled       |
| Store passwords using reversible encryption      | Disabled      |

### Account Lockout Policy

| Setting                            | Value                        |
| ---------------------------------- | ---------------------------- |
| Account lockout duration           | 15 minutes                   |
| Account lockout threshold          | 5 invalid logon attempts     |
| Reset account lockout counter after| 15 minutes                   |

Close the editor.

This password policy applies to every user in the domain. The values aren't arbitrary — they're roughly aligned with current NIST 800-63B guidance and what you'll see in most corporate environments.

## 8. Group Policy Object 2 — Security Auditing Baseline

This is the GPO that makes the lab actually visible to a SIEM. Without it, Wazuh agents will be forwarding very thin logs.

In Group Policy Management:

1. Right-click `corp.lab.local` → **Create a GPO in this domain, and Link it here...**
2. Name: `Lab - Security Auditing Baseline`
3. Right-click the new GPO → **Edit**

### Advanced Audit Policy Configuration

Navigate to **Computer Configuration → Policies → Windows Settings → Security Settings → Advanced Audit Policy Configuration → Audit Policies**.

Set the following sub-categories. For each one, double-click → tick **Configure the following audit events** → tick **Success** and/or **Failure** as shown:

| Category             | Sub-category                          | Success | Failure |
| -------------------- | ------------------------------------- | :-----: | :-----: |
| Account Logon        | Audit Credential Validation           |   ✓     |   ✓     |
| Account Logon        | Audit Kerberos Authentication Service |   ✓     |   ✓     |
| Account Logon        | Audit Kerberos Service Ticket Operations |  ✓  |   ✓     |
| Account Management   | Audit User Account Management         |   ✓     |   ✓     |
| Account Management   | Audit Security Group Management       |   ✓     |   ✓     |
| Detailed Tracking    | Audit Process Creation                |   ✓     |         |
| Logon/Logoff         | Audit Logon                           |   ✓     |   ✓     |
| Logon/Logoff         | Audit Logoff                          |   ✓     |         |
| Logon/Logoff         | Audit Account Lockout                 |         |   ✓     |
| Logon/Logoff         | Audit Special Logon                   |   ✓     |         |
| Policy Change        | Audit Audit Policy Change             |   ✓     |   ✓     |
| Privilege Use        | Audit Sensitive Privilege Use         |         |   ✓     |
| System               | Audit Security State Change           |   ✓     |         |

### Include command line in process creation events

Process creation auditing on its own logs that `cmd.exe` ran, not what arguments it was given. The interesting part — `powershell.exe -enc <base64>` or `whoami /priv` — lives in the command line. Turn it on:

**Computer Configuration → Policies → Administrative Templates → System → Audit Process Creation**

- **Include command line in process creation events** → **Enabled**

This populates the `CommandLine` field on Windows Event ID 4688, which is one of the highest-signal events you can give to a SIEM.

### PowerShell logging

PowerShell is the de facto language of post-exploitation on Windows. Logging it well is one of the highest-ROI things you can do for detection.

**Computer Configuration → Policies → Administrative Templates → Windows Components → Windows PowerShell**

- **Turn on Module Logging** → **Enabled** → in Options, set Module Names to `*`
- **Turn on PowerShell Script Block Logging** → **Enabled** (leave "Log script block invocation start / stop events" unticked for now — it's noisy)
- **Turn on PowerShell Transcription** → leave **Not Configured** for now (the transcript files balloon quickly without a sensible output path)

Close the editor.

## 9. Group Policy Object 3 — Workstation Lockdown

A small GPO scoped to the `Workstations` OU, so it applies to clients but not to the DC.

In Group Policy Management:

1. Right-click the `Workstations` OU → **Create a GPO in this domain, and Link it here...**
2. Name: `Lab - Workstation Lockdown`
3. Right-click the new GPO → **Edit**

### Screen lock

**User Configuration → Policies → Administrative Templates → Control Panel → Personalization**

- **Enable screen saver** → **Enabled**
- **Password protect the screen saver** → **Enabled**
- **Screen saver timeout** → **Enabled**, Seconds = `600` (10 minutes)
- **Force specific screen saver** → **Enabled**, Screen saver executable name = `scrnsave.scr` (the blank screen saver)

### User Account Control

**Computer Configuration → Policies → Windows Settings → Security Settings → Local Policies → Security Options**

- **User Account Control: Behaviour of the elevation prompt for administrators in Admin Approval Mode** → **Prompt for consent on the secure desktop**
- **User Account Control: Run all administrators in Admin Approval Mode** → **Enabled**

Close the editor.

## 10. Apply and verify

On `WIN-CLIENT01`, log in as `corp\james.harper` again (or stay logged in from earlier), open Command Prompt as Administrator and run:

```
gpupdate /force
```

You should see "Computer Policy update has completed successfully" and the same for User Policy. Then run:

```
gpresult /r
```

Look for the **Applied Group Policy Objects** section. You should see at least:

- `Default Domain Policy`
- `Lab - Security Auditing Baseline`
- `Lab - Workstation Lockdown` (under User Settings)

If `Lab - Workstation Lockdown` doesn't appear, the computer account is probably still sitting in the default `Computers` container — go back to step 6.

### Confirm audit policy is active

In an elevated Command Prompt:

```
auditpol /get /category:*
```

This dumps the effective audit policy for the machine. Confirm that the categories you enabled in Section 8 are showing **Success and Failure** (or whatever you set them to). If they all still say "No Auditing", the GPO hasn't applied yet — run `gpupdate /force` again and check the link order.

### Confirm process creation logging is working

In the same Command Prompt window:

```
whoami /all
```

Then open **Event Viewer → Windows Logs → Security**, filter on Event ID `4688`, and find the event for `whoami.exe`. The `Process Command Line` field should show the full `whoami /all` command, not just the binary name. If the command line is empty, the "Include command line" admin template setting didn't apply — re-check Section 8.

## 11. Take a VM snapshot

In Fusion: take a snapshot on `WIN-CLIENT01` named `Session 5 complete - domain joined, GPOs applied`. Also take a fresh snapshot of `WIN-DC01` named `Session 5 complete - GPOs configured` so the DC's policy state is captured too.

## 12. Verification checklist

- [ ] WIN-CLIENT01 running on the CORP segment, DHCP from pfSense, DNS pointing at `10.10.20.10`
- [ ] Domain-joined to `corp.lab.local`
- [ ] Computer account moved into the `Workstations` OU
- [ ] Logged in as `corp\james.harper`; `whoami /groups` shows `CORP\IT-Staff`
- [ ] Default Domain Policy edited with the password and lockout settings
- [ ] `Lab - Security Auditing Baseline` GPO created and linked at the domain root
- [ ] `Lab - Workstation Lockdown` GPO created and linked at the Workstations OU
- [ ] `gpresult /r` on the client lists all three GPOs as applied
- [ ] `auditpol /get /category:*` shows Success/Failure auditing where configured
- [ ] Event ID 4688 in the Security log shows the command line for a test process
- [ ] VM snapshots taken on both DC01 and WIN-CLIENT01

## Lessons learned

**Least privilege stops you doing things — and that's the point.** Signed in as `james.harper` (a standard user), several administrative checks refused to work: elevating a Command Prompt required admin credentials, and Event Viewer's Security log returned "Access is denied (5)". Both of those are the security model working correctly, not bugs. In a real environment, `james.harper` shouldn't be able to read Security events or self-elevate — if he could, any malware running under his session could too. The workaround is the same as it would be in production: use a separate admin account (`adm-luke`) for admin tasks.

**An elevated shell runs as whoever authenticated to UAC, not the logged-in user.** Running `gpresult /r` from a Command Prompt elevated with `CORP\Administrator` credentials returned "The user does not have RSoP data" — because Administrator has never logged onto the machine interactively and has no user-side policy result set cached. `gpresult /r /scope:computer` bypasses the user-side check entirely and dumps the machine's applied GPOs, which is what you actually want when verifying a computer-scoped policy.

**`gpresult /r` from a normal prompt only shows user settings.** Computer Settings are hidden unless the tool is elevated. That default is worth remembering — a lot of "the policy didn't apply" tickets are actually just people looking at the wrong half of the output.

**Windows 11's OOBE fights hard against local accounts.** The `Shift+F10 → start ms-cxh:localonly` trick worked on this build. On older ISOs it's `oobe\bypassnro` followed by a reboot. Every domain-joined enterprise deployment hits this — worth having both options memorised.

**Advanced audit policy proved unreliable to apply, and diagnosing it was the most instructive part of the session.** The GPO showed `Audit Process Creation = Success` in the editor, `gpresult /r /scope:computer` confirmed `Lab - Security Auditing Baseline` was applied, no legacy audit policy was defined anywhere to conflict with it — and yet `auditpol /get /subcategory:"Process Creation"` on the client reported `No Auditing`. Process creation events appeared briefly at boot and then stopped.

The diagnostic path was worth walking: check the GPO's actual settings, check the legacy `Local Policies → Audit Policy` node for conflicts, check the `Audit: Force audit policy subcategory settings` security option (which was `Not Defined` and is now explicitly `Enabled`, matching CIS benchmark guidance), then test whether a policy refresh clears the setting. Setting the subcategory manually with `auditpol /set /subcategory:"Process Creation" /success:enable` confirmed the logging pipeline itself works end to end — Event 4688 fired with the full command line populated.

**What fixed it** was defining `Audit: Force audit policy subcategory settings (Windows Vista or later)` explicitly as **Enabled**, rather than leaving it `Not Defined` and relying on the operating system default. The setting has survived reboots since. It appears in the CIS benchmarks for exactly this reason — an unstated default is not the same as a stated one, and depending on one is how configuration drifts without anyone noticing.

The lesson for a SOC context is the important one: **a GPO reporting as "applied" does not mean its settings are in effect on the endpoint.** `gpresult` tells you a policy object was delivered; only `auditpol` tells you what the machine is actually auditing. Anyone relying on Group Policy for detection coverage needs to verify at the endpoint, not the policy layer, or they end up with blind spots they don't know about.

**A computer account has to be moved out of the default `Computers` container before OU-linked GPOs will apply.** `Lab - Workstation Lockdown` did not reach the client until `WIN-CLIENT01` was moved into the `Workstations` OU. Anyone building a new AD environment will trip over this at least once.

---

**Next:** [Session 6 — Ubuntu Server and Wazuh SIEM](06-ubuntu-wazuh.md)
