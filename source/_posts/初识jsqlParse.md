---
title: 初识jsqlParse
abbrlink: d87f7e0c
date: 2021-04-26 11:39:03
tags: jsql,mybatis plus,tenant
top_img: /image/9.jpg
cover: /image/9.jpg
---

# jsqlParse初识
最近公司在整多租户的事情时遇到些问题，排查过程中认识到了jsqlParse这个东西，感觉比较好玩，就稍微研究一下。

# jsqlParse的官方描述
1. JSQLParser是一个基于 JavaCC 构建的 SQL 语句解析器。它将 SQL 转换为可遍历的 Java 类层次结构。
> JavaCC（Java Compiler Compiler）是一个开源的语法分析器生成器和词法分析器生成器。JavaCC根据输入的文法生成由Java语言编写的分析器。
2. JSqlParser是一个与 RDBMS 无关的 SQL 语句解析器。它将 SQL 语句转换为可遍历的 Java 类层次结构。{% label 纯机翻 %}
>  RDBMS的全拼是Relational Database Management System，从字面上可以理解为关系数据库管理系统。

看介绍可以了解到jsqlParse的功能就是把sql语句转化为Java中的类结构，所以在mybatis plus中很多增强处理中都是使用这个解析sql做处理。

# 应用方式
本文以mybties plus多租户拦截器为引入，写下一些多租户中使用jsql相关功能的理解。
以{% label com.baomidou.mybatisplus.extension.plugins.inner.TenantLineInnerInterceptor %}为例，可以看到这个拦截器继承了{% label JsqlParserSupport %}(注意这个抽象类是mybaties封装的)并在拦截器的{% label beforeQuery %}方法中调用了{% label parserSingle %}相关处理逻辑。
```java
    @Override
    public void beforeQuery(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
        if (InterceptorIgnoreHelper.willIgnoreTenantLine(ms.getId())) return;
        PluginUtils.MPBoundSql mpBs = PluginUtils.mpBoundSql(boundSql);
        mpBs.sql(parserSingle(mpBs.sql(), null));
    }
```

方法如下

```java
    public String parserSingle(String sql, Object obj) {
        if (logger.isDebugEnabled()) {
            logger.debug("original SQL: " + sql);
        }
        try {
            Statement statement = CCJSqlParserUtil.parse(sql);
            return processParser(statement, 0, sql, obj);
        } catch (JSQLParserException e) {
            throw ExceptionUtils.mpe("Failed to process, Error SQL: %s", e.getCause(), sql);
        }
    }
```
可以看到这里使用{% label net.sf.jsqlparser.parser.CCJSqlParserUtil.parse(String) %}方法将sql字符串转换为了java对象，接着往下看对象中包含哪些信息以及如何使用，也就是如何为为sql自动拼接租户相关条件。

```java
    protected String processParser(Statement statement, int index, String sql, Object obj) {
        if (logger.isDebugEnabled()) {
            logger.debug("SQL to parse, SQL: " + sql);
        }
        if (statement instanceof Insert) {
            this.processInsert((Insert) statement, index, sql, obj);
        } else if (statement instanceof Select) {
            this.processSelect((Select) statement, index, sql, obj);
        } else if (statement instanceof Update) {
            this.processUpdate((Update) statement, index, sql, obj);
        } else if (statement instanceof Delete) {
            this.processDelete((Delete) statement, index, sql, obj);
        }
        sql = statement.toString();
        if (logger.isDebugEnabled()) {
            logger.debug("parse the finished SQL: " + sql);
        }
        return sql;
    }
```
我们发现上面转换后的对象可以根据类型判断出是查询语句还是修改语句或插入语句等，不同的类型进行不同的sql处理，这里我们以查询语句为例往下观察{% label processSelect %}方法，其他都是类似的原理。

