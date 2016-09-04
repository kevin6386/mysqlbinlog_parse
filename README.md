# mysqlbinlog_parse 介绍
mysqlbinlog 对delete update insert 闪回操作，在tmp 生成对应闪回的库表的sql，不再担心delete等误操作。
MySQL binlog格式必须开启ROW格式
工具采用perl编写
##参数：
options :<br>
	-h,--help 				    # OUT : print help info   <br>
	-f	 			        # IN  : binlog file. [required]<br>
	--start-datetime 			# IN  : start datetime<br>
	--stop-datetime	 			# IN  : stop datetime<br>
	--start-position 			# IN  : start position<br>
	--stop-position	 			# IN  : stop position<br>
	-d, --database	 			# IN  : database, split comma<br>
	-t, --table		 		    # IN  : table, split comma. [required] set -d<br>
Sample :<br>
   mysqlbinlog_parse -f 'mysql-bin.xxx' #指定binlog文件<br>
   mysqlbinlog_parse -f 'mysql-bin.xxx'  --start-position=pos#开始点<br>
   mysqlbinlog_parse -f 'mysql-bin.xxx'  --start-position=pos --stop-position=pos#开始和结束<br>
   mysqlbinlog_parse -f 'mysql-bin.xxx'  -d 'db' #指定db<br>
   mysqlbinlog_parse -f 'mysql-bin.xxx'  -d 'db' -t 'table' #指定table<br>

##插入测试：<br>
(root:voole:)[cacti_data]> select * from c;<br>
Empty set (0.00 sec)<br>

(root:voole:)[cacti_data]> insert into c values(1,2);<br>
Query OK, 1 row affected (0.00 sec)<br>

(root:voole:)[cacti_data]> insert into c values(1,2);<br>
Query OK, 1 row affected (0.00 sec)<br>

(root:voole:)[cacti_data]> insert into c values(1,2);<br>
Query OK, 1 row affected (0.00 sec)<br>

(root:voole:)[cacti_data]> insert into c values(1,2);<br>
Query OK, 1 row affected (0.00 sec)<br>

(root:voole:)[cacti_data]> select * from c;<br>
+------+------+<br>
| a    | b    |<br>
+------+------+<br>
|    1 | 2    |<br>
|    1 | 2    |<br>
|    1 | 2    |<br>
|    1 | 2    |<br>
+------+------+<br>
4 rows in set (0.00 sec)<br>

##删除测试：
(root:voole:)[cacti_data]> select * from c;<br>
+------+------+<br>
| a    | b    |<br>
+------+------+<br>
|    1 | 2    |<br>
|    1 | 2    |<br>
|    1 | 2    |<br>
|    1 | 2    |<br>
|    1 | 2    |<br>
+------+------+<br>
5 rows in set (0.00 sec)<br>
(root:voole:)[cacti_data]> delete from c;<br>
Query OK, 5 rows affected (0.00 sec)<br>
(root:voole:)[cacti_data]> select * from c;<br>
Empty set (0.00 sec)<br>
<br>
##更新测试：
(root:voole:)[cacti_data]> update c set b='fdsafdas';<br>
Query OK, 4 rows affected (0.00 sec)<br>
Rows matched: 4  Changed: 4  Warnings: 0<br>
(root:voole:)[cacti_data]> select * from c;<br>
+------+----------+<br>
| a    | b        |<br>
+------+----------+<br>
|    1 | fdsafdas |<br>
|    1 | fdsafdas |<br>
|    1 | fdsafdas |<br>
|    1 | fdsafdas |<br>
+------+----------+<br>
4 rows in set (0.00 sec)<br>
(root:voole:)[cacti_data]> <br>

##执行恢复命令<br>
./mysqlbinlog_parse -f /data/mysql/mysql-bin.000019<br>
生成如下文件<br>
[root@localhost tmp]# ls<br>
binlog_out.txt  cacti_data.c-delete_to_insert.sql  cacti_data.c-insert_to_delete.sql  cacti_data.c-update_to_update.sql <br>

#恢复
##1、恢复插入操作<br>
cacti_data.c-insert_to_delete.sql，将插入的四行信息改为delete操作<br>
[root@localhost tmp]# cat cacti_data.c-insert_to_delete.sql <br>
DELETE FROM cacti_data.c<br>
WHERE<br>
a=1 and <br>
b='2';<br>
DELETE FROM cacti_data.c<br>
WHERE<br>
a=1 and <br>
b='2';<br>
DELETE FROM cacti_data.c<br>
WHERE<br>
a=1 and <br>
b='2';<br>
DELETE FROM cacti_data.c<br>
WHERE<br>
a=1 and <br>
b='2';<br>

