# RHEL8-CIS Hardening Exam - Practical Documentation
**Author:** aeron

## 1. Initial VM Setup & Pre-requisites
- **Networking:** After booting the AlmaLinux 8 VM, I enabled "automatically connect" for the network interface to ensure a DHCP IP was assigned.
- **SSH Access:** I addressed the IP configuration, generated SSH keys, and securely copied them to the VM using `ssh-copy-id` to enable passwordless login for automation.
- **Inventory Setup:** I created the `hosts.yml` inventory file to define the `almalinux8` target and configured the necessary SSH connection arguments.
- **GUI Installation:** Manually installed the "Server with GUI" group on the AlmaLinux 8 VM to retain a desktop environment.
- **Privilege Escalation:** Configured the `/etc/sudoers` file for the `aeron` user to allow initial privilege escalation without requiring a password (`NOPASSWD`).

## 2. Ansible Preparation
- **Role Installation:** Created a `requirements.yml` file and successfully installed the upstream `RHEL8-CIS` hardening role.

## 3. Variable Organization Strategy
To maintain a clean `playbook.yml` and adhere to Ansible best practices, I separated all rule modifications into two distinct variable files based on their logical purpose:

### A. `group_vars/all.yml` (Infrastructure Bugs & Limitations)
This file holds the block of bypasses related strictly to a Python 3.9 compatibility gap discovered during testing (detailed in Section 4). Placing these in `group_vars` ensures that any future RHEL 8 node added to this infrastructure will automatically inherit these necessary bug fixes.
- **SELinux Rules (1.3.1.x):** Bypassed due to the missing `python39-libselinux` bindings.
- **Package Managers (Sections 2, 3, 4, 5, 6):** Bypassed because Python 3.9 cannot use the DNF API to remove/install packages natively on this OS version.
- **Firewall Rules (4.1.x):** Bypassed due to the missing `python3-firewall` bindings.
- **Sudoers Lockout (5.2.4 & 5.2.5):** Bypassed because the CIS role strictly enforces `PASSWD` for sudo. While secure, this breaks Ansible automation mid-run if `-K` isn't actively utilized.
- **AIDE Config (6.1.2 & 6.1.3):** Bypassed because the initial AIDE package installation was skipped due to the aforementioned DNF module bug.

### B. `host_vars/almalinux8.yml` (Host-Specific Logical Bypasses)
This file contains overrides and bypasses specific to the `almalinux8` VM instance. These represent deliberate engineering choices made for this environment:
- **Authselect Profile:** Added `rhel8cis_authselect_custom_profile_name: aeron-profile` to satisfy custom audit profile requirements.
- **Desktop Requirement:** Added `rhel8cis_desktop_required: true` to prevent the GUI from being uninstalled during the hardening process.
- **GPG Keys:** Bypassed `rhel8cis_rule_1_2_1_1: false` to resolve a false alarm specific to this VM's repository setup.
- **GDM Banner Rules (1.8.x):** Bypassed due to broken or missing template files in the upstream CIS role repository.
- **Crypto-Policy (1.6.x):** Bypassed as the cryptography policies were already enforced locally.

## 4. Technical Challenges & Compatibility Resolution
During the execution of the playbook, a significant version mismatch was identified and documented between the control node and the target node.

**1. The Version Gap**
The control node is running a modern release of Ansible Core (2.20), while AlmaLinux 8 is an older enterprise OS built around Python 3.6.

**2. The Execution Chain Reaction**
Modern Ansible (2.20) refused to execute on Python 3.6 because the language version is too old to support modern Ansible syntax. To bypass these syntax errors, Ansible was forced to execute using Python 3.9 on the VM (via `ansible_python_interpreter: /usr/bin/python3.9`).

**3. The System Driver Limitation**
However, in AlmaLinux 8, native OS tools (SELinux, DNF, firewalld) are compiled strictly for Python 3.6. Because the Python 3.9 environment does not have these native Red Hat C-libraries attached to it, any Ansible task attempting to interact with the OS package manager or firewall (e.g., `ansible.builtin.package`, `ansible.posix.firewalld`, or `ansible.posix.selinux`) threw a "missing Python library" error. This was resolved by meticulously documenting and bypassing the affected tasks in `group_vars/all.yml` to ensure the rest of the security baseline could be successfully applied.

## 5. Post-Remediation Verification
After the playbook successfully completed, the following tests were conducted to verify the integrity of the CIS hardening:
- **Root SSH Denial:** Verified that direct `root` login over SSH is blocked (`PermitRootLogin no`).
- **USB Storage Block:** Verified that the USB storage kernel module is disabled (`modprobe -n -v usb-storage`).
- **PAM Faillock (Brute Force Protection):** Verified that the account lockout policies are active for repeated failed logins.
- **Audit Logging:** Verified that `auditd` is actively running and enforcing kernel-level tracking (`systemctl status auditd`).
- **Compliance Audit:** Reviewed the generated Goss Audit report located in `/opt/` to verify the final pass/fail metrics.
