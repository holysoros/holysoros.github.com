---
layout: post
title: "ActiveRecord 如何高效地获取随机 records"
description: ""
category:
tags: [rails activerecord db]
---

ActiveRecord 并没有直接提供随机获取的接口，有以下几种方法可以实现。

初级
------
```ruby
Model.all.sample(n)
```

返回 Model 的所有 records ，浪费带宽，浪费内存，效率奇差，无节操。

进阶
------

```ruby
ids = Model.pluck(:id).sample(n)
Model.where(id: ids)
```

先返回 Model 的所有 records 的 id ，然后随机选择 n 个，再次用 where 请求数据；效率不错，很有节操了。

装逼
------
数据库本身通常都提供 random 的语句，结合 ActiveRecord 的 order 可以用于获取随机 records ：

```ruby
# mysql
Model.order("RAND()").first(n)
# PostgreSQL and sqlite
Model.order("RANDOM()").first(n)
```

看起来很牛逼的样子，只需要因此 DB query ，但是违背了 ActiveRecord 的 database-agnostic 原则，而且数据库本身的 RANDOM 实现的效率并不高。

### 装逼也要有境界
尽管数据库的 random 语句各有不同，但 [randumb-Adds ability to pull back random records from Active Record](https://github.com/spilliton/randumb) 封装了不同数据库的 random ，如果感兴趣，可以尝试使用这个库。

如果只随机选择一个 record
------
如果只想随机选择一个 record ，这种方法也是一个可选项：

```ruby
offset = rand(Model.count)
Model.offset(offset).first
```

结论
------
因此，看起来进阶的方法是最靠谱的：

```ruby
ids = Model.pluck(:id).sample(n)
Model.where(id: ids)
```

达到了各方面平衡。

References
------
- [Random record in ActiveRecord on SO](http://stackoverflow.com/questions/2752231/random-record-in-activerecord)
- [Efficiently Getting Random Records in Active Record](http://easyactiverecord.com/blog/2014/03/27/efficiently-getting-random-records-in-active-record/)
- [Easily select random records in rails](http://thinkingeek.com/2011/07/04/easily-select-random-records-rails/)
