# Hibernate:Oracle数据库中Date类型的注解

在Hibernate的实体类（Entity）中对应Oracle数据库中的Date类型时，日期时间类型的属性字段设置不正确，很容易造成此类型数据存储不当，或者读取时只显示部分内容。

像IDEA中使用Persistence自动生成的实体类中就是使用的java.sql.Time，实际保存数据是正确的，但是使用Time读取输出后就只有时间没有日期。

而使用java.sql.Date类，是只有日期没有时间；使用java.util.Date类，需要@Temporal(TemporalType.DATE)注解，而且输出结果也是只有日期没有时间。

有网上说数据库中需使用Timestamp时间戳，但是在直接访问数据库时查看起来又不方便。

最终找到完美的解决方案如下：

1. 数据库字段为Date类型

2. Entity中字段注解如下，类为java.util.Date：

   ```java
   @Column(name="UPDATE_TIME",columnDefinition = "Date", nullable=true)
   @Temporal(TemporalType.TIMESTAMP)
   ```

3. 如果是jackson转换实体为json，增加注解格式化输出（这里加了时区防止输出时间偏差）：

   ```java
   @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss",timezone = "GMT+8")
   ```

   ​