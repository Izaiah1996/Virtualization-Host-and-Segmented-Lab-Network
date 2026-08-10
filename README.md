# Virtualization-Host-and-Segmented-Lab-Network
Taking a working Active Directory lab off my home network and rebuilding it as a proper isolated environment on consumer hardware. This is the foundation every other lab in the track plugs into.
Status: Complete
Date: July 2026
Track: The Frozen Curriculum, Phase 1, Lab 1
Cert pairing: CompTIA Network+

I'm a WGU Cybersecurity student headed for helpdesk, then SOC work, then detection
engineering. Before this, every lab I built was throwaway. It ran once, I grabbed a
screenshot, and it died. This one had to be different. It had to persist, and it
had to be something the next twenty-three labs could actually build on.

The curriculum I'm following originally called for Proxmox as the hypervisor. I cut
it. I already had a working VirtualBox setup on the laptop I use every day, and
tearing that down to install bare-metal Proxmox would have eaten my whole build
window without teaching me anything Lab 1 actually needed. The point of Lab 1 is a
segmented virtualization host, not a specific hypervisor. VirtualBox gets me there.
Proxmox stays on the list for when I have hardware to dedicate to it.

The setup

| Component | Detail |
|---|---|
| Host machine | HP laptop, Ryzen 7, 32 GB RAM |
| Host OS | Windows 11 |
| Hypervisor | Oracle VirtualBox |
| Lab storage | LaCie external 1 TB, mounted as CyberLab (E:), 931 GB usable |

The thing that actually limits me here is RAM, not disk. 931 GB holds way more VMs
than 32 GB of memory can run at the same time. So every lab from here on gets its
memory sized on purpose instead of just taking whatever VirtualBox defaults to.


Phase 0 — Getting the lab off my system drive

The lab used to live on my system drive, mixed in with everything else I do: music
production, content automation, coursework. That works right up until it doesn't. VM
disks grow, snapshots pile up, and a full system drive on the machine you use for
everything is just a bad day waiting to happen.

So I wiped and formatted the external drive and gave the lab its own home, with a
structure that keeps the related pieces together.

CyberLab (E:) 931 GB
├── VirtualBox VMs
│ ├── DC01
│ ├── CLIENT01
│ └── CyberLab-Ubuntu2-ROS
├── ISOs
├── Snapshots
├── PCAPs
├── Logs
├── Documentation
├── Music
└── Content Automation


The PCAPs, Logs, and Documentation folders are empty right now, but Labs 2, 4, and
5 are going to fill them. Building the structure before I need it means my captures
and logs land somewhere sensible from day one instead of scattering across my
desktop.

Making sure nothing broke in the move

Before touching anything else, I booted each VM one at a time, logged in, and shut
it down clean. If a move breaks something, it shows up at boot. Catching it now
beats catching it after I've stacked a network config on top and have to unwind two
things instead of one.

All three came up. Both Windows VMs threw the same warning:

The Windows Server ISO file can't be found.
C:\Users\izaia\Downloads\SERVER_EVAL_x64FRE_en-us.iso


Looks scary, but isn't. VirtualBox just remembers whatever was in the virtual DVD drive,
and the install ISO used to sit on my system drive, which is gone now. The VDI, the
disk with the actual installed OS on it, moved fine.

Fix: I detached the missing ISO from the DVD drive on both VMs instead of trying
to repoint it. These servers are already installed. They don't need the install disc
anymore, and leaving it attached just invites confusion later about boot order and
missing media. A clean lab VM has one disk and an empty DVD drive.

---

Phase 1 — Confirming Active Directory actually survived

Moving a domain controller is exactly where labs break quietly. It boots, you log in
on cached credentials, everything looks fine, and then three weeks later nothing can
authenticate and you have no idea why. So I checked before building anything on top
of it.

Is the domain healthy

```powershell
Get-ADDomain
```
<img width="876" height="621" alt="screenshots-03-services-dns" src="https://github.com/user-attachments/assets/01b75d12-c516-4fe7-b1f1-7a3b22e5260c" />

