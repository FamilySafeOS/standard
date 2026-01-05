# X-PP Vendor Implementation Guides
**Version:** 1.0  
**Target Audience:** OS Kernel Engineers, System Architects, and Driver Developers.

This directory contains the technical blueprints for implementing X-PP (Cross-Platform Policy Protocol) and achieving **FamilySafe OS Certification**.

---

## 1. Windows Implementation Guide (NT Kernel)
To achieve Level 3 compliance, Windows must utilize the **Windows Filtering Platform (WFP)** and **Kernel Callback Objects**.

### 1.1 Network Enforcement (WFP Callout)
Implement a kernel-mode callout driver that attaches to the `FWPM_LAYER_ALE_AUTH_CONNECT_V4` and `V6` layers.
- **Hook:** Intercept `ALE (Application Layer Enforcement)` connect requests.
- **Logic:** Validate the calling process's `AppID` and `UserSID` against the X-PP local cache.
- **Action:** If the policy state is `CHILD_SAFE_MODE` and the port/app is restricted, return `FWP_ACTION_BLOCK`.

### 1.2 Process Watchdog
Use `PsSetCreateProcessNotifyRoutineEx` to intercept process creation.
- Before a process is allowed to run, the X-PP Kernel driver must hash the executable and verify it against the `AppIdentity` manifest.
- Implement a **Job Object** for any process launched under a restricted profile to enforce hard memory and CPU limits defined in X-PP.

---

## 2. Linux & SteamOS Implementation Guide
Linux-based systems should leverage the **Linux Security Module (LSM)** framework combined with **eBPF**.

### 2.1 BPF-LSM Implementation
Use `BPF_PROG_TYPE_LSM` to hook into security critical paths without modifying kernel source.
- **Hook Points:** `socket_connect`, `bprm_check_security` (for execution), and `inode_permission`.
- **State Management:** Store the X-PP active manifest in a pinned **eBPF Map**. This allows the userspace agent to update policies (e.g., toggling ChildSafe Mode) by simply updating the map values, which the kernel reads with zero-latency.

### 2.2 Systemd Integration (X-PP Daemon)
The `xppd` daemon should manage a dedicated `family-safe.slice` using **cgroups v2**.
- When `CHILD_SAFE_MODE` is signaled, `xppd` moves all non-essential user processes into this slice to apply immediate resource throttling or suspension (`SIGSTOP`).

---

## 3. Tizen & webOS (Smart TV) Implementation Guide
For TV platforms, enforcement focuses on **Application Lifecycle** and **Hardware I/O Control**.

### 3.1 App Lifecycle Management
- **Tizen:** Use the `ApplicationManager` and `AppControl` APIs to intercept `launch()` requests.
- **webOS:** Implement a system-level service that monitors `com.webos.applicationManager`.
- **Action:** If `CHILD_SAFE_MODE` is active, the OS must intercept attempts to launch "Adult" or "Uncategorized" App IDs, redirecting the user to a "Restricted Access" UI.

### 3.2 Hardware Signaling (HDMI & Input)
- **HDMI Lock:** Use the `TVWindow` (Tizen) or `ExternalInput` (webOS) APIs to lock the active source. 
- **Remote Signal:** The OS must listen for the X-PP `SIG_MODE_CHANGE`. Upon receipt, it must programmatically switch the TV's **System Profile** to the "Kids/Supervised" account, which persists across reboots.

---

## 4. Universal Watchdog (The "Heartbeat" Protocol)
All certified OSes must implement a **Hardware-Software Heartbeat**:
1. The X-PP Userspace Agent must "kick" the hardware watchdog (e.g., `/dev/watchdog` on Linux or `WDIOC_KEEPALIVE`) every 10 seconds.
2. If the Agent is killed or tampered with, the Kernel must not only reboot (hardware default) but must set a **Boot Flag** in the TPM/Secure Enclave.
3. Upon next boot, the OS sees the flag and enters **Lockdown Mode** until the X-PP Controller re-authorizes the node.
