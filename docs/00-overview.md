# 00 — Overview

## Goals

This lab exists to give me hands-on experience that mirrors a real corporate environment, so I can study for Security+ in the right context and build a portfolio piece that shows employers what I can actually do, not just what's on my CV.

Specific outcomes I want from the finished lab:

1. A segmented network behind a firewall, with documented rules between segments — proves I understand network design and firewall policy, not just "set and forget" home routing.
2. A working Active Directory domain with users, groups, and OUs — proves I understand identity, the most common attack surface in real environments.
3. A SIEM (Wazuh) ingesting logs from Windows and Linux endpoints — proves I can build the visibility layer that any SOC role depends on.
4. An IDS (Suricata) generating alerts that get correlated in the SIEM — proves I understand detection beyond endpoint logging.
5. Documented attack/detect scenarios — proves I can think from both sides of the keyboard, which is what cybersecurity actually is.

## Design principles

A few rules I'm holding myself to throughout the build:

- **Default-deny between segments.** No flat network. If a host needs to talk to another segment, there's an explicit allow rule with a comment explaining why.
- **No production data, no real personal info.** All users, hostnames, and accounts are fake.
- **Snapshots before every major change.** Cheap insurance.
- **Document as I go, not "later".** Screenshots and notes are taken during the session, written up before the session ends.
- **Reproducibility over polish.** Anyone reading this should be able to follow along and build the same lab.

## Network design at a glance

```
                ┌──────────────────────┐
                │   Internet (NAT)     │
                └──────────┬───────────┘
                           │
                       WAN │
                ┌──────────▼───────────┐
                │      pfSense FW      │
                └──┬────┬────┬────┬────┘
        MGMT       │    │    │    │      DMZ
   10.10.10.0/24   │    │    │    │   10.10.40.0/24
                   │    │    │    │
                ┌──▼─┐  │    │  ┌─▼──┐
                │Kali│  │    │  │META│
                └────┘  │    │  └────┘
                        │    │  ┌────┐
                        │    │  │DVWA│
                        │    │  └────┘
              CORP      │    │    SERVERS
         10.10.20.0/24  │    │  10.10.30.0/24
                        │    │
                      ┌─▼─┐ ┌▼────────┐
                      │DC │ │Ubuntu   │
                      └───┘ │+ Wazuh  │
                      ┌───┐ └─────────┘
                      │W11│
                      └───┘
```

## Hardware envelope

Built on macOS with VMware Fusion 13.6.4, 32 GB RAM, 600 GB+ free disk. Memory budget when everything is running simultaneously:

| VM                 | Role              | RAM    |
| ------------------ | ----------------- | ------ |
| pfSense-FW01       | Firewall          | 2 GB   |
| WIN-DC01           | Domain Controller | 4 GB   |
| WIN-CLI01          | Win 11 client     | 4 GB   |
| LIN-SIEM01         | Ubuntu + Wazuh    | 8 GB   |
| LIN-WEB01          | Ubuntu + DVWA     | 1 GB   |
| META01             | Metasploitable2   | 0.5 GB |
| KALI01             | Kali Linux        | 4 GB   |
| **Total**          |                   | **~24 GB** |

Leaves ~8 GB for the macOS host. Comfortable.

## Next

See [`01-prep-and-planning.md`](01-prep-and-planning.md) for the Session 1 build steps.
