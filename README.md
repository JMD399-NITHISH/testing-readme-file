# User Group Management – Metadata Framework

![User Group Metadata](./images/user-group-metadata.png)

---

## 📘 Overview

This metadata file controls **Azure AD Group Management and Workspace Assignment**.

The pipeline uses this file to:

- Create AD security groups (if not already present)
- Add or remove users from AD groups
- Assign AD groups to Fabric workspaces (optional)
- Control membership lifecycle using effective dates

All operations are fully automated and metadata-driven.

---

# 🧩 Column-by-Column Definition
| Column Name      | Description                                                                                                                                                                               |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `user_name`      | Email ID of the user to be added or removed from the Azure AD group. Membership is controlled using effective dates.                                                                      |
| `adgrp_name`     | Azure AD Security Group name. If the group does not exist, it will be created. If it exists, users will be added or removed accordingly. All permissions are granted only via this group. |
| `description`    | Description of the AD group. Used only when creating a new group. Ignored if the group already exists.                                                                                    |
| `owners`         | Owner(s) of the AD group. Applied only when the group is newly created. Multiple owners must be separated by commas.                                                                      |
| `effective_from` | Start date from which the user should be part of the AD group. If the current date falls within the range, the user is added.                                                             |
| `effective_to`   | End date for group membership. If the current date exceeds this date, the user is removed automatically. Used for lifecycle-based access control.                                         |
| `workspace_name` | *(Optional)* Fabric workspace where the AD group should be assigned. If blank, no workspace assignment is performed.                                                                      |
| `role`           | *(Optional)* Workspace role (`Viewer`, `Contributor`, `Member`, `Admin`). Applied only if `workspace_name` is provided.                                                                   |


---

# 🔄 Execution Logic Flow

For each row:

1. Check if AD group exists.
2. If not:
   - Create group.
   - Set description.
   - Assign owners.
3. Check effective dates:
   - Add user if active.
   - Remove user if expired.
4. If workspace_name is provided:
   - Assign AD group to workspace with specified role.

---

# 🏗 Example Scenario

| user_name                   | adgrp_name           | description                                                                 | owners                      | effective_from | effective_to | workspace_name      | role   |
|-----------------------------|----------------------|-----------------------------------------------------------------------------|-----------------------------|----------------|--------------|---------------------|--------|
| nithish.y@jmangroup.com     | sg_dm_fbw_uk_north   | Members of UK north Region  | fabric.admin@jmangroup.com  | 2026-02-01     |  2027-09-17            | ws-fabric-sandbox   | Viewer |


Result:

- User nithish.y@jmangroup.com is added to the Azure AD group sg_dm_fbw_uk_north
- The AD group sg_dm_fbw_uk_north is assigned to the Fabric workspace ws-fabric-sandbox
- The group is granted the Viewer role in the workspace
- fabric.admin@jmangroup.com is configured as the group owner
- Access is effective from 2026-02-01

---

# 🏛 Governance Principles

- No direct workspace access to users.
- All access controlled via AD groups.
- Lifecycle managed through effective dates.
- Workspace assignment is optional.
- Centralized metadata-driven identity management.

---

# 📌 Notes

- Group description and owners are applied only during creation.
- Owners must be separated by commas.
- Workspace role assignment is optional.
- Effective dates control membership automatically.
- Supports multiple users per group.

---


# RBAC Metadata Framework – OLS & RLS (Fabric)

![RBAC Metadata](./images/rbac-metadata.png)

---

## 📘 Overview

This metadata file drives **Role-Based Access Control (RBAC)** automation in Microsoft Fabric for:

- **Object Level Security (OLS)** – Lakehouse & Warehouse
- **Row Level Security (RLS)** – Warehouse only

All access provisioning is handled through a metadata-driven Notebook.

---

# 🧩 Metadata Column Definitions

