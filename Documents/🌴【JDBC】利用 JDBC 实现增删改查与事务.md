# 项目构建
## maven 依赖
```xml
<!--  mysql driver      -->  
<dependency>  
    <groupId>mysql</groupId>  
    <artifactId>mysql-connector-java</artifactId>  
    <version>8.0.32</version>  
</dependency>
```
## 配置文件
```properties
url=jdbc:mysql://localhost:3306/user_db?serverTimezone=Asia/Shanghai&characterEncoding=utf8&useUnicode=true&useSSL=false&allowPublicKeyRetrieval=true  
username=root  
password=my-secret-pw
```
## 数据库连接获取
```java
public class JdbcUtil {  
    public static Connection getConnection(String resource) {  
        Connection conn = null;  
        try {  
  
            Properties properties = new Properties();  
            try (InputStream resourceAsStream = Thread.currentThread().getContextClassLoader().getResourceAsStream(resource);) {  
                properties.load(resourceAsStream);  
            } catch (IOException e) {  
                throw new RuntimeException(e);  
            }  
            String url = properties.getProperty("url");  
            String username = properties.getProperty("username");  
            String password = properties.getProperty("password");  
  
            conn = DriverManager.getConnection(url, username, password);  
        } catch (SQLException e) {  
            throw new RuntimeException(e);  
        }  
  
        return conn;  
    }  
}
```
# JDBC 实现事务
```java
public class TestTransaction {  
    public static void main(String[] args) throws SQLException {  
        Connection conn = JdbcUtil.getConnection("jdbc.properties");  
        PreparedStatement smt = null;  
        try {  
            // 1.开启事务  
            conn.setAutoCommit(false);  
            String sql = "UPDATE example SET age=? WHERE name=?";  
            smt = conn.prepareStatement(sql);  
            smt.setInt(1, 55);  
            smt.setString(2, "Greg");  
            smt.execute();  
            // 2.提交事务  
            conn.commit();  
        } catch (SQLException e) {  
            System.out.println("error");  
            // 3.回滚事务  
            conn.rollback();  
        }  
        assert smt != null;  
        smt.close();  
        conn.close();  
    }  
}
```
# JDBC 实现简单的 CRUD
## SELECT
```java
import org.junit.jupiter.api.Test;

import java.sql.*;

public class TestSqlQuery {
    @Test
    public void queryAll() throws SQLException {
        Connection conn = null;
        // 1.获取连接
        conn = JdbcUtil.getConn();
        // 2.sql语句
        String sql = "SELECT * FROM example";
        // 3.获取对象
        PreparedStatement preparedStatement = conn.prepareStatement(sql);
        // 4.执行语句
        ResultSet rs = preparedStatement.executeQuery();
        // 5.处理结果，多条结果用while，单条用if
        while (rs.next()) {
            // 通过字段名获取对象值
            int id = rs.getInt("id");
            // 通过字段列序号获取值
            String name = rs.getString(2);
            int age = rs.getInt("age");
            Date createTime = rs.getDate("create_time");
            System.out.println(String.format("id: %d, name: %s, age: %d, create_time: %s", id, name, age, createTime.toString()));
        }

        Util.close(conn, preparedStatement, rs);
    }

    @Test
    public void queryByName() throws SQLException{
        Connection conn = null;
        // 1.获取连接
        conn = Util.getConn();
        // 2.sql语句
        String sql = "SELECT * FROM example WHERE name=?";
        // 3.获取对象
        PreparedStatement preparedStatement = conn.prepareStatement(sql);
        preparedStatement.setString(1,"Greg");
        // 4.执行语句
        ResultSet rs = preparedStatement.executeQuery();
        // 5.处理结果，多条结果用while，单条用if
        if (rs.next()) {
            // 通过字段名获取对象值
            int id = rs.getInt("id");
            // 通过字段列序号获取值
            String name = rs.getString(2);
            int age = rs.getInt("age");
            Date createTime = rs.getDate("create_time");
            System.out.println(String.format("id: %d, name: %s, age: %d, create_time: %s", id, name, age, createTime.toString()));
        }

        Util.close(conn, preparedStatement, rs);
    }
}
```
## UPDATE
```java
import org.junit.jupiter.api.Test;

import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.SQLException;

public class TestSqlUpdate {

    @Test
    public void update() throws SQLException {
        Connection conn = null;
        // 1.获取conn
        conn = Util.getConn();
        // 2.sql语句
        String sql = "UPDATE example SET age=? WHERE id=?";
        // 3.获取prepareStatement
        PreparedStatement preparedStatement = conn.prepareStatement(sql);
        // 4.根据占位符填入值
        preparedStatement.setInt(1, 66);
        preparedStatement.setInt(2, 2);
        // 5.执行语句
        int result = preparedStatement.executeUpdate();
        System.out.println("影响行数：" + result);

        Util.close(conn, preparedStatement, null);
    }
}
```
## DELETE
```java
import org.junit.jupiter.api.Test;

import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.SQLException;

public class TestSqlDelete {
    @Test
    public void delete() throws SQLException {
        Connection conn = null;
        conn = Util.getConn();

        String sql = "DELETE FROM example WHERE age>?";

        PreparedStatement preparedStatement = conn.prepareStatement(sql);

        preparedStatement.setInt(1, 60);

        preparedStatement.executeUpdate();

        Util.close(conn, preparedStatement, null);
    }
}
```
## INSERT
```java
import org.junit.jupiter.api.Test;

import java.sql.*;
import java.util.Random;

public class TestSqlInsert {
    @Test
    public  void insert() throws SQLException {
        Connection conn = null;
        // 1.获取conn
        conn = Util.getConn();
        // 2.sql 语句，占位符
        String sql = "INSERT INTO example (`name`, `age`) VALUES (?, ?)";
        // 3.获取sql语句对象
        PreparedStatement preparedStatement = conn.prepareStatement(sql);
        // 4.填入字段内容
        // 第一个占位符字段，string类型
        preparedStatement.setString(1, "Gabriel");
        // 第二个占位符字段，int类型
        preparedStatement.setInt(2, 7);
        // 5.执行sql
        preparedStatement.executeUpdate();
        
        Util.close(conn, preparedStatement, null);
    }

    @Test
    public void insertBatch() throws SQLException {
        Connection conn = null;
        conn = Util.getConn();
        // 关闭自动提交
        conn.setAutoCommit(false);

        String sql = "INSERT INTO example (name, age) VALUES (?, ?)";
        PreparedStatement statement = conn.prepareStatement(sql);

        for (int i = 0; i < 100; i++) {
            statement.setString(1, "Galaxy" + i);
            statement.setInt(2, new Random().nextInt(18, 65));
            statement.addBatch();
            if (i % 10 == 0) {
                statement.executeBatch();
            }
        }
        statement.executeBatch();
        conn.commit();

        statement.close();
        conn.close();
    }
}
```