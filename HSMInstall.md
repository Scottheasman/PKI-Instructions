# PKI Lab - Production Deployment Runbook
## YubiHSM 2 Integration | Windows Server 2022 | AD CS

---

## Table of Contents
1. [Architecture Overview](#1-architecture-overview)
2. [Standards and Conventions](#2-standards-and-conventions)
3. [Prerequisites](#3-prerequisites)
4. [YubiHSM 2 Key Ceremony](#4-yubihsm-2-key-ceremony)
5. [Offline Root CA Installation](#5-offline-root-ca-installation)
6. [SubCA1 Installation](#6-subca1-installation)
7. [SubCA2 Installation](#7-subca2-installation)
8. [Certificate Template Configuration](#8-certificate-template-configuration)
9. [CA Properties Configuration](#9-ca-properties-configuration)
10. [Disaster Recovery](#10-disaster-recovery)
11. [Validation Checklist](#11-validation-checklist)

---

## 1. Architecture Overview

### PKI Topology
[Offline Root CA]
|
|-- [Lab Issuing CA 1 (SubCA1)] -- YubiHSM 2 (Independent)
|
|-- [Lab Issuing CA 2 (SubCA2)] -- YubiHSM 2 (Independent)

### CA Roles
| Role | Server OS | Domain Joined | HSM | Backup HSM |
| :--- | :--- | :--- | :--- | :--- |
| Offline Root CA | Windows Server 2022 | **No (Standalone)** | YubiHSM 2 (Primary) | YubiHSM 2 (Backup) |
| Lab Issuing CA 1 (SubCA1) | Windows Server 2022 | Yes | YubiHSM 2 | Wrap Key DR |
| Lab Issuing CA 2 (SubCA2) | Windows Server 2022 | Yes | YubiHSM 2 | Wrap Key DR |

### Key Design Decisions
- SubCA1 and SubCA2 are **independent** (different CA keys and certificates).
- Only **HTTP URLs** are embedded in issued certificates (no UNC/DFS paths).
- NDES RA template uses **Legacy CSP** regardless of HSM usage on the CA.
- CRL and Delta CRL are published to both local and UNC paths; only HTTP is embedded in certs.

---

## 2. Standards and Conventions

### Cryptographic Standards
| Component | Algorithm | Key Size | Hash |
| :--- | :--- | :--- | :--- |
| Root CA Signing Key | RSA | **4096** | SHA-256 |
| SubCA1 Signing Key | RSA | **3072** | SHA-256 |
| SubCA2 Signing Key | RSA | **3072** | SHA-256 |
| End-Entity Templates | RSA | 4096 | SHA-256 |

> **Why RSA 3072 for SubCAs?**
> RSA 3072 provides significantly better signing throughput on USB HSMs compared to RSA 4096,
> while remaining well above the 2048-bit security minimum. The Root CA uses 4096 because it
> signs infrequently (SubCA certs + CRLs only).

### HSM Labelling Standard
| HSM | Label | Key Label | Key ID |
| :--- | :--- | :--- | :--- |
| Root CA Primary | ROOT-PRIMARY | IEX-ROOTCA-KEY | 100 |
| Root CA Backup | ROOT-BACKUP | IEX-ROOTCA-KEY | 100 |
| SubCA1 | SUBCA1-PRIMARY | IEX-SUBCA1-KEY | 100 |
| SubCA2 | SUBCA2-PRIMARY | IEX-SUBCA2-KEY | 100 |

### Naming Conventions
| Object | Name |
| :--- | :--- |
| Root CA Common Name | Lab Root CA |
| SubCA1 Common Name | Lab Issuing CA 1 |
| SubCA2 Common Name | Lab Issuing CA 2 |
| Domain | lab.local |
| PKI Web URL | http://pki.lab.local/pkidata |
| OCSP URL | http://ocsp.lab.local/ocsp |
| DFS PKI Path | \\lab.local\share\PKIData |

---

## 3. Prerequisites

### 3.1 All CA Servers
- [ ] Windows Server 2022 installed and fully patched.
- [ ] Static IP address configured.
- [ ] Server name set per naming standard.
- [ ] Time sync verified (critical for certificate validity).
- [ ] YubiHSM 2 Connector Service installed.
- [ ] YubiHSM 2 KSP installed.
- [ ] YubiHSM 2 SDK/Tools installed (for `yubihsm-shell`).

### 3.2 Root CA Specific
- [ ] Server is **NOT** domain joined.
- [ ] Server is **offline** (no network connection except when performing CA operations).
- [ ] Two YubiHSM 2 devices available (Primary + Backup).
- [ ] Physical USB port secured (internal header recommended).

### 3.3 SubCA Servers Specific
- [ ] Server is domain joined.
- [ ] DNS resolves `pki.lab.local` and `ocsp.lab.local`.
- [ ] Service accounts created in AD:
  - `PKIEnrollSvc@lab.local`
  - `PKINDESSvc@lab.local`
- [ ] DFS share `\\lab.local\share\PKIData` is accessible.
- [ ] IIS installed and configured for PKI web publishing.

### 3.4 Verify YubiHSM Provider is Visible to Windows
Run this on **every CA server** before proceeding:

```powershell
certutil -csplist
Confirm the output contains an entry for the YubiHSM Key Storage Provider.
Record the exact provider name string as it appears in the output.
This exact string will be used in all AD CS installation commands.

Important: If the YubiHSM provider does not appear, do not proceed.
Verify the Connector Service is running and the HSM is inserted.

4. YubiHSM 2 Key Ceremony
This section must be completed BEFORE installing the AD CS role on any server.
Treat this as a formal "Key Ceremony." Document all actions, times, and personnel present.

4.1 Root CA Primary HSM Initialization
Insert the ROOT-PRIMARY HSM into the Root CA server.

Step 1: Reset and Initialize