
## 前言


最近在我的知识星球中，有个小伙伴问了这样一个问题：百万商品分页查询接口，如何保证接口的性能？


这就需要对该分页查询接口做优化了。


这篇文章从9个方面跟大家一起聊聊分页查询接口优化的一些小技巧，希望对你会有所帮助。


![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/ibJZVicC7nz5gDz0lqmzqK5YNRgD3cTqMaHK8ug5xWicWpsATmWY0sxFqRF5DpvT1MLKDibk4sUlU0BRfXLj3V6Qkg/640?wx_fmt=png&from=appmsg)
## 1 增加默认条件


对于分页查询接口，如果没有特殊要求，我们可以在输入参数中，给一些默认值。


这样可以缩小数据范围，避免每次都count所有数据的情况。


对于商品查询，这种业务场景，我们可以默认查询当天上架状态的商品列表。


例如：



```
select * from product 
where edit_date>='2023-02-20' and edit_date<'2023-02-21' and status=1
```

如果每天有变更的商品数量不多，通过这两个默认条件，就能过滤掉绝大部分数据，让分页查询接口的性能提升不少。



> 温馨提醒一下：记得给`时间`和`状态`字段增加一个`联合索引`。


## 2 减少每页大小


分页查询接口通常情况下，需要接收两个参数：`pageNo`（即：页码）和`pageSize`（即：每页大小）。


如果分页查询接口的调用端，没有传pageNo默认值是1，如果没有传pageSize也可以给一个默认值10或者20。


不太建议pageSize传入过大的值，会直接影响接口性能。


在前端有个下拉控件，可以选择每页的大小，选择范围是：10、20、50、100。


前端默认选择的每页大小为`10`。


不过在实际业务场景中，要根据产品需求而且，这里只是一个参考值。


## 3 减少join表的数量


有时候，我们的分页查询接口的查询结果，需要join多张表才能查出数据。


比如在查询商品信息时，需要根据商品名称、单位、品牌、分类等信息查询数据。


这时候写一条sql可以查出想要的数据，比如下面这样的：



```
select 
  p.id,
  p.product_name,
  u.unit_name,
  b.brand_name,
  c.category_name
from product p
inner join unit u on p.unit_id = u.id
inner join brand b on p.brand_id = b.id
inner join category c on p.category_id = c.id
where p.name='测试商品' 
limit 0,20;
```

使用product表去`join`了unit、brand和category这三张表。


其实product表中有unit\_id、brand\_id和category\_id三个字段。


我们可以先查出这三个字段，获取分页的数据缩小范围，之后再通过主键id集合去查询额外的数据。


我们可以把sql改成这样：



```
select 
  p.id,
  p.product_id,
  u.unit_id,
  b.brand_id,
  c.category_id
from product
where name='测试商品'
limit 0,20;
```

这个例子中，分页查询之后，我们获取到的商品列表其实只要20条数据。


再根据20条数据中的id集合，获取其他的名称，例如：



```
select id,name 
from unit
where id in (1,2,3);
```

然后在程序中填充其他名称。


伪代码如下：



```
List<Product> productList = productMapper.search(searchEntity);
List<Long> unitIdList = productList.stream().map(Product::getUnitId).distinct().collect(Collectors.toList());
List<Unit> unitList = UnitMapper.queryUnitByIdList(unitIdList);
for(Product product: productList) {
   Optional<Unit> optional = unitList.stream().filter(x->x.getId().equals(product.getId())).findAny();
   if(optional.isPersent()) {
      product.setUnitName(optional.get().getName());
   } 
}
```

这样就能有效的减少join表的数量，可以一定的程度上优化查询接口的性能。


## 4 优化索引


分页查询接口性能出现了问题，最直接最快速的优化办法是：`优化索引`。


因为优化索引不需要修改代码，只需回归测试一下就行，改动成本是最小的。


我们需要使用`explain`关键字，查询一下生产环境分页查询接口的`执行计划`。


看看有没有创建索引，创建的索引是否合理，或者索引失效了没。


索引不是创建越多越好，也不是创建越少越好，我们需要根据实际情况，到生产环境测试一下sql的耗时情况，然后决定如何创建或优化索引。


建议优先创建`联合索引`。


