# Cisco AAA TACACS+ Lab

## Overview

This project demonstrates the implementation of a centralized AAA (Authentication, Authorization, and Accounting) architecture using TACACS+ integrated with Cisco IOS in an EVE-NG environment.

The goal is to eliminate local user management on network devices and enforce centralized access control using Role-Based Access Control (RBAC), ensuring secure and auditable administrative access.

## 🎯 Objectives

The main objectives of this lab are:

- Implement centralized Authentication, Authorization, and Accounting (AAA) using TACACS+
- Replace local user management with a centralized access control system
- Enforce Role-Based Access Control (RBAC) on Cisco IOS devices
- Restrict user actions based on defined privilege levels
- Enable command-level authorization for different user roles
- Provide full accounting and auditability of all administrative actions
- Ensure fallback authentication using local credentials in case of TACACS+ failure

## 🏗️ Network Topology

The lab is built around a simple network architecture designed to simulate centralized AAA authentication using TACACS+.

A single LAN segment is used (`192.168.10.0/24`) where all devices communicate within a controlled environment.

The TACACS+ server acts as the central security component of the lab, handling all authentication, authorization, and accounting requests from network devices.

An admin workstation is used to simulate a real operator performing remote access.

![Topology](topology/topology.png)

| Device | Interface | IP Address | Role |
|--------|-----------|------------|------|
| TACACS Server (Ubuntu 22.04) | e0 | 192.168.10.10 | AAA Server |
| Cisco Router (IOS 7200) | fa0/0 | 192.168.10.3 | TACACS Client |
| Admin-PC (Ubuntu 22.04) | e0 | 192.168.10.50 | Management Host |

## 🔐 Role-Based Access Control (RBAC)

The lab implements Role-Based Access Control (RBAC) to enforce the principle of least privilege and separate administrative duties from basic monitoring tasks.

Two user roles are defined in the TACACS+ server:

#### Super Administrator

- Group: grp_superadmin  
- Privilege level: 15 (full administrative access)  
- Permissions: Read / Write / Configuration access  
- Cisco mapping: Enable mode + full EXEC privileges  

#### Junior Operator

- Group: grp_junior  
- Privilege level: 1 (read-only access)  
- Permissions: Monitoring and diagnostic commands only  
- Allowed commands: show commands  
- Cisco mapping: User EXEC mode restricted access  

## ⚙️ TACACS+ Server Deployment (Ubuntu)

The TACACS+ service is deployed on an Ubuntu Server and provides centralized Authentication, Authorization, and Accounting (AAA) for network devices.

### Installation

The TACACS+ daemon is installed using the official Ubuntu repositories:

```bash
sudo apt update
sudo apt install tacacs+
```

### Logging Configuration

A dedicated directory is created to store accounting logs:

```bash
sudo mkdir -p /var/log/tacacs
sudo chown tacacs:tacacs /var/log/tacacs
```

### TACACS+ Configuration

The main configuration file defines authentication, authorization, and accounting policies:

```bash
/etc/tacacs+/tac_plus.conf
```
This configuration includes:

- Shared secret key for device authentication
- Accounting file for session and command logging
- Role-Based Access Control (RBAC) groups
- User-to-role mapping

---

### Configuration Example

```text
# Shared secret key
key = "key1234"

# Accounting log file
accounting file = /var/log/tac_plus.acct

# Super Administrator group
group = grp_superadmin {
    default service = permit
    service = exec { priv-lvl = 15 }
}

# Junior Operator group
group = grp_junior {
    service = exec { priv-lvl = 1 }
    cmd = show { permit .* }
    cmd = exit { permit .* }
}

# Users
user = admin_lab { login = cleartext "admin123" member = grp_superadmin }
user = user_junior { login = cleartext "user123" member = grp_junior }
```

### Service Management

After applying the configuration, restart and verify the service:

```bash
sudo systemctl restart tacacs+
sudo systemctl status tacacs+
```

## 🔐 Cisco IOS AAA Configuration

The Cisco router is configured to use the TACACS+ server for centralized Authentication, Authorization, and Accounting (AAA).

### TACACS+ Server Definition

The TACACS+ server is declared using its IP address and the shared secret key previously configured on the Ubuntu server.

```text
conf t
aaa new-model

tacacs server TACACS-SRV
 address ipv4 192.168.10.10
 key key1234

exit
```

### AAA Policies

Default AAA method lists are configured for authentication, authorization, and accounting.

Local authentication is added as a fallback mechanism in case the TACACS+ server becomes unavailable.

```text
aaa authentication login default group tacacs+ local
aaa authorization exec default group tacacs+ local
aaa authorization commands 15 default group tacacs+ local
aaa authorization commands 1 default group tacacs+ local
aaa accounting exec default start-stop group tacacs+
aaa accounting commands 15 default start-stop group tacacs+
aaa accounting commands 1 default start-stop group tacacs+
```
> ⚠️ **Critical Fallback Note**: In production environments, it is imperative to maintain a local administrative account on the network device and append local as the final method in the AAA authentication list. This prevents an administrative lockout if the TACACS+ server becomes unreachable due to network failures or daemon crashes. The router will only fall back to the local database if the TACACS+ server fails to respond; if the server responds with a rejection (wrong credentials), access is denied, and fallback is not triggered.

