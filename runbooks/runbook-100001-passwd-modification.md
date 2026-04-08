# Runbook: Rule 100001 — Critical `/etc/passwd` modification

> **Trigger:** Wazuh Alert `100001` — Level 12  
> **Description:** `CRITICAL: /etc/passwd has been modified. Potential persistence or user creation detected.`  
> **ATT&CK mapping:** T1078.003 — Valid Accounts: Local Accounts (Persistence / Privilege Escalation)  
> **Parent SID:** 550 (FIM: integrity checksum changed) filtered by `/etc/passwd` path

---

## Context

Rule `100001` inherits from FIM SID `550` and narrows the scope to specifically `/etc/passwd`. This is a Level 12 critical alert because modification of this file is one of the strongest indicators of persistence (adversary creating a new local account) or privilege escalation (adversary modifying an existing account's UID/GID to gain root access).

Legitimate modifications to `/etc/passwd` are rare outside of planned administrative changes and should always be investigated immediately.

---

## Triage steps

### Step 1 — Identify what changed

Review the `syscheck.diff` field in the Wazuh alert to determine the exact modification. The three most dangerous patterns are:

**New user added:** A new line at the end of the file indicates an account was created.  
**UID/GID changed:** A modification to an existing user's UID (especially to `0`, which grants root) indicates privilege escalation.  
**Shell changed:** A user's login shell changed from `/usr/sbin/nologin` or `/bin/false` to `/bin/bash` indicates a previously disabled account was activated.

```bash
# View the current state of /etc/passwd
cat /etc/passwd

# Check for any UID 0 accounts (should only be root)
awk -F: '$3 == 0 {print $1}' /etc/passwd
```

### Step 2 — Correlate with administrative activity

Determine whether the change was the result of a legitimate `useradd`, `usermod`, or `adduser` command:

```bash
# Check recent sudo activity around the alert timestamp
grep "useradd\|usermod\|adduser\|passwd" /var/log/auth.log | tail -20

# Check who was logged in at the time
last | head -10
```

If the modification correlates with a known, authorized administrative action, document the justification and close the alert.

### Step 3 — Check for shadow file and group tampering

An adversary modifying `/etc/passwd` will often also tamper with related files:

```bash
# Check recent modification times
stat /etc/passwd /etc/shadow /etc/group

# Look for new groups or sudo membership changes
tail -5 /etc/group
grep "sudo" /etc/group
```

### Step 4 — Audit SSH persistence

If a new user was created, check for planted SSH keys:

```bash
# Check for authorized_keys files in all home directories
find /home -name "authorized_keys" -exec echo "--- {} ---" \; -exec cat {} \;

# Check root's authorized_keys
cat /root/.ssh/authorized_keys 2>/dev/null
```

### Step 5 — Containment

**If an unauthorized user was created:**

```bash
# Lock the account immediately
sudo usermod -L <malicious_user>

# Remove SSH keys if present
sudo rm -f /home/<malicious_user>/.ssh/authorized_keys

# Preserve evidence, then delete the account
sudo cp /etc/passwd /tmp/evidence_passwd_$(date +%Y%m%d_%H%M%S)
sudo userdel -r <malicious_user>
```

**If an existing account was escalated (UID changed to 0):**

```bash
sudo usermod -u <original_uid> <username>
```

**If a disabled shell was reactivated:**

```bash
sudo usermod -s /usr/sbin/nologin <username>
```

### Step 6 — Investigate the entry vector

The `/etc/passwd` modification is a persistence indicator — the adversary already had access. Investigate how:

```bash
# Check active sessions and recent logins
w
last -20

# Check for reverse shell processes (may also trigger Rule 100005)
ps -ef --forest | grep -E "bash|sh|nc|python|perl"

# Review sudo log for the escalation chain
grep sudo /var/log/auth.log | tail -30
```

### Step 7 — Post-incident documentation

Record: exact content of the `/etc/passwd` diff, whether a new user was created or an existing one modified, correlated login activity, the entry vector (if determined), and all containment actions taken.

---

## False positive guidance

Legitimate triggers for this rule include system administrators creating service accounts (`useradd`), package installations that create system users (e.g., `www-data` for Apache), and automated provisioning tools managing user accounts.

Even in these cases, the alert should be reviewed and closed with a documented justification — `/etc/passwd` modifications should never be silently ignored.

---

*Part of the [Wazuh SIEM Home Lab](../README.md) project · Rule definition in [Phase 5 § 4](../phases/phase-5-custom-rules.md#4-final-xml-configuration) · ATT&CK mapping in [detections/](../detections/mitre-attack-map.md)*
