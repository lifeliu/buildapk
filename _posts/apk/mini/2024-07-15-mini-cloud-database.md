---
layout: post
title: 微信小程序云数据库详解
categories: mini
tags: [小程序, 云数据库]
date: 2024/7/15 10:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_001.jpg)

## 引言

云数据库基于 MongoDB，采用文档型（JSON）存储，无需预先定义表结构。可在云开发控制台创建集合、设置索引和权限，在小程序端或云函数中通过 API 操作。支持 where 条件查询、orderBy 排序、limit 分页、aggregate 聚合等，满足大部分 CRUD 和统计需求。

与关系型数据库不同，云数据库的「表」对应「集合」，「行」对应「文档」。每个文档有唯一的 _id，可嵌套对象和数组。本文介绍云数据库的增删改查、权限配置与聚合用法。

## 增删改查

云数据库 API 采用链式调用，语义清晰。`db.serverDate()` 可插入服务端时间，避免客户端时间不准。`db.command.inc(1)` 实现原子自增，适合阅读量、点赞数等场景。注意：小程序端受权限限制，只能操作当前用户有权限的文档；云函数可绕过权限，以管理员身份操作。

```javascript
const db = wx.cloud.database();

// 新增
db.collection('articles').add({
  data: { title: '标题', content: '内容', createTime: db.serverDate() }
});

// 查询
db.collection('articles').where({ status: 1 })
  .orderBy('createTime', 'desc')
  .limit(10)
  .get();

// 更新
db.collection('articles').doc(id).update({
  data: { viewCount: db.command.inc(1) }
});

// 删除
db.collection('articles').doc(id).remove();
```

## 权限配置

在云开发控制台的「数据库」中，可为每个集合设置权限规则。常见规则包括：仅创建者可读写、所有用户可读仅创建者可写、仅管理端可写等。权限基于 openid 判断，适用于「用户只能改自己的数据」等场景。敏感数据或复杂业务逻辑建议在云函数中操作，云函数以管理员身份执行，不受权限限制。

## 聚合与统计

聚合管道（aggregate）支持 match、group、sort、limit 等阶段，可完成分组统计、多表关联等复杂查询。例如统计订单总金额、按日期分组计数等。聚合在云端执行，不占用小程序端资源，适合数据量较大的场景。

```javascript
db.collection('orders').aggregate()
  .match({ status: 'paid' })
  .group({ _id: null, total: db.command.sum('amount') })
  .end();
```

## 总结

云数据库提供灵活的文档存储与查询能力。合理设计集合结构（避免过深嵌套）、配置权限、为常用查询字段建立索引，可支撑大部分业务场景。注意单次查询默认最多返回 20 条，需分页时使用 limit + skip 或基于上次查询结果的分页方式。
