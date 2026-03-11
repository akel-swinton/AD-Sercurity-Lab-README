# AD-Sercurity-Lab-README
A hands-on Active Directory security lab demonstrating a real-world password spray attack, detection using Wazuh SIEM, and defense using Account Lockout Policy. This lab simulates techniques used by real threat actors and shows how defenders detect and stop them.
##  Lab Environment

|Component            |Details                                |
|---------------------|---------------------------------------|
|**Domain Controller**|Windows Server 2022 Standard Evaluation|
|**SIEM Platform**    |Wazuh 4.7.5                            |
|**Hypervisor**       |Proxmox VE                             |
|**Server IP**        |Redacted (private lab network)         |
|**Role**             |Primary Domain Controller              |

-----

##  Lab Objectives

- Create and manage Active Directory user accounts using PowerShell
- Simulate a password spray attack against multiple AD accounts
- Detect the attack using Wazuh SIEM
- Implement Account Lockout Policy to defend against password attacks
- Verify the lockout policy triggers and Wazuh alerts on the lockout event
- Map attack techniques to the MITRE ATT&CK framework

-----

##  Key Concepts Covered

|Concept                    |Security+ Domain                    |
|---------------------------|------------------------------------|
|Password Spray Attack      |Domain 1.0 - Threats & Attacks (24%)|
|Account Lockout Policy     |Domain 3.0 - Implementation (25%)   |
|SIEM Detection & Alerting  |Domain 4.0 - Incident Response (16%)|
|Active Directory Management|Domain 3.0 - Implementation (25%)   |
|MITRE ATT&CK Mapping       |Domain 1.0 - Threats & Attacks (24%)|

-----

## 📋 Lab Phases

-----

###  Phase 1 — Create Active Directory Users

**Objective:** Create test user accounts to simulate a real company environment with multiple employees.

**Commands Used:**

```powershell
New-ADUser -Name "jsmith" -AccountPassword (ConvertTo-SecureString "Summer2024!" -AsPlainText -Force) -Enabled $true
New-ADUser -Name "ssledge" -AccountPassword (ConvertTo-SecureString "Summer2024!" -AsPlainText -Force) -Enabled $true
New-ADUser -Name "smaddox" -AccountPassword (ConvertTo-SecureString "Summer2024!" -AsPlainText -Force) -Enabled $true
New-ADUser -Name "scoker" -AccountPassword (ConvertTo-SecureString "Summer2024!" -AsPlainText -Force) -Enabled $true
New-ADUser -Name "Adula" -AccountPassword (ConvertTo-SecureString "Summer2024!" -AsPlainText -Force) -Enabled $true
```

**Verified Users Created:**

```powershell
Get-ADUser -Filter * | Select-Object Name
```

**Result:**  5 AD user accounts successfully created and enabled.

-----

###  Phase 2 — Password Spray Attack Simulation

**Objective:** Simulate an attacker attempting one common password against multiple accounts to avoid triggering lockout thresholds.

**What Is A Password Spray?**

Unlike brute force attacks that try many passwords against one account, a password spray tries **one password against many accounts.** This is a common real-world technique because:

- It stays under account lockout thresholds
- It’s harder to detect than brute force
- It exploits weak/common passwords across large user bases

**Attack Commands Used:**

```powershell
net use \\localhost\IPC$ /user:jsmith WrongPassword1!
net use \\localhost\IPC$ /user:ssledge WrongPassword1!
net use \\localhost\IPC$ /user:smaddox WrongPassword1!
net use \\localhost\IPC$ /user:scoker WrongPassword1!
net use \\localhost\IPC$ /user:Adula WrongPassword1!
```

**Attack Results:**

```
jsmith   → System error 1326 - Login failure 
ssledge  → System error 1326 - Login failure 
smaddox  → System error 1326 - Login failure 
scoker   → System error 1326 - Login failure 
Adula    → System error 1326 - Login failure 
```

**MITRE ATT&CK Mapping:**

|Tactic           |Technique        |ID       |
|-----------------|-----------------|---------|
|Credential Access|Password Spraying|T1110.003|
|Initial Access   |Valid Accounts   |T1078    |

**Result:**  Password spray attack successfully simulated across all 5 accounts.

-----

###  Phase 3 — Wazuh SIEM Detection

**Objective:** Verify Wazuh detects the failed login attempts and account lockout events.

**What Wazuh Detected:**

|Event                |Windows Event ID|Wazuh Rule|Description                                    |
|---------------------|----------------|----------|-----------------------------------------------|
|Failed login attempts|4625            |60122     |Login failure - bad password                   |
|Multiple failures    |4625            |60204     |Multiple authentication failures               |
|Account locked out   |4740            |60113     |User account locked out (multiple login errors)|

**Key Alert:**

```
Time:    Mar 10, 2026 @ 22:48:45
Agent:   windows-server-2022
Rule:    User account locked out (multiple login errors)
Rule ID: 60113
Level:   8
```