##导入sql进行恢复：<br>

[root@localhost tmp]# mysql -uroot -p cacti_data <cacti_data.c-insert_to_delete.sql <br>
Enter password: <br>
[root@localhost tmp]# <br>

##查看
(root:voole:)[cacti_data]> select * from c;<br>
Empty set (0.00 sec)<br>
(root:voole:)[cacti_data]> <br>

已经不存在<br>

#2、测试删除操作<br>

cacti_data.c-delete_to_insert.sql：将delete操作生成insert<br>
[root@localhost tmp]# cat cacti_data.c-delete_to_insert.sql <br>
replace into cacti_data.c<br>
select<br>
1,<br>
'2';<br>
replace into cacti_data.c<br>
select<br>
1,<br>
'2';<br>
replace into cacti_data.c<br>
select<br>
1,<br>
'2';<br>
replace into cacti_data.c<br>
select<br>
1,<br>
'2';<br>
replace into cacti_data.c<br>
select<br>
1,<br>
'2';<br>
##查看表<br>
(root:voole:)[cacti_data]> select * from c;<br>
Empty set (0.00 sec)<br>
(root:voole:)[cacti_data]> <br>
##运行sql<br>
[root@localhost tmp]# mysql -uroot -p cacti_data <cacti_data.c-delete_to_insert.sql <br>
Enter password: <br>
[root@localhost tmp]# <br>
##查看<br>
(root:voole:)[cacti_data]> select * from c;<br>
+------+------+<br>
| a    | b    |<br>
+------+------+<br>
|    1 | 2    |<br>
|    1 | 2    |<br>
|    1 | 2    |<br>
|    1 | 2    |<br>
|    1 | 2    |<br>
+------+------+<br>
5 rows in set (0.00 sec)<br>
(root:voole:)[cacti_data]> <br>

##3、恢复更新操作<br>
 cacti_data.c-update_to_update.sql；将更新进行兑换生成更新操作<br>
[root@localhost tmp]# cat cacti_data.c-update_to_update.sql <br>
UPDATE cacti_data.c<br>
SET<br>
a=1,<br>
b='2'<br>
WHERE<br>
a=1 and <br>
b='fdsafdas';<br>
UPDATE cacti_data.c<br>
SET<br>
a=1,<br>
b='2'<br>
WHERE<br>
a=1 and <br>
b='fdsafdas';<br>
UPDATE cacti_data.c<br>
SET<br>
a=1,<br>
b='2'<br>
WHERE<br>
a=1 and <br>
b='fdsafdas';<br>
UPDATE cacti_data.c<br>
SET<br>
a=1,<br>
b='2'<br>
WHERE<br>
a=1 and <br>
b='fdsafdas';<br>
##查看表<br>
(root:voole:)[cacti_data]> select * from c;<br>
+------+----------+<br>
| a    | b        |<br>
+------+----------+<br>
|    1 | fdsafdas |<br>
|    1 | fdsafdas |<br>
|    1 | fdsafdas |<br>
|    1 | fdsafdas |<br>
|    1 | fdsafdas |<br>
+------+----------+<br>
5 rows in set (0.00 sec)<br>
(root:voole:)[cacti_data]> <br>
##导入sql<br>
[root@localhost tmp]# mysql -uroot -p cacti_data <cacti_data.c-update_to_update.sql <br>
Enter password: <br>
[root@localhost tmp]# <br>

##查看<br>
(root:voole:)[cacti_data]> select * from c;<br>
+------+------+<br>
| a    | b    |<br>
+------+------+<br>
|    1 | 2    |<br>
|    1 | 2    |<br>
|    1 | 2    |<br>
|    1 | 2    |<br>
|    1 | 2    |<br>
+------+------+<br>
5 rows in set (0.00 sec)<br>
(root:voole:)[cacti_data]> 

#4、恢复特定库或表<br>
如：cacti_data 下的kevin表。<br>
]# ./mysqlbinlog_parse -f /data/mysql/mysql-bin.000021 -d cacti_data -t kevin<br>



