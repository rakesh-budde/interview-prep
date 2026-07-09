# Ansible Interview Questions - Complete Guide

> **200+ Ansible Interview Questions for Senior DevOps Engineer, SRE, and Platform Engineer roles at FAANG companies**

---

## 📋 Table of Contents

- [Ansible Architecture](#ansible-architecture)
- [Playbooks & Plays](#playbooks--plays)
- [Inventory Management](#inventory-management)
- [Roles & Collections](#roles--collections)
- [Variables & Facts](#variables--facts)
- [Modules & Plugins](#modules--plugins)
- [Ansible Vault](#ansible-vault)
- [AWX/Tower](#awxtower)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)

---

## Ansible Architecture

### 🟢 Basic Questions

#### Q1: Explain Ansible architecture and why it's agentless.

**Basic Answer:**
Ansible uses a push-based, agentless architecture. It connects to managed nodes via SSH (Linux) or WinRM (Windows), executes tasks, and disconnects. No software needs to be installed on managed nodes.

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ANSIBLE ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  CONTROL NODE                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  ┌──────────────────────────────────────────────────┐   │    │
│  │  │              Ansible Engine                       │   │    │
│  │  │  ┌─────────┐ ┌─────────┐ ┌─────────┐            │   │    │
│  │  │  │Inventory│ │Playbooks│ │ Modules │            │   │    │
│  │  │  └─────────┘ └─────────┘ └─────────┘            │   │    │
│  │  │  ┌─────────┐ ┌─────────┐ ┌─────────┐            │   │    │
│  │  │  │ Plugins │ │  Roles  │ │  Vault  │            │   │    │
│  │  │  └─────────┘ └─────────┘ └─────────┘            │   │    │
│  │  └──────────────────────────────────────────────────┘   │    │
│  │                        │                                 │    │
│  │                        ▼                                 │    │
│  │              Connection Plugins                          │    │
│  │         (SSH, WinRM, Docker, Local)                      │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                     │
│            ┌───────────────┼───────────────┐                    │
│            ▼               ▼               ▼                    │
│  MANAGED NODES                                                  │
│  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐         │
│  │   Linux Host  │ │ Windows Host │ │  Network Dev  │         │
│  │               │ │               │ │               │         │
│  │ SSH (Port 22) │ │WinRM (5985/6)│ │ SSH/NETCONF  │         │
│  │    Python     │ │  PowerShell  │ │               │         │
│  └───────────────┘ └───────────────┘ └───────────────┘         │
│                                                                  │
│  EXECUTION FLOW:                                                │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  1. Parse inventory → identify target hosts              │    │
│  │  2. Parse playbook → determine tasks                     │    │
│  │  3. Generate Python code from modules                    │    │
│  │  4. Transfer code to managed nodes via SSH/WinRM         │    │
│  │  5. Execute code on managed nodes                        │    │
│  │  6. Capture output and return to control node            │    │
│  │  7. Clean up temporary files on managed nodes            │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  WHY AGENTLESS:                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  ✓ No agent installation/maintenance                     │    │
│  │  ✓ No agent resource consumption                         │    │
│  │  ✓ No agent security vulnerabilities                     │    │
│  │  ✓ Simpler network requirements                          │    │
│  │  ✓ Works with existing SSH infrastructure                │    │
│  │  ✓ Faster to start managing new systems                  │    │
│  │                                                          │    │
│  │  Trade-offs:                                             │    │
│  │  • Requires SSH/WinRM access                             │    │
│  │  • Push-based (no continuous agent monitoring)           │    │
│  │  • Python required on Linux managed nodes                │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### 🟡 Intermediate Questions

#### Q2: Explain Ansible execution strategies and how to optimize performance.

**Basic Answer:**
Ansible supports linear (default), free, and serial strategies. Performance can be improved through pipelining, fact caching, async tasks, and proper fork configuration.

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    EXECUTION STRATEGIES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  LINEAR (Default):                                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Task 1: ■■■ Host1, Host2, Host3 (wait for all)         │    │
│  │  Task 2: ■■■ Host1, Host2, Host3 (wait for all)         │    │
│  │  Task 3: ■■■ Host1, Host2, Host3 (wait for all)         │    │
│  │                                                          │    │
│  │  All hosts complete task before moving to next           │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  FREE:                                                          │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Host1: ■■■■■■■■■■■■■■■■■■ (runs as fast as possible)   │    │
│  │  Host2: ■■■■■■■■■■■■■■ (independent of others)          │    │
│  │  Host3: ■■■■■■■■■■■■■■■■■■■■■ (no waiting)              │    │
│  │                                                          │    │
│  │  Each host runs independently at its own pace            │    │
│  │  strategy: free                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  SERIAL (Batch/Rolling):                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Batch 1: Host1, Host2 (complete all tasks)             │    │
│  │  Batch 2: Host3, Host4 (complete all tasks)             │    │
│  │  Batch 3: Host5, Host6 (complete all tasks)             │    │
│  │                                                          │    │
│  │  serial: 2  # or "30%" or [1, 5, 10]                     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  PERFORMANCE OPTIMIZATION:                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  1. SSH Pipelining (ansible.cfg):                        │    │
│  │     [ssh_connection]                                     │    │
│  │     pipelining = True     # Reduces SSH operations       │    │
│  │                                                          │    │
│  │  2. Fact Caching:                                        │    │
│  │     [defaults]                                           │    │
│  │     gathering = smart                                    │    │
│  │     fact_caching = jsonfile                              │    │
│  │     fact_caching_connection = /tmp/ansible_facts         │    │
│  │     fact_caching_timeout = 86400                         │    │
│  │                                                          │    │
│  │  3. Increase Forks:                                      │    │
│  │     [defaults]                                           │    │
│  │     forks = 50            # Default is 5                 │    │
│  │                                                          │    │
│  │  4. Async Tasks:                                         │    │
│  │     - name: Long running task                            │    │
│  │       command: /usr/bin/long_task                        │    │
│  │       async: 3600         # Max runtime seconds          │    │
│  │       poll: 0             # Fire and forget              │    │
│  │                                                          │    │
│  │  5. Disable Fact Gathering (when not needed):            │    │
│  │     gather_facts: no                                     │    │
│  │                                                          │    │
│  │  6. Use Mitogen (third-party plugin):                    │    │
│  │     strategy_plugins = /path/to/mitogen/ansible_mitogen  │    │
│  │     strategy = mitogen_linear                            │    │
│  │     # Can provide 2-7x speedup                           │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Playbooks & Plays

### 🟡 Intermediate Questions

#### Q3: Explain idempotency in Ansible. How do you ensure tasks are idempotent?

**Basic Answer:**
Idempotency means running a task multiple times produces the same result as running it once. Ansible modules are designed to check current state before making changes.

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    IDEMPOTENCY                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  IDEMPOTENT PATTERN:                                            │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  Run 1: Check state → Change needed → Make change        │    │
│  │  Run 2: Check state → No change needed → Do nothing      │    │
│  │  Run 3: Check state → No change needed → Do nothing      │    │
│  │                                                          │    │
│  │  Result: Same end state regardless of run count          │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ✅ IDEMPOTENT EXAMPLES:                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  # File module - creates only if not exists              │    │
│  │  - name: Ensure directory exists                         │    │
│  │    file:                                                 │    │
│  │      path: /opt/myapp                                    │    │
│  │      state: directory                                    │    │
│  │      mode: '0755'                                        │    │
│  │                                                          │    │
│  │  # Package module - installs only if not present         │    │
│  │  - name: Ensure nginx installed                          │    │
│  │    apt:                                                  │    │
│  │      name: nginx                                         │    │
│  │      state: present                                      │    │
│  │                                                          │    │
│  │  # Service module - ensures desired state                │    │
│  │  - name: Ensure nginx running                            │    │
│  │    service:                                              │    │
│  │      name: nginx                                         │    │
│  │      state: started                                      │    │
│  │      enabled: yes                                        │    │
│  │                                                          │    │
│  │  # Template module - only changes if content differs     │    │
│  │  - name: Deploy config                                   │    │
│  │    template:                                             │    │
│  │      src: nginx.conf.j2                                  │    │
│  │      dest: /etc/nginx/nginx.conf                         │    │
│  │    notify: Restart nginx                                 │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ❌ NON-IDEMPOTENT EXAMPLES (Avoid):                            │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  # Shell/command without creates/removes                 │    │
│  │  - name: Run script (NOT IDEMPOTENT)                     │    │
│  │    shell: /opt/setup.sh                                  │    │
│  │                                                          │    │
│  │  # FIX: Use creates parameter                            │    │
│  │  - name: Run script (IDEMPOTENT)                         │    │
│  │    shell: /opt/setup.sh                                  │    │
│  │    args:                                                 │    │
│  │      creates: /opt/.setup_complete                       │    │
│  │                                                          │    │
│  │  # Lineinfile appending without check                    │    │
│  │  - name: Add line (might duplicate)                      │    │
│  │    lineinfile:                                           │    │
│  │      path: /etc/hosts                                    │    │
│  │      line: "192.168.1.1 myhost"                          │    │
│  │      # This is actually idempotent (checks for line)     │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  MAKING SHELL COMMANDS IDEMPOTENT:                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  # Option 1: creates/removes                             │    │
│  │  - name: Initialize database                             │    │
│  │    command: /usr/bin/db_init                             │    │
│  │    args:                                                 │    │
│  │      creates: /var/lib/db/.initialized                   │    │
│  │                                                          │    │
│  │  # Option 2: when condition with stat                    │    │
│  │  - name: Check if initialized                            │    │
│  │    stat:                                                 │    │
│  │      path: /var/lib/db/.initialized                      │    │
│  │    register: db_init                                     │    │
│  │                                                          │    │
│  │  - name: Initialize database                             │    │
│  │    command: /usr/bin/db_init                             │    │
│  │    when: not db_init.stat.exists                         │    │
│  │                                                          │    │
│  │  # Option 3: changed_when with check                     │    │
│  │  - name: Update config if needed                         │    │
│  │    shell: |                                              │    │
│  │      if ! grep -q "setting=value" /etc/config; then      │    │
│  │        echo "setting=value" >> /etc/config               │    │
│  │        echo "changed"                                    │    │
│  │      fi                                                  │    │
│  │    register: result                                      │    │
│  │    changed_when: "'changed' in result.stdout"            │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Roles & Collections

### 🔴 Advanced Questions

#### Q4: Explain Ansible Roles structure and best practices for creating reusable roles.

**Basic Answer:**
Roles provide a way to organize playbooks into reusable components with a standardized directory structure including tasks, handlers, templates, files, vars, defaults, and meta.

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ANSIBLE ROLES                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ROLE DIRECTORY STRUCTURE:                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  roles/                                                  │    │
│  │  └── nginx/                                              │    │
│  │      ├── tasks/                                          │    │
│  │      │   ├── main.yml        # Entry point               │    │
│  │      │   ├── install.yml     # Installation tasks        │    │
│  │      │   └── configure.yml   # Configuration tasks       │    │
│  │      ├── handlers/                                       │    │
│  │      │   └── main.yml        # Restart/reload handlers   │    │
│  │      ├── templates/                                      │    │
│  │      │   └── nginx.conf.j2   # Jinja2 templates          │    │
│  │      ├── files/                                          │    │
│  │      │   └── ssl-cert.pem    # Static files              │    │
│  │      ├── vars/                                           │    │
│  │      │   └── main.yml        # Role variables (high)     │    │
│  │      ├── defaults/                                       │    │
│  │      │   └── main.yml        # Default variables (low)   │    │
│  │      ├── meta/                                           │    │
│  │      │   └── main.yml        # Dependencies, metadata    │    │
│  │      ├── tests/                                          │    │
│  │      │   ├── inventory                                   │    │
│  │      │   └── test.yml        # Test playbook             │    │
│  │      └── README.md           # Documentation             │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  VARIABLE PRECEDENCE (lowest to highest):                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  1.  role defaults (defaults/main.yml)                   │    │
│  │  2.  inventory file or script group vars                 │    │
│  │  3.  inventory group_vars/all                            │    │
│  │  4.  playbook group_vars/all                             │    │
│  │  5.  inventory group_vars/*                              │    │
│  │  6.  playbook group_vars/*                               │    │
│  │  7.  inventory file or script host vars                  │    │
│  │  8.  inventory host_vars/*                               │    │
│  │  9.  playbook host_vars/*                                │    │
│  │  10. host facts / cached set_facts                       │    │
│  │  11. play vars                                           │    │
│  │  12. play vars_prompt                                    │    │
│  │  13. play vars_files                                     │    │
│  │  14. role vars (vars/main.yml)                           │    │
│  │  15. block vars                                          │    │
│  │  16. task vars                                           │    │
│  │  17. include_vars                                        │    │
│  │  18. set_facts / registered vars                         │    │
│  │  19. role params                                         │    │
│  │  20. include params                                      │    │
│  │  21. extra vars (-e) ← HIGHEST PRIORITY                  │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ROLE EXAMPLE - nginx/tasks/main.yml:                           │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  ---                                                     │    │
│  │  - name: Include OS-specific variables                   │    │
│  │    include_vars: "{{ ansible_os_family }}.yml"           │    │
│  │                                                          │    │
│  │  - name: Install nginx                                   │    │
│  │    package:                                              │    │
│  │      name: "{{ nginx_package }}"                         │    │
│  │      state: present                                      │    │
│  │                                                          │    │
│  │  - name: Deploy nginx configuration                      │    │
│  │    template:                                             │    │
│  │      src: nginx.conf.j2                                  │    │
│  │      dest: "{{ nginx_conf_path }}"                       │    │
│  │      validate: nginx -t -c %s                            │    │
│  │    notify: Reload nginx                                  │    │
│  │                                                          │    │
│  │  - name: Ensure nginx is running                         │    │
│  │    service:                                              │    │
│  │      name: nginx                                         │    │
│  │      state: started                                      │    │
│  │      enabled: yes                                        │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ROLE DEPENDENCIES (meta/main.yml):                             │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  ---                                                     │    │
│  │  dependencies:                                           │    │
│  │    - role: common                                        │    │
│  │    - role: firewall                                      │    │
│  │      vars:                                               │    │
│  │        firewall_allowed_ports:                           │    │
│  │          - 80                                            │    │
│  │          - 443                                           │    │
│  │                                                          │    │
│  │  galaxy_info:                                            │    │
│  │    author: Your Name                                     │    │
│  │    description: Install and configure nginx              │    │
│  │    license: MIT                                          │    │
│  │    min_ansible_version: 2.9                              │    │
│  │    platforms:                                            │    │
│  │      - name: Ubuntu                                      │    │
│  │        versions:                                         │    │
│  │          - focal                                         │    │
│  │          - jammy                                         │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Ansible Vault

### 🟡 Intermediate Questions

#### Q5: How do you manage secrets with Ansible Vault?

**Basic Answer:**
Ansible Vault encrypts sensitive data like passwords and keys. Use `ansible-vault create`, `edit`, `encrypt`, `decrypt` commands. Pass vault password via `--ask-vault-pass` or password file.

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ANSIBLE VAULT                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  VAULT COMMANDS:                                                │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  # Create new encrypted file                             │    │
│  │  ansible-vault create secrets.yml                        │    │
│  │                                                          │    │
│  │  # Encrypt existing file                                 │    │
│  │  ansible-vault encrypt vars/secrets.yml                  │    │
│  │                                                          │    │
│  │  # Edit encrypted file                                   │    │
│  │  ansible-vault edit secrets.yml                          │    │
│  │                                                          │    │
│  │  # View encrypted file                                   │    │
│  │  ansible-vault view secrets.yml                          │    │
│  │                                                          │    │
│  │  # Decrypt file                                          │    │
│  │  ansible-vault decrypt secrets.yml                       │    │
│  │                                                          │    │
│  │  # Rekey (change password)                               │    │
│  │  ansible-vault rekey secrets.yml                         │    │
│  │                                                          │    │
│  │  # Encrypt single string                                 │    │
│  │  ansible-vault encrypt_string 'mysecret' --name 'db_pass'│    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  MULTIPLE VAULT IDS:                                            │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  # Encrypt with vault ID                                 │    │
│  │  ansible-vault encrypt --vault-id dev@prompt secrets.yml │    │
│  │  ansible-vault encrypt --vault-id prod@~/.vault_prod \   │    │
│  │    prod_secrets.yml                                      │    │
│  │                                                          │    │
│  │  # Run with multiple vault IDs                           │    │
│  │  ansible-playbook site.yml \                             │    │
│  │    --vault-id dev@prompt \                               │    │
│  │    --vault-id prod@~/.vault_prod                         │    │
│  │                                                          │    │
│  │  # In encrypted file, vault ID is stored:                │    │
│  │  $ANSIBLE_VAULT;1.2;AES256;dev                           │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  INLINE ENCRYPTED VARIABLES:                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  # In vars file:                                         │    │
│  │  db_host: mydb.example.com                               │    │
│  │  db_user: appuser                                        │    │
│  │  db_password: !vault |                                   │    │
│  │    $ANSIBLE_VAULT;1.1;AES256                             │    │
│  │    32613064653338613361393639613838396138...             │    │
│  │                                                          │    │
│  │  # Generate inline encrypted string:                     │    │
│  │  ansible-vault encrypt_string 'P@ssw0rd!' \              │    │
│  │    --name 'db_password' >> vars/main.yml                 │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  BEST PRACTICES:                                                │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  1. Separate vault files:                                │    │
│  │     group_vars/                                          │    │
│  │     └── production/                                      │    │
│  │         ├── vars.yml          # Plain variables          │    │
│  │         └── vault.yml         # Encrypted secrets        │    │
│  │                                                          │    │
│  │  2. Prefix vault variables:                              │    │
│  │     # vault.yml                                          │    │
│  │     vault_db_password: encrypted_value                   │    │
│  │                                                          │    │
│  │     # vars.yml                                           │    │
│  │     db_password: "{{ vault_db_password }}"               │    │
│  │                                                          │    │
│  │  3. Use vault password file in CI/CD:                    │    │
│  │     ansible-playbook site.yml \                          │    │
│  │       --vault-password-file /run/secrets/vault_pass      │    │
│  │                                                          │    │
│  │  4. Never commit vault password files                    │    │
│  │     # .gitignore                                         │    │
│  │     .vault_pass                                          │    │
│  │     *.vault_password                                     │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## AWX/Tower

### 🔴 Advanced Questions

#### Q6: Explain AWX/Ansible Tower architecture and enterprise features.

**Basic Answer:**
AWX (open-source) and Ansible Tower (commercial) provide a web UI, REST API, RBAC, job scheduling, and centralized logging for Ansible automation.

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    AWX/TOWER ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  COMPONENTS:                                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  ┌──────────────────────────────────────────────────┐   │    │
│  │  │               Web Interface                       │   │    │
│  │  │          (React/Angular Frontend)                 │   │    │
│  │  └──────────────────────────────────────────────────┘   │    │
│  │                         │                                │    │
│  │                         ▼                                │    │
│  │  ┌──────────────────────────────────────────────────┐   │    │
│  │  │            REST API (Django)                      │   │    │
│  │  │    • Authentication (LDAP, SAML, OAuth)          │   │    │
│  │  │    • RBAC enforcement                             │   │    │
│  │  │    • Job management                               │   │    │
│  │  └──────────────────────────────────────────────────┘   │    │
│  │                         │                                │    │
│  │         ┌───────────────┼───────────────┐               │    │
│  │         ▼               ▼               ▼               │    │
│  │  ┌───────────┐   ┌───────────┐   ┌───────────┐         │    │
│  │  │  Redis    │   │ PostgreSQL│   │ RabbitMQ  │         │    │
│  │  │  (Cache)  │   │(Database) │   │  (Queue)  │         │    │
│  │  └───────────┘   └───────────┘   └───────────┘         │    │
│  │                         │                                │    │
│  │                         ▼                                │    │
│  │  ┌──────────────────────────────────────────────────┐   │    │
│  │  │            Execution Nodes                        │   │    │
│  │  │   ┌─────────┐ ┌─────────┐ ┌─────────┐           │   │    │
│  │  │   │ Worker 1│ │ Worker 2│ │ Worker N│           │   │    │
│  │  │   │(ansible)│ │(ansible)│ │(ansible)│           │   │    │
│  │  │   └─────────┘ └─────────┘ └─────────┘           │   │    │
│  │  └──────────────────────────────────────────────────┘   │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  KEY CONCEPTS:                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  Organizations:   Top-level tenant isolation             │    │
│  │       │                                                  │    │
│  │       ├── Teams:       Groups of users                   │    │
│  │       │                                                  │    │
│  │       ├── Projects:    Git repos with playbooks          │    │
│  │       │                                                  │    │
│  │       ├── Inventories: Host definitions                  │    │
│  │       │   ├── Static                                     │    │
│  │       │   ├── Dynamic (AWS, Azure, GCP)                  │    │
│  │       │   └── Smart (filtered)                           │    │
│  │       │                                                  │    │
│  │       ├── Credentials:  Stored securely                  │    │
│  │       │   ├── Machine (SSH)                              │    │
│  │       │   ├── Cloud (AWS, Azure)                         │    │
│  │       │   ├── SCM (Git)                                  │    │
│  │       │   └── Vault                                      │    │
│  │       │                                                  │    │
│  │       └── Job Templates: Playbook + Inventory + Creds    │    │
│  │           └── Workflow Templates: Chain job templates    │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ENTERPRISE FEATURES:                                           │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  • RBAC: Role-based access control                       │    │
│  │  • Audit: Complete job history and logs                  │    │
│  │  • Scheduling: Cron-like job scheduling                  │    │
│  │  • Notifications: Slack, email, webhooks                 │    │
│  │  • Surveys: Dynamic input forms for job templates        │    │
│  │  • Workflows: Complex multi-playbook orchestration       │    │
│  │  • HA: Clustered deployment for high availability        │    │
│  │  • API: Full REST API for integration                    │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Troubleshooting

### ⚫ Expert Questions

#### Q7: How do you debug Ansible playbook failures?

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ANSIBLE DEBUGGING                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  VERBOSITY LEVELS:                                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  ansible-playbook site.yml -v      # Basic output        │    │
│  │  ansible-playbook site.yml -vv     # More detail         │    │
│  │  ansible-playbook site.yml -vvv    # Connection debug    │    │
│  │  ansible-playbook site.yml -vvvv   # Full debug + SSH    │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  DEBUG STRATEGIES:                                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  1. Debug module:                                        │    │
│  │  - name: Print variable                                  │    │
│  │    debug:                                                │    │
│  │      var: my_variable                                    │    │
│  │      verbosity: 2                                        │    │
│  │                                                          │    │
│  │  - name: Print message                                   │    │
│  │    debug:                                                │    │
│  │      msg: "Value is {{ my_var }}"                        │    │
│  │                                                          │    │
│  │  2. Register and inspect:                                │    │
│  │  - name: Run command                                     │    │
│  │    command: whoami                                       │    │
│  │    register: result                                      │    │
│  │                                                          │    │
│  │  - debug:                                                │    │
│  │      var: result                                         │    │
│  │                                                          │    │
│  │  3. Assert module:                                       │    │
│  │  - name: Verify condition                                │    │
│  │    assert:                                               │    │
│  │      that:                                               │    │
│  │        - result.rc == 0                                  │    │
│  │        - "'expected' in result.stdout"                   │    │
│  │      fail_msg: "Assertion failed!"                       │    │
│  │                                                          │    │
│  │  4. Interactive debugger:                                │    │
│  │  ANSIBLE_STRATEGY=debug ansible-playbook site.yml        │    │
│  │  # Commands: p task, p result, c (continue), r (redo)    │    │
│  │                                                          │    │
│  │  5. Check mode (dry run):                                │    │
│  │  ansible-playbook site.yml --check --diff                │    │
│  │                                                          │    │
│  │  6. Step mode:                                           │    │
│  │  ansible-playbook site.yml --step                        │    │
│  │  # Prompts before each task                              │    │
│  │                                                          │    │
│  │  7. Start at specific task:                              │    │
│  │  ansible-playbook site.yml --start-at-task="Task Name"   │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  COMMON ISSUES:                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  SSH Connection Failed:                                  │    │
│  │  • Check SSH key permissions (600)                       │    │
│  │  • Verify ansible_user is set correctly                  │    │
│  │  • Test: ssh -vvv user@host                              │    │
│  │                                                          │    │
│  │  Module Not Found:                                       │    │
│  │  • Check collection is installed                         │    │
│  │  • Verify FQCN: ansible.builtin.file                     │    │
│  │  • ansible-galaxy collection list                        │    │
│  │                                                          │    │
│  │  Variable Undefined:                                     │    │
│  │  • Check variable precedence                             │    │
│  │  • Use default filter: {{ var | default('value') }}      │    │
│  │  • Debug: ansible-inventory --list                       │    │
│  │                                                          │    │
│  │  Timeout Issues:                                         │    │
│  │  • Increase timeout in ansible.cfg                       │    │
│  │  • Use async for long-running tasks                      │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📚 Documentation Links

- [Ansible Documentation](https://docs.ansible.com/)
- [Ansible Galaxy](https://galaxy.ansible.com/)
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/tips_tricks/ansible_tips_tricks.html)
- [AWX Project](https://github.com/ansible/awx)

---

**[← Back to Main README](../README.md)** | **[Next: Azure DevOps →](../azure-devops/README.md)**
