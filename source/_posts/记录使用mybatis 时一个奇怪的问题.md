---
title: 记录使用mybatis时一个奇怪的问题
abbrlink: d87f7e0p
date: 2024-01-26 11:39:03
tags: mybatis,ognl
top_img: /image/yaomeng.jpg
cover: /image/yaomeng.jpg
---

# 问题
群友发出求助，说代码明明给变量赋值为2但是执行完查询语句之后又变成4了，代码晒出来后一头问号。看起来就是正常的代码，怎么会这样呢。先来看一下代码示例：
```java
    public void select(PickingDTO pickingDTO) {
        pickingDTo.setDocumentstate("2");
        List<JneProductionPickingLine> pickingLineList = pickingLineMapper.findProductionPickingLineList(pickingDTO);
        for (JneProductionPickingLine jpl : pickingLineList) {
        // 一些处理...
        }
    }
```
大概是这样的一点代码，经过debug排查发现只要mapper的{% label findProductionPickingLineList %}方法一执行{% label pickingDTO %}的{% label  documentstate %}属性就会变成4。代码就这两行，看不出什么头绪但是又感觉mapper不太可能会影响到代码中的变量，不过也没别的地方可以排查了，就转到mapper对应的xml去看，对应的查询语句大概长这个样子。
```xml
    select * from table where delete_flag = 0
    <if test="dto.documentstate != null and dto.documentstate = '4'">
        and documentstate = #{dto.documentstate}
    </if>
```
Emmm...看完之后发现这里确实写的有点问题，难道是因为test条件中的判断把==少写了一个？但是为什么不报错呢，如果不支持这样写的话执行时应该就报错了。但是又没有发现有异常抛出，查询语句也是正常执行的。百思不得其解，就把==补全了一下，结果再次测试问题就消失了。原来真的是被xml中的判断条件把值改变了，具体是什么原因呢？

# OGNL表达式
自己想是想不通了，就去网上查询了一下mybatis判断条件的实现代码，发现使用的是ONGL表达式。以前都是学习判断条件怎么写，有不清楚的就直接百度具体用法，没有注意到过原理是怎样的。

