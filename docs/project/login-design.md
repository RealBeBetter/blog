---
title: 【项目】多种方式统一登录入口的设计方案
date: 2022-01-29 20:28:37
tags:
- Project
- Database
---

在编写项目的时候，通常会遇到很多情况下，需要实现统一登录入口。统一登录入口通常指的是：能够实现手机号、邮箱、用户名等信息登录，且共用一个登录入口。

这种登录方式现在属于一种主流的登录方式，除此之外，移动端通常还有本机号码一键登录。

## 登录账号鉴别

鉴于各种登录账号（指的是手机号、邮箱、用户名等可以唯一鉴别用户的信息，以下统称登陆账号）的组成不一样，我们可以在后端对数据进行区分。

我们在后台对用户的数据进行记录，一般要使用一个数据表来进行存储。表结构类似这种`id|name|phone|email|nick_name|desc` 。那么我们在规划登录的时候，设置了手机号和邮件均可登录，且要求统一登录入口，该如何进行设计呢？

在登录的时候，我们需要根据用户的输入来判断用户登录使用的是什么登录账号。如果使用的是手机号，我们可以根据用户的输入（国内用户）判断是否为电话号码（数字/长度）；如果使用的是邮箱，可以用特殊标识（`@/.`）来判断。但是这类的判断方法存在缺陷，如果还存在可以使用用户名进行登录的方式，需要进行各种验证，逻辑比较复杂，最终可能还需要进行登录字段的遍历操作。

如果使用初始字段进行登录设计，后续如果添加了一种登录方式，需要对数据库进行字段添加操作，修改困难，维护不便。所以，直接使用用户表进行统一登录功能的实现，并不灵活。

1. 判断登录方式逻辑复杂
2. 后续扩展便捷性差

## 数据库字段鉴别

在数据库的设计上，**加记录比加字段更容易扩展**。我们可以在登录的时候，创建一张专门用于维护登录的表，命名为 login_ticket 。在字段的设计上，我们可以记录：

| 字段名     | 类型         | 备注                                   |
| ---------- | ------------ | -------------------------------------- |
| id         | int/bigint   | 表格数据段唯一标识                     |
| user_id    | int/bigint   | 用户iD，关联用户表用户ID               |
| user_name  | varchar(255) | 用户名，用于登录时显示                 |
| login_type | int          | 登录方式，规定每一个值代表何种登录方式 |
| password   | varchar(255) | 用户密码，如果第三方登录则记录 token   |

之后，我们就可以在 user 表中只存储和登录无关的字段信息了。例如如下设计：

| 字段名     | 类型         | 备注               |
| ---------- | ------------ | ------------------ |
| id         | int/bigint   | 用户唯一标识       |
| nick_name  | varchar(255) | 用户昵称           |
| avatar_url | varchar(255) | 用户头像路径       |
| user_names | varchar(255) | 用户的所有登录方式 |

这样，我们就得到容易扩展的记录登录方式的数据表了。如果需要增加登录方式，只需要增加 login_ticket 表格中的数据即可。

例如：

- 在 user 表中，我们存储的数据为：

  | id   | nick_name | avatar_url        | user_names |
  | ---- | --------- | ----------------- | ---------- |
  | 1    | Real      | /avartar/Real.jpg |            |

- 在 login_ticket 表中，我们存储的数据为：

  | id    | user_id | user_name(登录账户) | login_type  | password |
  | ----- | ------- | ------------------- | ----------- | -------- |
  | 10001 | 1       | `12345678910`       | 0（邮箱）   | 123456   |
  | 10002 | 1       | `xxx@qq.com`        | 1（手机号） | 123456   |

有了这样的实际数据之后，我们如果新增登录方式，也可以通过添加字段来直接实现。但是一个 user 是存在多个字段的，那么修改密码，是需要将所有 user_id 相同的字段都修改密码的。

所以在具体的项目中，需要优先使用第二种方式来实现多种方式登录操作。如果项目数据比较大，我们还可以进行数据库的分库分表操作来完成数据库的访问优化。
