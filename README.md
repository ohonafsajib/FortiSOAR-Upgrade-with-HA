# FortiSOAR-Upgrade-with-HA
# FortiSOAR HA Upgrade Runbook  
**Upgrade Path:** 7.4.6 → 7.5.0 (OS Migration) → 7.6.5  
**Deployment Type:** 2-Node HA (Active / Passive)  
**Upgrade Strategy:** Secondary → Primary  

---

# 1. Environment Overview

Example HA Status Before Upgrade:

```bash
csadm ha list-nodes
```

Expected Output:

- Primary: Active
- Secondary: Passive
- Mode: operational

---

# 2. Pre-Upgrade Checklist (Mandatory)

- [ ] Full VM snapshot taken for BOTH nodes
- [ ] Stop all scheduled playbooks
- [ ] Confirm no active workflows running
- [ ] Export built-in connectors
- [ ] Verify repository connectivity
- [ ] Confirm disk space
- [ ] Confirm cluster is healthy

---

# 3. Upgrade Strategy (HA Best Practice)

Upgrade order:

1. Leave cluster on Secondary
2. Upgrade Secondary completely
3. Validate Secondary
4. Upgrade Primary (downtime expected)
5. Validate cluster
6. Confirm replication healthy

---

# PHASE 1 — Upgrade Secondary Node

---

## 3.1 Login to Secondary

```bash
csadm ha list-nodes
```

Confirm node role = `passive secondary`

---

## 3.2 Leave Cluster on Secondary

```bash
csadm ha leave-cluster
```

Verify:

```bash
csadm ha list-nodes
```

Expected:

- Node becomes standalone
- Status: active
- Comment: Left cluster

---

# PHASE 2 — Upgrade Secondary 7.4.6 → 7.5.0

---

## 4.1 Download Upgrade Package

```bash
cd /
wget https://repo.fortisoar.fortinet.com/7.5.0/fortisoar-inplace-upgrade-7.5.0.bin
chmod +x fortisoar-inplace-upgrade-7.5.0.bin
```

---

## 4.2 Execute Upgrade

```bash
sh fortisoar-inplace-upgrade-7.5.0.bin
```

---

## 4.3 Reboot (OS Migration Phase)

During reboot:

- Rocky Linux 8 → Rocky Linux 9
- Duration ~15–20 minutes
- FortiSOAR installs automatically

Monitor:

```bash
tail -f /var/log/leapp/leapp-upgrade.log
```

⚠ Do not run other commands during upgrade.

---

## 4.4 Validate Secondary (7.5.0)

```bash
cat /etc/redhat-release
uname -r
rpm -qi cyops | grep Version
csadm services --status
```

Expected:

- Rocky Linux 9.x
- FortiSOAR 7.5.0
- All services running

---

# PHASE 3 — Upgrade Secondary 7.5.0 → 7.6.5

---

## 5.1 Run Readiness Check

```bash
csadm upgrade check-readiness --version 7.6.5
```

---

## 5.2 Execute Upgrade

```bash
csadm upgrade execute --target-version 7.6.5
```

---

## 5.3 Monitor Logs

Primary:

```bash
tail -f /var/log/cyops/upgrade-fortisoar-7.6.5-<timestamp>.log
```

Elevate framework:

```bash
tail -f /var/log/cyops/fsr-elevate/fsr-elevate-7.6.5-<timestamp>.log
```

---

## 5.4 Post Upgrade Reboot (If Prompted)

Reboot if requested to apply:

- Kernel updates
- glibc
- systemd
- microcode

---

## 5.5 Validate Secondary (Final)

```bash
cat /etc/redhat-release
rpm -qi cyops | grep Version
csadm services --status
```

Expected:

- Rocky Linux 9.7
- FortiSOAR 7.6.5
- All services running

---

# PHASE 4 — Upgrade Primary Node

⚠ Downtime Expected

---

## 6.1 Perform Takeover (Optional Best Practice)

If needed:

```bash
csadm ha suspend-cluster
```

Or perform manual takeover from upgraded node.

---

## 6.2 Repeat Same Upgrade Steps on Primary

On Primary:

1. Download 7.5.0 installer  
2. Execute installer  
3. Reboot & monitor leapp logs  
4. Validate 7.5.0  
5. Run:

```bash
csadm upgrade check-readiness --version 7.6.5
csadm upgrade execute --target-version 7.6.5
```

6. Reboot if prompted  

---

# PHASE 5 — Rejoin Cluster

After both nodes are on 7.6.5:

On Secondary (or upgraded node):

```bash
csadm ha join-cluster <primary-ip>
```

Verify:

```bash
csadm ha list-nodes
```

Expected:

- Primary: active
- Secondary: passive
- Mode: operational
- Version: 7.6.5

---

# PHASE 6 — Final HA Validation

---

## 7.1 Verify Services

```bash
csadm services --status
```

All services must be Running.

---

## 7.2 Verify Replication

```bash
csadm ha get-replication-stat
```

Replication lag must be zero.

---

## 7.3 Validate GUI

- GUI accessible
- Playbooks working
- No HTTP 500
- Workflows execute successfully

---

# Rollback Plan (HA)

If failure occurs:

1. Power off affected node
2. Revert VM snapshot
3. Boot node
4. Rejoin cluster

---

# Activity Tracking Table

| Step | Node | Date | Engineer | Status | Notes |
|------|------|------|----------|--------|-------|
| Snapshot Taken | Primary | | | | |
| Snapshot Taken | Secondary | | | | |
| Secondary Leave Cluster | Secondary | | | | |
| Secondary 7.5.0 Upgrade | Secondary | | | | |
| Secondary 7.6.5 Upgrade | Secondary | | | | |
| Primary 7.5.0 Upgrade | Primary | | | | |
| Primary 7.6.5 Upgrade | Primary | | | | |
| Cluster Rejoin | Both | | | | |
| Final Validation | Both | | | | |

---

# Log Reference Summary

| Phase | Log Location |
|-------|--------------|
| OS Migration | /var/log/leapp/leapp-upgrade.log |
| 7.6.5 Upgrade | /var/log/cyops/upgrade-fortisoar-7.6.5-<timestamp>.log |
| Elevate Framework | /var/log/cyops/fsr-elevate/fsr-elevate-7.6.5-<timestamp>.log |
| RPM Logs | /var/log/cyops/install |

---

# Upgrade Completed

Final Version Example:

```
Rocky Linux Version: 9.7
FortiSOAR Version: 7.6.5-5662
Mode: operational
```

Confirm HA operational state before closing change.
