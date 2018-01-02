# SQL:insert加入Where条件

很多业务，尤其多对多关联，插入中间关系表数据时，经常会出现重复插入的问题。常用的解决方案有：

- 插入前删除全部关联数据
- 插入前提前查询数据是否存在
- 使用复合主键

这里再增加一个就是在insert时加入where条件限定，如Oracle中一个表User，有id(主键，使用序列test_id)、name，通过name来判断是否重复：

```sql
insert into member(name)
select nextval('TEST_ID'),'name' from USER
where not exists(select ID from USER where NAME = 'name');
```