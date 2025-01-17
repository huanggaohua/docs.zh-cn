# UPDATE

该语句用于更新一张主键模型表中的数据行。

3.0 版本之前，UPDATE 语句仅支持简单的语法，例如 `UPDATE <table_name> SET <column_name>=<expression> WHERE <where_condition>`。从 3.0 版本开始，StarRocks 丰富了 UPDATE 语法，支持使用多表关联和公用表表达式（CTE）。如果需要将待更新的表与数据库中其他表关联，则可以在 FROM 子句或 CTE 中引用其他的表。

## 使用说明

您需要确保 UPDATE 语句中 FROM 子句的表表达式可以转换成等价的 JOIN 查询语句。因为 StarRocks 实际执行 UPDATE 语句时，内部会进行这样的转换。假设 UPDATE 语句为 `UPDATE t0 SET v1=t1.v1 FROM t1 WHERE t0.pk = t1.pk;`，该 FROM 子句的表表达式可以转换为 `t0 JOIN t1 ON t0.pk=t1.pk;`。并且 StarRocks 根据 JOIN 查询的结果集，匹配待更新表的数据行，更新其指定列的值。如果 JOIN 查询的结果集中一条数据行存在多条关联结果，则该数据行的更新结果是随机的。

## 语法

```SQL
[ WITH <with_query> [, ...] ]
UPDATE <table_name>
SET <column_name> = <expression> [, ...]
[ FROM <from_item> [, ...] ]
WHERE <where_condition>
```

## 参数说明

`with_query`

一个或多个可以在 UPDATE 语句中通过名字引用的 CTE。CTE 是一个临时结果集，可以提高复杂语句的易读性。

`table_name`

待更新的表的名称。

`column_name`

待更新的列的名称。不需要包含表名，例如 `UPDATE t1 SET t1.col = 1` 是不合法的。

`expression`

给列赋值的表达式。

`from_item`

引用数据库中一个或者多个其他的表。该表与待操作的表进行连接，WHERE 子句指定连接条件，最终基于连接查询的结果集给待更新中匹配行的列赋值。 例如 FROM 子句为 `FROM t1 WHERE t0.pk = t1.pk;`，StarRocks 实际执行 UPDATE 语句时会将该 FROM 子句的表表达式会转换为 `t0 JOIN t1 ON t0.pk=t1.pk;`。

`where_condition`

只有满足 WHERE 条件的行才会被更新。该参数为必选，防止误更新整张表。如需更新整张表，请使用 `WHERE true`。

## 示例

假设存在如下两张表，表 `employees` 记录雇员信息，表 `accounts` 记录账户信息。

```SQL
CREATE TABLE employees
(
    id BIGINT NOT NULL,
    sales_count INT NOT NULL
) 
PRIMARY KEY (id)
DISTRIBUTED BY HASH(id) BUCKETS 1
PROPERTIES ("replication_num" = "1");

INSERT INTO employees VALUES (1,100),(2,1000);

CREATE TABLE accounts 
(
    accounts_id BIGINT NOT NULL,
    name VARCHAR(26) NOT NULL,
    sales_person INT NOT NULL
) 
PRIMARY KEY (accounts_id)
DISTRIBUTED BY HASH(accounts_id) BUCKETS 1
PROPERTIES ("replication_num" = "1");

INSERT INTO accounts VALUES (1,'Acme Corporation',2),(2,'Acme Corporation',3),(3,'Corporation',3);
```

如果需要给表 `employees` 中 Acme Corporation 公司管理帐户的销售人员的销售计数增加 1，则可以执行如下语句：

```SQL
UPDATE employees
SET sales_count = sales_count + 1
FROM accounts
WHERE accounts.name = 'Acme Corporation'
   AND employees.id = accounts.sales_person;
```

您也可以使用 CTE 改写上述语句，增加易读性。

```SQL
WITH acme_accounts as (
    SELECT * from accounts
     WHERE accounts.name = 'Acme Corporation'
)
UPDATE employees SET sales_count = sales_count + 1
FROM acme_accounts
WHERE employees.id = acme_accounts.sales_person;
```
