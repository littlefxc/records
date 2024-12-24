---
status: Done
---



## 表及字段信息

```sql
SELECT * FROM cols WHERE TABLE_NAME = 'xxx';
```

## 字段注释

```sql
SELECT * FROM user_col_comments WHERE TABLE_NAME = 'xxx';
```

## 表名称及说明

```sql
SELECT * FROM user_tab_comments WHERE TABLE_NAME = 'xxx';
```

## 建表信息 

```sql
SELECT * FROM user_objects WHERE rownum < 10;
```

## 查询Oracle数据库表结构

```sql
SELECT
	cols.Table_name 表名,
	user_tab_comments.comments 表说明,
	cols.Column_Name 字段名称,
	DATA_TYPE 数据类型,
	cols.DATA_LENGTH 数据长度,
	user_col_comments.Comments 字段说明,
	cols.NullAble 是否为空 
FROM
	cols
	LEFT JOIN user_tab_comments ON ( cols.Table_name = user_tab_comments.Table_name )
	LEFT JOIN user_col_comments ON ( cols.Table_name = user_col_comments.Table_name AND cols.Column_Name = user_col_comments.Column_Name ) 
GROUP BY
	cols.Table_Name,
	user_tab_comments.comments,
	cols.Column_Name,
	cols.DATA_TYPE,
	cols.DATA_LENGTH,
	user_col_comments.Comments,
	cols.NullAble 
ORDER BY
	cols.Table_Name;
```