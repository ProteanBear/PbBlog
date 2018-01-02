#Redis:批量删除Key

redis-cli中只有一个del命令可以用来删除key，但是无法用到keys命令的那个匹配模式。目前比较好的解决方案是使用linux系统的xargs，Mac中也有，但是Windows就不支持了。
简单说明一下：xargs 是一条Unix 和类 Unix 操作系统的常用命令；它的作用是将参数列表转换成小块分段传递给其他命令。那么使用它我们就可以使用如下方法来批量删除Redis中的key:

```
-- 全局，即redis-cli已经设置成系统变量
redis-cli keys "*" | xargs redis-cli del
-- 相对路径，即redis-cli没有设置成系统变量，但在当前目录
./redis-cli keys "*" | xargs ./redis-cli del
-- 绝对路径，redis-cli没有设置成系统变量，如位置在/opt/redis/
/opt/redis/redis-cli keys "*" | xargs /opt/redis/redis-cli del
-- 需要密码的情况
redis-cli -a password keys "*" | xargs redis-cli -a password del
```