**Result:**  Wazuh successfully detected both the password spray attempts and the subsequent account lockout event.

-----

###  Phase 4 — Defense: Account Lockout Policy

**Objective:** Implement and verify an Account Lockout Policy to prevent password spray and brute force attacks.

**Policy Applied:**

```powershell
net accounts /lockoutthreshold:3 /lockoutduration:30 /lockoutwindow:30
```

**Policy Settings Verified:**

```
Lockout threshold:              3 attempts
Lockout duration:               30 minutes
Lockout observation window:     30 minutes
Computer role:                  PRIMARY
```

**What Each Setting Does:**

|Setting          |Value |Purpose                               |
|-----------------|------|--------------------------------------|
|Lockout threshold|3     |Lock account after 3 wrong passwords  |
|Lockout duration |30 min|Keep account locked for 30 minutes    |
|Lockout window   |30 min|Reset attempt counter after 30 minutes|

**Policy Test — Triggering The Lockout:**

```powershell
# Attempt 1 - Wrong password
net use \\localhost\IPC$ /user:jsmith WrongPassword1!
→ "The user name or password is incorrect"

# Attempt 2 - Wrong password
net use \\localhost\IPC$ /user:jsmith WrongPassword1!
→ "The user name or password is incorrect"

# Attempt 3 - Wrong password
net use \\localhost\IPC$ /user:jsmith WrongPassword1!
→ "The user name or password is incorrect"

# Attempt 4 - Account now locked
net use \\localhost\IPC$ /user:jsmith WrongPassword1!
→ System error 1909
→ "The referenced account is currently locked out
   and may not be logged on to"
```

**Result:**  Account lockout policy successfully blocked further login attempts after 3 failures.

-----

###  Phase 5 — Wazuh Detects The Lockout

**Objective:** Confirm Wazuh fires an alert when an account gets locked out.

**Wazuh Alert Generated:**

```
Event ID:    4740
Rule ID:     60113
Description: User account locked out (multiple login errors)
Agent:       windows-server-2022
Severity:    Level 8
```

**Result:**  Wazuh automatically detected and alerted on the account lockout — exactly how a SOC analyst would be notified in a real environment.

-----

##  Lab Summary

|Phase  |Action                              |Result|
|-------|------------------------------------|------|
|Phase 1|Created 5 AD users via PowerShell   | PASS|
|Phase 2|Simulated password spray attack     | PASS|
|Phase 3|Wazuh detected failed login attempts| PASS|
|Phase 4|Implemented account lockout policy  | PASS|
|Phase 5|Wazuh detected account lockout event| PASS|

-----

##  Password Spray vs Brute Force

|                |Brute Force                 |Password Spray              |
|----------------|----------------------------|----------------------------|
|**Method**      |Many passwords → One account|One password → Many accounts|
|**Speed**       |Fast                        |Slow and deliberate         |
|**Lockout risk**|High - triggers quickly     |Low - stays under threshold |
|**Detection**   |Easier                      |Harder                      |
|**Defense**     |Account lockout             |Account lockout + MFA       |

-----

##  Key Takeaways

- **Password spray attacks** are a preferred technique because they avoid account lockouts
- **Account Lockout Policy** is the primary defense but must be paired with **MFA** for full protection
- **Wazuh SIEM** automatically detects both individual failures (Event 4625) and lockout events (Event 4740)
- **Windows Event IDs** are critical for SOC analysts — 4625 and 4740 are among the most important to know
- **Active Directory** is the primary identity management system in enterprise environments and a constant target for attackers
- A single compromised account can allow **lateral movement** across an entire network

-----

## MITRE ATT&CK Techniques Demonstrated

|Technique           |ID       |Tactic                      |
|--------------------|---------|----------------------------|
|Password Spraying   |T1110.003|Credential Access           |
|Valid Accounts      |T1078    |Initial Access / Persistence|
|Account Manipulation|T1098    |Persistence                 |

-----

##  Tools & Technologies Used

- Windows Server 2022 Active Directory
- PowerShell
- Wazuh SIEM 4.7.5
- Proxmox Virtual Environment
- MITRE ATT&CK Framework
- Windows Event Viewer (Event IDs 4625, 4740)

-----

## References

- [MITRE ATT&CK T1110.003 - Password Spraying](https://attack.mitre.org/techniques/T1110/003/)
- [Microsoft - Windows Security Event ID 4625](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/event-4625)
- [Microsoft - Windows Security Event ID 4740](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/event-4740)
- [Wazuh Documentation](https://documentation.wazuh.com)
- [CompTIA Security+ Exam Objectives](https://www.comptia.org/certifications/security)

-----

## 🔗 Related Labs

- [Wazuh SIEM Home Lab](../Wazuh-SIEM-Lab) — SIEM deployment, brute force detection, file integrity monitoring
