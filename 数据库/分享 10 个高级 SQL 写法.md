> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7209625823580766264)

*   本文所讲述 sql 语法都是基于 MySql 8.0
*   博主github地址：[github.com/wayn111 ](https://link.juejin.cn?target=http%3A%2F%2Fgithub.com%2Fwayn111%25C2%25A0 "http://github.com/wayn111%C2%A0") 欢迎大家关注，点个star

一、ORDER BY FIELD() 自定义排序逻辑
==========================

MySql 中的排序 ORDER BY 除了可以用 ASC 和 DESC，还可以通过 **ORDER BY FIELD(str,str1,...)** 自定义字符串/数字来实现排序。这里用 order_diy 表举例，结构以及表数据展示：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b59c27027de846adbe21dbe4715b1ee7~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) ORDER BY FIELD(str,str1,...) 自定义排序sql如下：

```
SELECT * from order_diy ORDER BY FIELD(title,'九阴真经', 
'降龙十八掌','九阴白骨爪','双手互博','桃花岛主',
'全真内功心法','蛤蟆功','销魂掌','灵白山少主');
复制代码
```

查询结果如下：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e0295cd52f7d420a899f874c83effb16~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) 如上，我们设置自定义排序字段为 title 字段，然后将我们自定义的排序结果跟在 title 后面。

二、CASE 表达式
==========

**case when then else end**表达式功能非常强大可以帮助我们解决 `if elseif else` 这种问题，这里继续用 order_diy 表举例，假如我们想在 order_diy 表加一列 level 列，根据money 判断大于60就是高级，大于30就是中级，其余显示低级，sql 如下：

```
SELECT *, 
case when money > 60 then '高级' 
when money > 30 then '中级' 
else '低级' END level 
from order_diy;
复制代码
```

查询结果：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6f74b300c02b4e3990e81ae1e0d98a63~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

> 需要注意的就是 **case when then** 语句不匹配如果没有写 **else end** 会返回 null，影响数据展示。

三、EXISTS 用法
===========

我猜大家在日常开发中，应该都对关键词 exists 用的比较少，估计使用 in 查询偏多。这里给大家介绍一下 exists 用法，引用官网文档:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d5291eaf2f67430d8bb78e9f9a7fe539~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) 可知 exists 后面是跟着一个子查询语句，它的作用是**根据主查询的数据，每一行都放到子查询中做条件验证，根据验证结果（TRUE 或者 FALSE），TRUE的话该行数据就会保留**，下面用 emp 表和 dept 表进行举例，表结构以及数据展示：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1be83b3e5d58457a81339b2b79cd2737~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) 计入我们现在想找到 emp 表中 dept_name 与 dept表 中 dept_name 对应不上员工数据，sql 如下：

```
SELECT * from emp e where exists (
SELECT * from dept p where e.dept_id = p.dept_id 
and e.dept_name != p.dept_name
)
复制代码
```

查询结果：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/63c5041d36144859803cec2d330ceb3c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) 我们通过 exists 语法将外层 emp 表全部数据 放到子查询中与一一与 dept 表全部数据进行比较，只要有一行记录返回true。画个图展示主查询所有记录与子查询交互如下：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8f5dc9c24781431d8f5225893a7deb5c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

*   第一条记录与子查询比较时，全部返回 false，所以第一行不展示。
*   第二行记录与子查询比较时，发现 `销售部门` 与 dept 表第二行 `销售部` 对应不上，返回 true，所以主查询该行记录会返回。
*   第二行以后记录执行结果同第一条。

四、GROUP_CONCAT(expr) 组连接函数
==========================

**GROUP_CONCAT(expr)** 组连接函数可以返回分组后指定字段的字符串连接形式，并且可以指定排序逻辑，以及连接字符串，默认为英文逗号连接。这里继续用 order_diy 表举例：sql 如下：

```
SELECT name, GROUP_CONCAT(title ORDER BY id desc  SEPARATOR '-') 
from order_diy GROUP BY name ORDER BY NULL;
复制代码
```

查询结果：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5b843a7302af41108bd87a08b900e686~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

如上我们通过 **GROUP_CONCAT(title ORDER BY id desc SEPARATOR '-')** 语句，指定分组连接 title 字段并按照 id 排序，设置连接字符串为 `-`。

五、自连接查询
=======

自连接查询是 sql 语法里常用的一种写法，掌握了自连接的用法我们可以在 sql 层面轻松解决很多问题。这里用 tree 表举例，结构以及表数据展示：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3661774b6e904dcc931d128bd9cbcdcc~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) tree 表中通过 pid 字段与 id 字段进行父子关联，假如现在有一个需求，我们想按照父子层级将 tree 表数据转换成 `一级职位 二级职位 三级职位` 三个列名进行展示，sql 如下：

```
SELECT t1.job_name '一级职位', t2.job_name '二级职位', t3.job_name '三级职位' 
from tree t1 join tree t2 on t1.id = t2.pid left join tree t3 on t2.id = t3.pid 
where t1.pid = 0;
复制代码
```

