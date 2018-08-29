1. 修改PostgreSQL数据库默认用户postgres的密码
PostgreSQL数据库创建一个postgres用户作为数据库的管理员，密码随机，所以需要修改密码，方式如下：

步骤一：登录PostgreSQL
1 sudo -u postgres psql
步骤二：修改登录PostgreSQL密码

1 ALTER USER postgres WITH PASSWORD 'postgres';

CREATE USER chater WITH PASSWORD '123456'; 

修改pg的配置文件：
# "local" is for Unix domain socket connections only
local   all             all                                     trust
# IPv4 local connections:
host    all             all             127.0.0.1/32           trust
重启pg服务


连接数据库, 默认的用户和数据库是postgres
psql -U chater -d chitchat

切换数据库,相当于mysql的use dbname
\c dbname
列举数据库，相当于mysql的show databases
\l
列举表，相当于mysql的show tables
\dt
查看表结构，相当于desc tblname,show columns from tbname
\d tblname

\di 查看索引 

创建数据库： 
create database [数据库名]; 
删除数据库： 
drop database [数据库名];  
*重命名一个表： 
alter table [表名A] rename to [表名B]; 
*删除一个表： 
drop table [表名]; 
*在已有的表里添加字段： 
alter table [表名] add column [字段名] [类型]; 
*删除表中的字段： 
alter table [表名] drop column [字段名]; 
*重命名一个字段：  
alter table [表名] rename column [字段名A] to [字段名B]; 
*给一个字段设置缺省值：  
alter table [表名] alter column [字段名] set default [新的默认值];
*去除缺省值：  
alter table [表名] alter column [字段名] drop default; 
在表中插入数据： 
insert into 表名 ([字段名m],[字段名n],......) values ([列m的值],[列n的值],......); 
修改表中的某行某列的数据： 
update [表名] set [目标字段名]=[目标值] where [该行特征]; 
删除表中某行数据： 
delete from [表名] where [该行特征]; 
delete from [表名];--删空整个表 
创建表： 
create table ([字段名1] [类型1] ;,[字段名2] [类型2],......<,primary key (字段名m,字段名n,...)>;); 
\copyright     显示 PostgreSQL 的使用和发行条款
\encoding [字元编码名称]
                 显示或设定用户端字元编码
\h [名称]      SQL 命令语法上的说明，用 * 显示全部命令
\prompt [文本] 名称
                 提示用户设定内部变数
\password [USERNAME]
                 securely change the password for a user
\q             退出 psql



可以使用pg_dump和pg_dumpall来完成。比如备份sales数据库： 
pg_dump drupal>/opt/Postgresql/backup/1.bak


附带一些指定给postgresql用户的常用命令:

默认用户
postgres安装完成后，会自动在操作系统和postgres数据库中分别创建一个名为postgres的用户以及一个同样名为postgres的数据库。

登录
方式1:指定参数登录
psql -U username -d database_name -h host -W

参数含义: -U指定用户 -d要连接的数据库 -h要连接的主机 -W提示输入密码。

方式2:切换到postgres同名用户后登录
su username
psql
当不指定参数时psql使用操作系统当前用户的用户名作为postgres的登录用户名和要连接的数据库名。所以在PostgreSQL安装完成后可以通过以上方式登录。

创建用户
方式1:在系统命令行中使用createuser命令中创建
createuser username 

方式2:在PostgresSQL命令行中使用CREATE ROLE指令创建
CREATE ROLE rolename;

方式3:在PostgresSQL命令行中使用CREATE USER指令创建
CREATE USER username;
CREATE USER和CREATE ROLE的区别在于，CREATE USER指令创建的用户默认是有登录权限的，而CREATE ROLE没有。

\du 指令显示用户和用户的用户属性

创建用户时设定用户属性
基本语法格式
CREATE ROLE role_name WITH  optional_permissions;

示例:在创建用户时设定登录权限。
CREATE ROLE username WITH LOGIN;

可以通过\h CREATE ROLE指令查看全部可设置的管理权限

修改用户属性
修改权限的命令格式
ALTER ROLE username WITH attribute_options;

例如:可通过以下方式禁止用户登录
ALTER ROLE username WITH NOLOGIN;

设置访问权限
语法格式如下:
GRANT permission_type ON table_name TO role_name;

实例:
GRANT UPDATE ON demo TO demo_role; --赋予demo_role demo表的update权限
GRANT SELECT ON ALL TABLES IN SCHEMA PUBLIC to demo_role; --赋予demo_role所有表的SELECT权限
特殊符号:ALL代表所访问权限，PUBLIC代表所有用户
GRANT ALL ON demo TO demo_role; --赋给用户所有权限
GRANT SELECT ON demo TO PUBLIC; --将SELECT权限赋给所有用户
\z或\dp指令显示用户访问权限。
\h GRANT显示所有可设置的访问权限

撤销用户访问权限
语法格式如下:
REVOKE permission_type ON table_name FROM user_name;
其中permission_type和table_name含义与GRANT指令中相同。

用户组
在postgres中用户实际上是role，同时组也是role。 包含其他role的role就是组。

创建组示例:
CREATE ROLE temporary_users;
GRANT temporary_users TO demo_role;
GRANT temporary_users TO test_user;

切换ROLE
SET ROLE role_name; --切换到role_name用户
RESET ROLE; --切换回最初的role

INHERIT权限：该属性使组成员拥有组的所有权限
ALTER ROLE test_user INHERIT;

删除用户和组
删除用户和组很简单:
DROP ROLE role_name;
DROP ROLE IF EXISTS role_name;

删除组role只会删除组的role本身，组的成员并不会被删除

https://blog.csdn.net/u010856284/article/details/70142810