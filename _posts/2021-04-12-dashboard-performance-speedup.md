---
layout: post
title:  "记录一下 dashboard 性能优化 (10s->1ms)"
date:   2021-04-12 20:00:00 +0800
---

[Hackershare](https://hackershare.dev/) 后台的一个Dashboard页面，由于很多统计类的查询，响应越来越慢，差不多要十几秒打开。主要是有两个50w左右的数据表，count非常慢，还有一部分原因就是这台2c4g的服务器部署了很多程序，CPU经常被其他服务占用。

![](https://l.ruby-china.com/photo/hooopo/b769b2c6-c102-481e-9dea-0ec4908b2ff5.png!large)

大概的数据量：

```txt
hackershare=# \dt+
                              List of relations
 Schema |            Name            | Type  | Owner  |  Size   | Description
--------+----------------------------+-------+--------+---------+-------------
 public | bookmarks                  | table | deploy | 1259 MB |
 public | clicks                     | table | deploy | 4360 kB |
 public | comments                   | table | deploy | 134 MB  |
 public | follows                    | table | deploy | 48 kB   |
 public | likes                      | table | deploy | 88 kB   |
 public | rss_sources                | table | deploy | 1280 kB |
 public | tag_subscriptions          | table | deploy | 88 kB   |
 public | taggings                   | table | deploy | 117 MB  |
 public | tags                       | table | deploy | 2200 kB |
 public | users                      | table | deploy | 2848 kB |
```

第一招，使用 union all，把多条count合并成一条语句：

```sql
select 'bookmark' as key, count(*) as count from bookmarks

UNION ALL

select 'comment' as key, count(*) as count from comments

UNION ALL

select 'click' as key, count(*) as count from clicks

UNION ALL 

others

```

返回结构大概这样：

```
   key    | count
----------+--------
 click    |  65103
 comment  | 421423
 bookmark | 465078
```

比之前的有提升，但效果不大...

第二招，使用explain

```ruby
# usage
# FastCount.new(User.all).call
# => 826
# FastCount.new(User.where("id > 200")).call
# => 665

class FastCount
  attr_reader :scope, :sql

  def initialize(scope)
    @scope = scope
    @sql = scope.to_sql
  end

  def call
    explain_sql = "explain (format json) #{sql}"
    result = ApplicationRecord.connection.execute(explain_sql)[0]["QUERY PLAN"]
    json = JSON.parse(result)
    json[0]["Plan"]["Plan Rows"].to_i
  end
end

```

看看效果：

```
explain (format json)  select * from bookmarks;
             QUERY PLAN
-------------------------------------
 [                                  +
   {                                +
     "Plan": {                      +
       "Node Type": "Seq Scan",     +
       "Parallel Aware": false,     +
       "Relation Name": "bookmarks",+
       "Alias": "bookmarks",        +
       "Startup Cost": 0.00,        +
       "Total Cost": 89629.30,      +
       "Plan Rows": 464730,         +
       "Plan Width": 1278           +
     }                              +
   }                                +
 ]
(1 row)

Time: 0.898 ms
```

不到1毫秒！！

另外，居然可以支持带过滤条件甚至带JOIN语句的count，比如：

```ruby
FastCount.new(User.where("id > 200")).call
```

* 适用场景：分页和dashboard之类不需要数据绝对准确，但对性能还有一些要求的场景。
* 缺点：并不能保证数据绝对准确，取决于你的auto vacuum设置，一般情况下如果你的表记录足够大，并且更新频繁，使用这种方案几乎误差范围都是很小的。

Full Code: https://github.com/hackershare/hackershare/pull/115/files



