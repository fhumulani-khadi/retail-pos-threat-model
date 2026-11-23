# 01. Asset & Trust Boundary Inventory

## 1. Critical System Assets
All high-value data stores, cryptographic credentials, cloud services, and edge hardware across the Multi-Branch Retail POS Cloud Sync System are cataloged below and classified by sensitivity and operational impact.

| Asset ID | Asset Name | Type | Description | Data Classification | Impact if Compromised |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **AST-01** | Payment Authorization Tokens & References | Data | Non-reversible payment tokens and transaction references returned by the PCI DSS-validated Payment Gateway (no raw PAN or SAD stored). | Restricted (PCI DSS-relevant) | Unauthorized void/refund replay, audit failure, loss of payment settlement integrity. |
| **AST-02** | Device mTLS Certificates & Entra ID JWTs | Credentials | Hardware-bound client certificates and short-lived OAuth2/JWT tokens authenticating branch sync agents to Azure API Management. | Confidential | Rogue terminal spoofing (TB2), fraudulent batch sales injection, cross-branch data access. |
| **AST-03** | Encrypted Offline Transaction Queue | Data Store | Local SQLite/encrypted store managed by the Store Edge Sync Service holding queued sales and per-record HMACs during WAN drops. | Confidential | Offline transaction tampering, replay attacks upon reconnect, lost branch revenue. |
| **AST-04** | Azure SQL Database (Sales & Inventory) | Data Store | Central relational database storing branch sales headers, SKU pricing, stock levels, and immutable audit logs. | Confidential | Price manipulation, inventory distortion, financial reconciliation discrepancies. |
| **AST-05** | Store Edge Sync & POS Terminal Software | Application | Local POS application, peripheral drivers, and background edge sync agent running on branch registers. | Internal | Unauthorized OS shell escape, malware injection through USB peripherals, POS downtime. |
| **AST-06** | Azure API Management & App Service | Cloud Service | Cloud ingress gateway (`APIM`) enforcing rate limits/mTLS and the `App Service` sync engine processing branch payloads. | Internal / Confidential | API denial of service during peak trading, broken object-level authorization (BOLA). |
| **AST-07** | Privileged Admin Identities (Entra ID / PIM) | Credentials | Cloud and security administrator accounts with rights over Azure infrastructure, APIM policy, Conditional Access, and Azure SQL. | Restricted | Full environment takeover, audit log tampering, mass terminal lock/wipe abuse. |

---

## 2. User Roles & Privilege Matrix (RBAC)
Access across branch registers and Microsoft Azure services is governed by **Microsoft Entra ID** using Role-Based Access Control (RBAC) and the **Principle of Least Privilege**.

| Role | Authentication Method | Branch POS Permissions | Azure Portal / API Permissions |
| :--- | :--- | :--- | :--- |
| **Cashier** | RFID Badge + 6-digit POS PIN | Process sales, scan items, accept tokenized card/cash payments, apply standard discounts (`<10%`). | **None** (Machine-level mTLS certificate + Entra ID service principal handles background sync). |
| **Store Manager** | Entra ID Username + Password + MFA | Approve refunds, voids, manual price overrides, and end-of-day (EOD) cash-up reconciliation. | Read-only access to single-branch sales and EOD summary dashboard. |
| **2nd-Line Support Analyst** | Entra ID SSO + FIDO2 Hardware MFA | Audited remote diagnostic access (read-only logs, restart Store Edge Sync Service); cannot alter sales records. | View APIM/App Service health telemetry, inspect failed batch sync queues, trigger branch resyncs. |
| **Cloud / Security Admin** | Entra ID SSO + FIDO2 MFA + PIM (JIT) | Emergency terminal lock/wipe through MDM; no routine store cashier privileges. | Full Azure infrastructure, APIM policy, Entra ID Conditional Access, and Azure SQL maintenance (logged). |

---

## 3. Trust Boundaries & Security Assumptions

Every boundary below represents a transition in network trust, physical control, or administrative domain where strict validation controls are enforced.

| ID | Trust Boundary | Crossing Components | Security Controls Enforced at Boundary |
| :--- | :--- | :--- | :--- |
| **TB1** | **Terminal / Peripherals to Store Edge Sync Service** | POS Terminal App, barcode scanners/printers ↔ Store Edge Sync Service & Encrypted Offline Queue | Local IPC restricted to signed binaries; OS Kiosk Mode; USB mass-storage blocked; AES-256 queue encryption (key held in TPM or OS key store) + per-record HMAC. |
| **TB2** | **Store Network to Azure (Internet, through API Management)** | Store Edge Sync Service ↔ Azure API Management (`APIM`) & Microsoft Entra ID | Mutual TLS (`mTLS` over `TLS 1.3`); Entra ID Conditional Access (compliant device check); APIM rate limiting and per-branch quotas. |
| **TB3** | **API Management to App Service & Azure SQL** | Azure API Management ↔ Azure App Service (Sync Engine) ↔ Azure SQL Database | App Service VNet integration; Azure SQL reachable only through a Private Endpoint (public access disabled); strict `branch_id` JWT claim enforcement (BOLA prevention); parameterized SQL queries. |
| **TB4** | **Card Reader to External Payment Gateway** | P2PE Card Reader ↔ PCI DSS-Validated Payment Gateway | Hardware-level Point-to-Point Encryption (`P2PE`): card data is encrypted at the reader, and the POS terminal and Azure hold no decryption keys; outbound TLS 1.3 to the gateway; only non-sensitive tokens returned. |

### Alignment with Key Architectural Assumptions
* **Assumption A1 (No Cleartext Card Data):** Enforced at **TB4** through hardware P2PE card readers. Threats `T11` and `T12` in [02_STRIDE_Threat_Matrix.md](./02_STRIDE_Threat_Matrix.md) evaluate residual risk if terminal compromise or reader tampering attempts to bypass this boundary.
* **Assumption A2 (Validated External Providers):** The external Payment Gateway and P2PE key-injection facility maintain their own independent PCI DSS validation.
* **Assumption A3 (Zero-Trust Branch Networks):** All traffic crossing **TB2** from store LANs is treated as untrusted and must pass device certificate validation, JWT verification, and payload schema checks before reaching **TB3**.
