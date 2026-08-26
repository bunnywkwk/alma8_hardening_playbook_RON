# RHEL8-CIS Hardening Exam - Practical Documentation

## 1. Initial VM Setup & Pre-requisites
- **Networking:** After booting the AlmaLinux 8 VM, enabled "automatically connect" for the network interface to ensure a DHCP IP was assigned.
- **SSH Access:** Fixed and addressed the IP, generated SSH keys, and copied them to the VM using `ssh-copy-id` to allow passwordless login.
- **Inventory Setup:** Created the `hosts.yml` inventory file to define the `almalinux8` target and SSH connection arguments.
- **GUI Installation:** Manually installed the "Server with GUI" group on the AlmaLinux 8 VM.
- **Privilege Escalation:** Configured the `/etc/sudoers` file for the `aeron` user to allow privilege escalation without requiring a password (`NOPASSWD`).

## 2. Ansible Preparation
- **Role Installation:** Created `requirements.yml` and installed the upstream `RHEL8-CIS` hardening role.

## 3. Playbook Execution & Troubleshooting
During the first runs of the playbook, several issues were encountered and resolved via variables in `playbook.yml`:

1. **Python Interpreter Issues:** The initial run failed because the default Python on AlmaLinux 8 lacked the required RPM modules. This required changing the Python directory/interpreter.
2. **Authselect Profile Missing:** The playbook failed because a custom authselect profile was required.
   - *Resolution:* Added `rhel8cis_authselect_custom_profile_name: aeron-profile` to `playbook.yml`.
3. **Missing Git:** Encountered an error because `git` was not installed on the VM.
4. **GPG Keys False Alarm:** An error regarding GPG keys blocked execution.
   - *Resolution:* Bypassed the rule by adding `rhel8cis_rule_1_2_1_1: false` to `playbook.yml`.

## 4. The Core Problem: The Python Compatibility Wall
*(Justification for the Mentor)*

**1. The Core Problem: A Version Gap**
My laptop is running the brand-new 2026 version of Ansible Core (2.20), while AlmaLinux 8 is an older enterprise OS built around Python 3.6.

**2. The Chain Reaction**
Modern Ansible refused to run on Python 3.6 because it is simply too old to support the new Ansible syntax. To bypass these syntax errors, we forced Ansible to run using Python 3.9 on the VM (via `ansible_python_interpreter: /usr/bin/python3.9`).

**3. The Missing System Drivers**
However, in AlmaLinux 8, Red Hat compiled all native OS tools (SELinux, DNF, firewalld) strictly for Python 3.6. Because Python 3.9 does not have these native Red Hat libraries attached to it, any Ansible task trying to touch the OS (like `ansible.builtin.package`, `ansible.posix.firewalld`, or `ansible.posix.selinux`) throws a "missing Python library" error.

## 5. Extensive Bypasses (The Workaround)
Because of the missing Python 3.9 system drivers mentioned above, the playbook crashed continuously on any task that installed/removed packages or configured the firewall/SELinux.

To continue progress natively from the laptop without rebuilding the entire environment, a massive block of bypasses was added to `playbook.yml`.

**Key Bypassed Categories in `playbook.yml`:**
- **SELinux Rules (1.3.1.x):** Bypassed due to missing `python39-libselinux`.
- **GDM Banner Rules (1.8.x):** Bypassed due to broken/missing template files in the CIS role.
- **Section 2 Package Removals (2.1.1 - 2.2.5):** Bypassed entirely because Python 3.9 cannot use the DNF API to remove packages like `autofs`, `avahi`, `dhcp-server`, etc.
- **Section 4 Firewall Rules (4.1.4 - 4.1.6):** Bypassed because Python 3.9 cannot use the `firewall` module to set active zones.
- **Sudoers Lockout (5.2.4 & 5.2.5):** Bypassed because the CIS role enforces `PASSWD` for sudo. Since Ansible was running without `-K`, it locked itself out mid-run and crashed on `Gathering Facts`.
