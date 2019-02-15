# 全国MySql二级考试 复习笔记

> 操作题因为本地题库mysql根本无法运行，只能手打记事本对照答案
> 做了几套题发现有些规律，题库总是会考察几类东西
> 所以做一下笔记，顺带复习一下sql
----
@[toc]

## 重点复习对象
* 视图
* 事件
* 触发器
* 储存过程
* 储存函数
* 用户的创建与权限
* 利用MySql进行备份和恢复
* PHP与MySql**面向过程**交互


符号含义
`[]`表示可选   `|`表示多选一

----
### 视图
视图就是临时按照自己要求制作的表，方便下次直接引用
#### 创建视图
```sql
CREATE [OR REPLACE]
    VIEW view_name[(column_list)]
    AS SELECT_statement
    [WITH [CASCADED | LOCAL] CHECK OPTION]
```
说明：
`or replace` 表示可选替换同名视图
`column_list` 表示替换视图内的对应列名称
`with check option` 用于可更新视图上的修改，`cascaded`为`默认` 表示更改会对仍需符合 `select_statement`的条件，`local`则只对定义视图进行检查

`select_statement`本身也存在一些限制：
1. 不能包含含有from的子查询
2. 用户除了create view权限外还需要有select_statement内部的相关权限
3. 不能引用系统变量或用户变量
4. 不能引用预处理语句参数
5. 引用的表或视图必须存在
6. 若select语句引用的不是当前数据库的基础表或源视图，需要加数据库前缀
7. 在由SELECT语句构造的视图定义中，允许使用ORDER BY 子句。但是，如果从特定视BY语句，则视图定义中的ORDER BY子句将被忽略
8. 对于SELECT语句中的其他选项或子句,若所创建的视图中也包含了这些选项，则语句执行效果未定义。`例如，如果在视图定义中包含了LIMIT子句，而SELECT语句也使用了自己的LIMIT子句那么MySQL对使用哪-个LIMT语句未做定义。`

#### 修改视图
```sql
ALTER VIEW view_name[(column_list)]
    AS SELECT_statement
    [WITH [CASCADED | LOCAL] CHECK OPTION]
```
就是重新写一遍

#### 删除视图

```sql
DROP VIEW [IF EXISTS]
    view_name[,view_name]
```
----
### 事件

 可以理解为`mysql计划任务`，根据周期来执行一段sql语句，可精确到每一秒执行一个任务，执行事件前`事件调度器`必须开启，其默认是关闭的
输入`SET GLOBAL event_scheduler = ON; `执行即可

