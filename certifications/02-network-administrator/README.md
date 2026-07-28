# Network Administrator Level 2 (네트워크관리사 2급)

| Item | Details |
|---|---|
| **Certification** | Network Administrator Level 2 (네트워크관리사 2급) |
| **Issuing authority** | ICQA — Korea Information & Communication Qualification Association |
| **Accreditation** | State-accredited private qualification, accredited by the Ministry of Science and ICT since January 2002 (accreditation no. 2023-02) |
| **Earned** | 2025.12.09 |
| **Credential ID** | `NT2081255` |
| **Verification** | [ICQA certificate verification](https://www.icqa.or.kr) |
| **Validity** | 5 years, renewable through continuing education (Level 1 has no expiry) |
| **Academic credit** | Recognised as 14 credits toward a computer engineering major under Korea's Academic Credit Bank System |

<img src="assets/certificate.png" width="420" alt="Certificate of qualification">

---

## TL;DR

South Korea's state-accredited qualification for network administration. It validates hands-on ability to configure, operate, and troubleshoot network infrastructure — physical cabling, router and switch CLI, Windows Server and Linux services, and baseline network security. This is where I stopped seeing a network as an abstraction and started seeing it as a set of devices someone has to configure, each with defaults, each with mistakes.

---

## Context for non-Korean readers

A widely held networking qualification in South Korea, issued by ICQA and accredited by the Ministry of Science and ICT. It is vendor-neutral and covers general network administration rather than a single vendor's product line — comparable in scope to entry-level vendor-neutral networking certifications such as CompTIA Network+. *(Informal comparison for context only.)*

**A note on grading.** The numbering is counter-intuitive to outside readers: **Level 2 is the state-accredited tier**, while Level 1 is a registered but non-accredited private qualification. Level 2 is also the open-entry tier — anyone may sit it. Level 1 has eligibility requirements that must be met before you are permitted to register: holding Level 2 plus several years of relevant professional experience, a related degree, or five or more years of employment at an IT organisation.

As a self-taught candidate without professional experience or a degree, Level 2 was the highest tier I was eligible to attempt.

---

## What it covers

### Written examination

| Subject | Scope |
|---|---|
| **Network fundamentals** | OSI seven-layer model, TCP/IP protocol suite, IP addressing and subnetting, topologies and network hardware |
| **Network operating systems** | Windows Server and Linux administration, user and account management, network service configuration (DNS, DHCP, IIS, FTP) |
| **Network and system security** | Security fundamentals, firewall concepts, malware countermeasures, security policy |
| **Computing and ICT fundamentals** | Computer architecture, operating system internals, current ICT trends |

**Format.** 50 multiple-choice questions in 50 minutes, computer-based, drawn from a
published question bank. Pass mark is 60 out of 100 with no per-subject minimum.
The practical stage must be passed within two years of clearing the written stage.

### Practical examination

**Format.** 18 task-based questions in 80 minutes, split into three timed segments:
10 minutes for physical cable termination, 20 minutes for router CLI, and 50 minutes
for the remainder. The cable task carries 6.5 points and every other question 5.5.
Pass mark is 60 out of 100 — in practice, 11 of the 18 questions.

| Area | Scope |
|---|---|
| **Infrastructure and cabling** | Physical UTP cable assembly to the TIA/568B standard, hardware diagnostics |
| **Server and service configuration** | Configuring Windows Server and Linux network services through interactive simulation |
| **Router and switch configuration** | Cisco CLI — interface configuration, IP setup, routing protocols |
| **Troubleshooting** | Diagnosing connectivity failures, IP conflicts, and service outages |

---

## How it connects to offensive security

Of the qualifications I hold outside the offensive security track, this one carries over the most directly. Penetration testing is largely the practice of finding where a network was configured imperfectly — which is difficult to do without first knowing how it is configured correctly.

| Competency | Relevance to security work |
|---|---|
| **IP addressing and subnetting** | Determining scope from a target range, recognising which hosts are reachable from where, and reasoning about pivot paths between segments. Subnetting fluency is the difference between guessing at a network's shape and deducing it. |
| **OSI model and TCP/IP behaviour** | Reading packet captures, interpreting scan results correctly, and understanding why a given probe produces a given response — the basis for distinguishing a filtered port from a closed one, or a firewall from a dead host. |
| **Router and switch CLI, routing protocols** | Network devices are targets, not just infrastructure. Knowing how routes and interfaces are configured is what makes routing manipulation, ACL gaps, and device misconfiguration legible as attack surface. |
| **Firewall concepts and security policy** | Understanding what a defender is trying to block, and where those controls are typically placed, is the starting point for reasoning about evasion and for writing remediation advice that a network team can actually act on. |
| **Windows Server services (DNS, DHCP, IIS, FTP)** | These are the services that appear in Active Directory environments. Having configured them is why enumeration of a Windows domain reads as a familiar system rather than an opaque one. |
| **Linux administration** | Users, permissions, services, and filesystem layout — the baseline knowledge that privilege escalation enumeration depends on. Recognising an abnormal system requires having administered a normal one. |
| **Structured troubleshooting** | The diagnostic loop taught for fault isolation — observe, hypothesise, test, narrow — is methodologically identical to enumeration. Learning it as a network administrator is why I approach an unfamiliar target systematically instead of by trial and error. |

---

## Preparation

[View study plan](assets/study-plan.png)

[View shared study materials](assets/study-materials.png)

| Item | Detail |
|---|---|
| **Total duration** | 2 months — 1 month written, 1 month practical |
| **Daily commitment** | ~5 hours per day |
| **Concurrent study** | Prepared alongside the Craftsman Information Processing certification during the same period |
| **Materials** | *Network Administrator Level 2 All-in-One* comprehensive textbook |
| **Study method** | Self-study supplemented by collaborative study groups and shared resources through KakaoTalk open chat |
| **Attempts** | Passed both stages on the first attempt |

Running two certification tracks in parallel was deliberate — the subject overlap
(operating systems, data communications) meant each reinforced the other rather than
competing for study time. Both were completed within the same month: this certification
on 9 December 2025, and Craftsman Information Processing on 24 December.

---

## What I still use

- **Subnetting and network topology reasoning** — used constantly when scoping a target range, interpreting scan output, and identifying which segments are reachable from a compromised host.
- **Windows Server service knowledge** — DNS and DHCP behaviour learned here is what makes Active Directory enumeration comprehensible rather than mechanical.
- **Cisco CLI familiarity** — enough grounding to assess network device configuration and understand routing and ACL misconfigurations when they appear.
- **The troubleshooting loop** — the habit of isolating variables systematically is the single most transferable thing this certification gave me, and it is what my enumeration methodology is built on.