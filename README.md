# Hardened VPS Bootstrap

## Overview
Automated hardening framework for Ubuntu/Debian server environments requiring compliance with CIS Benchmarks Level 1, BSI IT-Grundschutz (SYS.1.3), and DSGVO technical safeguards (Art. 32). 

This repository provides idempotent shell scripts and systemd unit configurations for achieving a defensible security baseline on cloud VPS or bare-metal installations within public sector or regulated industry contexts.

**Target Audience:** Infrastructure engineers managing multi-tenant environments, compliance officers requiring audit-ready system configurations, or DevOps teams eliminating manual hardening variance.

---

## Problem Statement
Standard cloud provider images (Ubuntu Server, Debian) ship with configurations optimized for ease-of-use, not security:
- SSH exposed on port 22 with password authentication enabled by default
- No mandatory access controls (AppArmor profiles inactive)
- Kernel parameters tuned for performance, not attack surface reduction
- Audit logging disabled (no forensic capability)

Manual hardening is error-prone, time-intensive, and fails to scale. Auditors demand **reproducible** security baselines with **documented deviations**.

---

## Features & Hardening Scope

### 1. SSH Hardening (`ssh-harden.sh`)
- Forces public-key authentication only (`PasswordAuthentication no`)
- Disables root login via SSH
- Non-standard port configuration (default: 2222, configurable)
- Rate limiting via `sshguard` or CrowdSec integration

### 2. Kernel & Network Stack (`sysctl-harden.sh`)
- IP forwarding disabled (unless explicitly required)
- SYN flood protection (`tcp_syncookies`)
- ICMP redirect rejection
- Source routing disabled
- Martian packet logging

**Mapped to:** CIS Ubuntu 22.04 Benchmark 3.3.x, BSI SYS.1.3.A10

### 3. Automated Patch Management (`auto-updates.sh`)
- Unattended security upgrades via `unattended-upgrades`
- Configurable reboot policies (manual by default for production stability)
- Email notifications on security patches

### 4. Intrusion Detection & Prevention
- **CrowdSec** deployment with firewall bouncer (optional)
- **Fail2Ban** fallback configuration for legacy environments
- Pre-configured jails for SSH, HTTP(S), and custom service logs

### 5. Audit Logging (`auditd-setup.sh`)
- `auditd` rules for monitoring:
  - User account modifications (`/etc/passwd`, `/etc/shadow`)
  - Privilege escalation (`sudo` usage)
  - System call anomalies (optional for high-security zones)
- Log rotation and centralization hooks (syslog/rsyslog)

**Mapped to:** BSI SYS.1.3.A12, CIS 4.1.x (Logging and Auditing)

### 6. Filesystem Hardening
- Automatic mount option enforcement:
  - `/tmp` → `noexec,nosuid,nodev`
  - `/var/tmp` → `noexec,nosuid,nodev`
  - `/var/log` → `nodev,nosuid`
- Immutable flag on critical config files (e.g., `/etc/ssh/sshd_config`)

---

## Architecture & Execution Flow

┌──────────────────────────────────┐
│ Fresh Ubuntu/Debian Install │
│ (Cloud Init / Manual Deploy) │
└────────────┬─────────────────────┘
│
┌──────────────────────────────────┐
│ bootstrap.sh (Orchestrator) │
│ - Validates OS version │
│ - Executes modules in order │
│ - Generates compliance report │
└────────────┬─────────────────────┘
│
┌────────┼────────┬─────────┬─────────┐
┌─────┐ ┌──────┐ ┌──────┐ ┌────────┐ ┌──────┐
│ SSH │ │Sysctl│ │Auditd│ │CrowdSec│ │Fail2B│
│Harden│ │Harden│ │ Setup│ │ Deploy │ │ Setup│
└─────┘ └──────┘ └──────┘ └────────┘ └──────┘
│ │ │ │ │
└────────┴────────┴─────────┴─────────┘
│
┌──────────────────────────────────┐
│ Compliance Report (JSON/CSV) │
│ - Applied controls checklist │
│ - Deviations log │
│ - Next audit date │
└──────────────────────────────────┘

---

## Installation & Usage

### Prerequisites
- **Supported OS:** Ubuntu 22.04 LTS, Ubuntu 24.04 LTS, Debian 12
- **Access:** Root or sudo privileges
- **Network:** Internet access for package installation (or air-gapped mirror configured)