<img width="920" height="498" alt="screenshots-01-get-aadomain" src="https://github.com/user-attachments/assets/4f9318fd-a236-4d28-bb9f-e392b557d0d0" />

Came back clean, single domain forest:

| Attribute | Value |
|---|---|
| Domain | zaylab.local |
| NetBIOS name | ZAYLAB |
| Forest | zaylab.local |
| Domain mode | Windows2016Domain |
| Domain SID | S-1-5-21-4013630350-1181435827-1393660964 |
| Domain GUID | a6c264a4-e678-4a55-875c-cf162323aa1a |
| Infrastructure Master | DC01.zaylab.local |
| PDC Emulator | DC01.zaylab.local |
| RID Master | DC01.zaylab.local |

All the FSMO roles are on DC01, the SYSVOL policy object is still linked, and the
references to ForestDnsZones and DomainDnsZones are intact. Nothing got lost in the
move.

Are the services up and is DNS answering

```powershell
Get-Service NTDS,DNS,Netlogon | Select Name,Status
Resolve-DnsName zaylab.local
```

<img width="876" height="621" alt="screenshots-03-services-dns" src="https://github.com/user-attachments/assets/fafd3a24-593e-4746-ba8f-38966645c682" />

NTDS, DNS, and Netlogon all Running, and zaylab.local resolved to 192.168.1.10.

These three are basically what make a domain controller a domain controller. NTDS is
the directory database, DNS is how clients find the domain in the first place, and
Netlogon publishes the records that make that discovery work. If even one of them is
down, the domain kind of half-works in ways that are genuinely miserable to track
down.

Does the client still trust the domain

```powershell
Test-ComputerSecureChannel -Verbose
nltest /dsgetdc:zaylab.local
```

<img width="952" height="533" alt="screenshots-04-securchannel-nltest" src="https://github.com/user-attachments/assets/d00602c5-aa8a-4d1d-bcd3-6d07be4fc819" />

Got True back, and it found DC01 at 192.168.1.10 with the full flag set: PDC, GC,
DS, LDAP, KDC, TIMESERV, GTIMESERV, WRITABLE, DNS_DC, and the rest.

The secure channel is the shared password between a workstation and the domain. VM
restores and snapshot rollbacks are the classic way to quietly break it, and when it
breaks you don't find out until someone can't log in. Worth confirming.

Snapshot taken: `BASELINE-POST-MIGRATION-CLEAN` on both VMs.

---

Phase 2 — Getting off my home network and designing the segments

At this point DC01 was sitting at 192.168.1.10 on my actual apartment network,
pulling its address from my regular router. That's a problem for a few reasons.

For one, it's just bad practice. A domain controller shouldn't be reachable from my
general network, and a lab where I fully intend to run malware and attack tools down
the line should never touch a network I use for real life. It's also fragile. Change
routers, change networks, get handed a different address, and the whole thing breaks
for reasons that have nothing to do with the lab. And most importantly, it makes the
later labs impossible. Lab 2 is a firewall routing between segments. You can't route
between segments that don't exist yet.

Internal network

VirtualBox Internal Networks are isolated segments. VMs on the same named internal
network can see each other, and nothing else can, not even the host. That's exactly
the isolation I want. Both VMs were already on an internal network called `LABNET`.

<img width="710" height="637" alt="screenshots=05=labnet-adapter" src="https://github.com/user-attachments/assets/d2b67c7e-60b1-45f3-9c29-d1203705aaf0" />

One thing to watch: VirtualBox matches these names literally, so `LABNET` and
`LabNet` are two completely different networks and it won't warn you. If the names
don't match exactly, your VMs just quietly can't see each other. One name,
everywhere.

The segment plan

The curriculum wants management, attacker, and victim segments eventually. But right
now there's nothing to put in three of them. Creating empty networks just to say I
did would be pointless. So instead I designed the full plan, reserved the address
ranges, and only built the one segment that actually has machines in it. I also
moved off 192.168.1.0/24, since that's the default range on basically every home
router and keeping it is asking for a collision later.

