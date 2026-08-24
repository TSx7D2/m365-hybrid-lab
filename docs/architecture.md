# Architecture and design decisions

Decisions worth explaining, and the reasoning that produced them. Placeholder names are used
throughout: `example.co.uk` for the public domain, `corp.example.co.uk` for the internal forest, and
`lab.example.co.uk` for the verified UPN suffix.

---

## Splitting servers and clients across two hosts

The NUC has 16 GB and runs continuously. The desktop has 32 GB but is a daily driver.

Servers live on the NUC so the domain controller and sync server stay up when the desktop is
rebooted. Clients live on the desktop, where the interactive work happens, such as Autopilot OOBE,
sign-in testing and checking whether a policy applied, and where there is RAM to spare.

Both hosts use an **external** virtual switch, which bridges VMs onto the physical LAN. That is what
makes the split possible, because a client on one hypervisor can only reach a domain controller on
another if both sit on the same subnet. An internal switch with NAT would isolate them.

`-AllowManagementOS $true` keeps the host's own networking alive through the shared adapter. Omit
it and the host drops off the network.

---

## Which machines are enrolled in Intune, and which are not

| Machine | Enrolled | Reasoning |
| --- | --- | --- |
| `W11-AP01` | Yes | The Autopilot target, cloud only |
| `W11-DOM01` | Yes | Hybrid joined, the realistic case for most estates |
| `MGMT01` | **No** | Administrative tooling, not an endpoint. Enrolling it would put a Tier 0 machine under the same policy as user devices |
| `DC01`, `AADC01` | **No** | Servers. Update-ring reboots would take out the directory |
| Physical hosts | **No** | The hypervisors. Compliance reboots would take every VM down with them |

Deciding what not to manage is as much of a design decision as deciding what to manage, and it is
one worth being able to justify.

---

## OU design

```
corp.example.co.uk
├── OU=Lab                 <- the sync boundary
│   ├── OU=Users
│   ├── OU=Groups
│   └── OU=Computers
├── OU=Admin               <- deliberately outside it
└── CN=Users, CN=Computers <- built-in containers, unused
```

Three things drove this shape.

**Containers cannot be targeted by Group Policy.** `CN=Users` and `CN=Computers` are containers, not
organisational units. GPOs link to sites, domains and OUs only, so anything left in the defaults
receives domain-level policy and nothing else.

**One parent means one sync boundary.** Entra Connect filters on OUs, and selecting a parent
automatically includes child OUs created later. Scope becomes something designed once rather than
policed.

**Privileged accounts must stay out of the cloud.** `CN=Users` holds `Administrator`, `krbtgt` and
the Domain and Enterprise Admins groups. Syncing those to Entra means a cloud compromise reaches the
forest. The named domain admin account lives in `OU=Admin` at the domain root, permanently outside
the synced subtree, and keeps the internal UPN suffix rather than the verified one. It has no
business having a cloud-shaped identity.

---

## Authentication method: password hash synchronisation

Chosen over pass-through authentication and federation.

**What actually syncs.** The NTLM hash is salted and run through PBKDF2-HMAC-SHA256, 1000
iterations. What lands in the cloud is a hash of a hash. It cannot be reversed to the password, and
it cannot be replayed against the on-premises directory. The common objection, that passwords end up
in the cloud, is answering a question nobody asked.

**Why it wins here.** Authentication happens entirely in Entra, so an on-premises outage does not
become a Microsoft 365 outage. It is also the only option that enables leaked credential detection
in Identity Protection. Password hashes ride their own two-minute cycle, separate from the
30-minute directory sync.

**The honest trade-off.** A disabled account can retain cloud access until the next sync cycle.
Pass-through authentication closes that gap by validating against a live domain controller, at the
cost of making cloud sign-in depend on on-premises health. Microsoft also advises enabling password
hash sync alongside it as a fallback, which reinstates what you chose PTA to avoid.

**Federation** was not considered seriously. AD FS means a farm, a proxy, and token-signing
certificates that take sign-in down when they expire, and a compromised AD FS lets an attacker forge
tokens for any user.

---

## Connect Sync rather than Cloud Sync

Cloud Sync is the lighter agent and the direction Microsoft is travelling. It was not used here
because **device sync reached only public preview in July 2026**, and device writeback remains
Connect-only. Hybrid join depends on computer objects reaching Entra, so this lab uses Connect Sync.

One operational note applies to any Connect Sync deployment. From **30 September 2026**, servers
running build **2.5.79.0 or below** stop synchronising altogether. The failure is quiet, because
authentication keeps working from cached password hashes, so nothing looks broken while disabled
accounts stay active in the cloud indefinitely.

---

## Identities and what each is for

| Identity | Scope | Purpose |
| --- | --- | --- |
| Break-glass admin | Cloud only, initial `.onmicrosoft.com` domain | Excluded from every Conditional Access policy. Never used routinely |
| Working Global Admin | Cloud only, verified domain | Day-to-day tenant administration |
| Hybrid Identity Admin | Cloud only, initial domain | Used at the Entra Connect wizard. The least-privileged role that can do it |
| Named domain admin | On-premises, `OU=Admin` | All domain administration. Never synced |
| Built-in `Administrator` | On-premises | Break-glass equivalent. Long password, not used |
| `Sync_*` and `MSOL_*` | Created automatically | The real service accounts, whose passwords rotate themselves |

The last row matters. The credentials entered into the Entra Connect wizard are used **once** and
never stored. They exist so the wizard can provision least-privilege service accounts at both ends.
Creating a permanent service account with a static password and exempting it from MFA would be the
wrong instinct, because the wizard sign-in is interactive with a person present, so MFA works and
should be enforced.

The break-glass account exists because every other administrative identity can be locked out by a
Conditional Access policy, including the one that wrote the policy.
