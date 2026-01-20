---
Time Synchronization Over SSH
---
> *Control the clock. Control the system.*

This document demonstrates how to **synchronize system time from a local machine to a remote host using SSH**, without NTP, without interactive shells — just raw command execution.

This approach is useful for:

* Embedded systems (e.g., **i.MX8**)
* Air-gapped environments
* Systems without NTP access
* One-shot forensic or controlled time alignment
---

## Concept

Instead of running a time daemon, we:

1. Read the **local system time**
2. Inject it into the **remote system** via SSH
3. (Optionally) Write the time to the **hardware clock (RTC)**

SSH allows **non-interactive command execution**, making this fast, scriptable, and deadly precise.

---

## Requirements

* SSH access to the remote host
* Root or sudo privileges on the remote system
* `date` and `hwclock` available on the remote host
* SSH key authentication recommended (no passwords)

---

## Linux → Linux (Generic)

Run **from your local Linux host**:

```bash
ssh <user>@<IP> "date -s \"$(date '+%Y-%m-%d %H:%M:%S')\""
ssh <user>@<IP> "hwclock -w"
```

### What’s happening

* `date '+%Y-%m-%d %H:%M:%S'`
  → Extracts local system time
* `date -s`
  → Sets remote system time
* `hwclock -w`
  → Writes system time to RTC

---

## Linux → i.MX8 (Embedded Target)

Most i.MX8 boards run as `root`, so the command is even cleaner:

```bash
ssh root@IMX_IP "date -s \"$(date '+%Y-%m-%d %H:%M:%S')\""
ssh root@IMX_IP "hwclock -w"
```

### Notes for i.MX8

* RTC may be external or battery-backed
* If `hwclock` fails, verify RTC driver:

  ```bash
  ls /dev/rtc*
  ```

---

## Windows → i.MX8 (PowerShell)

Run **from PowerShell** on Windows:

```powershell
ssh root@IMX_IP "date -s '$(Get-Date -Format "yyyy-MM-dd HH:mm:ss")'"
ssh root@IMX_IP "hwclock -w"
```

### Why this works

* `Get-Date` replaces Linux `date`
* PowerShell string expansion happens **locally**
* SSH transmits the resolved timestamp

---

## One-Liner (Linux)

If you want it **tight and silent**:

```bash
ssh root@IMX_IP "date -s \"$(date '+%F %T')\" && hwclock -w"
```

---

## Common Pitfalls

### Timezone mismatch

This method copies **local wall-clock time**, not timezone info.

Verify on remote:

```bash
timedatectl
```

Set timezone if needed:

```bash
timedatectl set-timezone UTC
```

---

### NTP overriding your time

Disable NTP on the remote system if necessary:

```bash
timedatectl set-ntp false
```

---

## Security Notes

* Use **SSH keys**, not passwords
* Consider restricting SSH command execution
* Time manipulation affects:

  * Logs
  * TLS certificates
  * Forensics
  * Distributed systems

Use responsibly.

---

## TL;DR

You don’t need NTP.

You don’t need an interactive shell.

You just need SSH.

```text
Local time → SSH → Remote date → RTC
```

> *When you own the clock, you own the timeline.*


---