| Segment | VirtualBox name | Subnet | Purpose | Status |
|---|---|---|---|---|
| Lab / AD | LABNET | 10.10.10.0/24 | DC01, CLIENT01, future domain hosts | Built |
| Management | LABNET-MGMT | 10.10.20.0/24 | SIEM, monitoring, admin plane | Planned |
| Attacker | LABNET-RED | 10.10.30.0/24 | Kali, offensive tooling | Planned |
| DMZ / victim | LABNET-DMZ | 10.10.40.0/24 | Exposed services, web targets | Planned |

The third number in each subnet matches the segment number, so an address tells you
where it lives at a glance. 10.10.10.10 is DC01 on the lab segment. That sounds minor
but it's a lifesaver once you're staring at logs or a Wireshark capture trying to
figure out what talked to what. I also reserved `.1` in every subnet for the OPNsense
firewall that shows up in Lab 2.

So when I say I "designed a segmented network," that's true. The design is real and
the reasoning is written down. Spinning up empty VirtualBox entries wouldn't have
made it any more true.

---

Phase 3 — Actually doing the migration to 10.10.10.0/24

Order really matters here. DC01 runs DNS, and Active Directory stores its own address
inside DNS. If I change the IP without fixing those records, the domain ends up
half-broken.

Step 1: Find the adapter

```powershell
Get-NetAdapter
```

<img width="891" height="472" alt="screenshots-06-get-netadapter" src="https://github.com/user-attachments/assets/518176a0-1fb0-4332-ab7d-98a236f055bc" />

| Name | Description | ifIndex | Status |
|---|---|---|---|
| Ethernet | Intel(R) PRO/1000 MT Desktop Adapter | 6 | Up |

Never guess the interface name. Configure an adapter that doesn't exist and it fails
in ways that look like a networking problem when there isn't one.

Step 2: check the old state, clean up the junk

```powershell
Get-NetIPConfiguration -InterfaceIndex 6
```

<img width="834" height="647" alt="screenshots-07-preconfig-gateway" src="https://github.com/user-attachments/assets/06dd4305-9079-44c4-be50-af50c11e115a" />

| Field | Value | What it meant |
|---|---|---|
| IPv4Address | 192.168.1.10 | Already static, good |
| IPv4DefaultGateway | 192.168.1.1 | Dead. That router doesn't exist on LABNET |
| DNSServer | 127.0.0.1 | Correct, the DC asks itself |
| NetProfile | Unidentified network | Side effect of the dead gateway |

I pulled the phantom gateway:

```powershell
Remove-NetRoute -InterfaceIndex 6 -DestinationPrefix "0.0.0.0/0" -Confirm:$false
Get-NetConnectionProfile -InterfaceIndex 6
```

<img width="869" height="571" alt="screenshots-08-remove-route-profile" src="https://github.com/user-attachments/assets/1bfdcb41-3103-4ecd-b734-51ea72ba9df3" />

Here's a thing that surprised me: a default gateway pointing at nothing is actually
worse than having no gateway at all. Windows keeps trying to reach that dead next
hop, and you get slow DNS, sluggish service startups, and hangs that all look like
app problems. Once I removed it, the network profile sorted itself out to
DomainAuthenticated, which is the right classification.

Step 3: the actual IP change

```powershell
Remove-NetIPAddress -InterfaceIndex 6 -IPAddress 192.168.1.10 -Confirm:$false
New-NetIPAddress -InterfaceIndex 6 -IPAddress 10.10.10.10 -PrefixLength 24
Set-DnsClientServerAddress -InterfaceIndex 6 -ServerAddresses 127.0.0.1
```

<img width="928" height="650" alt="screenshots-12-ipchange-idempotent" src="https://github.com/user-attachments/assets/4c7fe1f7-9662-45e3-bf71-3dcea5eaf800" />

