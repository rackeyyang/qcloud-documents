#### UPDATE

更新一个表的行。

##### 概要

```sql
UPDATE [ONLY] table [[AS] alias]
   SET {column = {expression | DEFAULT} |
   (column [, ...]) = ({expression | DEFAULT} [, ...])} [, ...]
   [FROM fromlist]
   [WHERE condition | WHERE CURRENT OF cursor_name ]
```

##### 描述

UPDATE更改满足条件的所有行中指定列 的值。只有要被修改的列需要在 SET 子句中提及， 没有被显式修改的列保持它们之前的值。

默认地，UPDATE 命令将会更新指定表的所有列以及它的所有子表。如果只想要更新提到的指定表，必须使用ONLY子句。

有两种方法使用包含在数据库其他表中的信息来修改一个表：使用子选择 或者在FROM 子句中指定额外的表。这种技术只适合特定的环境。

如果WHERE CURRENT OF子句被指定，被更新的行则为最近从指定游标获取的行。

用户必须拥有在要更新表上的UPDATE 特权。如果任何一列的值需要被expressions或者 condition读取， 用户还必须拥有该列上的SELECT特权。

**输出**

成功完成时，一个UPDATE命令返回命令标签如下形式：

```sql
UPDATE count
```

count是被更新的行数。 如果count 为0，没有行被该查询更新（这不是一个错误）。

##### 参数

ONLY

如果指定，则只会更新所提及表中的匹配行。如果没有指定，任何从所提及表继承得到的表中的匹配的行也会被更新。

table

要更新的表的名称（可以是方案限定的）。

alias

目标表的一个替代名称。在提供了一个别名时，它会完全隐藏表的真实 名称。例如，给定UPDATE foo AS f，UPDATE语句的剩余部分必须用 f而不是foo引用该表。

column

所指定的表的一列的名称。 如果需要，该列名可以用一个子域名称或者数组下标限定。不要在目标列的说明中包括表的名称。

expression

A要被赋值给该列的一个表达式。该表达式可以使用该表中这一列或者其他列的旧值。

DEFAULT

将该列设置为它的默认值（如果没有为它指定默认值表达式，默认值将会为NULL）。

fromlist

表表达式的列表，允许来自其他表的列出现在WHERE条件和更新表达式中。 这类似于可以在SELECT语句的FROM子句中指定的表列表。注意目标表不能出现在fromlist中，除非用户想做自连接（这种情况下它必须以alias（别名）出现在fromlist中）。

condition

一个返回boolean类型值的表达式。让这个 表达式返回true的行将会被更新。

cursor_name

要在WHERE CURRENT OF条件中使用的游标名。要被更新的是从这个游标中最近取出的行。该游标必须是一个在UPDATE目标表上的简单（非连接，非聚集）的查询。WHERE CURRENT OF 不能和一个布尔条件一起指定。

UPDATE...WHERE CURRENT OF语句只能在服务器端执行，例如在一个交互式的psql会话中或者一个脚本中。 语言扩展，例如PL/pgSQL不支持可更新游标。

output_expression

在每一行被更新后，要被UPDATE命令计算并且返回的表达式。该表达式可以使用FROM列出的表中的任何列名。写*可以返回所有列。

output_name

用于一个被返回列的名称。

##### 注解

SET在数据库一个表的分布主键列上是不允许的。

当存在FROM 子句时，实际发生的是：目标表被连接到 from_list中的表，并且该连接的每一个输出行表示对目标表的一个更新操作。在使用FROM时，用户应该确保该连接对每一个要修改的行产生至多一个输出行。换句话说，一个目标行不应该连接到来自其他表的多于一行上。如果发生这种情况，则只有一个连接行将被用于更新目标行，但是将使用哪 一行是很难预测的。

由于这种不确定性，只在一个子选择中引用其他表更安全，不过这种语句通常很难写并且也比使用连接慢。

直接在一个分区表的特定分区（子表）执行UPDATE 和 DELETE还不支持。相反，可以在根分区表（由CREATE TABLE命令创建的表）上执行这些命令。

##### 示例

把表films的列kind中的单词Drama改成Dramatic：

```sql
UPDATE films SET kind = 'Dramatic' WHERE kind = 'Drama';
```

在表weather的一行中调整温度项并且 把沉淀物重置为它的默认值：

```sql
UPDATE weather SET temp_lo = temp_lo+1, temp_hi = 
temp_lo+15, prcp = DEFAULT
WHERE city = 'San Francisco' AND date = '2016-07-03';
```

使用另一种列列表语法来做同样的更新：

```sql
UPDATE weather SET (temp_lo, temp_hi, prcp) = (temp_lo+1, 
temp_lo+15, DEFAULT)
WHERE city = 'San Francisco' AND date = '2016-07-03';
```

为管理Acme Corporation账户的销售人员增加销售量，使用 FROM 子句语法（假定所有连接的表在数据库中是通过id列进行分布的）：

```sql
UPDATE employees SET sales_count = sales_count + 1 FROM 
accounts
WHERE accounts.name = 'Acme Corporation'
AND employees.id = accounts.id;
```

执行相同的操作，在 WHERE子句中使用子选择：

```sql
UPDATE employees SET sales_count = sales_count + 1 WHERE id =
  (SELECT id FROM accounts WHERE name = 'Acme Corporation');
```

尝试插入一个新库存项及其库存量。如果该项已经存在，则转而更新 已有项的库存量。要这样做并且不让整个事务失败，可以使用保存点：

```sql
BEGIN;
-- 其他操作
SAVEPOINT sp1;
INSERT INTO wines VALUES('Chateau Lafite 2003', '24');
-- 假定上述语句由于未被唯一键失败，
-- 那么现在我们发出这些命令：
ROLLBACK TO sp1;
UPDATE wines SET stock = stock + 24 WHERE winename = 'Chateau 
Lafite 2003';
-- 继续其他操作，并且最终
COMMIT;
```

##### 兼容性

这个命令符合SQL标准，不过 FROM子句是数据库的扩展。

根据标准，列语法应该允许列为从一个单一的行值表达式赋值，如子选择：

```sql
UPDATE accounts SET (contact_last_name, contact_first_name) =
    (SELECT last_name, first_name FROM salesmen
     WHERE salesmen.id = accounts.sales_id);
```

当前还没有实现 — 源必须是独立的表达式列表。

有些其他数据库系统提供了一个FROM选项，在其中在其中目标表 可以在FROM中被再次列出。但数据库不是这样解释FROM的。在移植使用这种扩展的应用时要小心。

##### 另见

DECLARE, DELETE, SELECT, INSERT