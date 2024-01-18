---
title: Mybatis查询参数传入StringBuilder
date: 2024-01-18 11:00
categories:
- 框架
tags:
- MyBatis
- 源码
---

# 问题

故事要从我实习期间说起，当时使用 MyBatis 写了一个查询方法，并且在对应的 xml 文件中正确配置了 SQL 语句，虽然确定一切都没配置错误，但是查询语句始终无法查出任何信息，日志打印也是正常的

以下为问题复现

Mapper

```Java
@Mapper
public interface SBTestMapper {
    String queryByName(@Param("username") StringBuilder username);
}
```

```XML
<select id="queryByName" resultType="java.lang.String" parameterType="java.lang.StringBuilder" >
    SELECT `password` FROM user_info WHERE username = #{username}
</select>
```

很简单的查询语句，但是查不出信息，日志打印出来是这样的

```log
==>  Preparing: SELECT `password` FROM user_info WHERE username = ?
==> Parameters: pserimal(StringBuilder)
<==      Total: 0
```

数据库中直接运行是有结果的，非常疑惑对吧，但是我们注意到，一般情况下我们传的字符参数是 String 类型，我这里传的是 StringBuilder，难道是这个原因？，遂修改参数类型为 String 进行尝试，成功查询，这里就不再放代码赘述

# 解决方案

实习时只修改到了这一步便忙于其他工作了，最近开始尝试理解发生问题的原因，一开始通过在 MyBatis 源码中各种打断点，一团迷糊依然没有找到 MyBatis 到底是在哪里把参数设置到查询语句中去的

## 使用 ${} 进行字符串拼接的方式

我们换个思路，既然自己找不行，那网上有吗，没有，在网上并没有找到相关问题

那再换个思路，官方代码库的 issue 列表中有无相关问题？很遗憾，截止我写这篇博客（2024-01-18），issue 列表中用 **StringBuilder** 作为关键词查询我并没有找到相关的问题，但有条 issue 中的解决方案确实能解决问题，[unable to use dynamic sql to update value with dynamic column #2369](https://github.com/mybatis/mybatis-3/issues/2369)，其中建议使用 `${}` 来代替 `#{}`，尝试了下，将查询语句修改如下：

```SQL
SELECT `password` FROM user_info WHERE username = "${username}"
```

这次确实查询也出结果了，但我们都知道，使用 `${}` 的方式就是字符串拼接，有 SQL 注入的风险

## 自定义 TypeHandler

那如果我们非要用 StringBuilder 并且不想用 `${}` 的方式还有其他方式可以解决这个问题吗，当然！

查看官方文档，能找到这样一条说明 [XML 映射器#参数](https://mybatis.org/mybatis-3/zh_CN/sqlmap-xml.html#Parameters)

![Alt text](../assets/images/blogs/MyBatis%E6%96%87%E6%A1%A3XML%E6%98%A0%E5%B0%84%E5%99%A8%E5%8F%82%E6%95%B0TypeHandler.jpg)

哦？那我们不妨试试自定义一个 TypeHandler 来使用吧！

它在在 `org.apache.ibatis.type.TypeHandler` 这个位置，是一个带泛型的接口，其中 `setParameter` 方法首先引起我的注意，直觉告诉我，答案一定在这里

那么是直接继承这个接口吗，里面的方法怎么实现呢，不急，我们先看看 MyBatis 中对这个接口中 `setParameter` 方法的实现，只有一个 `org.apache.ibatis.type.BaseTypeHandler` 类实现了这个方法

```Java
@Override
public void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException {
    if (parameter == null) {
        if (jdbcType == null) {
        throw new TypeException("JDBC requires that the JdbcType must be specified for all nullable parameters.");
        }
        try {
        ps.setNull(i, jdbcType.TYPE_CODE);
        } catch (SQLException e) {
        throw new TypeException("Error setting null for parameter #" + i + " with JdbcType " + jdbcType + " . "
            + "Try setting a different JdbcType for this parameter or a different jdbcTypeForNull configuration property. "
            + "Cause: " + e, e);
        }
    } else {
        try {
        setNonNullParameter(ps, i, parameter, jdbcType);
        } catch (Exception e) {
        throw new TypeException("Error setting non null for parameter #" + i + " with JdbcType " + jdbcType + " . "
            + "Try setting a different JdbcType for this parameter or a different configuration property. " + "Cause: "
            + e, e);
        }
    }
}
```

方法并不复杂，除了判空和一系列抛出错误等操作外，唯一能解决我们疑问的方法是其中调用的 `setNonNullParameter`，但是该方法是一个抽象方法，看来这个类是用来实现模板模式的，不急，看看这个方法有哪些类做了实现

嗯，很多，但我们发现这些实现类都是以 Java 中某个类的名称开头的，如 `ArrayTypeHandler`，并没有找到类似于 `StringBuilderTypeHandler` 的实现，好，那看来我们的任务就是实现一个 `StringBuilderTypeHandler` 并且在 XML 的 SQL 中进行指定

仿照 `StringTypeHandler` 来写一个吧！

```Java
package com.imal.springtrick.mapper.typehandler;

import org.apache.ibatis.type.BaseTypeHandler;
import org.apache.ibatis.type.JdbcType;

import java.sql.CallableStatement;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;

/**
 * @author pserimal 2024/1/18 10:22
 * @version 1.0
 */
public class StringBuilderHandler extends BaseTypeHandler<StringBuilder> {

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, StringBuilder parameter, JdbcType jdbcType) throws SQLException {
        ps.setString(i, parameter.toString());
    }

    @Override
    public StringBuilder getNullableResult(ResultSet rs, String columnName) throws SQLException {
        String string = rs.getString(columnName);
        return new StringBuilder(string);
    }

    @Override
    public StringBuilder getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        return new StringBuilder(rs.getString(columnIndex));
    }

    @Override
    public StringBuilder getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        return new StringBuilder(cs.getString(columnIndex));
    }
}
```

然后在 SQL 中进行设置

```SQL
SELECT `password` FROM user_info WHERE username = #{username,typeHandler=com.imal.springtrick.mapper.typehandler.StringBuilderHandler}
```

重启，试试看，成功了！

```log
==>  Preparing: SELECT `password` FROM user_info WHERE username = ?
==> Parameters: pserimal(String)
<==    Columns: password
<==        Row: 123456
<==      Total: 1
```

并且注意到这次打印的日志是 `pserimal(String)`，说明我们重写的 setNonNullParameter 方法生效了！