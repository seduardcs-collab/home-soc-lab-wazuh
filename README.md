# Home SOC Lab: Wazuh Detection and Incident Report

A home-built Security Operations Centre (SOC) lab using Wazuh, built in VirtualBox. I simulated reconnaissance and an SSH brute-force attack against a monitored endpoint, then investigated and reported on the resulting detections.

**Full incident report:** incident-report.md

---

## What this project demonstrates

- Building a monitored environment from scratch (SIEM manager, monitored endpoint, attacker machine)
- Deploying and configuring a Wazuh agent
- Simulating real attack techniques (reconnaissance scanning, credential brute-forcing)
- Investigating alerts in a SIEM dashboard, including filtering, correlation and MITRE ATT&CK mapping
- Writing up findings in a clear, structured incident report
- Diagnosing and fixing real infrastructure problems during setup (see Troubleshooting below)

---

## Lab architecture

| Role | System | Purpose |
| --- | --- | --- |
| Wazuh manager/dashboard | Ubuntu Server 22.04, 8 GB RAM | Central SIEM: collects logs, generates alerts |
| Victim (monitored endpoint) | Ubuntu Server 22.04, Wazuh agent installed | Target machine being monitored and attacked |
| Attacker | Kali Linux | Used to simulate reconnaissance and brute-force attacks |

All three machines ran in VirtualBox on an isolated host-only network, with no access to or from the internet or any other device. Attacks were only ever run against machines I owned and controlled.

---

## What I did

1. Installed Wazuh (manager, indexer, dashboard) on an Ubuntu Server VM.
2. Built a second Ubuntu Server VM as a monitored "victim" and deployed the Wazuh agent to it.
3. Set up a Kali Linux VM as the attacker, on the same isolated network.
4. Ran an Nmap scan from Kali against the victim to identify open services (found SSH open on port 22).
5. Ran a simulated SSH brute-force attack using Hydra against a deliberately weak test account.
6. Investigated the resulting alerts in the Wazuh dashboard: filtered by severity, found the key detection (a level 12 alert for multiple authentication failures followed by a success), and mapped it to MITRE ATT&CK techniques.
7. Wrote up the full investigation as a formal incident report.

---

## Key finding

Wazuh correctly escalated 567 individual authentication failures into a single high-severity correlated alert (Rule ID 40112, level 12) once a successful login followed the failed attempts, rather than leaving an analyst to manually spot the pattern across hundreds of low-level log lines. Full detail in the incident report.

---

## Troubleshooting (setup problems and fixes)

Building this lab didn't go smoothly, and fixing these problems was as instructive as the attack itself.

### 1. VM files wouldn't copy ("Access is denied")

**Symptom:** copying the Kali VM folder failed with error `0x80070005: Access is denied`, and the folder appeared empty when browsed. **Cause:** the destination folder was inside OneDrive, which was trying to sync multi-gigabyte VM files that were actively in use, and blocking access to them. **Fix:** moved all VM files to a local, non-synced folder (`C:\VMs`) instead of Documents.

### 2. Wazuh installation failed: "Installation cleaned"

**Symptom:** the Wazuh install script ran for several minutes, then rolled itself back with no clear error. **Cause:** the Ubuntu Server installer had only allocated about half of the virtual disk (roughly 19 GB of a 40 GB disk), leaving too little free space for Wazuh's installer checks to pass. **Fix:** extended the logical volume to use the full disk with `lvextend -l +100%FREE` and `resize2fs`, confirmed with `df -h`, then reran the installer.

### 3. Install failed again: ports already in use

**Symptom:** a second install attempt failed, reporting that ports 1515 and 55000 were already in use by other processes. **Cause:** leftover Wazuh processes from the first failed install hadn't fully stopped. **Fix:** stopped and purged the Wazuh services and packages, removed leftover config directories, rebooted the VM, confirmed the ports were free, then reinstalled.

### 4. Wazuh manager still wouldn't start after cleanup

**Symptom:** the manager package reported "installation finished" but immediately failed to start, with errors like `wazuh-keystore: No such file or directory`. **Cause:** the package manager (dpkg) still believed wazuh-manager was fully installed even though its files had been deleted during cleanup, so it skipped reinstalling them. **Fix:** force-purged the package with `dpkg --purge --force-all`, removed the remaining directories, rebooted, and reinstalled from a genuinely clean state.

### 5. SSH brute-force attack reported "0 valid passwords found"

**Symptom:** Hydra completed a full run against the victim with no successful login found. **Cause:** the attacker's wordlist file didn't actually contain the victim's test password; a `~` (home directory shortcut) wasn't expanding correctly in the file path, so Hydra wasn't reading the file I thought it was. **Fix:** used the full absolute path (`/home/kali/passwords.txt`) instead of `~/passwords.txt`, and verified the file's contents with `cat` before rerunning the attack.

---

## Tools used

- VirtualBox
- Ubuntu Server 22.04
- Kali Linux
- Wazuh (SIEM/XDR platform)
- Nmap
- Hydra

---

## What I'd do next

- Add a network-based detection tool (e.g. Suricata) alongside Wazuh's host-based monitoring, since the port scan generated little to no alerting on its own.
- Configure Wazuh's Active Response to automatically block an attacking IP after repeated failures, rather than only alerting.
- Expand the victim environment with a second monitored endpoint to practise correlating activity across multiple hosts.
