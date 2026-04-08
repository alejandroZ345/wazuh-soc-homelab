# Security Policy

## Scope

This repository contains documentation, detection rules (XML), triage runbooks, and configuration references for a Wazuh SIEM home lab environment. It does not distribute executable software, container images, or compiled binaries.

The primary security risks for this project are:

- **Accidental credential exposure** — Bcrypt hashes, plaintext passwords, API keys, or session tokens inadvertently committed to version control.
- **Sensitive infrastructure details** — Internal IP addresses, hostnames, or network topology information that could facilitate targeting of the lab environment.
- **Detection rule evasion** — While detection logic is intentionally public (to demonstrate detection engineering skills), reports of trivially bypassable rule patterns are welcome.

---

## Supported Versions

This project follows a continuous documentation model rather than versioned releases. Security reports are accepted against the current state of the `main` branch.

---

## Reporting a Vulnerability

If you discover any of the following in this repository, please report it:

- Exposed credentials, password hashes, or API tokens (even if they appear to be lab/test values).
- Sensitive infrastructure details that should have been redacted.
- Configuration patterns that could mislead someone into deploying an insecure setup if followed as-is.

### How to report

**For credential exposure or sensitive data leaks** — Please report privately using [GitHub's private vulnerability reporting feature](https://github.com/alejandroZ345/-Cloud-Native-Security-Operations-Wazuh-SIEM-XDR-Deployment/security/advisories/new) so the information can be scrubbed from version history before public disclosure.

**For general security improvements** — Open a standard [issue](https://github.com/alejandroZ345/-Cloud-Native-Security-Operations-Wazuh-SIEM-XDR-Deployment/issues) describing the concern. No private channel is needed for non-sensitive recommendations.

### Response timeline

I will acknowledge reports within **72 hours** and aim to resolve credential exposure issues within **24 hours** of confirmation, since these require immediate history rewriting.

---

## Security Practices in This Project

The following practices are maintained to prevent accidental exposure:

- The `.gitignore` file excludes common credential and configuration files.
- All documented procedures use placeholder values (e.g., `<MANAGER_IP>`, `"Your_Custom_Password"`) rather than real credentials.
- Password hashes shown in documentation are either truncated or from destroyed lab instances.

---

## Disclaimer

This is an educational home lab project. The detection rules, runbooks, and configurations documented here are designed for controlled lab environments. They should be reviewed and adapted by a qualified security professional before being deployed in any production environment.
