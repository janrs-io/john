---
title: "RBAC是什么？怎么实现RBAC ?附表结构"
author: "John Yang"
lang: zh
pubDatetime: 2021-04-02T16:04:00Z
tags:
  - Security
  - Architecture
  - RBAC
description: "role管理员表就是用来保存该系统中需要有的管理员。比如用户管理员、财务管理员、运维人员等等 role角色表就是用来保存该系统中需要用到的角色。比如用户管理角色、财务管理角色、运维角色 permission权限表就是用来保存整套系统中表示权限请求的入口。在常见的开发中就是API接口或者URI地址。就是用..."
---

## RBAC在现在的系统中依然是常用权限控制，有必要掌握

### **什么是RBAC**
#### RBAC。英文名称Role-Based Access Control。顾名思义，就是基于角色的权限控制系统。理解的基本思路就是，在角色的前提下，按照角色不同将权限分配到对应的角色，再将管理员设置成对应的角色，进而管理员获得权限。在管理员请求某个业务权限的时候，从数据库中查询该管理员所拥有的权限，并将请求的业务权限跟数据库比对。以此进行权限控制

### **实现思路**
#### RBAC通常是通过数据库来实现的。也就是说将权限的数据保存在数据库中，然后在请求时拿出来进行比对
#### 在设计RBAC的时候有以下几张权限表：
#### **`admin`** 管理员表
`role`**管理员表就是用来保存该系统中需要有的管理员。比如用户管理员、财务管理员、运维人员等等**
####  **`role`** 角色表
`role`**角色表就是用来保存该系统中需要用到的角色。比如用户管理角色、财务管理角色、运维角色**
#### **`permission`** 权限表
`permission`**权限表就是用来保存整套系统中表示权限请求的入口。在常见的开发中就是API接口或者URI地址。就是用来保存整套系统中对外暴露业务操作的接口**
#### **`role_permission`** 角色权限关联表
`role_permission`**角色关联表就是用来保存各个不同角色所拥有的权限**
#### **`role_admin`** 角色管理员关联表
`role_admin`**角色管理员关联表就是用来保存管理员所属哪个角色的**
#### **`menu`** 菜单表
`menu` **这个菜单表其实从RBAC的角度出发，并不需要这个表。这个表只是从业务的层面的来表示的，并不是真正意义上权限的逻辑控制。很多时候很多开发人员把`menu`表等同于`role`表，从RBAC的角度来理解其实并不正确**

### **表结构**
#### `admin`管理员表结构
```sql
#code start

CREATE TABLE `backend_admin` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '自增id',
  `username` varchar(255) DEFAULT '' COMMENT '用户名',
  `password` varchar(255) DEFAULT '' COMMENT '密码',
  `register_time` int(11) DEFAULT '0' COMMENT '注册时间',
  `update_time` int(11) DEFAULT '0' COMMENT '更新时间',
  `status` tinyint(1) DEFAULT '1' COMMENT '账户状态0=禁用1=正常',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='管理员表';

#code end
```
#### `role`角色表结构
```sql
#code start

CREATE TABLE `rbac_role` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '自增id',
  `role_name` varchar(255) DEFAULT '' COMMENT '角色名称',
  `status` tinyint(1) DEFAULT '1' COMMENT '角色状态0=禁用1=启用',
  `create_time` int(11) DEFAULT '0' COMMENT '添加时间',
  `update_time` int(11) DEFAULT '0' COMMENT '更新时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='角色表';

#code end
```
#### `permission`权限表结构
```sql
#code start

CREATE TABLE `rbac_permission` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '自增id',
  `permission_name` varchar(255) DEFAULT '' COMMENT '权限名称',
  `route_controller` varchar(255) DEFAULT '' COMMENT '控制器',
  `route_action` varchar(255) DEFAULT '' COMMENT '执行动作',
  `parent_id` int(11) DEFAULT '0' COMMENT '父级id',
  `status` tinyint(1) NOT NULL DEFAULT '1' COMMENT '状态0=禁用1=启用',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='权限表';

#code end
```
#### `role_permission`角色权限关联表结构
```sql
#code start

CREATE TABLE `rbac_permission_role` (
  `permission_id` int(11) DEFAULT '0' COMMENT '权限id',
  `role_id` int(11) DEFAULT '0' COMMENT '角色id'
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='角色权限关联表';

#code end
```
#### `role_admin`角色管理员关联表结构
```sql
#code start

CREATE TABLE `rbac_admin_role` (
  `role_id` int(11) DEFAULT '0' COMMENT '角色id',
  `admin_id` int(11) DEFAULT '0' COMMENT '管理员id'
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='角色管理员关联表';

#code end
```
#### `menu`菜单表结构
```sql
#code start

CREATE TABLE `rbac_menu` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '自增id',
  `menu_name` varchar(255) DEFAULT '' COMMENT '菜单名称',
  `visible` tinyint(1) DEFAULT '1' COMMENT '是否可见0=不可见1=可见',
  `parent_id` int(11) DEFAULT '0' COMMENT '父级id',
  `status` tinyint(1) DEFAULT '1' COMMENT '状态0=可用1=不可用',
  `admin_id` int(11) NOT NULL DEFAULT '0' COMMENT '管理员id',
  `sort` int(11) NOT NULL DEFAULT '0' COMMENT '排序',
  `permission_id` int(11) DEFAULT '0' COMMENT '权限id',
  `icon` varchar(255) DEFAULT '' COMMENT '图标',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='菜单表';

#code end
```

### **实际开发怎么实现**

#### 而在常规的后台开发中，权限其实对应的就是业务的控制权，再细化下去就是业务开发中，对应实现业务的代码。比如MVC模式中，就是对应的每一个Controller的Action，因为Action实现的就是暴露给外界的控制入口。那么实现过程就是，将这些实现业务的每一个代码接口【控制入口】列出来，每一个接口就是一个**`permission`**，将这些接口保存到**`permission`**表；在**`admin`**表中设置一个默认的管理员。比如：**`super_admin`**；在**`role`**表个中设置一个拥有所有权限的角色。比如：**`super_role`**；在**`role_admin`**表中添加角色管理员关联的数据；在**`role_permission`**表中添加对应角色拥有的权限。那么在管理员登陆后要对后台的某个业务进行操作时，获取该操作的业务接口；在**`role_admin`**中查询获取该管理员对应的角色；在**`role_permission`**表中查询对应角色拥有的权限；最后将请求的业务接口权限是否存在查询的数据中，比对一下就可以得知该管理员是否有操作权限

### **在PHP项目中怎么实现**
#### 在这里只大概的说一下在php项目中怎么实现。主要有几个知识概念要掌握，一个是命名空间；一个是php的反射机制。具体怎么实现的，在其他博文中再做详细举例
