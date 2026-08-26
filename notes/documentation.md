# RHEL8-CIS Hardening Exam - Practical Documentation

## 1. Initial VM Setup & Pre-requisites
- **Networking:** After booting the AlmaLinux 8 VM, enabled "automatically connect" for the network interface to ensure a DHCP IP was assigned.
- **SSH Access:** Fixed and addressed the IP, generated SSH keys, and copied them to the VM using `ssh-copy-id` to allow passwordless login.
- **Inventory Setup:** Created the `hosts.yml` inventory file to define the `almalinux8` target and SSH connection arguments.
- **GUI Installation:** Manually installed the "Server with GUI" group on the AlmaLinux 8 VM.
- **Privilege Escalation:** Configured the `/etc/sudoers` file for the `aeron` user to allow privilege escalation without requiring a password (`NOPASSWD`).

## 2. Ansible Preparation
- **Role Installation:** Created `requirements.yml` and installed the upstream `RHEL8-CIS` hardening role.

## 3. Variable Organization Strategy
To maintain a clean `playbook.yml` and follow Ansible best practices, all rule modifications were separated into two distinct variable files based on their logical purpose:

### A. `group_vars/all.yml` (Infrastructure Bugs & Limitations)
This file holds the massive block of bypasses related strictly to the Python 3.9 compatibility gap. Placing these in `group_vars` ensures that any future RHEL 8 node added to this infrastructure will automatically inherit these required bug fixes.
- **SELinux Rules (1.3.1.x):** Bypassed due to missing `python39-libselinux`.
- **Package Managers (Sections 2, 3, 4, 5, 6):** Bypassed entirely because Python 3.9 cannot use the DNF API to remove/install packages natively.
- **Firewall Rules (4.1.x):** Bypassed due to missing `python3-firewall`.
- **Sudoers Lockout (5.2.4 & 5.2.5):** Bypassed because the CIS role enforces `PASSWD` for sudo, which breaks automation if `-K` isn't utilized.
- **AIDE Config (6.1.2 & 6.1.3):** Bypassed because the AIDE package installation itself was skipped due to the DNF bug.

### B. `host_vars/almalinux8.yml` (Host-Specific Logical Bypasses)
This file contains overrides and bypasses specific to the `almalinux8` VM instance. These are logical choices made by the engineer.
- **Authselect Profile Missing:** Added `rhel8cis_authselect_custom_profile_name: aeron-profile`.
- **Desktop Requirement:** Added `rhel8cis_desktop_required: true` to prevent the GUI from being uninstalled.
- **GPG Keys False Alarm:** Bypassed `rhel8cis_rule_1_2_1_1: false`.
- **GDM Banner Rules (1.8.x):** Bypassed due to broken/missing template files in the upstream CIS role.
- **Crypto-Policy (1.6.x):** Bypassed as it was already enforced locally.

## 4. The Core Problem: The Python Compatibility Wall
*(Justification for the Mentor)*

**1. The Core Problem: A Version Gap**
My laptop is running the brand-new 2026 version of Ansible Core (2.20), while AlmaLinux 8 is an older enterprise OS built around Python 3.6.

**2. The Chain Reaction**
Modern Ansible refused to run on Python 3.6 because it is simply too old to support the new Ansible syntax. To bypass these syntax errors, we forced Ansible to run using Python 3.9 on the VM (via `ansible_python_interpreter: /usr/bin/python3.9`).

**3. The Missing System Drivers**
However, in AlmaLinux 8, Red Hat compiled all native OS tools (SELinux, DNF, firewalld) strictly for Python 3.6. Because Python 3.9 does not have these native Red Hat libraries attached to it, any Ansible task trying to touch the OS (like `ansible.builtin.package`, `ansible.posix.firewalld`, or `ansible.posix.selinux`) throws a "missing Python library" error.