[这个博客写的比较好](https://www.cnblogs.com/qlqwjy/p/7954175.html)
#### 创建事件
```sql
CREATE 
    [DEFINER = { user | CURRENT_USER }] 
    EVENT 
    [IF NOT EXISTS] 
    event_name 
    ON SCHEDULE schedule 
    [ON COMPLETION [NOT] PRESERVE] 
    [ENABLE | DISABLE | DISABLE ON SLAVE] 
    [COMMENT 'comment'] 
    DO event_body; 
   
schedule: 
    AT timestamp [+ INTERVAL interval] ... 
  | EVERY interval 
    [STARTS timestamp [+ INTERVAL interval] ...] 
    [ENDS timestamp [+ INTERVAL interval] ...] 
   
interval: 
    quantity {YEAR | QUARTER | MONTH | DAY | HOUR | MINUTE | 
              WEEK | SECOND | YEAR_MONTH | DAY_HOUR | DAY_MINUTE | 
              DAY_SECOND | HOUR_MINUTE | HOUR_SECOND | MINUTE_SECOND}
```
名词解释：
   ` event_name `：创建的event名字（唯一确定的）。
    `ON SCHEDULE`：计划任务。
   `schedule`: 决定event的执行时间和频率（注意时间一定要是将来的时间，过去的时间会出错），有两种形式 AT和EVERY。
    `[ON COMPLETION [NOT] PRESERVE]`： 可选项，默认是ON COMPLETION NOT PRESERVE 即计划任务执行完毕后自动drop该事件；ON COMPLETION  PRESERVE则不会drop掉。
    `[COMMENT 'comment']` ：可选项，comment 用来描述event；相当注释，最大长度64个字节。
    `[ENABLE | DISABLE]` ：设定event的状态，默认ENABLE：表示系统尝试执行这个事件， DISABLE：关闭该事情，可以用alter修改
    `DO event_body`: 需要执行的sql语句（可以是复合语句）。CREATE EVENT在存储过程中使用时合法的。

#### 修改事件
```sql
ALTER 
    [DEFINER = { user | CURRENT_USER }] 
    EVENT event_name 
    [ON SCHEDULE schedule] 
    [ON COMPLETION [NOT] PRESERVE] 
    [RENAME TO new_event_name] 
    [ENABLE | DISABLE | DISABLE ON SLAVE] 
    [COMMENT 'comment'] 
    [DO event_body] 
```

#### 删除事件
```sql
DROP EVENT [IF EXISTS] event_name
```
----
### 触发器
顾名思义，一旦满足什么条件就只执行语句
有点类似于外键on delete 中的cascade restrict等

**触发器的四要素**
 
监视地点（table），监视事件（insert/update/delete），触发时间（after/before），触发事件（insert/update/delete）

#### 创建触发器
```sql
CREATE TRIGGER 触发器名称
    after|before（触发时间）
    insert|update|delete（触发事件）
    on 表名（监视地址）
    FOR EACH ROW--此句在MySQL中是写死的，只有行触发器，在oracle中还有表触发器
BEGIN
    sql1
    ...
    sqlN
END;
```
语句解释：
触发时间Before|After：表示触发器是在激活语句之前还是之后出发。
    - before是**先完成触发**，再进行数据的增、删、改`若希望验证新数据是否满足使用限制，则使用before`，
    - after是**先完成数据的增、删、改**，再触发`若希望在触发器的语句执行之后完成几个或更多的改变通常使用after选项。`
触发事件：
    1. INSERT 将新行插入表示激活触发器。
    2. UPDATE 更改表中某一行时激活触发器。 
    3. DELETE 从表中删除某一行时激活触发器。

**如何在使用触发器时引用行值**
    * 对于`insert`而言，新增的行用`new`来表示；行中的每一列的值，用`new.列名`来表示。
    * 对于`delete`而言，删除的行用`old`来表示，行中的每一列的值，用`old.列名`来表示。
    * 对于`update`而言，被修改的行，修改前的数据用`old`来表示，`old.列名`引用被修改前的行中的值；修改后的数据用`new`来表示，`new.列名`引用被修改后的行中的值。

#### 删除触发器
```sql
DROP TRIGGER [IF EXISTS] [数据库名.]触发器名
```
注意drop trigger需要super权限

---
### 储存过程
储存过程是mysql中一组为了完成某特定功能的sql语句集，实质是一段存放在数据库中的代码

#### 将分号；转义
在建立储存过程中常常需要用到`DELIMITER`来进行`;`的转义，避免在执行多条语句结束前提前结束
```sql
DELIMITER $$
```
· `$$` 是用户自定义的结束符号，可自行定义，但应该避免使用 反斜杠`\`因为是mysql的转义字符

#### 创建储存过程

```sql
DELIMITER $$
CREATE 
    PROCEDURE sp_name([[IN|OUT|INOUT] param_name type][,...]])  
    //这里类似于function sp_name(a as int,b as int)
    BEGIN
        SQL1;
        SQL2;
        ...
        SQLN;
    END $$
```
语句解释：
· `输入参数IN` 使得数据可以传递给一个储存过程 `输出参数OUT`用于储存过程需要返回一个操作结果的情形 `输入输出参数INOUT` 前面两者的集合

##### DECLARE声明局部变量
```sql
DECLARE var_name[,...]type[DEFAULT value]
```
例如`DECLARE sno CHAR(10) DEFAULT '3';`
注意点
· 只能在begin end 头部中声明
· 作用域为begin end中
· 为了和mysql区分，可以使用`@变量名`作为变量

##### SET语句进行赋值
利用set可以为局部变来给你赋值
```sql
SET var_name=expr[,var_name=expr]
```
例如`set sno='10000'`

##### 流程判语句
1. 条件判断
    * if-else
    ```sql
    IF condition THEN statement
        [ELSEIF condition THEN statement]
        [ELSE statement]
    END IF
    ```
    * case
    ```sql
    CASE case_value
        WHEN when_value THEN statement
        WHEN when_value THEN statement
    END CASE
    ```
    该种方法是使用case_value和when_value进行比较，若为真才执行
    第二种方法
    ```sql
    CASE 
        WHEN condition THEN statement
        WHEN condition THEN statement
        ELSE statement
    END CASE
    ```
    这种方法更加方便

2. 循环语句
    * WHILE循环
    ```sql
    WHILE search_condition DO
        statement
    END WHILE
    ```
    * REPEAT循环
    ```sql
    REPEAT
        statement
    UNTIL search_condition
    END REPEAT
    ```
    * LOOP循环
    ```sql
    LOOP
        statement
    END LOOP
    ```
    这种不太清楚怎么用

##### 游标
游标是一个被SELECT语句检索出来的结果集，储存了游标之后就可以对其数据进行浏览

1. 声明游标
    ```sql
    DECLARE cursor_name CURSOR FOR select_statement
    ```
2. 打开游标
    ```sql
    DECLARE cursor_name CURSOR FOR select_statement
    ```
3. 读取数据
    ```sql
    FETCH cursor_name INTO var_name[,...]
    ```
4. 关闭游标
    ```sql
    CLOSE cursor_name
    ```

#### 调用储存过程
直接像Visual Basic一样`call`就行
    ```sql
    CALL sp_name([parameter[,...]])
    ```
#### 删除储存过程
```sql
DROP PROCEDURE FUNCTION[IF EXISTS]sp_name
```
---
### 储存函数
区别   |储存过程  |储存函数
|--------|--------|--------|
输出参数|需要     |不需要，自己有return
调用    |要用CALL|直接引用

#### 创建储存函数
```sql
DELIMITER $$;
CREATE 
    FUNCTION sp_name([param_name type[,...]])
    RETURNS type
