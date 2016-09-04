# mysqlbinlog_parse 介绍
mysqlbinlog 对delete update insert 闪回操作，在tmp 生成对应闪回的库表的sql，不再担心delete等误操作。
MySQL binlog格式必须开启ROW格式
工具采用perl编写
参数：
options :
	-h,--help			    # OUT : print help info   
	-f				        # IN  : binlog file. [required]
	--start-datetime		# IN  : start datetime
	--stop-datetime			# IN  : stop datetime
	--start-position		# IN  : start position
	--stop-position			# IN  : stop position
	-d, --database			# IN  : database, split comma
	-t, --table			    # IN  : table, split comma. [required] set -d
Sample :
   mysqlbinlog_parse -f 'mysql-bin.xxx' #指定binlog文件
   mysqlbinlog_parse -f 'mysql-bin.xxx'  --start-position=pos#开始点
   mysqlbinlog_parse -f 'mysql-bin.xxx'  --start-position=pos --stop-position=pos#开始和结束
   mysqlbinlog_parse -f 'mysql-bin.xxx'  -d 'db' #指定db
   mysqlbinlog_parse -f 'mysql-bin.xxx'  -d 'db' -t 'table' #指定table

插入测试：
(root:voole:)[cacti_data]> select * from c;
Empty set (0.00 sec)

(root:voole:)[cacti_data]> insert into c values(1,2);
Query OK, 1 row affected (0.00 sec)

(root:voole:)[cacti_data]> insert into c values(1,2);
Query OK, 1 row affected (0.00 sec)

(root:voole:)[cacti_data]> insert into c values(1,2);
Query OK, 1 row affected (0.00 sec)

(root:voole:)[cacti_data]> insert into c values(1,2);
Query OK, 1 row affected (0.00 sec)

(root:voole:)[cacti_data]> select * from c;
+------+------+
| a    | b    |
+------+------+
|    1 | 2    |
|    1 | 2    |
|    1 | 2    |
|    1 | 2    |
+------+------+
4 rows in set (0.00 sec)

删除测试：
(root:voole:)[cacti_data]> select * from c;
+------+------+
| a    | b    |
+------+------+
|    1 | 2    |
|    1 | 2    |
|    1 | 2    |
|    1 | 2    |
|    1 | 2    |
+------+------+
5 rows in set (0.00 sec)
(root:voole:)[cacti_data]> delete from c;
Query OK, 5 rows affected (0.00 sec)
(root:voole:)[cacti_data]> select * from c;
Empty set (0.00 sec)

更新测试：
(root:voole:)[cacti_data]> update c set b='fdsafdas';
Query OK, 4 rows affected (0.00 sec)
Rows matched: 4  Changed: 4  Warnings: 0
(root:voole:)[cacti_data]> select * from c;
+------+----------+
| a    | b        |
+------+----------+
|    1 | fdsafdas |
|    1 | fdsafdas |
|    1 | fdsafdas |
|    1 | fdsafdas |
+------+----------+
4 rows in set (0.00 sec)
(root:voole:)[cacti_data]> 

执行恢复命令
./mysqlbinlog_parse -f /data/mysql/mysql-bin.000019
生成如下文件
[root@localhost tmp]# ls
binlog_out.txt  cacti_data.c-delete_to_insert.sql  cacti_data.c-insert_to_delete.sql  cacti_data.c-update_to_update.sql 

恢复
1、恢复插入操作
cacti_data.c-insert_to_delete.sql，将插入的四行信息改为delete操作
[root@localhost tmp]# cat cacti_data.c-insert_to_delete.sql 
DELETE FROM cacti_data.c
WHERE
a=1 and 
b='2';
DELETE FROM cacti_data.c
WHERE
a=1 and 
b='2';
DELETE FROM cacti_data.c
WHERE
a=1 and 
b='2';
DELETE FROM cacti_data.c
WHERE
a=1 and 
b='2';

导入sql进行恢复：

[root@localhost tmp]# mysql -uroot -p cacti_data <cacti_data.c-insert_to_delete.sql 
Enter password: 
[root@localhost tmp]# 

查看
(root:voole:)[cacti_data]> select * from c;
Empty set (0.00 sec)
(root:voole:)[cacti_data]> 

已经不存在

2、测试删除操作

cacti_data.c-delete_to_insert.sql：将delete操作生成insert
[root@localhost tmp]# cat cacti_data.c-delete_to_insert.sql 
replace into cacti_data.c
select
1,
'2';
replace into cacti_data.c
select
1,
'2';
replace into cacti_data.c
select
1,
'2';
replace into cacti_data.c
select
1,
'2';
replace into cacti_data.c
select
1,
'2';
查看表
(root:voole:)[cacti_data]> select * from c;
Empty set (0.00 sec)
(root:voole:)[cacti_data]> 
运行sql
[root@localhost tmp]# mysql -uroot -p cacti_data <cacti_data.c-delete_to_insert.sql 
Enter password: 
[root@localhost tmp]# 
查看
(root:voole:)[cacti_data]> select * from c;
+------+------+
| a    | b    |
+------+------+
|    1 | 2    |
|    1 | 2    |
|    1 | 2    |
|    1 | 2    |
|    1 | 2    |
+------+------+
5 rows in set (0.00 sec)
(root:voole:)[cacti_data]> 

3、恢复更新操作
 cacti_data.c-update_to_update.sql；将更新进行兑换生成更新操作
[root@localhost tmp]# cat cacti_data.c-update_to_update.sql 
UPDATE cacti_data.c
SET
a=1,
b='2'
WHERE
a=1 and 
b='fdsafdas';
UPDATE cacti_data.c
SET
a=1,
b='2'
WHERE
a=1 and 
b='fdsafdas';
UPDATE cacti_data.c
SET
a=1,
b='2'
WHERE
a=1 and 
b='fdsafdas';
UPDATE cacti_data.c
SET
a=1,
b='2'
WHERE
a=1 and 
b='fdsafdas';
查看表
(root:voole:)[cacti_data]> select * from c;
+------+----------+
| a    | b        |
+------+----------+
|    1 | fdsafdas |
|    1 | fdsafdas |
|    1 | fdsafdas |
|    1 | fdsafdas |
|    1 | fdsafdas |
+------+----------+
5 rows in set (0.00 sec)
(root:voole:)[cacti_data]> 
导入sql
[root@localhost tmp]# mysql -uroot -p cacti_data <cacti_data.c-update_to_update.sql 
Enter password: 
[root@localhost tmp]# 

查看
(root:voole:)[cacti_data]> select * from c;
+------+------+
| a    | b    |
+------+------+
|    1 | 2    |
|    1 | 2    |
|    1 | 2    |
|    1 | 2    |
|    1 | 2    |
+------+------+
5 rows in set (0.00 sec)
(root:voole:)[cacti_data]> 

4、恢复特定库或表
如：cacti_data 下的kevin表。
]# ./mysqlbinlog_parse -f /data/mysql/mysql-bin.000021 -d cacti_data -t kevin



