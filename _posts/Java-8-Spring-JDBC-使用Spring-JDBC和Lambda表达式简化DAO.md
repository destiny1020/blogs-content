---
title: '[Java 8 & Spring JDBC] 使用Spring JDBC和Lambda表达式简化DAO'
date: 2015-07-09 10:48:00
categories: [编程语言, Java]
tags: [Java, Java 8, Spring, Lambda]
---

## 使用Spring JDBC和Lambda表达式简化DAO

如果你需要向数据库中插入一条Item记录，那么会有类似下面的代码：

Item对应的实体类型为：

```java
public class Item {
	public int name;
	public BigDecimal price;
}
```

<!-- More -->

```java
public void create(Item item) throws IOException {
  PreparedStatement ps = null;

  try {
    Connection con = template.getDataSource().getConnection();
    ps = con.prepareStatement("insert into items (name, price, prc_date) values (?, ?, ?, now())");
    ps.setString(1, item.name);
    ps.setBigDecimal(2, item.price);
    ps.executeUpdate();
  } catch (SQLException e) {
    throw new IOException(e);
  } finally {
    if (ps != null) {
      try {
        ps.close();
      } catch (SQLException e) {
        logger.warn(e.getMessage(), e);
      }
    }
  }
}
```

其中的template的类型为`org.springframework.jdbc.core.JdbcTemplate`。
如果使用JdbcTemplate类型提供的update方法，可以使上述代码大幅简化：

```java
public void create(Item item) throws IOException {
  template.update(
      "insert into items (name, price, prc_date) values (?, ?, now())",
      item.name, item.price);
}
```

但是，直接使用update方法的这一重载并不是最快的。可以使用`public int update(String sql, PreparedStatementSetter pss)`这一重载来得到更佳的运行速度：

```java
public void create(CartItemRelation item) throws IOException {
  template.update(
      "insert into item (name, price, prc_date) values (?, ?, now())",
      new PreparedStatementSetter() {
        @Override
        public void setValues(PreparedStatement ps) throws SQLException {
          ps.setString(1, item.name);
          ps.setBigDecimal(2, item.price);
        }
      });
}
```

如果使用Java 8的Lambda表达式，上述代码仍然有简化的空间：

```java
public void create(final Item item) throws IOException {
  template.update(
      "insert into items (name, price, prc_date) values (?, ?, now())",
      ps -> {
        ps.setString(1, item.name);
        ps.setBigDecimal(2, item.price);
      });
}
```

同样的，对于SELECT语句也可以通过JdbcTemplate和Lambda表达式简化，简化后繁琐的try-catch-finally语句可以被有效消除：

```java
public Item findByItemName(String name) throws IOException {
  PreparedStatement ps = null;
  ResultSet rs = null;

  try {
    Connection con = template.getDataSource().getConnection();
    ps = con.prepareStatement("select name, price from items where name = ?");
    ps.setString(1, name);
    rs = ps.executeQuery();

    if (rs.next()) {
      return new Item(rs.getString(1), rs.getBigDecimal(2));
    }

    return null;
  } catch (SQLException e) {
    throw new IOException(e);
  } finally {
    if (rs != null) {
      try {
        rs.close();
      } catch (SQLException e) {
        logger.warn(e.getMessage(), e);
      }
    }

    if (ps != null) {
      try {
        ps.close();
      } catch (SQLException e) {
        logger.warn(e.getMessage(), e);
      }
    }
  }
}
```

简化后的代码如下所示：

```java
public Item findItemByName(String name) throws IOException {
  return DataAccessUtils.requiredSingleResult(
      template.query("select name, price from items where name = ?",
          ps -> {
              ps.setString(1, name);
          },
          (rs, rowNum) -> new Item(rs.getString(1), rs.getBigDecimal(2))
          ));
}
```

由于`template.query`返回的是一个List集合，所以还需要使用`DataAccessUtils.requiredSingleResult`来取得唯一对象。

对于其他类型的SQL语句，如update和delete等，都可以通过使用Spring JdbcTemplate和Lambda表达式进行大幅简化。