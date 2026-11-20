# Cybersecurity Threat Model & Risk Assessment: Multi-Branch Retail POS Cloud Sync System

## 1. Executive Summary
This repository presents a **STRIDE Threat Model, Risk Assessment, and ITIL-Aligned Incident Response Playbook** for a distributed retail Point-of-Sale (POS) cloud synchronization architecture.

The modeled system supports real-time sales transaction processing, inventory updates, and user authentication across **25+ retail branches** and a centralized cloud backend, while maintaining offline transaction queuing during branch connectivity drops. The goal of this assessment is to map trust boundaries, identify high-impact threats, assign risk ratings, and define practical preventative controls alongside 2nd-line operational triage and escalation steps.

> **Scope & Assessment Note:** This is a structured security architecture and operational risk assessment modeled on realistic multi-branch retail POS workflows. Third-party payment gateway internals and physical facility security outside the POS/network boundary are treated as out-of-scope dependencies.

## 2. System Architecture Overview
The architecture spans three primary zones:
* **Branch Store Network (Edge / Lower-Trust Zone):** Includes POS terminals, peripheral devices, a local store edge sync service, and an encrypted offline transaction queue for intermittent connectivity.
* **Cloud Backend (Application & Data Zone):** Hosted on **Microsoft Azure** (or **AWS**), comprising an API Gateway, Identity Provider (IdP), Sales & Inventory Sync Service, and an encrypted relational database.
* **External Third-Party Zone:** An external **PCI-DSS-compliant Payment Gateway** handling payment authorization and tokenization so raw cardholder data is minimized within the retail environment.

## 3. Data Flow Diagram (DFD) & Trust Boundaries
Data flows between store terminals, local edge queues, cloud services, and external payment endpoints across defined **Trust Boundaries** (where trust levels, network segments, or administrative domains change).

```mermaid
flowchart LR
    %% External Entities
    Cashier([Cashier / Store Manager])
    PayGW([3rd-Party Payment Gateway])

    %% Trust Boundary 1: Branch Store LAN
    subgraph TB1 [Trust Boundary 1: Branch Store Network]
        POS[POS Terminal App]
        LocalDB[(Local Offline SQLite Cache)]
        POS <-->|Encrypted Local Sync| LocalDB
    end

    %% Trust Boundary 2: Cloud VPC
    subgraph TB2 [Trust Boundary 2: Cloud Infrastructure VPC]
        APIGW[Cloud API Gateway]
        IdP[Identity Provider / MFA]
        SyncSvc[POS Sync & Inventory Service]
        CloudDB[(Central Sales & Inventory DB)]

        APIGW -->|Validate JWT| IdP
        APIGW -->|HTTPS / TLS 1.3| SyncSvc
        SyncSvc <-->|Read/Write SQL| CloudDB
    end

    %% Cross-Boundary Flows
    Cashier -->|PIN / Badge Login| POS
    POS -->|Auth & Sales Sync - TLS 1.3| APIGW
    POS -->|Card Token Request - TLS 1.3| PayGW
    PayGW -->|Payment Approval Token| POS
```

## 4. Repository Structure
* **[01_Asset_and_Trust_Boundary_Inventory.md](./01_Asset_and_Trust_Boundary_Inventory.md)** — Critical data assets, role/privilege matrix, and network trust boundary definitions.
* **[02_STRIDE_Threat_Matrix.md](./02_STRIDE_Threat_Matrix.md)** — 12 structured threat scenarios evaluated across STRIDE categories using a Likelihood × Impact scoring matrix.
* **[03_Incident_Response_and_ITIL_Playbook.md](./03_Incident_Response_and_ITIL_Playbook.md)** — Preventative security controls mapped to 2nd-line detection, containment, escalation, and post-incident review workflows.
* **`diagrams/`** — Source diagram files (`.drawio` / `.json`) and exported architecture/DFD images (`.png`).

## 5. Frameworks & Standards Referenced
* **STRIDE** (Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege)
* **NIST Cybersecurity Framework (CSF 2.0)** (Identify, Protect, Detect, Respond, Recover)
* **ITIL v4 Incident Management** (Triage, Escalation, Containment, Root Cause Analysis)
* **PCI-DSS v4.0 Concepts** (Scope reduction through tokenization and encrypted transport)
