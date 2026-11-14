---
title: "What Is RBAC and How to Implement It (with MySQL Schema)"
author: "John Yang"
lang: en
pubDatetime: 2021-04-02T16:04:00Z
tags:
  - Security
  - Architecture
  - RBAC
description: "An introduction to Role-Based Access Control (RBAC), core table relationships (admin, role, permission, and menu), MySQL DDL schemas, and enforcement flow."
---

Role-Based Access Control (RBAC) remains one of the most widely used authorization models in modern backend systems.

## What Is RBAC?

**RBAC** stands for **Role-Based Access Control**. Instead of assigning permissions directly to individual users, permissions are grouped into **roles**, and administrators (users) are assigned to one or more roles.

When an administrator makes a request to perform a business action, the backend queries the permissions associated with their assigned roles from the database and checks whether the requested operation is permitted.

## Core Design & Tables

RBAC is typically backed by relational database tables:

- **`admin` (Administrators Table)**: Stores system administrator accounts (e.g., user manager, finance admin, operations engineer).
- **`role` (Roles Table)**: Defines roles used in the system (e.g., User Management, Finance, Ops).
- **`permission` (Permissions Table)**: Stores every protected action or entry point in the system—typically API endpoints, URIs, or controller/action pairs.
- **`role_permission` (Role-Permission Mapping)**: Maps which permissions belong to each role.
- **`role_admin` (Admin-Role Mapping)**: Maps which roles are assigned to each administrator.
- **`menu` (Navigation Menu Table)**: Strictly speaking, RBAC does not require a `menu` table—it is a UI/business presentation concern rather than core authorization logic. Many developers conflate `menu` with `role`, which blurs the boundary between UI visibility and actual access control.

## MySQL Table Schemas

### `backend_admin`

```sql
CREATE TABLE `backend_admin` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT 'Auto-increment ID',
  `username` varchar(255) DEFAULT '' COMMENT 'Username',
  `password` varchar(255) DEFAULT '' COMMENT 'Password hash',
  `register_time` int(11) DEFAULT '0' COMMENT 'Registration timestamp',
  `update_time` int(11) DEFAULT '0' COMMENT 'Last updated timestamp',
  `status` tinyint(1) DEFAULT '1' COMMENT 'Account status: 0=disabled, 1=active',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='Administrators';
```

### `rbac_role`

```sql
CREATE TABLE `rbac_role` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT 'Auto-increment ID',
  `role_name` varchar(255) DEFAULT '' COMMENT 'Role name',
  `status` tinyint(1) DEFAULT '1' COMMENT 'Role status: 0=disabled, 1=enabled',
  `create_time` int(11) DEFAULT '0' COMMENT 'Created timestamp',
  `update_time` int(11) DEFAULT '0' COMMENT 'Last updated timestamp',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='Roles';
```

### `rbac_permission`

```sql
CREATE TABLE `rbac_permission` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT 'Auto-increment ID',
  `permission_name` varchar(255) DEFAULT '' COMMENT 'Permission name',
  `route_controller` varchar(255) DEFAULT '' COMMENT 'Controller',
  `route_action` varchar(255) DEFAULT '' COMMENT 'Action',
  `parent_id` int(11) DEFAULT '0' COMMENT 'Parent permission ID',
  `status` tinyint(1) NOT NULL DEFAULT '1' COMMENT 'Status: 0=disabled, 1=enabled',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='Permissions';
```

### `rbac_permission_role`

```sql
CREATE TABLE `rbac_permission_role` (
  `permission_id` int(11) DEFAULT '0' COMMENT 'Permission ID',
  `role_id` int(11) DEFAULT '0' COMMENT 'Role ID'
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='Role-permission mapping';
```

### `rbac_admin_role`

```sql
CREATE TABLE `rbac_admin_role` (
  `role_id` int(11) DEFAULT '0' COMMENT 'Role ID',
  `admin_id` int(11) DEFAULT '0' COMMENT 'Admin ID'
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='Admin-role mapping';
```

### `rbac_menu`

```sql
CREATE TABLE `rbac_menu` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT 'Auto-increment ID',
  `menu_name` varchar(255) DEFAULT '' COMMENT 'Menu label',
  `visible` tinyint(1) DEFAULT '1' COMMENT 'Visibility: 0=hidden, 1=visible',
  `parent_id` int(11) DEFAULT '0' COMMENT 'Parent menu ID',
  `status` tinyint(1) DEFAULT '1' COMMENT 'Status: 0=enabled, 1=disabled',
  `admin_id` int(11) NOT NULL DEFAULT '0' COMMENT 'Admin ID',
  `sort` int(11) NOT NULL DEFAULT '0' COMMENT 'Sort order',
  `permission_id` int(11) DEFAULT '0' COMMENT 'Linked permission ID',
  `icon` varchar(255) DEFAULT '' COMMENT 'Icon identifier',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='UI menus';
```

## Runtime Enforcement Flow

In a typical MVC backend, each exposed operation maps to a `Controller` and `Action`:

1. Enumerate every protected controller/action endpoint and store it in `rbac_permission`.
2. Seed a default administrator (e.g., `super_admin`) in `backend_admin` and a full-access role (e.g., `super_role`) in `rbac_role`.
3. Link the administrator to their role(s) in `rbac_admin_role`, and grant the corresponding permissions in `rbac_permission_role`.
4. On each authenticated request, resolve the target `Controller`/`Action`, look up the admin's roles in `rbac_admin_role`, fetch the granted permissions via `rbac_permission_role`, and check whether the requested endpoint is in the granted set.

## Note on PHP Implementations

In PHP frameworks, automatic permission discovery and route-based enforcement commonly rely on **namespaces** and **PHP Reflection** to scan controller classes and methods cleanly.
