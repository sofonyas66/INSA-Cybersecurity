# Day 25 — Introduction to Active Directory & Initial Enumeration

Topics: Red Teaming Foundations, AD Architecture, Network Protocols, Authentication & Access Control, AD Attack Surface

**Date:** Saturday, June 20, 2026

---

## Module 1: Red Teaming Foundations & Ethics

The session opened by drawing a hard line between red teaming and standard pentesting — red teaming is about thinking, moving, and operating like an advanced adversary, not just finding vulnerabilities.

**The Red Team Mandate**
Red team engagements are goal-oriented, not vulnerability-oriented. The objective isn't "find every flaw" — it's to use realistic adversary techniques to validate whether the organization's overall security readiness holds up, and to improve the security architecture as a result.

**Physical vs. Digital Security**
Physical and digital security aren't separate disciplines — they're connected. Physical access to a building can lead directly to system compromise (badge cloning, dropped USB devices, unattended workstations). Engagements are split into external (attacking from outside the network) and internal (assuming a foothold already exists). Security has to be assessed as a whole system, not as isolated digital controls.

**Rules of Engagement (RoE)**
Every engagement requires written authorization (NDA/RoE) before any action is taken. This defines scope, objectives, and which attack methods are authorized. Data protection requirements and compliance with Ethiopian cyber law are part of the RoE, not an afterthought.

**Legal & Ethical Considerations**
Work strictly within written authorization. Respect scope boundaries even when a path outside scope looks interesting. Protect confidentiality of everything encountered during the engagement. No persistence after the engagement ends, no disruption, no harm beyond the stated objectives.

**The Red Team Operator Mindset**
This is the part that matters most going forward: OPSEC and stealth awareness, patient and disciplined execution, thinking like a real adversary rather than a tool operator, minimizing noise and detection, and documenting everything clearly as you go — not after the fact from memory.

---

## Module 2: Active Directory Fundamentals

### Why Protocols Matter First

Before touching AD itself, the session covered the protocols AD runs on. The reasoning: you can't understand attack surface in an AD environment without understanding what's actually talking to what, and over which protocol.

| Protocol | Port | Role |
|---|---|---|
| SMB | 445 | File/printer sharing, resource access |
| LDAP | 389/636 | Directory queries, AD communication |
| Kerberos | 88 | Ticket-based authentication, SSO |
| DNS | 53 | Name resolution, service discovery |
| DHCP | 67/68 | Automatic IP assignment |
| NTP | 123 | Time sync — critical because Kerberos fails without it |
| HTTP/HTTPS | 80/443 | Web app communication |
| FTP/SFTP | 21/22 | File transfer |
| MSSQL | 1433 | Database backend |
| SMTP/POP3/IMAP | 25/110/143 | Mail send/retrieve/sync |
| RDP | 3389 | Remote desktop, graphical admin |
| WinRM | 5985/5986 | PowerShell remoting, admin automation |
| SSH | 22 | Encrypted remote shell |

The point of memorizing this isn't trivia — every one of these is a potential identity entry point or a way to understand what's normal traffic in the environment.

### What Active Directory Actually Is

AD is Microsoft's directory service for centrally managing users, computers, and network resources. It's the backbone of identity management in most enterprise networks — centralized authentication, authorization, and policy enforcement instead of managing access per-machine.

**Core components:**

- **Domains** — logical boundary containing users, groups, computers, policies; shares one database and security policy set. Example: `company.local`, `INSA.corp`.
- **Domain Controllers (DCs)** — the servers that host AD itself. They authenticate and authorize users, store the AD database (`NTDS.dit`), enforce security policy, and replicate directory data between each other.
- **Organizational Units (OUs)** — containers used to organize objects and delegate administration. Lets you apply Group Policy to just `OU=HR` without touching `OU=IT`.
- **Users and Groups** — individual identities and the groups that simplify permission management. Permissions are almost always assigned to groups, not individual users.
- **Computers** — domain-joined devices, authenticated by AD and managed through Group Policy.
- **Group Policy Objects (GPOs)** — centralized config enforcement (password policy, USB restrictions, etc.) applied at the Site, Domain, or OU level.
- **Trust Relationships** — allow resource sharing between domains. Can be one-way or two-way, transitive or non-transitive.
- **Forests and Trees** — a forest is the top-level AD structure and the actual security boundary; a tree is a collection of related domains sharing a common schema. Trust exists by default between domains in the same forest.
- **Sites and Replication** — sites represent physical locations (e.g. Addis Ababa Site, Hawassa Site); replication keeps AD data consistent across DCs at different sites.

**Common protected groups worth knowing by name:** Domain Admins, Enterprise Admins, Schema Admins, Administrators, Backup Operators, Account Operators, Cert Publishers, RODCs. These are the high-value targets in any AD environment — compromising membership in one of these is effectively game over for that domain.

---

## Authentication & Access Control

### Identity in AD

Every identity in AD — user, computer, or service account — gets a unique **SID (Security Identifier)**. This matters because Windows applies permissions to SIDs internally, not usernames. Rename an account and the SID stays the same.

