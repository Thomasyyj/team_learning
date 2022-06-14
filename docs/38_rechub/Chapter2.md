# Task2:数据库的基本使用

![](material/ch1-overview.png)

本节中我们主要来了解各个数据库的基本使用方法

## 1.MySQL的基本使用方法

### 1.1 SQL基本语句规范

- 所有语句以分号（;）结尾
- 语句不区分大小写
- 插入到表中的数据区分大小写
- 常数：
  * sql中直接写的**字符串，数字或者日期**叫做常数
  * 字符串与日期：用单引号（''）分隔开，如:('abc')
  * 数字：直接写即可
- SQL中的注释主要采用`--`和`/* ... */`的方式，第二种方式可以换行。在MySQL下，还可以通过`#`来进行注释。

### 1.2 命名规则

- 在数据库中，只能使用半角英文字母、数字、下划线（_）作为数据库、表和列的名称 。
- 名称必须以半角英文字母作为开头。
- 名称不能重复，同一个数据库下不能有2张相同的表。

### 1.3 数据类型

Mysql支持所有SQL数值数据类型，其中包括：数值类，日期时间类（datetime，date，timestamp等），字符串类

### 1.4 数据库的基本操作

#### 1.4.1 数据库的创建：

通过create语句创建

```mysql
create database [if not exists] <数据库名称>;
```

MySQL 的数据存储区将以目录方式表示 MySQL 数据库，因此数据库名称必须符合操作系统的文件夹命名规则，不能以数字开头，尽量要有实际意义。e.g:

```mysql
create database if not exists shop;
```

#### 1.4.2数据库的查看：

- 查看所有存在的数据库

  ```mysql
  show databases [like '数据库名'];;
  ```

  `LIKE`从句是可选项，用于匹配指定的数据库名称。`LIKE` 从句可以部分匹配，也可以完全匹配。

  e.g:

  ```mysql
  SHOW DATABASES LIKE 'S%';
  ```

- 查看创建的数据库

  ```mysql
  SHOW CREATE DATABASES <数据库名>;
  ```

  e.g:

  ```mysql
  SHOW CREATE DATABASES shop;
  ```

#### 1.4.3 选择数据库

在操作数据库前，必须指定所要操作的数据库。可以通过`USE`命令切换到对应的数据库下。

```mysql
USE <数据库名>
```

#### 1.4.4 删除数据库

```mysql
DROP DATABASE [IF EXISTS] <数据库名>;
```

### 1.5 表的基本操作

表相当于文件，表中的一条记录就相当于文件的一行内容，不同的是，表中的一条记录有对应的标题，称为表的字段。

#### 1.5.1 表的创建

语法结构为

```mysql
CREATE TABLE <表名> （<字段1> <数据类型> <该列所需约束>，
   <字段2> <数据类型> <该列所需约束>，
   <字段3> <数据类型> <该列所需约束>，
   <字段4> <数据类型> <该列所需约束>，
   .
   .
   .
   <该表的约束1>， <该表的约束2>，……）；
```

e.g:

```mysql
CREATE TABLE Product(
  product_id CHAR(4) NOT NULL,
  product_name VARCHAR(100) NOT NULL,
  procut_type VARCHAR(32) NOT NULL,
  sale_price INT,
  purchase_price INT,
  regist_date DATE,
  PRIMARY KEY(product_id)
);
```

在第二章中，我们介绍过不同的数据类型:

其中`CHAR`为定长字符，这里`CHAR`旁边括号里的数字表示该字段最长为多少字符，少于该数字将会使用空格进行填充。

`VARCHAR`表示变长字符，括号里的数字表示该字段最长为多少字符，存储时只会按照字符的实际长度来存储，但会使用额外的1-2字节来存储值长度。

简单介绍一下该语句中出现的约束条件，约束条件在后面会详细介绍：

- `PRIMARY KEY`：主键，表示该字段对应的内容唯一且不能为空。
- `NOT NULL`：在 `NULL` 之前加上了表示否定的` NOT`，表示该字段不能输入空白。

通过`SHOW TABLES`命令来查看当前数据库下的所有的表名：

```mysql
SHOW TABLES;
```

通过DESE<表名>来查看表的结构：

```mysql
DESC Product;
```



#### 1.5.2 表的删除

删除表的情况，直接用drop指令

```mysql
DROP TABLE <表名>;

-- 例如：DROP TABLE Product;
```

说明：通过`DROP`删除的表示无法恢复的，在删除表的时候请谨慎。

#### 1.5.3 表的字段的修改

通过`ALTER TABLE`语句，我们可以对表字段进行不同的操作，下面通过示例来具体学习用法。

e.g:

首先我们创建一张名为student的表

```mysql
CREATE TABLE Student(
  id INT PRIMARY KEY,
  name CHAR(15)
); 
```

```mysql
DESE Student;
```

1. 操作：更改表名（通过rename）

```mysql
ALTER TABLE Student RENAME Students;
```

2. 操作：插入新的字段（通过add）

```mysql
-- 不同字段通过逗号分开
ALTER TABLE Students ADD sex CHAR(1), ADD age INT;
```

3. 其它插入

```mysql
-- 在表首插入字段
ALTER TABLE Students ADD stu_num INT FIRST;

-- 在某指定字段后插入height
ALTER TABLE Students ADD height INT AFTER sex;
```

4. 字段的删除

```mysql
-- 删除字段stu_num
ALTER TABLE Students DROP stu_num;
```

5. 字段的修改

   1. d通过`MODIFY`修改字段的数据类型。

      ```mysql
      -- 修改字段age的数据类型
      ALTER TABLE Students MODIFY age CHAR(3);
      ```

   2. 通过change命令，修改字段名或类型

      ```mysql
      -- 修改字段name为stu_name，不修改数据类型
      ALTER TABLE Students CHANGE name stu_name CHAR(15);
      
      -- 修改字段sex为stu_sex，数据类型修改为int
      ALTER TABLE Students CHANGE sex stu_sex INT;
      ```

#### 1.5.4 表的查询

sql语句主要通过select语句来查询，语法结构为

```mysql
SELECT <字段名>, ……
 FROM <表名>;
```

如果要直接查询全部字段

```mysql
SELECT *
 FROM <表名>;
```

其中，**星号（*）**代表全部字段的意思。



#### 1.5.5 表的复制





