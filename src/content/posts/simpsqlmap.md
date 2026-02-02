---
title: 好靶场实战-SQLMap与数字型SQL注入
published: 2026-02-02
description: 'SQLMap快速破解简单注入题'
image: ''
tags: [SQL]
category: '好靶场实战'
draft: false 
lang: 'zh_CN'
---

# 前言
这是我在好靶场的一个简单练习 *[sql注入-数字型](http://www.loveli.com.cn/see_bug_one?id=106 "sql注入-数字型")* 推送的WP，现已过审并给我发放了奖励，我把教程归档到这里以供后续使用
除了可以使用MySQL的查询参数，还可以使用第三方工具SQLMap来一步步获取本题的flag，首先需要先安装SQLMap（推荐在Linux进行，Windows也可）
同时，**本教程以学习目的为主，可能只对本靶场有针对性，同时稍带一些扩展知识，仅供参考**
## 操作步骤
首先假设靶场的URL为`xxx.loveli.com.cn:8888`
启动靶场页面，在输入框输入1并搜索，然后把地址栏的URL复制下来
启动终端（假设环境为Linux且已安装SQLMap），执行以下命令：
`sqlmap -u "http://xxx.loveli.com.cn:8888/search?id=1" --batch` (此举是为了确认数据库类型，一般情况下靶场可以忽略这个步骤，batch参数可以自动选择默认选项，避免多余的交互)
有效输出内容如图：
[![1.png](https://img.simpmc.org/file/1770024862437_1.png)](http://mrlee.w1.luyouxia.net/files/uploads/1.png "此图可以看到数据库信息")
本图中，可以确定：
1. 数据库类型：MySQL，这意味着我们可以针对MySQL/MariaDB的特性来设计后续攻击或利用
2. 版本信息：大于等于5.0，这一点可以确定我们可以对本数据库使用哪些功能，本教程先不研究这个
3. MariaDB fork：靶场数据库运行的是MariaDB，但不影响后续操作，因为SQLMap已经支持MariaDB的语法

接下来执行：
`sqlmap -u "http://xxx.loveli.com.cn:8888/search?id=1" --dbs`（此举是为了枚举数据库）
有效输出内容截图：
[![2.png](https://img.simpmc.org/file/1770024914505_2.png)](http://mrlee.w1.luyouxia.net/files/uploads/2.png "这里有6个数据库")

这6个数据库中，有几个可以直接排除：**mysql, information_schema, performance_schema, sys**.因为这些数据库主要存放系统信息，基本不会有其他数据在里面
那么剩下2个数据库：**mydb,sql_injection_lab**,如果是第一次做，也不能确定哪个包含我们想要的flag
我也以第一次做的角度，先试试最上面的数据库: mydb
执行以下命令：
 `sqlmap -u "http://xxx.loveli.com.cn:8888/search?id=1" -D mydb --table`（此举是为了查看mydb数据库的表）
 有效内容截图如下：
[![3.png](https://img.simpmc.org/file/1770024930301_3.png)](http://mrlee.w1.luyouxia.net/files/uploads/3.png "报错了，因为找不到表")
 呃呃呃，看上去mydb的表内容是空的，那就可以直接更换目标了
` sqlmap -u "http://xxx.loveli.com.cn:8888/search?id=1" -D sql_injection_lab --tables`
 有效内容截图：
[![4.png](https://img.simpmc.org/file/1770024954313_4.png)](http://mrlee.w1.luyouxia.net/files/uploads/4.png "找到了flag")
不错，找到我们要找的flag了，游戏要过关喽！
`sqlmap -u "http://xxx.loveli.com.cn:8888/search?id=1" -D sql_injection_lab -T flag --dump`
最终结果如图：
[![6.png](https://img.simpmc.org/file/1770024986771_6.png)](http://mrlee.w1.luyouxia.net/files/uploads/6.png "已获取到flag")
看！flag内容出现了！（我在后面几个字打码了）
而且--dump参数还能把该数据导出为csv文件，可以用任意表格软件打开，例如WPS:
[![7.png](https://img.simpmc.org/file/1770025002437_7.png)](http://mrlee.w1.luyouxia.net/files/uploads/7.png "WPS读取SQLMap的导出文件")
SQLMap就这样，省去了我们苦苦查参数的时间
## 扩展（坑）
`sqlmap -u "http://xxx.loveli.com.cn:8888/search?id=1" -D sql_injection_lab -T flag --columns`
有效内容截图：
[![5.png](https://img.simpmc.org/file/1770024972597_5.png)](http://mrlee.w1.luyouxia.net/files/uploads/5.png "并没有出现flag的具体值")
这样是看不到具体内容的，--columns的作用是枚举表结构，即告诉你某个表有哪些字段、字段名和字段类型
--dump的作用才是导出表的实际数据
