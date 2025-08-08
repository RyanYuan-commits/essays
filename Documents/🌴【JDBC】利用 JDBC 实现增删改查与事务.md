### 1 项目构建
#### maven 依赖
```xml
<!-- mysql driver -->  
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.32</version>
</dependency>
```
#### 配置文件
```properties
url=jdbc:mysql://localhost:3306/user_db?serverTimezone=Asia/Shanghai&characterEncoding=utf8&useUnicode=true&useSSL=false&allowPublicKeyRetrieval=true
username=root
password=my-secret-pw
```
### 2 数据库连接获取
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
### 3 JDBC 实现事务
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
### 4 JDBC 实现简单的 CRUD
源码: https://github.com/RyanYuan-commits/frame-source-read/blob/main/mybatis-source-read/src/main/java/com/ryan/jdbc_test/TestSql.java
#### 查询
```java
public void queryAll() throws SQLException {
	try (Connection conn = JDBCUtil.getConnection("jdbc_test/jdbc.properties")) {
		String sql = "SELECT * FROM users";
		PreparedStatement preparedStatement = conn.prepareStatement(sql);
		ResultSet rs = preparedStatement.executeQuery();
		while (rs.next()) {
			int id = rs.getInt("id");
			String name = rs.getString(2);
			int age = rs.getInt("age");
			Date createTime = rs.getDate("create_time");
			System.out.printf("id: %d, name: %s, age: %d, create_time: %s%n", id, name, age, createTime.toString());
		}
	}
}
```
#### 更新
```java
public void update() throws SQLException {
	try (Connection connection = JDBCUtil.getConnection("jdbc_test/jdbc.properties")) {
		String sql = "UPDATE users SET age=? WHERE id=?";
		PreparedStatement preparedStatement = connection.prepareStatement(sql);
		preparedStatement.setInt(1, 66);
		preparedStatement.setInt(2, 2);
		int result = preparedStatement.executeUpdate();
		System.out.println("影响行数：" + result);
	}
}
```
#### 删除
```java
public void delete() throws SQLException {
	try (Connection conn = JDBCUtil.getConnection("jdbc_test/jdbc.properties")) {
		String sql = "DELETE FROM users WHERE age>?";
		PreparedStatement preparedStatement = conn.prepareStatement(sql);
		preparedStatement.setInt(1, 60);
		preparedStatement.executeUpdate();
	}
}
```
#### 插入
```java
public  void insert() throws SQLException {
	try (Connection conn = JDBCUtil.getConnection("jdbc_test/jdbc.properties")) {
		String sql = "INSERT INTO users (`name`, `age`) VALUES (?, ?)";
		PreparedStatement preparedStatement = conn.prepareStatement(sql);
		preparedStatement.setString(1, "Gabriel");
		preparedStatement.setInt(2, 7);
		preparedStatement.executeUpdate();
	}
}
```