BEGIN
    sql1;
    sql2;
    ...
    sqln;
END $$
```
#### 调用储存函数
```sql
    SELECT sp_name(param)

```
#### 删除储存函数
```sql
    DROP FUNCTION sp_name;
```
------
### 用户的创建与权限
#### 创建用户
```sql
CREATE USER 'name'@'localhost' [IDENTIFIED BY 'password']
```
#### 权限设置
##### 授予权限GRANT
```sql
GRANT 权限名
ON 表名 
TO name@localhost
[WITH with_statement]
```
注意点：
`with`后常见
`WITH GRANT OPTION` 
##### 撤销权限REVOKE
```sql
REVOKE 权限名
ON 表名
FROM name@localhost
```
-----
### MySql数据备份与恢复
这个非常重要！以往因为不知道可以直接从mysql备份，总是要到`phpmyadmin`里面去导出导入，如果利用自带的`mysqldump`功能就会快很多

#### mysqldump备份数据
1. 数据表导出
```sql
mysqldump -u username -p password 数据库 数据表 > 导出文件地址文件名.sql
```
2. 备份数据库系统
```sql
mysqldump -u username -p password --all-databases > 导出filename.sql
```
3. 分别备份表结构和数据
```sql
mysqldump -u root -p **** --tab=filename.sql
可以分别备份表数据和结构
```
#### mysql恢复数据

注意恢复数据`不是``mysqldump`了
1. 恢复数据库的结构和数据
```sql
mysql -u root -p **** db_database< database.sql
```
2. 仅恢复数据
```sql
mysqlimport -u root -p **** database.sql
```
-----
### PHP与MySql**面向过程**交互
找到两个非常好的文章！
[MySQLi基于面向过程的编程](https://blog.csdn.net/koastal/article/details/50650496)
[MySQLi基于面向对象的编程](https://blog.csdn.net/koastal/article/details/50650500)

因为mysql二级考试的题目中几乎是面向过程的，所以必须用面向过程的编程，否则拿不了分
除此之外，因为考级的语句都很早，而且明显一步能完成的非要分成两步，所以一定要注重背下语句

#### 连接数据库
```php
<?php
header("Content-type:text/html;charset=utf-8");
$link  =  mysqli_connect( 'localhost' ,  'root' ,  '' ,  'test' ) or die ('Connect Error:'.mysqli_connect_error());
//这是一般的面向过程的php使用mysqli连接法
$con=mysql_connect("localhost:3306","root","")
or die("数据库服务器连接失败！<br>");
mysql_select_db('test',$con) or die( "数据库选择失败！<br>");
//这是傻不拉几的连接法，考察了选择数据库的语句
?>
```
#### 设定字符集
```php
mysqli_set_charset($link,'UTF8');
//mysqli面向过程
mysql_query("set column 'uft8");
//给某一列设置属性
```
#### 执行sql语句(插入读取更新删除)
##### 插入数据
```php
$sql = "INSERT INTO MyGuests (firstname, lastname, email)
VALUES ('John', 'Doe', 'john@example.com')";
 
if (mysqli_query($conn, $sql)) {
    echo "新记录插入成功";
} else {
    echo "Error: " . $sql . "<br>" . mysqli_error($conn);
}
```

##### 读取数据
```php
$sql = "SELECT id, firstname, lastname FROM MyGuests";
$result = mysqli_query($conn, $sql);
 
if (mysqli_num_rows($result) > 0) {
    // 输出数据
    while($row = mysqli_fetch_assoc($result)) {
        echo "id: " . $row["id"]. " - Name: " . $row["firstname"]. " " . $row["lastname"]. "<br>";
    }
} else {
    echo "0 结果";
}
```
解释：
`mysql_fetch_array(data[,array_type])`
    **array_type类型**
    1. MYSQL_NUM 数字数组
    2. MYSQL_ASSOC 关联数组（键值对）
    3. MYSQL_BOTH 默认值，同时产生数字和关联数组
`mysql_fetch_row(data)` 产生数字数组
`mysql_fetch_assoc(data) `产生关联数组（键值对）



差不多是这样了，其他就是选择题慢慢磨和刷题即可












