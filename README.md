# IBM i Security Runbook — Powered by Bob

## Description

An interactive security runbook for IBM i systems, designed to be used with **IBM Bob** as an AI assistant. Covers connecting to an IBM i system via SSH, auditing security settings and system values, managing user profiles and group profiles, enforcing Separation of Duty (SoD) through object-level authority, and demonstrating the difference between library and object authority levels.

Also includes the full process for publishing assets to the **Bob CE Marketplace**, including the pre-flight overlap checklist, `README.md` requirements, PR submission steps, and the automated AI review workflow.

## Usage

Open [`ibmi-security-runbook.md`](./ibmi-security-runbook.md) and follow each section interactively with Bob. For each step, copy the **💬 Bob Prompt** block directly into the Bob chat window. Bob will connect to your IBM i system via SSH and execute the commands or SQL queries on your behalf.

Sections:
1. Prerequisites & Connecting to the IBM i System
2. Explore Current Security Settings & Statuses
3. View Current Users, Roles & Library Access
4. Create a User Profile with Separation of Duty
5. Demonstrate the Difference Between Library Access
6. Publishing to the Bob CE Marketplace

## Prerequisites

- IBM i 7.4 or 7.5 (tested; earlier releases may work with minor adjustments)
- A valid IBM i user profile with `*SECADM` or `*ALLOBJ` special authority
- SSH enabled on the target IBM i system (`STRTCPSVR SERVER(*SSHD)`)
- IBM Bob configured with SSH access to the target IBM i system
- IBM i Access Client Solutions (ACS) installed on your workstation (optional, for SQL Script runs)
- An IBM GitHub (`github.ibm.com`) account to contribute to the CE Bob Marketplace

## Pre-flight Overlap Check (Step 2 — Marketplace Publishing)

Before submitting a PR to the CE Bob Marketplace:

- **Asset name:** `ibmi-security-runbook` — verify no existing asset in `GUIDES/` or `LABS/` uses this name in the upstream [`ClientEngineering/bob`](https://github.ibm.com/ClientEngineering/bob) repository
- **Content overlap:** This guide covers IBM i security hardening with Bob. Verify no existing Marketplace asset covers the same IBM i security audit + SoD topic combination
- **Marketplace contract:** Follow the structure defined at [`marketplace-contract`](https://github.ibm.com/ClientEngineering/bob/tree/main/marketplace-contract) — place this asset under `GUIDES/ibmi-security-runbook/`

## Author

IBM Power Demos with Bob — `ibm-power-demos-with-bob`