如果你对explain关键字的用法比较感兴趣，可以看看我的这篇文章《[explain \| 索引优化的这把绝世好剑，你真的会用吗？](https://github.com)》。


如果你对索引失效的问题比较感兴趣，可以看看我的这篇文章《[聊聊索引失效的10种场景，太坑了](https://github.com):[milou加速器](https://xinminxuehui.org)》。


## 5 用straight\_join


有时候我们的业务场景很复杂，有很多查询sql，需要创建多个索引。


在分页查询接口中根据不同的输入参数，最终的查询sql语句，MySQL根据一定的抽样算法，却选择了不同的索引。


不知道你有没有遇到过，某个查询接口，原本性能是没问题的，但一旦输入某些参数，接口响应时间就非常长。


这时候如果你此时用`explain`关键字，查看该查询sql执行计划，会发现现在走的索引，跟之前不一样，并且驱动表也不一样。


之前一直都是用表a驱动表b，走的索引c。


此时用的表b驱动表a，走的索引d。


为了解决Mysql选错索引的问题，最常见的手段是使用`force_index`关键字，在代码中指定走的索引名称。


但如果在代码中硬编码了，后面一旦索引名称修改了，或者索引被删除了，程序可能会直接报错。


这时该怎么办呢？


答：我们可以使用`straight_join`代替`inner join`。


straight\_join会告诉Mysql用左边的表驱动右边的表，能改表优化器对于联表查询的执行顺序。


之前的查询sql如下：



```
select p.id from product p
inner join warehouse w on p.id=w.product_id;
...
```

我们用它将之前的查询sql进行优化：



```
select p.id from product p
straight_join warehouse w on p.id=w.product_id;
...
```

## 6 数据归档


随着时间的推移，我们的系统用户越来越多，产生的数据也越来越多。


单表已经到达了几千万。


这时候分页查询接口性能急剧下降，我们不能不做分表处理了。


做简单的分表策略是将历史数据归档，比如：在`主表`中只保留最近三个月的数据，三个月前的数据，保证到`历史表`中。


我们的分页查询接口，默认从主表中查询数据，可以将数据范围缩小很多。


如果有特殊的需求，再从历史表中查询数据，最近三个月的数据，是用户关注度最高的数据。


## 7 使用count(\*)


在分页查询接口中，需要在sql中使用`count`关键字查询`总记录数`。


目前count有下面几种用法：


* count(1\)
* count(id)
* count(普通索引列)
* count(未加索引列)


那么它们有什么区别呢？


* count(\*) ：它会获取所有行的数据，不做任何处理，行数加1。
* count(1\)：它会获取所有行的数据，每行固定值1，也是行数加1。
* count(id)：id代表主键，它需要从所有行的数据中解析出id字段，其中id肯定都不为NULL，行数加1。
* count(普通索引列)：它需要从所有行的数据中解析出普通索引列，然后判断是否为NULL，如果不是NULL，则行数\+1。
* count(未加索引列)：它会全表扫描获取所有数据，解析中未加索引列，然后判断是否为NULL，如果不是NULL，则行数\+1。


由此，最后count的性能从高到低是：



> count(\*) ≈ count(1\) \> count(id) \> count(普通索引列) \> count(未加索引列)


所以，其实`count(*)`是最快的。



> 我们在使用count统计总记录数时，一定要记得使用count(\*)。


## 8 从ClickHouse查询


有些时候，join的表实在太多，没法去掉多余的join，该怎么办呢？


答：可以将数据保存到`ClickHouse`。


ClickHouse是基于`列存储`的数据库，不支持事务，查询性能非常高，号称查询十几亿的数据，能够秒级返回。


为了避免对业务代码的嵌入性，可以使用`Canal`监听`Mysql`的`binlog`日志。当product表有数据新增时，需要同时查询出单位、品牌和分类的数据，生成一个新的结果集，保存到ClickHouse当中。


查询数据时，从ClickHouse当中查询，这样使用count(\*)的查询效率能够提升N倍。



> 需要特别提醒一下：使用ClickHouse时，新增数据不要太频繁，尽量批量插入数据。


其实如果查询条件非常多，使用ClickHouse也不是特别合适，这时候可以改成`ElasticSearch`，不过它跟Mysql一样，存在`深分页`问题。


## 9 数据库读写分离


有时候，分页查询接口性能差，是因为用户并发量上来了。


在系统的初期，还没有多少用户量，读数据请求和写数据请求，都是访问的同一个数据库，该方式实现起来简单、成本低。


刚开始分页查询接口性能没啥问题。


但随着用户量的增长，用户的读数据请求和写数据请求都明显增多。


我们都知道数据库连接有限，一般是配置的空闲连接数是100\-1000之间。如果多余1000的请求，就只能等待，就可能会出现接口超时的情况。


因此，我们有必要做数据库的`读写分离`。写数据请求访问`主库`，读数据请求访问`从库`，从库的数据通过binlog从主库同步过来。


根据不同的用户量，可以做一主一从，一主两从，或一主多从。


数据库读写分离之后，能够提升查询接口的性能。


如果你对性能优化比较感兴趣，可以看看《[性能优化35讲](https://github.com)》，里面有更多干货内容。


 


## 最后说一句(求关注，别白嫖我)


如果这篇文章对您有所帮助，或者有所启发的话，帮忙扫描下发二维码关注一下，您的支持是我坚持写作最大的动力。求一键三连：点赞、转发、在看。关注公众号：【苏三说技术】，在公众号中回复：进大厂，可以免费获取我最近整理的10万字的面试宝典，好多小伙伴靠这个宝典拿到了多家大厂的offer。


