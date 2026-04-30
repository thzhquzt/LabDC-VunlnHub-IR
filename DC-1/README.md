# 🛡️ Incident Response Report: DC-1 (Drupalgeddon2)

## 1. Executive Summary
- **Target:** DC-1 (Vulnerable Web Server - IP: `192.168.80.100`).
- **Vulnerability:** Drupal 7.x Remote Code Execution (CVE-2018-7600 / Drupalgeddon2).
- **Impact:** System Compromised, Root access gained, Web Defacement, Sensitive Data Exfiltration (`/etc/shadow`).
- **Defense Stack:** Security Onion (SIEM Gateway), Wazuh (HIDS), `iptables` (Network Firewall).
- **Resolution:** Isolated the host via Gateway Firewall (Zero Trust approach), eradicated all backdoors, safely patched the CMS via an internal staging server, and reset enterprise credentials before restoring services.

---

## 2. Architecture & Environment
The lab environment is built on an **In-the-middle Traffic Analysis** architecture. Security Onion acts as the routing Gateway, allowing the Blue Team to monitor all network traffic and enforce Containment measures at the network level without interacting directly with the compromised host.

![Network Topology](Images/0_Architecture/01_network_topology.png)

The Wazuh Agent was successfully deployed on the DC-1 machine and actively forwarded logs to the SIEM (Security Onion).

![Wazuh Agent Active](Images/0_Architecture/02_wazuh_agent_active.png)
![Kibana Agent Check](Images/0_Architecture/03_kibana_agent_check.png)

---

## 3. MITRE ATT&CK Mapping

| Tactic | Technique | ID | Description |
| :--- | :--- | :--- | :--- |
| **Initial Access** | Exploit Public-Facing App | `T1190` | Exploited Drupal RCE vulnerability (CVE-2018-7600). |
| **Execution** | Command and Scripting Interpreter | `T1059` | Established a Reverse Shell using Metasploit. |
| **Privilege Escalation**| Abuse Elevation Control Mechanism | `T1548.001`| Abused SUID misconfiguration on the `find` binary to gain Root privileges. |
| **Defense Evasion** | Indicator Removal on Host | `T1070.001`| Anti-Forensics: Overwrote `/var/log/auth.log` to 0 bytes. |
| **Credential Access** | OS Credential Dumping | `T1003.008`| Exfiltrated the `/etc/shadow` password hash file. |
| **Impact** | Defacement | `T1491.001`| Modified the `index.php` homepage to display a defacement flag. |

---

## 4. Phase 1: Attack & Detection

### 4.1. Reconnaissance
The attacker initiated a deep Nmap scan (`-sV -A`) to discover open ports and running services.

![Nmap Scan](Images/1_Attack_and_Detection/01_nmap_scan_version.png)

**SOC Detection:** The Wazuh Agent detected abnormal connection spikes, and Kibana successfully flagged the Nmap User-Agent.

![Alert Nmap Kibana](Images/1_Attack_and_Detection/02_alert_nmap_kibana.png)

### 4.2. Exploitation & Establishing C2
The attacker utilized the `exploit/unix/webapp/drupal_drupalgeddon2` module in Metasploit to compromise the server.

![Metasploit Exploit](Images/1_Attack_and_Detection/03_exploit_drupalgeddon2.png)

**SOC Detection:** Kibana logs revealed the DC-1 machine actively initiating a reverse connection (Reverse Shell) back to the attacker's IP on port 4444.

![Alert Reverse Shell](Images/1_Attack_and_Detection/04_alert_kibana_reverse_shell.png)

### 4.3. Privilege Escalation & Defacement
The attacker escalated privileges to root via the SUID vulnerability in the `find` command and subsequently defaced the website.

![SUID Privilege Escalation](Images/1_Attack_and_Detection/05_privesc_find_suid.png)
![Web Defacement Command](Images/1_Attack_and_Detection/08_attacker_echo_defacement.png)
![Web Defacement Browser](Images/1_Attack_and_Detection/09_browser_defacement_flag.png)

### 4.4. Defense Evasion (Anti-Forensics) & Backdoors
To cover their tracks, the attacker cleared the `/var/log/auth.log` file and created hidden rogue users (`hacker_lord`, `sys_update`) with root privileges (UID=0).

![Create Rogue Users](Images/1_Attack_and_Detection/06_attacker_create_rogue_users.png)
![Clear Log](Images/1_Attack_and_Detection/10_anti_forensics_clear_log.png)

**SOC Detection:** Squert immediately triggered two High-Priority alerts: a massive reduction in log file size and a CIS Benchmark violation (Detection of unauthorized UID 0 accounts).

