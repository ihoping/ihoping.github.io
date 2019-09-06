---
title: php代码规范和项目规范
date: 2019-09-06 17:33:40
tags:
---

最近总结了一下php的项目代码规范：

0. 代码规范遵循PSR
1. model里对某个表进行查询时只调用一个方法，且这个方法唯一（使用特殊前缀?），所有对数据的操作都建立在该方法之上。
2. 需要调用外部接口时需经过API层。
3. 不能跨级调用，例如可以controller->model->dao，而不能直接controller->dao
3. 不能在Components或者Dao层里直接查询数据库。
4. 访问数据库需要经过model。
<!-- more -->  
5. 使用ORM查询数据库，结果需要返回数组，多条数据toArray()，一条数据toArray()[0]，尽量不要使用first()->toArray()。
6. dao层按模块分目录。
7. 配置都放Common里？公用。
8. 事务使用transaction回调的方法代替手动begin和commit。
9. 定时脚本需要try catch，异常后发邮件到负责人。
10. 重要的地方try catch异常后记日志。
11.	除了临时脚本handleController外，控制器不能直接调用DAO层，
12.	参数验证放在控制器里，业务验证放在model里？
13.	和业务无关的公用方法放在FUnctils里
14.	避免N+1，优化N+1
15.	使用强等于
16.	尽量少用联表查询
17.	小心ORM，它会记住参数
18.	函数尽量单一职责，不用考虑多次查询数据库的问题
19.	能使用array_map、array_filter、array_reduce、array_walk等函数来处理数组或构建数据就不要使用foreach