```java
    @Override
    protected void processSelect(Select select, int index, String sql, Object obj) {
        processSelectBody(select.getSelectBody());
        List<WithItem> withItemsList = select.getWithItemsList();
        if (!CollectionUtils.isEmpty(withItemsList)) {
            withItemsList.forEach(this::processSelectBody);
        }
    }
```
这里发现查询中可以获取一个{% label SelectBody %}和一个{% label withItem %}的集合，其中{% label SelectBody %}即代表查询本身，而{% label withItem %}则代表SQL语句中的with语句，即对with语句内的子查询遍历处理，接着往下看处理过程。
```java
    protected void processSelectBody(SelectBody selectBody) {
        if (selectBody == null) {
            return;
        }
        if (selectBody instanceof PlainSelect) {
            processPlainSelect((PlainSelect) selectBody);
        } else if (selectBody instanceof WithItem) {
            WithItem withItem = (WithItem) selectBody;
            processSelectBody(withItem.getSubSelect().getSelectBody());
        } else {
            SetOperationList operationList = (SetOperationList) selectBody;
            List<SelectBody> selectBodys = operationList.getSelects();
            if (CollectionUtils.isNotEmpty(selectBodys)) {
                selectBodys.forEach(this::processSelectBody);
            }
        }
    }
```
这里看到也有一些判断，如果是查询是子查询则进行递归处理。我们直接看{% label processPlainSelect %}的处理。
```java
    protected void processPlainSelect(PlainSelect plainSelect) {
        FromItem fromItem = plainSelect.getFromItem();
        Expression where = plainSelect.getWhere();
        processWhereSubSelect(where);
        if (fromItem instanceof Table) {
            Table fromTable = (Table) fromItem;
            if (!tenantLineHandler.ignoreTable(fromTable.getName())) {
                //#1186 github
                plainSelect.setWhere(builderExpression(where, fromTable));
            }
        } else {
            processFromItem(fromItem);
        }
        //#3087 github
        List<SelectItem> selectItems = plainSelect.getSelectItems();
        if (CollectionUtils.isNotEmpty(selectItems)) {
            selectItems.forEach(this::processSelectItem);
        }
        List<Join> joins = plainSelect.getJoins();
        if (CollectionUtils.isNotEmpty(joins)) {
            processJoins(joins);
        }
    }
```
第一二行分别获取了{% label FormItem %}和{% label Expression %}，根据变量名其实可以猜到一些，{% label FormItem %}对应SQL语句中from后的信息，{% label Expression %}则代表where后的信息。这里因为{% label FormItem %}也有可能是子查询所以有判断是否是{% label Table %}类型，然后判断如果是租户管理表就拼接条件，往下看具体拼接实现代码怎么实现。
```java
    protected Expression builderExpression(Expression currentExpression, Table table) {
        EqualsTo equalsTo = new EqualsTo();
        equalsTo.setLeftExpression(this.getAliasColumn(table));
        equalsTo.setRightExpression(tenantLineHandler.getTenantId());
        if (currentExpression == null) {
            return equalsTo;
        }
        if (currentExpression instanceof OrExpression) {
            return new AndExpression(new Parenthesis(currentExpression), equalsTo);
        } else {
            return new AndExpression(currentExpression, equalsTo);
        }
    }
```
这里面逻辑也很简单，就是获取租户字段信息和当前租户ID的值，然后组装为{% label EqualsTo %}对象，最后判断当前条件是否是or如果是or的话给当前条件加上一层框号，否则直接把条件and拼接上。这个方法结束后返回值被set到查询的where条件中，整个流程差不多就结束了，一个租户ID的条件就这样被拼接上去了。
还有很多详细的地方没说到，比如查询中selectItem中的子查询递归处理等，感兴趣的可以自己翻阅一下，这里不展开说了。

# 回顾总结
jsql将sql转换为Java类结构
```java
    Statement parse = CCJSqlParserUtil.parse(sql);
```
常见的Statement有：Select,Insert,Delete,Update
SQL语句基本划分为：select    SelectItem   from   FromItem   where   Expression
获取查询语句的表名称
```java
((Table)plainSelect.getFromItem()).getName()
```
获取查询语句中的查询元素
```java
    PlainSelect.getSelectItems();
```
条件拼接
```java
    new AndExpression(leftExpression, rightExpression)
```
# 最后
一点浅显理解，第一次见到这玩意儿。以前有批量生成SQL的需求都是用字符串直接拼，很多复杂点的就比较难处理，现在了解到jsqlParse之后感觉可以尝试使用起来构建一定复杂度的SQL语句。
然后上面的例子是针对mybatis plus的多租户功能，可以看到里面很多实现并不完整，比如查询语句中在CASE WHEN中写子查询语句，就会导致不能按预期把租户条件加到条件中去。还有很多实际使用过程中其他比较复杂的sql都可能会产生这样的问题，如果有类似的需求还是需要大家进行很多细化的定制开发的。