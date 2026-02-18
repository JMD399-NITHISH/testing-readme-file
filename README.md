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

---

## 🔹 Column A – `user_name`

Specifies the **user email address**.

Example:

```
nithish.y@jmangroup.com
```

This user will be:

- Added to the AD group (if within effective date)
- Removed from the AD group (if expired)

---

## 🔹 Column B – `adgrp_name`

Specifies the **Azure AD Security Group name**.

### Behavior:

- If the group does NOT exist → it will be created.
- If the group already exists → members will be added/removed accordingly.
- Users are never granted direct permissions — only via AD groups.

---

## 🔹 Column C – `description`

Specifies the description for the AD group.

### Behavior:

- Used only when the AD group is newly created.
- If the group already exists → this value is ignored.
- Helps document business purpose of the group.

Example:

```
Finance SQL readers from north region
```

---

## 🔹 Column D – `owners`

Specifies the AD group owners.

### Rules:

- Only applied when the group is newly created.
- Ignored if group already exists.
- Multiple owners must be separated by commas

Owners are assigned as Entra ID group owners.

---

## 🔹 Column E – `effective_from`

Start date for user membership in the AD group.

- If current date ≥ `effective_from`
- And current date ≤ `effective_to`
→ User will be added to the group.

---

## 🔹 Column F – `effective_to`

End date for user membership in the AD group.

If:

```
current_date > effective_to
```

→ User will be automatically removed from the AD group.

### Revocation Process:

To revoke a user:
1. Update `effective_to` to yesterday’s date.
2. Run the pipeline.
3. User will be removed from the AD group automatically.

---

## 🔹 Column G – `workspace_name` (Optional)

Specifies the Fabric workspace where the AD group should be assigned.

### Behavior:

- If value is provided → group will be added to workspace.
- If blank → no workspace-level assignment will occur.

---

## 🔹 Column H – `role` (Optional)

Specifies the workspace role.

Allowed values:

```
Viewer
Contributor
Member
Admin
```

### Behavior:

- Only applied if `workspace_name` is specified.
- If workspace is blank → this column is ignored.

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

| user_name | adgrp_name | workspace_name | role |
|------------|------------|----------------|------|
| kavin.n@jmangroup.com | sg_dm_fbw_uk_north | ws-fabric-sandbox | Viewer |

Result:

- User added to `sg_dm_fbw_uk_north`
- Group assigned to workspace `ws-fabric-sandbox`
- Role granted as Viewer

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

---

## 🔹 Column A – `ad_group_name`

Specifies the **Azure AD Security Group**.

- Users are added to this AD group in the `user_group_management` stage of the pipeline.
- All OLS and RLS permissions are applied **only to this AD group**.
- No direct user-level permissions are granted.
- Serves as the central security identity.

---

## 🔹 Column B – `workspace_name`

Specifies the Fabric workspace.

The pipeline uses this value to:
- Locate the target artifact
- Apply permissions within the correct workspace

---

## 🔹 Column C – `artifact_type`

Allowed values:

```
Warehouse
Lakehouse
```

### Behavior

| Artifact Type | OLS | RLS |
|---------------|-----|-----|
| Lakehouse     | ✅ Yes | ❌ No |
| Warehouse     | ✅ Yes | ✅ Yes |

- Lakehouse → Only Object-Level Security
- Warehouse → Object-Level + Row-Level Security

---

## 🔹 Column D – `object_type`

Allowed values:

```
table
schema
file
```

### 1️⃣ `table`
- Works for Warehouse & Lakehouse
- Format required:
  ```
  schema.table_name
  ```

### 2️⃣ `schema`
- Works for Warehouse & Lakehouse
- Format:
  ```
  schema_name
  ```

### 3️⃣ `file`
- Only supported for **Lakehouse**
- Applies file/folder-level security
- Example:
  ```
  Customer Tool - Final/Input
  ```

---

## 🔹 Column E – `artifact_name`

Specifies the target Lakehouse or Warehouse name.

Examples:

```
lh_core
dwh_report
```

---

## 🔹 Column F – `object_path`

Specifies object location depending on object type.

### If `schema`
```
dbo
```

### If `table`
```
dbo.fact_invoice
```

### If `file`
```
Customer Tool - Final/Input
```

---

## 🔹 Column G – `access_level`

Allowed values:

```
Read
ReadWrite
```

### Permission Mapping

| Access Level | Permissions |
|--------------|------------|
| Read         | SELECT |
| ReadWrite    | SELECT, INSERT, UPDATE, DELETE |

---

## 🔹 Column H – `rls_level`

Applicable only when:

- `artifact_type = Warehouse`
- `object_type = table`

Specifies the column used for Row-Level Security.

Example:
```
region
source
```

---

## 🔹 Column I – `rls_data`

Specifies the filter value for RLS.

Example:

| rls_level | rls_data |
|-----------|----------|
| region    | UK North |
| source    | sap |

If blank → No RLS applied.

---

## 🔹 Column J – `effective_from`

Date when the permission becomes active.

---

## 🔹 Column K – `effective_to`

Date until which permission remains valid.

### Revoking Access

To revoke access:
1. Update `effective_to` to yesterday’s date.
2. Run the Notebook.
3. Permissions will be revoked automatically.

⚠️ Date-based revocation currently implemented for Warehouse.

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