### Quick Start (Interactive Mode)
git clone https://github.com/patrick-bloem/hardened-vps-bootstrap.git
cd hardened-vps-bootstrap
chmod +x bootstrap.sh
sudo ./bootstrap.sh

The script will:
1. Detect OS version and validate compatibility
2. Prompt for non-default SSH port (default: 2222)
3. Execute hardening modules in sequence
4. Generate `/root/hardening-report-$(date).json`

### Non-Interactive Deployment (CI/CD / Ansible)
sudo ./bootstrap.sh --non-interactive
--ssh-port=2222
--enable-crowdsec
--disable-fail2ban
--auto-reboot=false

**Environment Variables Override:**
export SSH_PORT=2222
export CROWDSEC_ENROLL_KEY="your-console-key"
sudo -E ./bootstrap.sh --non-interactive

---

## Compliance Mapping

| Standard | Control ID | Implemented By | Status |
|----------|-----------|----------------|--------|
| CIS Ubuntu 22.04 Benchmark | 5.3.4 (SSH Strong Ciphers) | `ssh-harden.sh` | ✅ Full |
| CIS Ubuntu 22.04 Benchmark | 3.3.1 (IP Forwarding) | `sysctl-harden.sh` | ✅ Full |
| BSI IT-Grundschutz SYS.1.3 | A10 (Network Hardening) | `sysctl-harden.sh` | ✅ Full |
| BSI IT-Grundschutz SYS.1.3 | A12 (Logging) | `auditd-setup.sh` | ✅ Full |
| DSGVO Art. 32 | Technical Safeguards | Combined modules | ⚠️ Partial* |

*DSGVO compliance requires additional organizational measures beyond technical hardening.

---

## Configuration Files

### `config/ssh_config_template`
Hardened SSH daemon configuration enforcing:
- `PermitRootLogin no`
- `PasswordAuthentication no`
- `PubkeyAuthentication yes`
- `Protocol 2`
- `Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com`

### `config/sysctl_hardening.conf`
Kernel parameter overrides for network stack protection:
net.ipv4.conf.all.rp_filter = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.accept_redirects = 0
kernel.dmesg_restrict = 1


### `config/auditd_rules.d/`
Predefined audit rules for common attack vectors:
- `10-privileged-commands.rules`: Tracks `sudo`, `su`, `chsh`
- `20-file-integrity.rules`: Monitors `/etc/passwd`, `/etc/shadow`, `/etc/sudoers`

---

## Testing & Validation

### Automated Testing (Serverspec)
cd tests/
bundle install
rake spec

**Test Coverage:**
- SSH port changed and password auth disabled
- Sysctl parameters correctly applied
- CrowdSec service running and parsing logs
- Auditd generating entries on privileged commands

### Manual Validation Checklist
Verify SSH hardening
sudo sshd -T | grep -E 'port|passwordauthentication|permitrootlogin'

Check kernel parameters
sysctl -a | grep net.ipv4.tcp_syncookies

Confirm auditd status
sudo systemctl status auditd
sudo ausearch -m USER_AUTH -sv no # Failed login attempts

---

## Maintenance & Updates

### Updating Hardening Rules
cd hardened-vps-bootstrap
git pull origin main
sudo ./bootstrap.sh --update-only

The `--update-only` flag skips already-applied configurations and only updates changed modules.

### Rollback Procedure
All modified configuration files are backed up to `/root/hardening-backups/$(date)/`.

sudo ./rollback.sh --restore-ssh
sudo ./rollback.sh --restore-all # Full system rollback

---

## Roadmap
- [ ] Rocky Linux / AlmaLinux support
- [ ] Immutable infrastructure integration (Packer templates)
- [ ] STIG compliance profile (DISA STIGs)
- [ ] Automated CIS-CAT Lite integration

---

## Contributing
This repository follows a strict review process:
1. All PRs must include Serverspec tests
2. Changes to hardening modules require CIS/BSI mapping updates in README
3. No "security by obscurity" practices (e.g., random port changes without documentation)

---

## License
MIT License. Provided for infrastructure hardening in compliance-critical environments.

**Disclaimer:** This framework assists with technical baseline hardening. It does not replace security audits, penetration testing, or organizational security policies.

---

## Author
**Patrick Bloem** | Senior Infrastructure Engineer  
GitHub: [@patrick-bloem](https://github.com/patrick-bloem)
