# Access Levels and Permissions

This document explains the access control system used throughout the API, including how access levels work, how permissions are checked, and how the `access` property in API responses should be interpreted.

## Table of Contents

1. [Overview](#overview)
2. [Access Levels](#access-levels)
3. [Access Level Hierarchy](#access-level-hierarchy)
4. [User Groups](#user-groups)
5. [How Access Checks Work](#how-access-checks-work)
6. [The `access` Property in API Responses](#the-access-property-in-api-responses)
7. [Public Resources](#public-resources)
8. [Sharing Objects via Links](#sharing-objects-via-links)
9. [Parent-Child Access Inheritance](#parent-child-access-inheritance)

## Overview

The API uses a hierarchical permission system based on access levels. Every resource (object) in the system can have permissions assigned to user groups, and users gain access to resources through their group memberships.

Access control is based on three key concepts:

1. **Access Levels** - A hierarchical set of permission levels (O, A, D, W, C, R, N)
2. **User Groups** - Collections of users with shared access rights
3. **Object Permissions** - Mappings between objects and user groups with specific access levels

## Access Levels

The system defines the following access levels, from highest to lowest:

| Level | Name    | Description |
|-------|---------|-------------|
| **O** | Owner   | Full control over the object. Can transfer ownership to another user or group. Only one owner per object. |
| **A** | Admin   | Can manage access rights for other users, perform administrative tasks, and grant lower access levels to others. |
| **D** | Delete  | Can delete the object permanently. |
| **W** | Write   | Can modify the object's data and properties. |
| **C** | Create  | Can create child objects under this object. |
| **R** | Read    | Can view and read the object's full content. |
| **N** | Notify  | Receives notifications about the object but cannot read its full content. This is a special level outside the main hierarchy. |

Additionally, there is a "View" level (displayed as lowercase `r` in responses) which indicates minimal access - the user can see the object exists but cannot read its contents. This typically applies to public resources.

## Access Level Hierarchy

Access levels follow a hierarchical structure where higher levels implicitly include the capabilities of all lower levels:

```
O (Owner)
 └── A (Admin)
      └── D (Delete)
           └── W (Write)
                └── C (Create)
                     └── R (Read)
```

For example:
- A user with **W** (Write) access automatically has **C** (Create) and **R** (Read) access
- A user with **A** (Admin) access has all permissions except **O** (Owner)
- A user with **O** (Owner) access has complete control

**Special case: Notify (N)**

The **N** (Notify) level is separate from the main hierarchy. Users with **N** access:
- Receive notifications about the object
- Do NOT automatically have **R** (Read) access
- Cannot read the object's full content unless explicitly granted **R** or higher

## User Groups

Permissions are assigned to **User Groups** rather than individual users. This provides flexible access management:

### Group Types

| Type | Description |
|------|-------------|
| **user** | A native/private group automatically created for each user. Cannot be deleted. Every user has exactly one native group. |
| **group** | A regular collaborative group that can be created, managed, and deleted. Multiple users can be members with different access levels. |

### Group Membership

Users can belong to multiple groups with different access levels in each:

- **O** (Owner) - Full control over the group itself
- **A** (Admin) - Can manage members (except owner), add/remove users with R/C/W/D access
- **D** (Delete) - Can remove resources from the group
- **W** (Write) - Can modify resources in the group
- **C** (Create) - Can create new resources in the group context
- **R** (Read) - Can view group information and member list

Important rules:
- Only **one owner** per group
- Owners **cannot leave** their group (must transfer ownership first)
- Only owners can add or remove admin members
- Members can voluntarily leave a group (except owners)

## How Access Checks Work

When you access a resource through the API, the system performs access checks as follows:

### 1. Determine Required Access Level

Each API operation requires a minimum access level:

| Operation | Method | Typical Required Level |
|-----------|--------|------------------------|
| List resources | GET | R (Read) |
| View resource | GET | R (Read) |
| Create resource | POST | Depends on parent object |
| Update resource | PATCH | W (Write) |
| Delete resource | DELETE | A (Admin) |
| Custom methods | Varies | Defined by the method |

### 2. Calculate Effective Access

The system determines your effective access level by:

1. Finding all your group memberships
2. For each group, checking if it has access to the target object
3. Calculating the effective access as the **minimum** of:
   - Your access level within the group
   - The group's access level to the object
4. Taking the **highest** effective access across all your groups

**Example:**
- You have **A** (Admin) access in Group X
- Group X has **W** (Write) access to Object Y
- Your effective access to Object Y through Group X = **W** (the minimum)

If you're also in Group Z with **W** access, and Group Z has **O** (Owner) access to Object Y, your effective access through Group Z would be **W**. The system uses whichever gives you higher access.

### 3. Compare and Grant/Deny

If your effective access level is equal to or higher than the required level, access is granted. Otherwise, an `error_access_denied` (403) error is returned.

## The `access` Property in API Responses

When you query resources, the API response includes an `access` object that provides detailed information about your access rights to each returned object.

### Structure

```json
{
  "result": "success",
  "data": { ... },
  "access": {
    "object-id-here": {
      "required": "R",
      "available": "W",
      "expires": null,
      "user_group": "usg-xxxxx-xxxx-xxxx-xxxx-xxxxxxxx",
      "group": {
        "User_Group__": "usg-xxxxx-xxxx-xxxx-xxxx-xxxxxxxx",
        "Name": "My Team",
        "Type": "group",
        ...
      }
    }
  }
}
```

### Fields Explained

| Field | Description |
|-------|-------------|
| **required** | The access level that was required for this operation (e.g., "R" for a GET request) |
| **available** | Your actual/effective access level to this object |
| **expires** | Expiration date/time of your access, or `null` if permanent |
| **user_group** | The ID of the group through which you have access |
| **group** | Full details of the group (when expanded) |

### Understanding the Values

- If `available` >= `required`, the operation succeeded
- If `available` is higher than `required`, you have more permissions than needed
- A lowercase `r` in `available` indicates view-only access (public resource)
- The `user_group` field shows which of your groups provided this access

### Access Without Group Information

For public resources or link-based access, the response may show access without group details:

```json
"access": {
  "object-id": {
    "required": "R",
    "available": "r"
  }
}
```

This indicates you have view access to a public resource.

## Public Resources

Some resources are marked as **public** (or "open"). These resources:

- Can be accessed without authentication
- Do not require any group membership
- Show as `"available": "r"` (lowercase) in access responses
- Still respect higher access levels if you're authenticated and have group access

Public resources typically include:
- Reference data (countries, currencies, languages)
- Public catalog items
- Shared templates

## Sharing Objects via Links

Objects can be shared using special **share links** that grant temporary or permanent access:

### Link Types

| Type | Description |
|------|-------------|
| **user** | Temporary access link that expires |
| **permuser** | Permanent access link (no expiration) |
| **automatic** | System-generated link |
| **support** | Link for support access |

### Link Access Levels

Share links can grant the following access levels:
- **R** (Read)
- **C** (Create)
- **W** (Write)
- **A** (Admin)

Note: Links cannot grant **O** (Owner) or **D** (Delete) access.

### How Link Sharing Works

1. A user with sufficient access (typically **W** or higher) creates a share link
2. The link contains an encrypted token with the access level
3. When someone uses the link, they receive temporary access at the specified level
4. Link access is recorded in the session and combined with any group-based access
5. The higher of link access or group access is used

### Restrictions

- You cannot create a link with access equal to or higher than your own
- You need at least **W** access to create share links
- Link access can expire based on the link's configuration

## Parent-Child Access Inheritance

Objects can have parent-child relationships. When checking access:

1. The system first checks for direct access to the object
2. If no direct access exists, it checks the parent object
3. This continues up the hierarchy until access is found or the root is reached

This allows for hierarchical permission structures where access to a parent grants access to all children.

### Example

```
Folder A (you have W access)
  └── Document B (no direct access)
       └── Comment C (no direct access)
```

In this case, you would have **W** access to both Document B and Comment C through inheritance from Folder A.

---

## Quick Reference

### Access Level Summary

| Level | Can Read | Can Create | Can Write | Can Delete | Can Admin | Can Transfer |
|-------|----------|------------|-----------|------------|-----------|--------------|
| O     | Yes      | Yes        | Yes       | Yes        | Yes       | Yes          |
| A     | Yes      | Yes        | Yes       | Yes        | Yes       | No           |
| D     | Yes      | Yes        | Yes       | Yes        | No        | No           |
| W     | Yes      | Yes        | Yes       | No         | No        | No           |
| C     | Yes      | Yes        | No        | No         | No        | No           |
| R     | Yes      | No         | No        | No         | No        | No           |
| N     | No*      | No         | No        | No         | No        | No           |

*N (Notify) level receives notifications but cannot read full content.

### Common Error Responses

| Error Token | HTTP Code | Meaning |
|-------------|-----------|---------|
| `error_authentication_required` | 403 | Login required |
| `error_access_denied` | 403 | Insufficient permissions |
| `error_not_found` | 404 | Resource not found or not accessible |
