# IBM i Security Runbook — Powered by Bob

> **Connecting, Exploring, and Hardening Security on IBM i using IBM Bob as your AI assistant**

This runbook is designed to be followed interactively with **Bob** (IBM's AI coding and operations assistant). Each section includes the exact prompt to give Bob, what Bob will do, and the commands or queries that result. Bob connects to your IBM i system via SSH and executes all checking, user creation, and authority management tasks on your behalf.

---

## Table of Contents

1. [Prerequisites & Connecting to the IBM i System](#1-prerequisites--connecting-to-the-ibm-i-system)
2. [Explore Current Security Settings & Statuses](#2-explore-current-security-settings--statuses)
3. [View Current Users, Roles & Library Access](#3-view-current-users-roles--library-access)
4. [Create a User Profile with Separation of Duty](#4-create-a-user-profile-with-separation-of-duty)
5. [Demonstrate the Difference Between Library Access](#5-demonstrate-the-difference-between-library-access)
- [Appendix A — Useful Commands Quick Reference](#appendix-a--useful-commands-quick-reference)
- [Appendix B — SQL Catalog Views for Security Reporting](#appendix-b--sql-catalog-views-for-security-reporting)

---

## How to Use This Runbook with Bob

Each task in this runbook follows a consistent pattern:

| Symbol | Meaning |
|--------|---------|
| 💬 **Bob Prompt** | Copy this prompt exactly into the Bob chat window |
| ⚙️ **What Bob Does** | What Bob will execute on the IBM i system |
| 📋 **Command / SQL** | The underlying CL command or SQL Bob runs |
| ✅ **Expected Result** | What you should see when the step succeeds |
| ⚠️ **Security Note** | Important security considerations |

Bob uses SSH to connect to your IBM i system and runs commands via the PASE shell or through the IBM i CL/SQL interfaces. Make sure Bob's MCP or SSH tool configuration points to your target system before starting.

---

## 1. Prerequisites & Connecting to the IBM i System

### 1.1 Prerequisites Checklist

Before starting, confirm you have:

- [ ] A valid IBM i user profile with `*SECADM` or `*ALLOBJ` special authority
- [ ] The hostname or IP address of the target system (e.g. `192.168.1.100` or `as400.company.local`)
- [ ] SSH enabled on IBM i — verify with `STRTCPSVR SERVER(*SSHD)` from a 5250 session
- [ ] IBM i Access Client Solutions (ACS) installed on your workstation (for SQL Script runs)
- [ ] Network access to port `22` (SSH), `449` (ACS), or `23` (5250/Telnet — avoid in production)
- [ ] Bob configured with SSH access to the target IBM i system

> **Security note:** Telnet (port 23) transmits credentials in plaintext. Use SSH or TLS-encrypted ACS connections in any non-isolated environment. Never bind SSH to `0.0.0.0`; bind to the specific system IP.

---

### 1.2 Connect to IBM i via SSH

#### 💬 Bob Prompt

```
Connect to the IBM i system at [YOUR_IP_OR_HOSTNAME] using SSH with my user [YOUR_PROFILE].
Confirm the connection is active and show me the current system name and OS version.
```

#### ⚙️ What Bob Does

Bob establishes an SSH session to your IBM i system and runs the system identity commands.

#### 📋 Commands

```bash
# Bob connects via SSH
ssh YOURID@192.168.1.100

# Confirm connection and show system identity
system "DSPSYSVAL SYSVAL(QSYSNAME)"
system "DSPSYSVAL SYSVAL(QMODEL)"

# Show OS version
DSPPTF  # or
system "DSPJOBLOG"
```

To start an interactive PASE session after connecting:

```bash
bash
```

To access a 5250 green-screen session from within SSH:

```bash
call qp2term
```

#### ✅ Expected Result

Bob confirms the SSH session is open and returns the system name (e.g. `DEVIBMI`) and OS release (e.g. `IBM i 7.5`).

---

### 1.3 Verify SSH Service is Running

#### 💬 Bob Prompt

```
Check whether the SSH daemon (SSHD) is active on the IBM i system.
If it is not running, show me the command to start it.
```

#### ⚙️ What Bob Does

Bob queries the active server list and reports SSH status.

#### 📋 Commands

```sql
-- Check active TCP/IP servers via SQL
SELECT SERVER_NAME, CURRENT_JOBS, MAXIMUM_JOBS
FROM QSYS2.SERVER_JOB_INFO
WHERE SERVER_NAME = 'SSHD';
```

If SSH is not active:

```cl
/* Start the SSH daemon — requires *IOSYSCFG authority */
STRTCPSVR SERVER(*SSHD)
```

---

## 2. Explore Current Security Settings & Statuses

### 2.1 Check the System Security Level

#### 💬 Bob Prompt

```
Check the current IBM i security level (QSECURITY system value) on the connected system
and tell me whether it meets IBM's recommended minimum for a production system.
```

#### ⚙️ What Bob Does

Bob reads `QSECURITY` via SQL and interprets the result against IBM's recommendations.

#### 📋 Command

```cl
DSPSYSVAL SYSVAL(QSECURITY)
```

Or via SQL:

```sql
SELECT SYSTEM_VALUE_NAME, CURRENT_NUMERIC_VALUE
FROM QSYS2.SYSVAL_INFO
WHERE SYSTEM_VALUE_NAME = 'QSECURITY';
```

#### ✅ Expected Result

| Level | Meaning | Production Safe? |
|-------|---------|-----------------|
| 10 | No security | ❌ Never use |
| 20 | Password security only | ❌ |
| 30 | Resource security | ⚠️ Minimum only |
| 40 | OS integrity protection | ✅ Recommended minimum |
| 50 | Enhanced integrity | ✅ Recommended for production |

---

### 2.2 Audit All Critical Security System Values

#### 💬 Bob Prompt

```
Run a full security audit of the IBM i system values on the connected system.
Show me all password policy settings, lockout settings, and auditing configuration.
Flag any values that do not meet IBM security best practices.
```

#### ⚙️ What Bob Does

Bob queries all security-related system values from `QSYS2.SYSVAL_INFO` and compares them to recommended baselines.

#### 📋 SQL Query

```sql
SELECT
    SYSTEM_VALUE_NAME,
    CURRENT_CHARACTER_VALUE,
    CURRENT_NUMERIC_VALUE
FROM QSYS2.SYSVAL_INFO
WHERE SYSTEM_VALUE_NAME IN (
    'QSECURITY',
    'QPWDMINLEN',
    'QPWDMAXLEN',
    'QPWDEXPITV',
    'QPWDLVL',
    'QMAXSIGN',
    'QMAXSGNACN',
    'QLMTSECOFR',
    'QRETSVRSEC',
    'QAUDLVL',
    'QAUDCTL',
    'QPWDRQDDIF',
    'QPWDRQDDGT',
    'QPWDRQDMXRPT',
    'QPWDVLDPGM'
)
ORDER BY SYSTEM_VALUE_NAME;
```

#### ✅ Recommended Values Reference

| System Value | Recommended | Description |
|---|---|---|
| `QSECURITY` | `40` or `50` | Security level |
| `QPWDMINLEN` | `8` or more | Minimum password length |
| `QPWDEXPITV` | `90` or less | Password expiration (days) |
| `QMAXSIGN` | `3` | Max failed sign-on attempts |
| `QMAXSGNACN` | `*DISABLE` | Disable profile after lockout |
| `QLMTSECOFR` | `1` | Restrict SECOFR to console only |
| `QAUDCTL` | `*AUDLVL *OBJAUD` | Enable auditing |
| `QAUDLVL` | `*AUTFAIL *SECURITY *CREATE *DELETE *PGMFAIL` | Audit events |
| `QRETSVRSEC` | `0` | Do not retain server security data |
| `QPWDLVL` | `2` or `3` | Support longer passwords |

---

### 2.3 Check the Security Audit Journal

#### 💬 Bob Prompt

```
Check whether the IBM i Security Audit Journal (QAUDJRN) exists and is active on the connected system.
If it does not exist, show me the commands to create and enable it.
```

#### ⚙️ What Bob Does

Bob checks for the existence of `QAUDJRN` in `QSYS` and reads the `QAUDCTL` and `QAUDLVL` system values.

#### 📋 Commands

```cl
/* Check if the audit journal object exists */
DSPOBJD OBJ(QSYS/QAUDJRN) OBJTYPE(*JRN)

/* Display current auditing settings */
DSPSYSVAL SYSVAL(QAUDCTL)
DSPSYSVAL SYSVAL(QAUDLVL)
```

If the journal does not exist, Bob will run:

```cl
/* Create the journal receiver and journal */
CRTJRNRCV JRNRCV(QSYS/QAUDJRNCV) THRESHOLD(100000)
CRTJRN JRN(QSYS/QAUDJRN) JRNRCV(QSYS/QAUDJRNCV) MNGRCV(*SYSTEM) DLTRCV(*YES)

/* Enable auditing — requires *SECADM authority */
CHGSYSVAL SYSVAL(QAUDCTL) VALUE('*AUDLVL *OBJAUD')
CHGSYSVAL SYSVAL(QAUDLVL) VALUE('*AUTFAIL *SECURITY *PGMFAIL *CREATE *DELETE')
```

---

### 2.4 Check Active Network Services

#### 💬 Bob Prompt

```
List all active TCP/IP network services on the IBM i system and their port numbers.
Flag any services that are commonly disabled for security hardening (e.g. Telnet, FTP, HTTP unencrypted).
```

#### 📋 SQL Query

```sql
SELECT
    LOCAL_ADDRESS,
    LOCAL_PORT,
    REMOTE_ADDRESS,
    REMOTE_PORT,
    CONNECTION_STATE,
    TCP_STATE
FROM QSYS2.NETSTAT_INFO
WHERE CONNECTION_STATE = 'LISTEN'
ORDER BY LOCAL_PORT;
```

Also list active server jobs:

```sql
SELECT JOB_NAME, FUNCTION, ELAPSED_TIME
FROM TABLE(QSYS2.ACTIVE_JOB_INFO(
    SUBSYSTEM_LIST_FILTER => 'QSERVER,QUSRWRK'
))
ORDER BY JOB_NAME;
```

#### ✅ Services to Review

| Port | Service | Recommendation |
|------|---------|---------------|
| 21 | FTP | ❌ Disable — use SFTP instead |
| 23 | Telnet (5250) | ⚠️ Disable if SSH available |
| 80 | HTTP | ⚠️ Disable if not needed |
| 22 | SSH | ✅ Keep active |
| 449 | ACS / DDM | ✅ Keep for ACS users |
| 8470–8476 | IBM i web services | Review if in use |

---

## 3. View Current Users, Roles & Library Access

### 3.1 List All User Profiles

#### 💬 Bob Prompt

```
List all user profiles on the IBM i system. Include their status, user class,
special authorities, and days until password expires. Highlight any profiles
that are enabled, have no password expiry set, or hold elevated authorities.
```

#### 📋 SQL Query

```sql
SELECT
    USER_NAME,
    STATUS,
    USER_CLASS_NAME,
    SPECIAL_AUTHORITIES,
    INITIAL_MENU,
    INITIAL_LIBRARY,
    LIMIT_CAPABILITIES,
    NO_PASSWORD_INDICATOR,
    DAYS_UNTIL_PASSWORD_EXPIRES,
    PREVIOUS_SIGNON
FROM QSYS2.USER_INFO
ORDER BY USER_CLASS_NAME, USER_NAME;
```

> 💡 **Tip:** Run this query in ACS Run SQL Scripts for a sortable, exportable view of all users.

---

### 3.2 Identify Users with Elevated Special Authorities

#### 💬 Bob Prompt

```
Show me all enabled IBM i users who hold any special authority (*ALLOBJ, *SECADM,
*AUDIT, *SAVSYS, *SERVICE, *IOSYSCFG, *JOBCTL, or *SPLCTL).
Group them by authority type and flag any unexpected accounts.
```

#### 📋 SQL Query

```sql
SELECT
    USER_NAME,
    STATUS,
    USER_CLASS_NAME,
    SPECIAL_AUTHORITIES,
    PREVIOUS_SIGNON,
    DAYS_UNTIL_PASSWORD_EXPIRES
FROM QSYS2.USER_INFO
WHERE SPECIAL_AUTHORITIES <> ' '
  AND STATUS = '*ENABLED'
ORDER BY USER_CLASS_NAME, USER_NAME;
```

#### ✅ Special Authorities Reference

| Special Authority | What It Allows |
|---|---|
| `*ALLOBJ` | Access ALL objects regardless of object authority |
| `*SECADM` | Create and manage user profiles |
| `*JOBCTL` | Control jobs, subsystems, output queues |
| `*SPLCTL` | Full control over all spooled files |
| `*SAVSYS` | Save and restore any object |
| `*SERVICE` | Run system service tools |
| `*AUDIT` | Manage audit journal and object auditing |
| `*IOSYSCFG` | Configure I/O devices and network settings |

---

### 3.3 Show Library Access for a Specific User

#### 💬 Bob Prompt

```
Show me all libraries that the user [USERNAME] has authority to on this IBM i system.
Include the authority level (USE, CHANGE, ALL, EXCLUDE) and whether they have
read, add, update, or delete rights on the data within those libraries.
```

#### 📋 SQL Query

```sql
-- Replace 'MYUSER' with the actual username
SELECT
    OBJECT_NAME           AS LIBRARY,
    OBJECT_TYPE,
    AUTHORIZATION_NAME    AS GRANTED_TO,
    OBJECT_AUTHORITY,
    DATA_READ,
    DATA_ADD,
    DATA_UPDATE,
    DATA_DELETE,
    DATA_EXECUTE
FROM QSYS2.OBJECT_PRIVILEGES
WHERE OBJECT_TYPE = '*LIB'
  AND AUTHORIZATION_NAME = 'MYUSER'
ORDER BY OBJECT_NAME;
```

Display all authorities on a specific library:

```sql
SELECT
    AUTHORIZATION_NAME,
    OBJECT_AUTHORITY,
    DATA_READ,
    DATA_ADD,
    DATA_UPDATE,
    DATA_DELETE,
    DATA_EXECUTE
FROM QSYS2.OBJECT_PRIVILEGES
WHERE OBJECT_NAME = 'PRODLIB'
  AND OBJECT_TYPE = '*LIB'
ORDER BY AUTHORIZATION_NAME;
```

---

### 3.4 Check Group Profile Membership

#### 💬 Bob Prompt

```
Show me all IBM i group profiles and which users belong to each group.
Include supplemental group memberships. I want to understand the full
authority inheritance chain for each user.
```

#### 📋 SQL Query

```sql
SELECT
    U.USER_NAME,
    U.STATUS,
    U.USER_CLASS_NAME,
    U.GROUP_PROFILE_NAME,
    U.SUPPLEMENTAL_GROUP_LIST,
    U.SPECIAL_AUTHORITIES
FROM QSYS2.USER_INFO U
WHERE U.GROUP_PROFILE_NAME <> '*NONE'
   OR U.SUPPLEMENTAL_GROUP_LIST <> '*NONE'
ORDER BY U.GROUP_PROFILE_NAME, U.USER_NAME;
```

---

## 4. Create a User Profile with Separation of Duty

Separation of Duty (SoD) ensures no single user has end-to-end control over a critical business process. On IBM i, this is enforced through User Class, Special Authorities, and object-level authority — granting only what is needed for each role.

### 4.1 Design the SoD Role Model

This example implements three roles for a payroll workflow. No single person can both **enter payroll data**, **approve it**, and **view the audit trail**.

| Role | Group Profile | User Class | Special Authorities | Library Access |
|------|--------------|------------|--------------------|-|
| Payroll Clerk | `GRPPAYROLL` | `*USER` | `*NONE` | `PAYLIB: *USE`, `TEMPLIB: *CHANGE` |
| Finance Approver | `GRPAPPROVER` | `*USER` | `*NONE` | `PAYLIB: *CHANGE`, `APPROVELIB: *ALL` |
| Internal Auditor | `GRPAUDITOR` | `*USER` | `*AUDIT` | `PAYLIB: *USE` (read-only), `AUDITLIB: *USE` |

> **SoD best practice:** Always assign authorities to **Group Profiles**, never to individual users. When someone changes role, change their group membership — not individual authority grants.

---

### 4.2 Create the Group Profiles

#### 💬 Bob Prompt

```
On the connected IBM i system, create three group profiles for a payroll SoD model:
1. GRPPAYROLL — Payroll Data Entry group, no special authorities, no password (not for sign-on)
2. GRPAPPROVER — Finance Approval group, no special authorities, no password
3. GRPAUDITOR — Internal Audit group, *AUDIT special authority only, no password
Use appropriate TEXT descriptions for each.
```

#### ⚙️ What Bob Does

Bob runs `CRTUSRPRF` for each group profile via SSH to the IBM i system.

#### 📋 Commands

```cl
/* Payroll Clerk Group Profile */
CRTUSRPRF USRPRF(GRPPAYROLL)
           PASSWORD(*NONE)
           USRCLS(*USER)
           SPCAUT(*NONE)
           TEXT('Group Profile - Payroll Data Entry')
           GRPAUT(*NONE)

/* Finance Approver Group Profile */
CRTUSRPRF USRPRF(GRPAPPROVER)
           PASSWORD(*NONE)
           USRCLS(*USER)
           SPCAUT(*NONE)
           TEXT('Group Profile - Finance Approval')
           GRPAUT(*NONE)

/* Internal Auditor Group Profile */
CRTUSRPRF USRPRF(GRPAUDITOR)
           PASSWORD(*NONE)
           USRCLS(*USER)
           SPCAUT(*AUDIT)
           TEXT('Group Profile - Internal Audit Read Only')
           GRPAUT(*NONE)
```

#### ✅ Verify Group Profiles Were Created

#### 💬 Bob Prompt

```
Verify that the three group profiles GRPPAYROLL, GRPAPPROVER, and GRPAUDITOR
were successfully created on the IBM i system. Show their status, user class,
and special authorities.
```

```sql
SELECT USER_NAME, STATUS, USER_CLASS_NAME, SPECIAL_AUTHORITIES, NO_PASSWORD_INDICATOR
FROM QSYS2.USER_INFO
WHERE USER_NAME IN ('GRPPAYROLL', 'GRPAPPROVER', 'GRPAUDITOR')
ORDER BY USER_NAME;
```

---

### 4.3 Create Individual User Profiles

#### 💬 Bob Prompt

```
Create three individual user profiles on the IBM i system and assign them to
the correct SoD group profiles:
1. JSMITH — Jane Smith, Payroll Clerk, assigned to GRPPAYROLL, limit command line (*YES), force password change
2. MWILSON — Mike Wilson, Finance Approver, assigned to GRPAPPROVER, limit command line (*PARTIAL), force password change
3. SAUDIT — System Auditor, Internal Auditor, assigned to GRPAUDITOR, limit command line (*YES), force password change
All profiles should require a password change on first sign-on. Do not assign any special authorities directly.
```

#### ⚙️ What Bob Does

Bob runs `CRTUSRPRF` for each individual user, assigning them to their group profile.

#### 📋 Commands

```cl
/* Payroll Clerk — JSMITH */
CRTUSRPRF USRPRF(JSMITH)
           PASSWORD(*USRPRF)
           USRCLS(*USER)
           SPCAUT(*NONE)
           GRPPRF(GRPPAYROLL)
           INLPGM(*NONE)
           INLMNU(MAIN)
           LMTCPB(*YES)
           PWDEXP(*YES)
           TEXT('Jane Smith - Payroll Clerk')

/* Finance Approver — MWILSON */
CRTUSRPRF USRPRF(MWILSON)
           PASSWORD(*USRPRF)
           USRCLS(*USER)
           SPCAUT(*NONE)
           GRPPRF(GRPAPPROVER)
           LMTCPB(*PARTIAL)
           PWDEXP(*YES)
           TEXT('Mike Wilson - Finance Approver')

/* Internal Auditor — SAUDIT */
CRTUSRPRF USRPRF(SAUDIT)
           PASSWORD(*USRPRF)
           USRCLS(*USER)
           SPCAUT(*NONE)
           GRPPRF(GRPAUDITOR)
           LMTCPB(*YES)
           PWDEXP(*YES)
           TEXT('System Auditor - Read Only Access')
```

---

### 4.4 Grant Library Authorities to Group Profiles

#### 💬 Bob Prompt

```
Grant the following library authorities on the IBM i system for the SoD payroll model.
Assign authorities to the GROUP profiles only, not the individual users:
- GRPPAYROLL: *USE on PAYLIB, *CHANGE on TEMPLIB
- GRPAPPROVER: *CHANGE on PAYLIB, *ALL on APPROVELIB
- GRPAUDITOR: *USE (read-only) on PAYLIB, *USE on AUDITLIB
Also explicitly revoke *CHANGE, *ADD, and delete rights from GRPAUDITOR on PAYLIB
to enforce separation of duty.
```

#### 📋 Commands

```cl
/* GRPPAYROLL: Read access to payroll library, change access to temp staging library */
GRTOBJAUT OBJ(PAYLIB)    OBJTYPE(*LIB) USER(GRPPAYROLL) AUT(*USE)
GRTOBJAUT OBJ(TEMPLIB)   OBJTYPE(*LIB) USER(GRPPAYROLL) AUT(*CHANGE)

/* GRPAPPROVER: Change access to payroll (can update/approve), full access to approval library */
GRTOBJAUT OBJ(PAYLIB)     OBJTYPE(*LIB) USER(GRPAPPROVER) AUT(*CHANGE)
GRTOBJAUT OBJ(APPROVELIB) OBJTYPE(*LIB) USER(GRPAPPROVER) AUT(*ALL)

/* GRPAUDITOR: Read-only access to payroll (audit visibility), use access to audit library */
GRTOBJAUT OBJ(PAYLIB)   OBJTYPE(*LIB) USER(GRPAUDITOR) AUT(*USE)
GRTOBJAUT OBJ(AUDITLIB) OBJTYPE(*LIB) USER(GRPAUDITOR) AUT(*USE)

/* ENFORCE SoD: Explicitly revoke modification rights from auditor */
RVKOBJAUT OBJ(PAYLIB) OBJTYPE(*LIB) USER(GRPAUDITOR) AUT(*CHANGE *ADD *DLT *UPD)
```

#### 💬 Bob Prompt — Verify SoD Setup

```
Verify the authority setup for the SoD payroll model on IBM i.
Show me a full authority matrix for PAYLIB, TEMPLIB, APPROVELIB, and AUDITLIB
so I can confirm each group has exactly the right level of access and nothing more.
```

```sql
SELECT
    OBJECT_NAME           AS LIBRARY,
    AUTHORIZATION_NAME    AS GRANTED_TO,
    OBJECT_AUTHORITY,
    DATA_READ,
    DATA_ADD,
    DATA_UPDATE,
    DATA_DELETE,
    DATA_EXECUTE
FROM QSYS2.OBJECT_PRIVILEGES
WHERE OBJECT_NAME IN ('PAYLIB', 'TEMPLIB', 'APPROVELIB', 'AUDITLIB')
  AND OBJECT_TYPE = '*LIB'
  AND AUTHORIZATION_NAME IN ('GRPPAYROLL', 'GRPAPPROVER', 'GRPAUDITOR')
ORDER BY OBJECT_NAME, AUTHORIZATION_NAME;
```

---

## 5. Demonstrate the Difference Between Library Access

IBM i object authority has two independent layers:

1. **Library Authority** — controls whether a user can find and navigate to objects inside a library
2. **Object Authority** — controls what a user can actually do with a specific object (file, program, etc.)

Both layers must allow access for a user to successfully use an object.

### 5.1 Authority Levels Explained

| Authority | Library Level | Object (File/Program) Level |
|-----------|--------------|----------------------------|
| `*EXCLUDE` | Cannot see or use the library | Cannot open, run, read, or modify |
| `*USE` | Can find and navigate to objects | Read-only (open INPUT, CALL program) |
| `*CHANGE` | Can use and navigate library | Read + Write (open I-O, INSERT, UPDATE, DELETE) |
| `*ALL` | Full management (rename, delete library) | Full object control including delete, save/restore |
| `*AUTL` | Controlled by an Authorization List | Controlled by an Authorization List |

---

### 5.2 Setup: Create Test Libraries and Objects

#### 💬 Bob Prompt

```
On the IBM i system, create two demonstration libraries called DEMOREADLIB and DEMOWRITELIB.
In each library create a simple physical file. Then remove all *PUBLIC authority from both
libraries and their files so we start from a fully locked-down state.
```

#### 📋 Commands

```cl
/* Create demonstration libraries */
CRTLIB LIB(DEMOREADLIB)  TEXT('Demo - Read Only Access Library')
CRTLIB LIB(DEMOWRITELIB) TEXT('Demo - Read Write Access Library')

/* Create a physical file (table) in each library */
CRTPF FILE(DEMOREADLIB/EMPLOYEES)  RCDLEN(132)
CRTPF FILE(DEMOWRITELIB/ORDERS)    RCDLEN(132)

/* Remove *PUBLIC authority from both libraries — lock down baseline */
RVKOBJAUT OBJ(DEMOREADLIB)  OBJTYPE(*LIB) USER(*PUBLIC) AUT(*ALL)
RVKOBJAUT OBJ(DEMOWRITELIB) OBJTYPE(*LIB) USER(*PUBLIC) AUT(*ALL)

/* Remove *PUBLIC authority from the files too */
RVKOBJAUT OBJ(DEMOREADLIB/EMPLOYEES)  OBJTYPE(*FILE) USER(*PUBLIC) AUT(*ALL)
RVKOBJAUT OBJ(DEMOWRITELIB/ORDERS)    OBJTYPE(*FILE) USER(*PUBLIC) AUT(*ALL)
```

---

### 5.3 Scenario A — Read-Only Access (`*USE`)

#### 💬 Bob Prompt

```
Grant JSMITH *USE authority on DEMOREADLIB and its EMPLOYEES file on the IBM i system.
Then show me what operations JSMITH is allowed to perform and which ones will fail
with a CPF9802 Not Authorized error.
```

#### 📋 Commands — Grant Access

```cl
/* Grant *USE to JSMITH on the library and the file */
GRTOBJAUT OBJ(DEMOREADLIB)           OBJTYPE(*LIB)  USER(JSMITH) AUT(*USE)
GRTOBJAUT OBJ(DEMOREADLIB/EMPLOYEES) OBJTYPE(*FILE) USER(JSMITH) AUT(*USE)
```

#### ✅ What JSMITH Can Do (Will Succeed)

```cl
/* Display file description — OK */
DSPFD FILE(DEMOREADLIB/EMPLOYEES)

/* Query/read the file — OK */
RUNQRY QRYFILE((DEMOREADLIB/EMPLOYEES))
```

```sql
-- Read records via SQL — OK
SELECT * FROM DEMOREADLIB.EMPLOYEES;
```

#### ❌ What JSMITH Cannot Do (Will Fail — CPF9802)

```sql
-- Cannot insert a record
INSERT INTO DEMOREADLIB.EMPLOYEES VALUES('Test');
```

```cl
/* Cannot delete the file */
DLTF FILE(DEMOREADLIB/EMPLOYEES)

/* Cannot create objects in the library */
CRTPF FILE(DEMOREADLIB/NEWFILE) RCDLEN(80)
```

---

### 5.4 Scenario B — Read/Write Access (`*CHANGE`)

#### 💬 Bob Prompt

```
Grant MWILSON *CHANGE authority on DEMOWRITELIB and its ORDERS file on the IBM i system.
Show which operations succeed (read, insert, update) and which ones still fail
(deleting the file object itself, saving the library).
```

#### 📋 Commands — Grant Access

```cl
/* Grant *CHANGE to MWILSON on the write library and its file */
GRTOBJAUT OBJ(DEMOWRITELIB)        OBJTYPE(*LIB)  USER(MWILSON) AUT(*CHANGE)
GRTOBJAUT OBJ(DEMOWRITELIB/ORDERS) OBJTYPE(*FILE) USER(MWILSON) AUT(*CHANGE)
```

#### ✅ What MWILSON Can Do (Will Succeed)

```sql
-- Read records
SELECT * FROM DEMOWRITELIB.ORDERS;

-- Insert a record
INSERT INTO DEMOWRITELIB.ORDERS VALUES('ORD001', 'PENDING');

-- Update a record
UPDATE DEMOWRITELIB.ORDERS SET STATUS = 'APPROVED' WHERE ORDNO = 'ORD001';
```

#### ❌ What MWILSON Cannot Do (Will Fail)

```cl
/* Cannot delete the file object itself — needs *ALL */
DLTF FILE(DEMOWRITELIB/ORDERS)

/* Cannot save or restore the library — needs *SAVSYS or *ALL */
SAVLIB LIB(DEMOWRITELIB) DEV(*SAVF) SAVF(QGPL/MYSAVF)
```

---

### 5.5 Scenario C — No Access (`*EXCLUDE`)

#### 💬 Bob Prompt

```
Explicitly exclude SAUDIT from DEMOWRITELIB on the IBM i system to enforce
separation of duty — the auditor must not be able to access the write library at all.
Then grant SAUDIT *USE read-only access to DEMOREADLIB for audit visibility.
Show me the errors that SAUDIT would see when attempting to access DEMOWRITELIB.
```

#### 📋 Commands

```cl
/* Ensure SAUDIT has no access to the write library (*PUBLIC is already *EXCLUDE) */
RVKOBJAUT OBJ(DEMOWRITELIB) OBJTYPE(*LIB) USER(SAUDIT) AUT(*ALL)

/* Grant SAUDIT read-only access to DEMOREADLIB for audit purposes */
GRTOBJAUT OBJ(DEMOREADLIB)           OBJTYPE(*LIB)  USER(SAUDIT) AUT(*USE)
GRTOBJAUT OBJ(DEMOREADLIB/EMPLOYEES) OBJTYPE(*FILE) USER(SAUDIT) AUT(*USE)
```

#### ❌ What SAUDIT Cannot Do (SoD enforced)

```cl
/* DEMOWRITELIB will not appear in library list */
DSPLIBL

/* Error: Not authorized to library DEMOWRITELIB — CPF9802 */
WRKOBJLIB LIB(DEMOWRITELIB)
```

---

### 5.6 Verify the Full Authority Setup

#### 💬 Bob Prompt

```
Run a full authority verification report for DEMOREADLIB and DEMOWRITELIB
on the IBM i system. Show me who has what access, confirm JSMITH can only read,
MWILSON can read and write, and SAUDIT is excluded from DEMOWRITELIB.
Format it as a clear access matrix.
```

#### 📋 CL Command

```cl
/* Display all authorities on a library */
DSPOBJAUT OBJ(DEMOREADLIB) OBJTYPE(*LIB)
```

#### 📋 SQL Query — Full Authority Matrix

```sql
SELECT
    OBJECT_NAME           AS LIBRARY,
    AUTHORIZATION_NAME    AS USER_OR_GROUP,
    OBJECT_AUTHORITY,
    DATA_READ,
    DATA_ADD,
    DATA_UPDATE,
    DATA_DELETE,
    DATA_EXECUTE
FROM QSYS2.OBJECT_PRIVILEGES
WHERE OBJECT_NAME IN ('DEMOREADLIB', 'DEMOWRITELIB')
  AND OBJECT_TYPE = '*LIB'
ORDER BY OBJECT_NAME, AUTHORIZATION_NAME;
```

Also verify file-level authorities:

```sql
SELECT
    OBJECT_NAME,
    OBJECT_TYPE,
    AUTHORIZATION_NAME,
    OBJECT_AUTHORITY,
    DATA_READ,
    DATA_ADD,
    DATA_UPDATE,
    DATA_DELETE
FROM QSYS2.OBJECT_PRIVILEGES
WHERE OBJECT_SCHEMA IN ('DEMOREADLIB', 'DEMOWRITELIB')
ORDER BY OBJECT_SCHEMA, OBJECT_NAME, AUTHORIZATION_NAME;
```

---

### 5.7 Summary — Authority Comparison Matrix

| User / Group | DEMOREADLIB | DEMOWRITELIB | Can Read | Can Write / Delete |
|---|---|---|---|---|
| `JSMITH` (GRPPAYROLL) | `*USE` | `*EXCLUDE` | ✅ DEMOREADLIB only | ❌ No |
| `MWILSON` (GRPAPPROVER) | `*EXCLUDE` | `*CHANGE` | ✅ DEMOWRITELIB only | ✅ Insert/Update (no object delete) |
| `SAUDIT` (GRPAUDITOR) | `*USE` | `*EXCLUDE` | ✅ DEMOREADLIB only (audit view) | ❌ No (SoD enforced) |
| `*PUBLIC` | `*EXCLUDE` | `*EXCLUDE` | ❌ No | ❌ No |
| Administrator | `*ALL` | `*ALL` | ✅ Yes | ✅ Full control |

> **Key takeaway:** IBM i authority is additive through group membership and explicit grants. Having `*USE` on a library grants list and navigation access, but each object inside still requires its own authority grant. This two-layer model — library authority + object authority — is fundamental to IBM i access control.

---

## Appendix A — Useful Commands Quick Reference

| Command | Purpose |
|---|---|
| `DSPSYSVAL SYSVAL(*SECU)` | Display all security system values |
| `DSPUSRPRF USRPRF(*ALL)` | List all user profiles |
| `CRTUSRPRF` | Create a new user profile |
| `CHGUSRPRF` | Modify an existing user profile |
| `DLTUSRPRF` | Delete a user profile (use `OWNOBJOPT` to handle owned objects) |
| `GRTOBJAUT` | Grant object authority to a user or group |
| `RVKOBJAUT` | Revoke object authority |
| `DSPOBJAUT` | Display current authorities on an object |
| `CHGOBJOWN` | Change object ownership |
| `WRKOBJ / WRKOBJLST` | Work with objects in a library |
| `DSPLIBAUT` | Display library authority |
| `GO SECURITY` | Open IBM i Security management menu |
| `GO SECTOOLS` | Open IBM i Security Tools menu (reporting) |
| `PRTUSRPRF USRPRF(*ALL) TYPE(*BASIC)` | Print security summary report for all users |
| `STRTCPSVR SERVER(*SSHD)` | Start SSH daemon |
| `ENDTCPSVR SERVER(*SSHD)` | Stop SSH daemon |
| `NETSTAT *CNN` | Display TCP/IP connection status |

---

## Appendix B — SQL Catalog Views for Security Reporting

| View | What It Shows |
|---|---|
| `QSYS2.USER_INFO` | All user profiles, status, authorities, and settings |
| `QSYS2.OBJECT_PRIVILEGES` | Object-level authorities for all objects |
| `QSYS2.GROUP_PROFILE_ENTRIES` | Group membership — who is in which group profile |
| `QSYS2.ACTIVE_JOB_INFO` | Currently active jobs (function, subsystem, user) |
| `QSYS2.NETSTAT_INFO` | Network connections — active TCP/IP sessions |
| `QSYS2.SYSVAL_INFO` | All system values and their current settings |
| `QSYS2.JOURNAL_ENTRIES` | Query audit journal entries programmatically |
| `QSYS2.SERVER_JOB_INFO` | Active TCP/IP server information |

---

## Quick-Reference Bob Prompts

Copy and paste these prompts directly into Bob for each task:

```
# Security Audit
"Run a full IBM i security audit on [HOSTNAME]. Check QSECURITY, all password
system values, audit journal status, and active network services. Flag any
settings that do not meet IBM production security recommendations."

# User Report
"List all user profiles on the IBM i system at [HOSTNAME] including their
special authorities, group memberships, and last sign-on date. Highlight any
enabled accounts with no password expiry or with *ALLOBJ or *SECADM authority."

# Library Access Report
"Show me a complete authority matrix for the library [LIBNAME] on the IBM i
system. List every user and group with any authority level, including *EXCLUDE
entries, and show read/write/delete rights at the data level."

# Create SoD User
"Create a new IBM i user profile [USERNAME] for [FULL NAME] on [HOSTNAME].
Assign them to the group profile [GRPNAME]. Set user class *USER with no
special authorities. Limit command line access and force a password change on
first sign-on. Add descriptive text '[ROLE DESCRIPTION]'."

# Verify SoD
"Verify the separation of duty configuration on IBM i [HOSTNAME]. Confirm that
GRPAUDITOR cannot modify PAYLIB, GRPPAYROLL cannot access APPROVELIB, and
GRPAPPROVER cannot access AUDITLIB. Show me the full authority matrix."
```

---

## Document Information

- **Version:** 1.0
- **Platform:** IBM i (OS/400) — tested on IBM i 7.4 and 7.5
- **Tool:** IBM Bob AI Assistant
- **Authority required:** `*SECADM` or `*ALLOBJ` for most administrative commands
- **Security note:** Always test in a non-production environment first. Review IBM i security documentation for your specific OS version and compliance requirements.

---
