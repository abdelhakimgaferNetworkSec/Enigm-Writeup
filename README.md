# Hack The Box: Enigma - Comprehensive Penetration Testing & Write-Up Report

## 1. Executive Summary
**Enigma** is an intermediate-level Linux machine hosted on Hack The Box. The assessment methodology follows a standard penetration testing lifecycle: reconnaissance, enumeration, initial access via credential harvesting, local enumeration, and root privilege escalation through a service-layer vulnerability (**CVE-2026-27626** in **OliveTin**). This write-up details the step-by-step vector analysis, toolsets, code execution strategies, and remediation guidelines required to completely compromise the target and secure administrative control.

---

## 2. Information Gathering & Reconnaissance

### 2.1 Network Port Scanning (`Nmap`)
The assessment commenced with a TCP SYN scan to identify open ports and services running on the target host.

```bash
nmap -p- --min-rate=1000 -T4 <TARGET_IP>
```

**Scan Findings:**

* **Port 22/tcp (SSH)**: Open, OpenSSH daemon running.
* **Port 80/tcp (HTTP)**: Open, Nginx/Apache web server hosting a web application.
* **Port 2049/tcp (NFS)**: Open, network file sharing service.

### 2.2 Web Enumeration & Directory Brute-Forcing

Using directory discovery tools (`gobuster` / `ffuf`), we scanned the web service on port 80 to uncover hidden endpoints, administrative portals, or configuration artifacts.

* Discovered a public-facing web presence and developer blog posts outlining internal infrastructure components.
* Identified potential application paths related to database configuration files and automation scripts.

---

## 3. Initial Access (User-Level Compromise)

### 3.1 Credential Discovery

Through detailed analysis of the web application files and auxiliary NFS shares, we uncovered valid developer credentials.

* **Username**: `haris`
* **Password**: `bestfriends`

### 3.2 Establishing the User Shell via SSH

With the discovered credentials, we authenticated against the SSH service to establish an interactive low-privileged shell on the target system:

```bash
ssh haris@<TARGET_IP>
```

* **Interactive Shell Acquired**: `haris@enigma:~$`
* Successfully located and retrieved the user flag (`user.txt`).

---

## 4. Local Enumeration & Privilege Escalation Discovery

### 4.1 Internal Service Auditing

Upon establishing a user-level shell, we reviewed local listening ports and active processes to identify potential attack surfaces that are not exposed externally:

```bash
ss -tulpn
```

**Key Discovery:**

* The **OliveTin** automation management web service was running locally on `127.0.0.1:1337` under `root` privileges.

### 4.2 Analyzing OliveTin Configuration and Vulnerabilities

Further inspection of the OliveTin application configuration file (`/opt/OliveTin/config.yaml`) revealed that database backup actions were executed via a shell wrapper. Specifically, the configuration utilized the `db_pass` parameter within a `mysqldump` command string:

```yaml
shell: "mysqldump -u {{ db_user }} -p'{{ db_pass }}' {{ db_name }} > /opt/backups/backup.sql"
arguments:
  - name: db_pass
    type: password
```

This configuration introduces a critical command injection vulnerability (**CVE-2026-27626**) because the `db_pass` parameter is enclosed in single quotes within a shell context, but does not properly sanitize input containing shell control operators (e.g., semicolons `;`, pipes `|`, or backticks).

---

## 5. Exploitation Strategy & Execution (CVE-2026-27626)

### 5.1 Setting up an SSH Port Forwarding Tunnel

Since the vulnerable OliveTin service is bound strictly to `127.0.0.1:1337`, external requests from the Kali attacking machine are blocked with a `Connection refused` error. To interface with the service, we established a local port-forwarding tunnel via SSH:

```bash
ssh -L 1337:127.0.0.1:1337 haris@<TARGET_IP>
```

### 5.2 Crafting and Executing the Exploit Payload

Using a Python exploit script targeting the OliveTin API endpoint (`/api/olivetin.api.v1.OliveTinApiService/StartAction`), we injected arbitrary system commands into the vulnerable parameter vector.

Due to strict shell syntax and directory dependencies in the original `mysqldump` command wrapper, direct file creation could sometimes result in exit status errors (such as missing target directories or permission constraints). By crafting precise execution strings or querying the file system directly through the API vulnerability, we bypassed execution hurdles.

Running the command payload via the exploit interface:

```bash
python3 CVE-2026.27626.py -u http://127.0.0.1 -x "cat /root/root.txt"
```

**Execution Output:**

```text
[+] Execution ID: 7fd15259-3996-4e03-9ebd-329034ee4e0d
======================================================================
Command Output
======================================================================
exit status 127
Usage: mysqldump [OPTIONS] database [tables]
...
ae9306f5690c4812c6f9445027d0d060
sh: 1: : Permission denied
======================================================================
```

The root flag was successfully extracted directly through the command output stream.

---

## 6. Post-Exploitation & Evidence of Compromise

* **Root Flag Acquired**: `ae9306f5690c4812c6f9445027d0d060`
* **Administrative Persistence**: An alternative persistence mechanism involves appending an SSH public key to `/root/.ssh/authorized_keys` via the command injection vector, granting seamless administrative terminal access.

---

## 7. Remediation & Defense Recommendations

1. **Input Sanitization & Parameterization**: Ensure that application parameters passed to system commands (such as database passwords in OliveTin) are strictly validated against alphanumeric character sets, prohibiting raw shell metacharacters.
2. **Principle of Least Privilege**: Avoid running management automation software like OliveTin under the `root` security context unless absolutely necessary. Run services under dedicated service accounts with minimal permissions.
3. **Software Updates**: Upgrade OliveTin and associated automation wrappers to patched versions where command injection vulnerabilities via templated parameters are mitigated.
4. **Network Segmentation**: Restrict management interfaces from binding to loopback interfaces if administrative oversight requires multi-factor authentication or network-level access control lists (ACLs).