| Column Name      | Description                                                                                                                                                                             |
| ---------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ad_group_name`  | Azure AD Security Group to which Object-Level Security (OLS) and Row-Level Security (RLS) permissions are applied. Users are managed separately through the User Group metadata.        |
| `workspace_name` | Fabric workspace where the target artifact (Lakehouse or Warehouse) exists. Used to locate and apply permissions.                                                                       |
| `artifact_type`  | Type of artifact. Allowed values: `Lakehouse` or `Warehouse`. Lakehouse supports OLS only. Warehouse supports both OLS and RLS.                                                         |
| `object_type`    | Security level of the object. Allowed values: `schema`, `table`, `file`. File-level security applies only to Lakehouse. Schema and table apply to both Lakehouse and Warehouse.         |
| `artifact_name`  | Name of the Lakehouse or Warehouse where access must be applied (e.g., `lh_core`, `dwh_report`).                                                                                        |
| `object_path`    | Target object location. For schema-level → specify schema name. For table-level → specify `schema.table_name`. For file-level → specify folder path inside the Lakehouse Files section. |
| `access_level`   | Permission level. Allowed values: `Read` (SELECT) or `ReadWrite` (SELECT, INSERT, UPDATE, DELETE).                                                                                      |
| `rls_level`      | *(Warehouse only, table-level only)* Column name on which Row-Level Security filter should be applied (e.g., `region`, `source`).                                                       |
| `rls_data`       | *(Warehouse only)* Value used in RLS filtering. Users in the group will see only rows matching this value. Leave blank if RLS is not required.                                          |
| `effective_from` | Date from which the permission becomes active for the AD group.                                                                                                                         |
| `effective_to`   | Date until which permission remains valid. To revoke access, set this date to a past date and re-run the notebook.                                                                      |

---

# 🔐 Security Model Summary

---

## ✅ Lakehouse

Supports:
- Schema-level OLS
- Table-level OLS
- File-level OLS

Does NOT support:
- RLS

---

## ✅ Warehouse

Supports:
- Schema-level OLS
- Table-level OLS
- Row-Level Security (RLS)

Does NOT support:
- File-level security

---

# 🔄 Access Lifecycle

1. Users are added to Azure AD groups.
2. Metadata defines access rules.
3. Pipeline:
   - Applies Object-Level Security (OLS)
   - Applies Row-Level Security (RLS) for Warehouse
4. Effective dates manage activation & revocation.

---

# 🏗 Example Scenarios

---

## Example 1 – Warehouse RLS

| Group | Object | RLS Column | Value |
|-------|--------|------------|-------|
| sg_dm_fbw_uk_north | dbo.fact_sales | region | UK North |

Result:
Users in the group see only:

```
region = 'UK North'
```

---

## Example 2 – Lakehouse File-Level

| Group | Object Type | Path |
|-------|------------|------|
| sg_dm_fbw_analyst | file | Customer Tool - Final/Input |

Users can access only that folder inside the Lakehouse.

---

## Example 3 – Schema-Level OLS

| Group | Object Type | Object Path |
|-------|------------|------------|
| sg_dm_fbw_analyst | schema | ip_data |

Users can access all tables under schema `ip_data`.

---

# 🔧 Prerequisites

---

1️⃣ Azure AD App Registration (Service Principal)

- An Azure AD Application (Service Principal) must be created and configured with the required API permissions.

✅ Microsoft Graph – Application Permissions

The following Application permissions must be granted with Admin Consent:

- Directory.ReadWrite.All – Read and write directory data
- Group.Create – Create security groups
- Group.ReadWrite.All – Read and write all groups
- GroupMember.ReadWrite.All – Manage group memberships
- RoleManagement.ReadWrite.Directory – Manage directory RBAC settings
- User.Read.All – Read all users’ full profiles

⚠️ Admin consent is mandatory for these permissions.

✅ Power BI / Fabric – Delegated Permissions

The Service Principal must also have the following Fabric / Power BI API permissions:

- Workspace.ReadWrite.All
- Lakehouse.ReadWrite.All
- Lakehouse.Execute.All
- Warehouse.Execute.All
- Dataset.ReadWrite.All
- Report.ReadWrite.All
- SQLendpoint.Read.All
- OneLake.ReadWrite.All
- Item.ReadWrite.All
- Item.Execute.All
- Capacity.Read.All
- Capacity.ReadWrite.All
- StorageAccount.ReadWrite.All
- UserState.ReadWrite.All
- Code.AccessFabric.All

Ensure all required permissions show “Granted for <Tenant Name>”.

2️⃣ Fabric Workspace Requirements

The Service Principal must be added to the Fabric workspace.

- It must have Admin or Contributor role (depending on required operations).

3️⃣ Environment Variables / Secrets

- Client ID
- Client Secret (stored securely in Key Vault or Fabric secrets)
- Tenant ID
- Workspace ID
- Lakehouse name
- 
## OneLake Security (Required for OLS in SQL Endpoint)

To ensure Object Level Security (OLS) reflects correctly in the SQL Analytics Endpoint, the following configuration is mandatory:

### Enable OneLake Security for Tables

Before creating any custom SQL roles for OLS:

1. Navigate to the **SQL Analytics Endpoint**
2. Open **Data access mode (Preview)**
3. Select:

   ✅ **Use OneLake security for tables (User’s identity access mode)**

4. Click **Apply**

---

### Important Notes

- This setting must be enabled **before creating SQL roles**
- OLS will not reflect in the SQL endpoint if this option is not enabled
- The user's identity will be used to access table data
- Table permissions will be controlled via **OneLake security**
- Views, Stored Procedures, and Functions will continue to use **SQL security**

---

### Required Permissions

To change this setting:

- The user must have **Admin role** in the Fabric workspace
- Only Workspace Admins can modify the Data Access Mode


⚠️ If this setting is not enabled prior to role creation, security synchronization errors may occur and OLS may not function as expected.

---

## ⚠️ Known Limitations & Considerations

This implements metadata-driven RBAC for Fabric Lakehouse and Warehouse. However, there are certain platform-level limitations and behavioral constraints to be aware of.

---

### 🔹 1️⃣ Lakehouse Does Not Support Native RLS

- **Lakehouse supports Object-Level Security (OLS) only int his release.**
- **Row-Level Security (RLS) is not supported in Lakehouse.**

---

### 🔹 2️⃣ SQL Endpoint Metadata Refresh Issue (Lakehouse)

When applying OLS roles via API:
- ✅ Role creation works
- ✅ Members are added correctly
- ✅ Permissions reflect in Lakehouse view
- ❌ **But SQL Analytics Endpoint may not immediately recognize role membership**

#### Observed Behavior
Even after adding all members to a role:
- SQL endpoint sometimes does not refresh security metadata
- Users cannot query tables immediately

#### Workaround Used
To force SQL endpoint metadata refresh:
1. Add a temporary (dummy) member to the role
2. Remove the dummy member
3. This triggers a metadata refresh in the SQL endpoint
4. Permissions then work as expected

**Impact:**
- Additional operational step required
- May introduce slight automation complexity

---

### 🔹 3️⃣ Effective-Date Revocation Applies Per Execution

- Access revocation depends on pipeline execution
- If pipeline is not run, expired permissions remain active
- No real-time revocation without execution

**Impact:**  
Operational discipline required to run pipeline regularly.

---

## 🧠 Architectural Considerations

- **Azure AD Group-based model** ensures centralized identity management
- **Metadata-driven approach** improves governance
- However, Fabric SQL endpoint security relies on internal synchronization mechanisms
- Certain behaviors (like dummy member refresh) are platform-dependent and may change in future updates
---

# Row-Level Security (RLS) – Implementation Guide

**Scope:** Semantic Model Security Configuration
**Applies To:** Microsoft Fabric Semantic Models and reports 

---

## Overview

This document explains how to configure **Row-Level Security (RLS)** using **metadata-driven logic** and the `nb_object_level_security` notebook.

The RLS implementation dynamically generates a security mapping table and applies it to the Semantic Model using relationships and role assignments.

---

# Step 1: Update Metadata with Access Details

Before running the notebook, update the security metadata with appropriate access mappings.

### What to update:

* User / Group Identifier
* Entity / Business Key
* Access Level 
* Any filtering attributes used in the fact/dimension tables

⚠️ Ensure:

* No duplicate or conflicting access entries
* All required keys match the keys used in the semantic model
* Email IDs are accurate

This metadata drives the entire RLS logic.

---

# Step 2: Run the Security Notebook

Run the notebook:

```
nb_object_level_security
```

### What this notebook does:

* Reads the metadata
* Processes access mappings
* Generates a security table:

```
security.rls_metadata
```

### Output:

A table created under:

```
Schema: security  
Table: rls_metadata
```

This table will contain:

* User Identifier (email)
* Security Key(s)

---

# Step 3: Add the RLS Table to the Semantic Model

1. Open the Semantic Model.
2. Add the table:

```
security.rls_metadata
```

3. Create relationships based on your model design.

---

# Step 4: Configure Relationships

You must create **one of the following relationships**:

### Option A – Many-to-Many (Fact-Level Filtering)

Connect:

```
security.rls_metadata  ⇄  Final Fact Table
```

Use this when:

* Security keys directly filter the fact table.
* Multiple access rows per user are expected.

---

### Option B – Many-to-One (Dimension-Level Filtering)

Connect:

```
security.rls_metadata  →  Dimension Table
```

Then ensure:

* The Dimension table is already related to the Fact table.
* Filter direction allows propagation to the Fact table.

Use this when:

* Security is dimension-driven.
* Cleaner star schema enforcement is preferred.

---

# Step 5: Create Roles in Semantic Model

1. Open the Semantic Model.
2. Go to **Manage Roles**.
3. Create a new role (e.g., `Dynamic_RLS`).

### Apply Filter on:

```
security.rls_metadata
```

Use:

```
[User] = USERPRINCIPALNAME()
```

This ensures the logged-in user only sees their mapped data.

---

# Step 6: Assign Users to Roles

1. Go to **Manage Roles**
2. Add:

   * Users
   * Security Groups
3. Assign them to the created role.

---

# Validation Checklist

- Metadata updated
- Notebook executed successfully
- `security.rls_metadata` table created
- Relationship configured correctly
- Role created with USERPRINCIPALNAME() filter
- Users assigned to role
- Tested using “View As Role”

---

# Architecture Flow

```
Metadata → Notebook → security.rls_metadata → Semantic Model Relationship → Role → User Access
```

---

# Important Notes

* Always re-run the notebook after metadata changes.
* If access issues occur, validate:

  * Relationship cardinality
  * Cross-filter direction
  * UPN format
* Many-to-One via Dimension is recommended for clean modeling.
* Many-to-Many should be used only when necessary.

---

# Reference

[Microsoft Fabric row-level security documentation](https://learn.microsoft.com/en-us/fabric/security/service-admin-row-level-security)



