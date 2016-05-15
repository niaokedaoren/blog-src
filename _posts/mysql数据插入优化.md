title: mysql数据插入优化
date: 2015-07-21 21:59:38
categories:
- database
tags:
- mysql
---

### 缘起
之前项目，有个任务需要将每天的批量任务运算结果插入到mysql数据库中，以备后续查询，运算结果条数大概在100w左右，如果一条条插入，需要几十分钟，竟然和产生数据的批量任务运行时间相当。经过优化之后，插入性能差不多有10倍的提升。在这里记录一下调研调优的整个过程，权当自我总结。^_^

### mysql数据插入方法
- 调用Statement相关方法，有单个调用和批量处理两种
- 调用PreparedStatement相关方法，同样有两种方式
- load方法，很好很强大

下面一一介绍这些方法，并随带样例代码加以说明。
#### Database Connection
为了方便说明，只是简单做了一下包装。
```java
public class DbConnectionManager {
    private static DbConfiguration dbConfiguration = null;
    static {
        dbConfiguration = DbConfiguration.instance();
        try {
            Class.forName(dbConfiguration.getDriver());
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }   
    }   

    public static Connection getConnection() throws SQLException {
        String url = dbConfiguration.getUrl();
        String username = dbConfiguration.getUsername();
        String password = dbConfiguration.getPassword();
        Connection connection = null;
        connection = DriverManager.getConnection(url, username, password);
        connection.setAutoCommit(false);  // 为了提高批量插入的性能

        return connection;
    }   
}
```

#### Statement
调用Statement相关方法，单个插入。
```java
public long runInsertViaStatement(List<DataUnit> data) {
    long costTime = 0;
    Connection connection = null;
    Statement statement = null;
    try {
        connection = DbConnectionManager.getConnection();
        statement = connection.createStatement();
        long begin = System.currentTimeMillis();
        for (DataUnit record : data) {
            String sql = String.format("insert ignore into term_uv values ('%s', '%s', %s, %s)",
                    record.term, record.dt, record.uv, record.platform);
            statement.execute(sql);
        }   
        connection.commit();
        costTime += (System.currentTimeMillis() - begin);
    } catch (SQLException e) {
        e.printStackTrace();
    } finally {
        try {
            statement.close();
            connection.close();
        } catch (SQLException e) {
            e.printStackTrace();
        }   
    }   

    return costTime;
}
```

#### StatementBatch
批量插入数据，每1000条数据执行一次executeBatch。
```java
statement = connection.createStatement();
int count = 0;
for (DataUnit record : data) {
    count += 1;
    String sql = String.format("insert ignore into term_uv values('%s', '%s', %s, %s)",
            record.term, record.dt, record.uv, record.platform);
    statement.addBatch(sql);
    if (count % 1000 == 0) {
        statement.executeBatch();
    }
}
statement.executeBatch();
```

#### PreparedStatement
这种方式也是一条条插入，但是不用重复为每条sql解析语法，所以相对于Statement方法，会快一些，但是在实验中，并没有差多少，可能是sql太过简单。用PreparedStatement也有缺点，就是一个statement只能为一条sql语句服务。
```java
String sql = "insert ignore into term_uv (term, dt, uv, platform) values (?, ?, ?, ?)";
statement = connection.prepareStatement(sql);
for (DataUnit record : data) {
    statement.setString(1, record.term);
    statement.setString(2, record.dt);
    statement.setString(3, record.uv);
    statement.setString(4, record.platform);
    statement.executeUpdate();
}
```

#### PreparedStatementBatch
这个就是上面几种方法的终极组合。
```java
String sql = "insert ignore into term_uv (term, dt, uv, platform) values (?, ?, ?, ?)";
statement = connection.prepareStatement(sql);
int count = 0;
for (DataUnit record : data) {
    count += 1;
    statement.setString(1, record.term);
    statement.setString(2, record.dt);
    statement.setString(3, record.uv);
    statement.setString(4, record.platform);
    statement.addBatch();
    if (count % 1000 == 0) {
        statement.executeBatch();
    }
}
statement.executeBatch();
```

#### Load
直接通过文件方式load到mysql中，这应该是最快的方式，实验结果也是如此。但是缺点就是要把数据写到文件中，然后load到mysql中。
```java
statement = connection.createStatement();
String sql = String.format("load data local infile '%s' replace into table term_uv", filename);
statement.executeUpdate(sql);
```

###实验结果
| 数据量 | Statement | PreparedStatement | StatementBatch | PreparedStatementBatch | Load |
|-----|-------|------|-------|------|------|
| 1w  | 273231 ms  | 273045 ms | 274518 ms | 273063 ms | 1002 ms |

除了load，其它方法基本差不多，当时也没有多调查，就直接采用load。

#### rewriteBatchedStatements
做完项目之后，google了一把，才发现原来默认情况下，jdbc对批量执行还是一条条处理。要开启批量模式，需要在建立connection的url参数中设置rewriteBatchedStatements为true。
```bash
jdbc:mysql://<host>:<port>/<db>?rewriteBatchedStatements=true
```
新的实验结果如下：

| 数据量 | Statement | PreparedStatement | StatementBatch | PreparedStatementBatch | Load |
|-----|-------|------|-------|------|------|
| 1w  | 273231 ms  | 273045 ms | 2555 ms | 1181 ms | 1002 ms |
| 10w  | --  | -- | 18361 ms | 7850 ms | 6812 ms |

P.S. 对于term字段中包含单引号的需要转义
```java
term.replace("'", "''");
```
