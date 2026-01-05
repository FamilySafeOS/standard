# RFC: Cross-Platform Policy Protocol (X-PP) v2.4.0

**Authors:** David Emanuel Oprea (Lead), Dr. Gemini (Co-Author)  
**Date:** January 5, 2026  
**Status:** Standards Track / FamilySafe OS Core Specification  

## 1. Abstract
The **Cross-Platform Policy Protocol (X-PP)** is a standardized orchestration framework for digital governance. It addresses the fragmentation of household security by providing a unified "Household Control Plane" to manage heterogeneous endpoints, including Desktop PCs, Mobile Devices, Gaming Consoles, Handhelds, and Smart TVs. X-PP is the technical foundation for the **FamilySafe OS** certification.

## 2. System Architecture
X-PP operates through a 3-tier hybrid trust-chain, ensuring that policies are resilient even in offline scenarios.

### 2.1 The Three Tiers
1.  **The Controller:** The "Source of Truth" (Local NAS, Router, or Cloud) managing the encrypted Household Manifest.
2.  **The Satellite:** A verified administrator device (Mobile App) used for **QR Trust-Pairing** and secure remote signaling.
3.  **The Node:** Any FamilySafe-certified endpoint (PC, Phone, TV, Console) that natively enforces the policy via its kernel or system services.

## 3. Remote Mode Signaling (RMS) & ChildSafe Mode
A certified FamilySafe OS must implement a native, atomic listener for state transitions.

### 3.1 ChildSafe Mode (CSM) State Machine
When a `CHILD_SAFE_MODE` signal is received, the Node must perform the following atomic operations:
* **Process Quiescence:** Identify and suspend non-compliant active processes.
* **Network Reconfiguration:** Enforce strict DNS-over-HTTPS (DoH) to a supervised endpoint and apply L4 port-filtering.
* **UI/Shell Transition:** Trigger a native OS-level UI switch to a restricted environment (e.g., Kids Profile).
* **Input/Output Lockdown:** Restrict hardware-level access to unmonitored peripherals (USB Storage, HDMI-in).

## 4. Platform-Specific Enforcement Mappings

### 4.1 Windows (NT Kernel)
Windows implementation relies on the **Windows Filtering Platform (WFP)**.
* **Networking:** Kernel-mode callout drivers intercept `ALE (Application Layer Enforcement)` connect requests at the transport layer.
* **Processes:** Use `PsSetCreateProcessNotifyRoutineEx` to verify application hashes against the X-PP manifest before execution.

### 4.2 Linux / SteamOS
Linux systems leverage the **Linux Security Module (LSM)** and **eBPF**.
* **Hooks:** Use `BPF_PROG_TYPE_LSM` to mediate access to kernel objects (inodes, tasks, files).
* **State:** Policies are stored in pinned **eBPF Maps**, allowing the `xppd` daemon to update kernel-level enforcement without context-switch overhead.

### 4.3 Smart TVs (Tizen / webOS)
Enforcement focuses on the application lifecycle and hardware I/O.
* **App Control:** TVs must intercept launch requests via `ApplicationManager` (Tizen) or `com.webos.applicationManager` (webOS).
* **Hardware Lock:** Remote signals trigger a lockdown of HDMI external inputs and system-wide volume/brightness caps.

## 5. Security & Tamper Protection
### 5.1 The Kernel Watchdog
To prevent bypass, the X-PP Node implements a **State Consistency Watchdog**. 
* **Heartbeat:** If the userspace agent fails to report to the Kernel, the system enters a **"Default Deny"** state (blocking all non-emergency traffic).
* **Immutability:** Policy manifests must be stored in a partition protected by the **Hardware Root of Trust** (TPM/Secure Enclave).

## 6. Universal Schema (JSON-LD)
```json
{
  "@context": "[https://familysafeos.org/v2](https://familysafeos.org/v2)",
  "@type": "PolicyManifest",
  "subject": {
    "id": "household_user_alpha",
    "mode": "CHILD_SAFE_MODE"
  },
  "enforcement": {
    "timeQuota": { "value": 7200, "unit": "seconds", "shared": true },
    "contentFilter": "urn:xpp:filter:strict",
    "signals": { "lockdownOnIdle": true }
  },
  "emergency": {
    "breakGlassEnabled": true,
    "auditRequired": true
  },
  "signature": {
    "type": "Ed25519Signature2020",
    "proofValue": "..."
  }
}
```
## 7. Compliance Levels (FS-CL)
* **FS-CL1 (Basic):** Network-level enforcement via X-PP Gateway/Proxy (Legacy devices).
* **FS-CL2 (Standard):** Native system-service support + Remote Mode Signaling.
* **FS-CL3 (Full):** Kernel-integrated enforcement + Hardware Root of Trust + Watchdog.

---
&copy; 2026 David Emanuel Oprea