You can chain these with `;` to save typing in the VM console, but `;` is not `&&`.
If the first command fails, the rest still run. I found this out by running the whole
chain twice and getting a wall of red both times, except both errors actually meant
"this already worked." The remove found nothing to remove, and the new address
refused to duplicate itself. The lesson was to slow down and read the errors instead
of panicking at red text.

```powershell
Get-NetIPConfiguration -InterfaceIndex 6
```

<img width="952" height="669" alt="screenshots-13-postconfig-10net" src="https://github.com/user-attachments/assets/b5188848-01e2-4081-8626-67ce0af41bb7" />

| Field | Value |
|---|---|
| IPv4Address | 10.10.10.10 |
| PrefixLength | 24 |
| IPv4DefaultGateway | (none) |
| DNSServer | 127.0.0.1 |
| NetProfile | zaylab.local |

No gateway on purpose. There's nowhere to route to on an isolated segment, and adding
one that doesn't answer just recreates the exact problem I cleaned up a minute ago.

Step 4: fixing DNS

AD had registered itself as 192.168.1.10. Those records were now wrong.

```powershell
ipconfig /registerdns
Restart-Service Netlogon
Get-DnsServerResourceRecord -ZoneName zaylab.local -RRType A
```

<img width="952" height="510" alt="screenshots-14-dns-records" src="https://github.com/user-attachments/assets/05f8765b-ee42-4239-8f11-ea227b984591" />

`ipconfig /registerdns` re-registers the host record, and restarting Netlogon forces
it to rewrite all the SRV records that tell clients where the domain controller, KDC,
and LDAP live. Those SRV records are what `nltest` reads, and stale ones are the
single most common reason a lab domain half-works after you change subnets.

| HostName | RecordData | Status |
|---|---|---|
| @ | 10.10.10.10 | Updated |
| dc01 | 10.10.10.10 | Updated |
| DESKTOP-MRJGVU8 | 192.168.1.100 | Stale, deleted |
| DomainDnsZones | 10.10.10.10 | Updated |
| ForestDnsZones | 10.10.10.10 | Updated |

Four of the five updated themselves. The last one was the client's old lease record,
so I deleted it by hand:

```powershell
Remove-DnsServerResourceRecord -ZoneName zaylab.local -RRType A -Name "DESKTOP-MRJGVU8" -Force
```

### Step 5: rebuilding DHCP

The old scope was 192.168.1.0/24, named "Office LAN," handing out .100 through .200
with option 003 (Router) still set to the old 192.168.1.1.

<img width="939" height="550" alt="screenshots-09-old-dhcp-scope" src="https://github.com/user-attachments/assets/0b8ec30a-84c1-4610-b35c-e9d45a2814b5" />

You can't just edit a scope onto a different subnet. It has to come down and get
rebuilt.

```powershell
Remove-DhcpServerv4Scope -ScopeId 192.168.1.0 -Force

Add-DhcpServerv4Scope -Name "LABNET" `
  -StartRange 10.10.10.100 -EndRange 10.10.10.200 `
  -SubnetMask 255.255.255.0 -State Active

Set-DhcpServerv4OptionValue -ScopeId 10.10.10.0 `
  -DnsServer 10.10.10.10 -DnsDomain zaylab.local