结果如下：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/172447faa2e24abeaba9bc65af0f6f1f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) 我们通过 **tree t1 join tree t2 on t1.id = t2.pid** 自连接展示 `一级职位 二级职位`，再用 **left join tree t3 on t2.id = t3.pid** 自连接展示 `二级职位 三级职位`，最后通过**where 条件 t1.pid = 0**过滤掉非一级职位的展示，完成这个需求。

六、更新 emp 表和 dept 表关联数据
======================

这里继续使用上文提到的 emp 表和 dept 表，数据如下： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b46cf0ce241940c893fe9a95cce90e99~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) 可以看到上述 emp 表中 jack 的部门名称与 dept 表实际不符合，现在我们想将 jack 的部门名称更新成 dept 表的正确数据，sql 如下：

```
update emp, dept set emp.dept_name = dept.dept_name
where emp.dept_id = dept.dept_id;
复制代码
```

查询结果： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8166d1e27a8741ddbd30a348279c2a6b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

我们可以直接关联 emp 表和 dept 表并设置关联条件，然后更新 emp 表的 dept_name 为 dept 表的 dept_name。

七、ORDER BY 空值 NULL 排序
=====================

ORDER BY 字句中可以跟我们要排序的字段名称，但是当字段中存在 null 值时，会对我们的排序结果造成影响。我们可以通过 **ORDER BY IF(ISNULL(title), 1, 0)** 语法将 null 值转换成0或1，来达到将 null 值放到前面还是后面进行排序的效果。这里继续用 order_diy 表举例，sql 如下：

```
SELECT * FROM order_diy ORDER BY  IF(ISNULL(title), 0, 1), money;
复制代码
```

查询结果：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6dc5041031984649a2e1486ecd846588~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

八、with rollup 分组统计数据的基础上再进行统计汇总
===============================

MySql 中可以使用 with rollup 在分组统计数据的基础上再进行统计汇总，即用来得到 group by 的汇总信息。这里继续用order_diy 表举例，sql 如下：

```
SELECT name, SUM(money) as money 
FROM order_diy GROUP BY name WITH ROLLUP;
复制代码
```

查询结果：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b2a664e19d704d498ba6ae8ed4ff849b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) 可以看到通过 **GROUP BY name WITH ROLLUP** 语句，查询结果最后一列显示了分组统计的汇总结果。但是 name 字段最后显示为 null，我们可以通过 `coalesce(val1, val2, ...)` 函数，这个函数会返回参数列表中的第一个非空参数。

```
SELECT coalesce(name, '总金额') name, SUM(money) as money 
FROM order_diy GROUP BY name WITH ROLLUP;
复制代码
```

查询结果：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8dde6116f2304609b3f9475db7bd8036~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

九、with as 提取临时表别名
=================

with as 语法需要 MySql 8.0以上版本，它有一个别名叫做 CTE，官方对它的说明如下

> 公用表表达式 (CTE) 是一个命名的临时结果集，它存在于单个语句的范围内，稍后可以在该语句中引用，可以多次引用。

它的作用主要是提取子查询，方便后续共用，更多情况下会用在数据分析的场景上。

如果一整句查询中**多个子查询都需要使用同一个子查询**的结果，那么就可以用 with as，将共用的子查询提取出来，加个别名。后面查询语句可以直接用，对于大量复杂的SQL语句起到了很好的优化作用。这里继续用 order_diy 表举例，这里使用 with as 给出 sql 如下：

```
-- 使用 with as
with t1 as (SELECT * from order_diy where money > 30),
t2 as (SELECT * from order_diy where money > 60)
SELECT * from t1 
where t1.id not in (SELECT id from  t2) and t1.name = '周伯通';
复制代码
```

查询结果：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0370dce17d184d66bee5a0ce16a11c2e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) 这个 sql 查询了 order_diy 表中 money 大于30且小于等于60之间并且 name 是周伯通的记录。可以看到使用 CTE 语法后，sql写起来会简洁很多。

10、存在就更新，不存在就插入
===============

MySql 中通过**on duplicate key update**语法来实现存在就更新，不存在就插入的逻辑。插入或者更新时，它会根据表中主键索引或者唯一索引进行判断，如果主键索引或者唯一索引有冲突，就会执行**on duplicate key update**后面的赋值语句。 这里通过 news 表举例，表结构和说数据展示，其中 news_code 字段有唯一索引：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/75cbb094c7274346ad2e7b61c0986452~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) 添加sql：

```
-- 第一次执行添加语句
INSERT INTO `news` (`news_title`, `news_auth`, `news_code`) 
VALUES ('新闻3', '小花', 'wx-0003') 
on duplicate key update news_title = '新闻3';
-- 第二次执行修改语句
INSERT INTO `news` (`news_title`, `news_auth`, `news_code`) 
VALUES ('新闻4', '小花', 'wx-0003') 
on duplicate key update news_title = '新闻4';
复制代码
```

结果如下：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/77ede005a5f34b369843f1ebb45b4761~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

总结
==

到这里，本文所分享的10个高级sql写法就全部介绍完了，希望对大家日常开发 sql 编写有所帮助，喜欢的朋友们可以点赞加关注😘。

> 公众号【waynblog】每周更新博主最新技术文章，欢迎大家关注。