**SID structure**, broken down from the example `S-1-5-21-3623811015-3361044348-30300820-514`:
- `S-1` — revision 1, current SID format
- `5` — NT Authority
- `21` — non-unique authority, meaning a domain/computer identifier follows
- the long number sequence — unique domain identifier
- `514` — the RID (Relative Identifier), specific to the account/group

**Well-known RIDs worth memorizing:**
- 500 → built-in Administrator
- 501 → Guest
- 512 → Domain Admins
- 513 → Domain Users
- 514 → Domain Guests
- 515 → Domain Computers

Knowing these means you can spot a privileged account from its SID alone without needing a name lookup.

### Authentication Flow

Authentication verifies identity to a Domain Controller. Two protocols exist: **Kerberos** (modern, default) and **NTLM** (legacy, still around for compatibility). After successful authentication, the system generates an **access token** containing the user's SID and group memberships — this token is what's actually checked for every subsequent resource access, not the original credentials.

**Kerberos**, in simplified flow:
1. User logs in → receives a **TGT** (Ticket Granting Ticket) from the **AS** (Authentication Service)
2. TGT is used to request access to a specific service
3. **KDC** (Key Distribution Center) issues a **Service Ticket** via the **TGS** (Ticket Granting Service)
4. Ticket is presented to the target service → access granted without re-entering credentials

This is what makes Single Sign-On work — log in once, access file servers, databases, and apps without re-authenticating each time.

**NTLM**, for comparison: stateless, challenge-response based, relies on password hash validation rather than tickets. Still around for legacy compatibility, but it's where relay attacks and replay attacks live — a meaningfully weaker design than Kerberos.

**SAM vs AD accounts**: SAM is the local machine account database — machine-specific. AD accounts are domain-wide and centralized. A "Local Admin" lives in SAM; a "Domain Admin" lives in AD. Both can exist on the same box.

### Access Control

**ACL (Access Control List)** is the full permission list on an object. **ACE (Access Control Entry)** is a single rule inside that list. The authorization flow end to end:

> Login → Kerberos/NTLM authentication → access token generated (SID + group memberships) → resource checks its ACL → matching ACE found → access allowed or denied

Why this layer matters: it controls *all* access decisions in AD. A misconfiguration here doesn't stay contained — it can affect the entire domain. A single compromised identity with the wrong group membership can mean domain-wide access. This is the foundation everything else in AD security analysis builds on.

---

## AD Attack Surface & Enumeration

### What the Attack Surface Actually Is

Every identity, service, and system exposed in AD — users, computers, groups, applications — makes up the attack surface, and it expands with every permission and misconfiguration layered on top. The central question during enumeration is simple: **what can be discovered or accessed?**

**Core identity exposure points:**
- User accounts (standard and privileged)
- Service accounts (application identities — often over-privileged and forgotten)
- Computer accounts (domain-joined systems)
- Groups (where privilege actually aggregates)
- Disabled or stale accounts — frequently overlooked, frequently still have valid permissions

**Service attack surfaces:** SPNs (Service Principal Names), and every network service running — HTTP, SMB, LDAP, RDP, MSSQL, WinRM, SSH. The key idea repeated here: every running service is a potential identity entry point.

**SMB shares specifically** deserve attention — they're a common internal exposure point that often contains sensitive files: configs, backups, even credentials sitting in plaintext on a share nobody's audited (`\\fileserver\finance`, `\\dc01\sysvol`).

### How Enumeration Should Actually Be Done

The flow taught: **Network → Hosts → Services → Identities → Relationships**

1. Identify reachable systems first
2. Discover exposed services on those systems
3. Identify users and groups
4. Map relationships between identities
5. Build a mental model of the full environment

This isn't a checklist to rush through — the goal at each step is building an actual mental map, not just collecting raw data for its own sake.

**User & Group Discovery** — users exist across domains and OUs, groups define the privilege structure, and group nesting adds complexity fast. Privileged groups are the high-value targets. The goal is understanding *who exists and what they belong to*, not just listing names.

**Computer & Host Discovery** — domain-joined workstations and servers. Servers tend to host the sensitive services; workstations may have live user sessions sitting on them. Both matter for building the environment map.

**Relationship Mapping** — this is where everything connects:

> User → Group → Permission → Service → Data

Understanding *relationships* between users, groups, computers, and permissions matters more than raw enumeration data on its own. This is also where **BloodHound** was named directly as the key tool for analyzing this kind of relationship data — which lines up directly with the graph-based approach in the red team planning platform project.

---

## Key Takeaways

- AD attack surface = identity + services + relationships, not any one of those alone.
- SMB shares are an easy, frequently overlooked way to expose confidential internal data.
- Enumeration is fundamentally about building a mental map of the environment, not just running tools and collecting output.
- Users, groups, and computers form interconnected trust layers — the relationships between them matter more than any single data point.
- Authentication and authorization is the central control layer of AD: a single compromised identity with the wrong group membership can mean domain-wide access.

## Connection to the Graduation Project

This session is the direct theoretical foundation for the GNN attack path engine. BloodHound's relationship-mapping model — User → Group → Permission → Service → Data — is exactly the graph structure the platform's knowledge graph is built on top of. The platform takes this same enumeration output and adds Ethiopian threat intelligence plus learned path prediction on top of it.
