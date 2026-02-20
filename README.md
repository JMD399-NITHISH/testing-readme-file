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

