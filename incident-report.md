# Incident Report: SSH Brute-Force Attack Detection

**Lab:** Home SOC Lab (Wazuh, VirtualBox) **Analyst:** \[Your Name\] **Date of exercise:** 17–18 October 2026

---

## 1. Summary

A simulated SSH brute-force attack was launched from an attacker virtual machine (Kali Linux) against a monitored victim virtual machine (Ubuntu Server) running a Wazuh agent. The attack was preceded by a port scan to identify open services. Wazuh detected the resulting authentication failures and correctly escalated them to a high-severity brute-force alert once the pattern was recognised, shortly before the attacker successfully authenticated using a weak password from a custom wordlist.

**Verdict:** Successful simulated compromise via weak credentials; correctly detected by Wazuh.

---

## 2. Environment

| Role | System | IP Address |
| --- | --- | --- |
| Wazuh manager/dashboard | Ubuntu Server 22.04 | 192.168.56.102 |
| Victim (monitored endpoint) | Ubuntu Server 22.04, Wazuh agent installed | 192.168.56.101 |
| Attacker | Kali Linux | 192.168.56.103 |

All systems were isolated on a VirtualBox host-only network (no access to or from the internet or any other device).

---

## 3. Timeline

| Time | Event |
| --- | --- |
| 19:17:23 | Port scan launched from Kali against victim (`nmap -sS -sV 192.168.56.10``1`) |
| 19:18:26 | Scan completed: port 22/tcp (SSH) found open |
| 19:54:18 | SSH brute-force attack launched from Kali against victim's SSH service, targeting user `testuser` with a 101-entry password list |
| 00:55:37 | Wazuh raised a level 12 alert (Rule ID 40112) for multiple authentication failures followed by a successful login |
| 19:55:45 (Hydra log) \* | Hydra reported a valid password found: `testuser` / `winter2024` |
| Attack duration | Approximately 1 minute 27 seconds from first attempt to successful login |

\* *Note: Hydra's log timestamp and the Wazuh alert timestamp were recorded from different system clocks (VM vs dashboard) and may be offset; both refer to the same event.*

---

## 4. Evidence

- **Reconnaissance:** Nmap scan from Kali identified port 22 (SSH) open on the victim, indicating a viable attack surface.
- **Attack:** Hydra was used to attempt 101 SSH logins for user `testuser` against the victim. The 101st attempt, using the password `winter2024`, succeeded.
- **Detection:** The Wazuh Threat Hunting dashboard recorded 799 total alerts for the victim agent within the observation window, including 569 authentication failures and 7 authentication successes. One alert reached level 12.
- **Key alert:** Rule ID 40112, level 12, described as multiple authentication failures followed by a success, fired at 00:55:37. This is Wazuh's standard pattern for detecting a successful brute-force compromise, not just a single failed login.
- **MITRE ATT&CK mapping:** Wazuh tagged the activity under Password Guessing, Brute Force, and Valid Accounts, consistent with the attack technique used.


*(Insert screenshots here: Nmap output, Hydra success line, Wazuh dashboard overview, filtered high-severity alert list, expanded Rule 40112 alert.)*

---

## 5. MITRE ATT&CK Mapping

| Technique | ID | Notes |
| --- | --- | --- |
| Brute Force | T1110 | Hydra used to attempt multiple SSH passwords |
| Valid Accounts | T1078 | Attacker obtained working credentials after brute-forcing |
| (Reconnaissance) Active Scanning | T1595 | Nmap scan preceded the attack |

---

## 6. Impact

In this lab scenario, the attacker gained interactive SSH access to the victim machine using a weak password. In a real environment, this would grant an attacker access on the system, from which they could escalate privileges, move laterally, or exfiltrate data, depending on what `testuser` has access to.

---

## 7. Recommendations

- **Disable password authentication for SSH** and require key-based authentication instead.
- **Disable or restrict root login over SSH.**
- **Enforce a strong password policy** so weak passwords like `winter2024` cannot be set in the first place.
- **Install and configure fail2ban** (or Wazuh's Active Response) to automatically block an IP after a small number of failed login attempts, rather than allowing 100+ attempts.
- **Alert thresholds:** confirm Wazuh (or an equivalent control) is configured to notify an analyst in real time on a level 10+ alert, rather than relying on a manual dashboard check.
- **Network segmentation:** in a production environment, SSH should not be exposed to untrusted networks at all; access should go through a bastion host or VPN.

---

## 8. Lessons Learned

- A port scan alone (without a network-level tool like Suricata) generated little to no Wazuh alerting, since Wazuh's agent monitors host logs rather than network traffic directly. This is a useful finding in itself: host-based monitoring and network-based monitoring catch different things, and a real SOC would want both.
- Wazuh's escalation from individual failed-login alerts to a single high-severity "multiple failures then success" alert (Rule 40112) is a good example of alert correlation reducing noise for an analyst, rather than having to manually spot the pattern across hundreds of low-level events.
- [Add your own reflection here once you've reviewed everything, e.g. anything that surprised you, anything you'd do differently next time.]