![Alert UID 0](Images/1_Attack_and_Detection/07_alert_squert_uid0.png)
![Alert Size Reduced](Images/1_Attack_and_Detection/11_alert_squert_size_reduced.png)

### 4.5. Data Exfiltration
The attacker prepared the sensitive `/etc/shadow` file by copying it into the web-accessible directory and modifying permissions.

![Copy Shadow](Images/1_Attack_and_Detection/14_cp_shadow.png)

Subsequently, the file was exfiltrated to the Kali machine using `wget`.

![Wget Shadow](Images/1_Attack_and_Detection/12_data_exfiltration_shadow.png)

**SOC Detection:** The SIEM successfully detected the unique signature of the shadow file being transferred over unencrypted HTTP protocol.

![Alert Shadow HTTP](Images/1_Attack_and_Detection/13_alert_wazuh_shadow_http.png)

---

## 5. Phase 2: Containment

Since the system was fully compromised (Root access), the IR Team strictly followed the **Zero Trust** principle. No actions were performed directly on the DC-1 OS initially to avoid triggering potential booby traps and to preserve RAM/Disk states for forensics. All containment actions were executed Out-of-Band via the SIEM Gateway.

### 5.1. Attacker IP Blocking (Active Response)
A command was pushed from Security Onion to the local Agent to block the attacker's IP.

![Active Response Deny](Images/2_Containment/01_active_response_deny.png)

### 5.2. Total Host Quarantine (Network Isolation)
Leveraging the Gateway architecture, the IR team configured `iptables` rules to DROP all inbound and outbound traffic for DC-1.

![Iptables Block Inbound](Images/2_Containment/02_iptables_block_inbound.png)
![Iptables Block Outbound](Images/2_Containment/03_iptables_block_outbound.png)

---

## 6. Phase 3: Eradication
After securing a snapshot of the machine, the IR team accessed the internal console of DC-1 to manually clean the malware and backdoors.

*   **Removing Malicious Files:** Deleted the defaced `index.php` and the leaked `shadow.txt`.
    *   ![Remove Deface](Images/3_Eradication/01_remove_deface_index.png)
    *   ![Remove Shadow Leak](Images/3_Eradication/02_remove_shadow_leak.png)
*   **Stripping SUID Permissions:**
    *   ![Remove SUID](Images/3_Eradication/04_remove_suid.png)
*   **Removing Rogue Users:** Encountered an OS self-defense mechanism (Process 1 error) when attempting to use `userdel`. The team manually cleaned the `/etc/passwd` file and rebooted the system.
    *   ![Find Rogue Users](Images/3_Eradication/03_find_rogue_users.png)
    *   ![Userdel Error PID 1](Images/3_Eradication/05_userdel_pid1_error.png)
    *   ![Manual Passwd Cleanup](Images/3_Eradication/06_manual_passwd_cleanup.png)
    *   ![System Reboot](Images/3_Eradication/07_reboot.png)

---

## 7. Phase 4: Recovery Plan

During this phase, the IR Team executed an "Air-gapped Patching" procedure to ensure the host was completely secure before reconnecting it to the network.

| Step | Action | Technical Details | Evidence |
| :---: | :--- | :--- | :--- |
| **1** | **Prepare Patch** | Downloaded the CMS update and set up an internal Staging Server on Security Onion. | ![Stage Patch](Images/4_Recovery/01_stage_patch_on_seconion.png) |
| **2** | **Pull Patch Internally** | DC-1 fetched the patch from the Staging Server via the internal network. | ![Pull Patch](Images/4_Recovery/02_pull_patch_to_dc1.png) |
| **3** | **Extract Patch** | Extracted the updated source code locally. | ![Extract Patch](Images/4_Recovery/03_extract_patch_file.png) |
| **4** | **Restore Service** | Overwrote the web directory with the clean source code and restarted Apache2. | ![Restore Web](Images/4_Recovery/04_restore_web_structure.png) |
| **5** | **Reset Credentials** | Reset the Root password immediately to invalidate the stolen hashes. | ![Reset Password](Images/4_Recovery/05_reset_root_password.png) |
| **6** | **Network Unblocking** | Removed the `DROP` rules on the Gateway `iptables` to restore external access. | ![Unblock Firewall](Images/4_Recovery/06_unblock_gateway_firewall.png) |
| **7** | **Verification** | Verified that the website was functioning properly from an external network. | ![Verify Website](Images/4_Recovery/07_verify_website_restored.png) |
| **8** | **Hyper-Care Monitoring** | Created a dedicated Kibana Dashboard to closely monitor DC-1 post-incident. | ![Hyper Care](Images/4_Recovery/08_hyper_care_monitoring.png) |
