---
title: mysql批量更新的几种方法.md
date: 2019-11-01 23:18:17
tags:
---
# mysql 批量update
创建初始数据
```sql
-- 创建表
create table `fruit` (
    id int(11) not null auto_increment primary key,
    fruit_name varchar(32) not null default ''
);

-- 插入初始数据

INSERT into `fruit` (`id`, `fruit_name`)
    VALUES
    (1,'apple'),
    (2, 'orange'),
    (3, 'peach');

```

## 方法一： 使用on duplicate key update ==推荐==

注意 1:插入的值中必须含有一列有唯一索引的列，比如id
2：其他的列必须有默认值

当主键或唯一索引重复时更新数据，否则插入新数据，不需要维护索引，效率很高更新一张十几万条数据的表只需几秒到十几秒。
```sql
INSERT into `fruit` (`id`, `fruit_name`)
    VALUES
    (1, 'grape'),
    (2, 'banana'),
    (3, 'strawbe')
    ON DUPLICATE KEY UPDATE `fruit_name` = VALUES(`fruit_name`);

```

## 方法二 case when

```sql
UPDATE fruit
SET fruit_name = (CASE id WHEN 1 THEN 'grape'
                 WHEN 2 THEN 'banana'
                 WHEN 3 THEN 'strawberry'
         END)
WHERE id IN(1, 2 ,3);

```

## 方法三 创建临时表

创建临时表，联结临时表和需要更新的表，更新之后删除临时表
例如：
``` sql
-- 示例：

update table1, table2 set table1.fruit_name = table2.fruit_name where table1.id = table2.id;
```

# 方法四：replace into

当主键或唯一索引重复时删除旧的数据并插入新的数据，需要维护索引效率较慢

```sql
    replace into fruit (id,fruit_name) values (1,'grape'),(2,'banana'),(3,'strawberry');
```