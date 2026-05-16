# Session 2 — VMware Fusion custom networking

> **Goal:** create four custom virtual networks in VMware Fusion that match the MGMT, CORP, SERVERS, and DMZ segments from the network design, and configure each so that pfSense — not Fusion — handles DHCP and routing.
>
> **Time:** ~30 minutes. No VMs are installed in this session.

## What you'll have at the end

- Four custom virtual networks in Fusion, one per segment
- Each set to host-only mode with VMware's built-in DHCP server disabled
- The Mac host able to reach the MGMT network directly (so you can SSH or RDP into Kali from a Mac terminal later)
- The Mac host walled off from CORP, SERVERS, and DMZ, forcing all traffic between those segments through pfSense
- The default `vmnet8` (Share with my Mac / NAT) left in place — pfSense will use this for its WAN interface in Session 3

## Why we disable VMware's DHCP

pfSense will hand out DHCP addresses on each internal segment when we install it in Session 3. If VMware Fusion is also running a DHCP server on the same subnet, you'll get a race condition between the two and VMs will end up with the wrong gateway. Turning Fusion's DHCP off on every custom network is the cleaner long-term setup, and it's how a real network would be — one authoritative DHCP server per segment.

## Why MGMT is the only network the Mac can reach

The Mac is effectively the physical management workstation sitting outside the lab. Letting it reach MGMT directly means you can drop into Kali quickly without having to console in via VMware. Walling it off from CORP, SERVERS, and DMZ keeps every other segment routed through pfSense, which makes the firewall the single chokepoint for inter-segment traffic — exactly like a real corporate network. It also means the firewall rules you build in Session 3 actually get exercised.

## 1. Confirm your Fusion version

You need VMware Fusion **13.5 or newer** for the custom network UI used in this guide. Open Fusion, then in the menu bar choose **VMware Fusion → About VMware Fusion**. The version should read 13.5 or higher (mine is 13.6.4).

If you're on an older version, update through Fusion → Check for Updates before continuing.

## 2. Open Network Settings

In the menu bar, choose **VMware Fusion → Settings → Network**. You'll be asked for your Mac admin password before any changes can be made — Fusion needs elevated permissions to modify network interfaces on the host. Click the padlock at the bottom-left to unlock.

## 3. Add the four custom networks

For each segment in the table below, click the **+** button at the bottom of the network list and configure the network as follows.

| Name in Fusion | Subnet            | Allow Mac to connect | Provide addresses (DHCP) |
| -------------- | ----------------- | -------------------- | ------------------------ |
| MGMT           | 10.10.10.0/24     | **Yes**              | **No** (off)             |
| CORP           | 10.10.20.0/24     | No                   | **No** (off)             |
| SERVERS        | 10.10.30.0/24     | No                   | **No** (off)             |
| DMZ            | 10.10.40.0/24     | No                   | **No** (off)             |

For each new network:

1. Click **+** to create a new vmnet. Fusion will assign it a number (vmnet2, vmnet3, etc.) automatically.
2. Rename it to the segment name (MGMT, CORP, SERVERS, or DMZ) in the **Name** field.
3. Leave **Provide addresses on this network via DHCP** ticked for now. Enter the subnet in the **Subnet IP** field — e.g. `10.10.10.0` — and leave the netmask as `255.255.255.0`. Click **Apply**.
4. Now untick **Provide addresses on this network via DHCP** and click **Apply** again. This disables Fusion's built-in DHCP server for that vmnet while keeping the subnet configured.
5. Set **Connect the host Mac to this network** to **on** for MGMT only, **off** for the other three.
6. Leave **Allow virtual machines on this network to connect to external networks (using NAT)** unticked on all four. If you tick this, Fusion will NAT the segment to your Mac's internet connection and bypass pfSense entirely.

> **Fusion UI quirk:** the Subnet IP and Subnet Mask fields are only editable while DHCP is enabled. If you uncheck DHCP first, the fields grey out and you can't enter the subnet. The two-step "Apply, then disable DHCP, then Apply again" sequence above works around this — the subnet configuration persists on the vmnet after DHCP is turned off, even though the UI no longer shows it.

## 4. Confirm the existing networks

Two networks should already exist from the default Fusion install. Leave them as they are:

- **vmnet0** — bridged. Not used by this lab but harmless to leave.
- **vmnet8** — NAT, labelled "Share with my Mac". This is what pfSense will use for its WAN interface in Session 3, so the firewall has a path to the internet.

## 5. Apply and verify

After all four custom networks are configured, click **Apply** on the Network settings panel. Fusion may briefly disconnect existing VMs while the new networks come up — that's expected.

The most reliable way to verify the configuration on macOS Fusion is to read Fusion's networking config file directly. `ifconfig | grep vmnet` does not work on Fusion 13.x for macOS — the underlying network stack uses Apple's vmnet framework, and the host-side interfaces only appear as `bridge` interfaces (and only for networks where the Mac is connected).

From a terminal:

```bash
cat /Library/Preferences/VMware\ Fusion/networking
```

For each custom segment you should see a block of lines like the following (the `VNET_n` number depends on the order Fusion assigned):

```
answer VNET_2_DISPLAY_NAME MGMT
answer VNET_2_HOSTONLY_SUBNET 10.10.10.0
answer VNET_2_HOSTONLY_NETMASK 255.255.255.0
answer VNET_2_DHCP no
answer VNET_2_VIRTUAL_ADAPTER yes
```

What to check across all four custom networks:

- `DISPLAY_NAME` matches the segment name (MGMT, CORP, SERVERS, DMZ).
- `HOSTONLY_SUBNET` matches the planned subnet (10.10.10.0, 10.10.20.0, 10.10.30.0, 10.10.40.0).
- `HOSTONLY_NETMASK` is `255.255.255.0` on every one.
- `DHCP no` appears for every custom segment, confirming Fusion's DHCP is disabled.
- `VIRTUAL_ADAPTER yes` appears only on MGMT. On CORP, SERVERS, and DMZ it should be `no` (or absent).

You can also confirm the Mac's connection to MGMT specifically from the host side:

```bash
ifconfig | grep -E "bridge|vmenet"
```

You should see at least one `bridge` interface with an IP in `10.10.10.0/24`. The other three networks won't appear in `ifconfig` output because the Mac isn't connected to them, which is by design.

## 6. Verification checklist

- [ ] Four custom networks created in Fusion (MGMT, CORP, SERVERS, DMZ)
- [ ] Each custom network has DHCP disabled
- [ ] MGMT is the only custom network the Mac host can connect to
- [ ] `vmnet8` (Share with my Mac) still exists for pfSense's WAN
- [ ] The Fusion networking config file shows all four custom networks with the correct subnets, DHCP off, and the host adapter enabled only on MGMT

## Lessons learned

_Filled in at the end of the session._

---

**Next:** [Session 3 — pfSense install and firewall rules](03-pfsense.md)