Get-DhcpServerv4Scope; Get-DhcpServerv4OptionValue -ScopeId 10.10.10.0
```

<img width="912" height="644" alt="screenshots-15-dhcp-rebuild" src="https://github.com/user-attachments/assets/0ff8d2ce-2284-4695-a324-c41f2ed4fdd2" />

| OptionId | Name | Value |
|---|---|---|
| 006 | DNS Servers | 10.10.10.10 |
| 015 | DNS Domain Name | zaylab.local |
| 051 | Lease | 691200 (8 days) |

I left option 003 (Router) off on purpose. On an isolated segment with no router,
handing out a gateway just makes every client waste time trying to reach a next hop
that's never going to answer. Option 006 is the one that really matters for a domain,
though: if a client gets an address but the wrong DNS server, it'll grab a lease just
fine and then completely fail to find the domain. And that looks exactly like a
broken domain controller when it's really just a bad DNS handout.

---

Phase 4 — Confirming the client works

CLIENT01 was still clinging to a lease from a scope that didn't exist anymore, so I
forced it to grab a fresh one.

```powershell
ipconfig /release
ipconfig /renew
ipconfig /all
```

<img width="892" height="625" alt="screenshots-16-client-10net" src="https://github.com/user-attachments/assets/2bf65fba-5ca3-478c-8c4c-b64899b53a7e" />

<img width="952" height="653" alt="screenshots-16-client-10net 2" src="https://github.com/user-attachments/assets/506b1d57-89d8-4871-b9f6-dfca78fe4e01" />

<img width="952" height="674" alt="screenshots-16-client-10net3" src="https://github.com/user-attachments/assets/44e60614-c76a-4585-a103-46a3f4581603" />


| Field | Value |
|---|---|
| IPv4 Address | 10.10.10.100 (Preferred) |
| Subnet Mask | 255.255.255.0 |
| DHCP Server | 10.10.10.10 |
| DNS Servers | 10.10.10.10 |
| Connection suffix | zaylab.local |
| Default Gateway | (none) |
| Lease | 8 days |

Then confirmed the domain still worked on the new subnet:

```powershell
nltest /dsgetdc:zaylab.local
Test-ComputerSecureChannel
```

<img width="889" height="600" alt="screenshots-17-final-domain-verify-stopattrue" src="https://github.com/user-attachments/assets/50bb1091-4227-48f4-96b9-767bad5e7221" />

Found DC01 at 10.10.10.10 with the full flag set, secure channel True, and a ping
that came back 4 for 4 with zero loss at 1 ms. Done.

---

## Phase 5 — Fixing the hostname (this one fought me)

The VM was named CLIENT01 in VirtualBox, but Windows itself still thought it was
`DESKTOP-MRJGVU8`. That's cosmetic today, but it becomes a real headache in Labs 4
and 5 when a SIEM is pulling in events and not one hostname in the logs matches
anything on my diagram.

Honestly, this took longer than the entire network migration, which is worth being
upfront about.

### What went wrong

```powershell
Rename-Computer -NewName CLIENT01 -Restart
```

<img width="750" height="474" alt="screenshots-18-rename-denied" src="https://github.com/user-attachments/assets/bbb6185d-7b46-410e-9ebc-6cdcb987a936" />

It kept failing with `Access is denied`, and the command itself was fine. Three
different problems were stacked on top of each other:

First, the shell wasn't actually elevated. The title bar said "Windows PowerShell,"
not "Administrator: Windows PowerShell," and `Rename-Computer` needs local admin no
matter what domain credential you feed it. Second, I was logged in on a standard
employee account, which was the right account for testing domain logins but has zero
local admin rights, so UAC kept demanding credentials instead of a simple yes. And
third, my admin account, zay.admin, wasn't even in Domain Admins. It was only in
Domain Users and a custom IT_Admins group. Domain Admins is automatically a local
admin on every domain-joined machine, and without it zay.admin had no more power on
CLIENT01 than any random user.

The fix

Over on DC01, in Active Directory Users and Computers (`dsa.msc`), I reset the
zay.admin password, then went to Properties, the Member Of tab, and added **Domain
Admins**. Then Apply, then OK. It doesn't actually save until you do that.

<img width="952" height="569" alt="screenshots-19-domain-admins" src="https://github.com/user-attachments/assets/98803b99-9ad5-4db4-9343-af975e550b09" />

| Group | Location |
|---|---|
| Domain Admins | zaylab.local/Users |
| Domain Users | zaylab.local/Users |
| IT_Admins | zaylab.local/Groups |

Back on CLIENT01 I signed all the way out, not just locked it. Group membership only
gets read at logon, so a session that's already running will never see the new
rights. Signed back in as `ZAYLAB\zay.admin`, opened an elevated shell, and this time
got a plain yes/no prompt with no credential box.

```powershell
Rename-Computer -NewName CLIENT01 -Restart
```

After the reboot:

```powershell
hostname                      # CLIENT01
Test-ComputerSecureChannel    # True
```

<img width="892" height="671" alt="screenshots-20-rename-success" src="https://github.com/user-attachments/assets/15ef58ea-db50-4280-8c1c-6584deeaab90" />

The rename also updates the machine's account in AD, which is why I checked the
secure channel again right after. That part isn't optional.

---

Where it landed

| Host | FQDN | Address | Method | Roles |
|---|---|---|---|---|
| DC01 | DC01.zaylab.local | 10.10.10.10 | Static | AD DS, DNS, DHCP |
| CLIENT01 | CLIENT01.zaylab.local | 10.10.10.100 | DHCP | Domain workstation |
                Windows 11 Host (HP, Ryzen 7, 32 GB)
                             |
                    VirtualBox Hypervisor
                             |
      +----------------------+----------------------+
      |                      |                      |
  LABNET               LABNET-MGMT            LABNET-RED

10.10.10.0/24 10.10.20.0/24 10.10.30.0/24
(BUILT) (PLANNED) (PLANNED)
|
+----+----+ LABNET-DMZ
| | 10.10.40.0/24
DC01 CLIENT01 (PLANNED)
.10 static .100 DHCP
AD/DNS/DHCP


No gateway on any host, no route off the segment. Completely walled off from both my
laptop and my home network.

Snapshots

| Name | What it captures |
|---|---|
| BASELINE-POST-MIGRATION-CLEAN | After the drive move, AD verified, ISO detached |
| BASELINE-LABNET-ISOLATED | On the internal network, still 192.168.1.0/24 |
| BASELINE-10.10.10-RENAMED | Final. 10.10.10.0/24, renamed, verified |

Three restore points, all taken with the VMs powered off. Snapshotting a running
domain controller grabs its RAM state too and can restore into a weird, inconsistent
condition, so I always shut down first. Every future lab starts from
`BASELINE-10.10.10-RENAMED`. That's the whole reason I built this carefully: I can go
break things on purpose now and get back to a known-good domain in under a minute.

---

What this actually demonstrates

- Hypervisor admin: migrating VMs across storage, managing virtual media,
  building a real snapshot strategy
- Network design: subnet planning, thinking through segmentation, and an
  addressing scheme built to be readable in logs
- Active Directory admin: verifying FSMO roles, repairing DNS records, validating
  secure channels, delegating privileges
- DHCP admin: building scopes, setting options, and deliberately leaving one off
  with a documented reason
- Troubleshooting instincts: verify before you build, change one thing at a time,
  snapshot before anything destructive, and read the error instead of reacting to it

---

What I took away

Verify before you build. Booting each VM by itself after the move cost me fifteen
minutes and would've saved me hours if something had actually been broken.

A dead gateway is worse than no gateway. It turns instant, obvious failures into
slow timeouts that are a nightmare to diagnose.

Errors are information, not alarms. Two walls of red during the IP change both
just meant "already done."

Being an admin isn't the same as running as one. Domain Admin rights mean nothing
if the shell isn't elevated. UAC hands you a filtered token. Check the title bar.

Group membership is read at logon. Add someone to a group and their current
session still has no idea.

Design and build are separate things. Four documented segments with one actually
built is more honest than four empty networks with no reasoning behind them.

Knowing what to cut is a skill. Dropping Proxmox was the right move. The goal was
a segmented virtualization host, not a particular hypervisor.

---

Next up

Lab 2: an OPNsense firewall routing between LABNET, LABNET-RED, and LABNET-DMZ with
default-deny, plus Suricata running as an IDS first and then an IPS. Those reserved
`.1` addresses and the planned subnets from this lab are sitting there waiting for
exactly that.