### AAA Verification

The active AAA configuration can be verified using:

```text
show run | include aaa
```

Expected output:

```text
aaa new-model
aaa authentication login default group tacacs+ local
aaa authorization exec default group tacacs+ local
aaa authorization commands 1 default group tacacs+ local
aaa authorization commands 15 default group tacacs+ local
aaa accounting exec default start-stop group tacacs+
aaa accounting commands 1 default start-stop group tacacs+
aaa accounting commands 15 default start-stop group tacacs+
aaa session-id common
```

### Remote Access Configuration (Optional)

To apply AAA policies to remote administrative access (Telnet/SSH), the VTY lines must reference the AAA method lists.

```text
line vty 0 4
 login authentication default
 authorization exec default
 authorization commands 1 default
 authorization commands 15 default
 transport input ssh telnet
exit
```

#### SSH support is enabled with:

```text
ip domain-name lab.com
crypto key generate rsa modulus 1024
ip ssh version 2
```

### Console Authorization (Optional)

AAA authorization can also be extended to console access:

```text
aaa authorization console

line console 0
 login authentication default
 authorization exec default
exit
```

### TACACS+ Connectivity Verification

Communication between the Cisco router and the TACACS+ server can be validated using:

```text
show tacacs
```

Example output:

```text
Tacacs+ Server - public :
            Server address: 192.168.10.10
               Server port: 49
              Socket opens:         40
             Socket closes:         40
             Socket aborts:          0
             Socket errors:          0
           Socket Timeouts:          0
   Failed Connect Attempts:          0
        Total Packets Sent:         56
        Total Packets Recv:         56
```

## 🧪 Verification & Testing

To validate the AAA implementation, operational scenarios were tested from a management workstation using PuTTY.

> ⚠️ **Production Note:** This lab uses Telnet (Port 23) exclusively for rapid prototyping in a controlled environment. In production networks, security standards mandate disabling unencrypted management protocols and strictly enforcing **SSH (v2)**.

---

#### 1. Super Administrator Access & Full Privilege Authorization

When connecting with the `admin_lab` credentials, the user is directly placed into **privilege level 15**, as indicated by the `#` prompt.

![Admin Authentication](assets/admin-auti.png)

To verify full authorization to apply changes, global configuration mode was accessed to create a loopback interface without restrictions:

![Admin Command Authorization](assets/admin-auto.png)

---

#### 2. Junior Operator Access & Command Restrictions

When connecting with the Junior Operator credentials (`user_junior`), the system grants **privilege level 1**, restricting the environment to the `>` prompt.

![Junior Authentication](assets/user-auti.png)

Under this restricted profile, privilege escalation via the `enable` command is completely blocked by the TACACS+ server. The user is permitted to perform monitoring tasks (such as `show ip interface brief`) but cannot make global system modifications.

![Junior Command Restrictions](assets/user-auto.png)

---

#### 3. Centralized Audit Trail (Accounting Logs)

Every single connection state and privilege execution is logged on the central TACACS+ daemon. To audit administrative actions, we inspect the central tracking file:

```text
tacacs@tacacs:~$ sudo nano /var/log/tacacs/tac_plus.acct
```

![Terminal ScreenShot](assets/accounting.png)

```text
May 19 13:22:07 192.168.10.3    admin   tty2    192.168.10.50   start   task_id=20      timezone=UTC    service=shell
May 19 13:22:11 192.168.10.3    admin   tty2    192.168.10.50   stop    task_id=20      timezone=UTC    service=shell   priv-lvl=15     cmd=configure terminal <cr>
May 19 13:22:17 192.168.10.3    admin   tty2    192.168.10.50   stop    task_id=21      timezone=UTC    service=shell   priv-lvl=15     cmd=interface Loopback 1 <cr>
May 19 13:22:25 192.168.10.3    admin   tty2    192.168.10.50   stop    task_id=20      timezone=UTC    service=shell   disc-cause=1    disc-cause-ext=9        pre-session-time=10     elap$
May 19 13:22:34 192.168.10.3    user    tty2    192.168.10.50   start   task_id=23      timezone=UTC    service=shell
May 19 13:23:04 192.168.10.3    user    tty2    192.168.10.50   stop    task_id=23      timezone=UTC    service=shell   disc-cause=1    disc-cause-ext=9        pre-session-time=5
```

## 🔍 Troubleshooting

If authentication or authorization failures occur during testing, use the following Cisco IOS diagnostic commands to isolate the root cause:

* `debug aaa authentication` – Monitors the authentication exchange step-by-step to check user matching.
* `debug aaa authorization` – Validates command processing and privilege level attributes received from TACACS+.
* `debug tacacs` – Tracks raw packet synchronization, timing, and transport layer connectivity with the Ubuntu Server.