---
title: Oracle 12c特性解读-容器数据库和灾备-03 创建与管理CDB和PDB
date: 2017-05-17
tags:
- oracle
- 12c
---

## 手工创建cdb数据库(create database语句)
### 创建前的准备工作
``` perl
# 创建密码文件
orapwd file=$ORACLE_HOME/dbs/orapworcl1 password=oracle force=y format=12

# 创建参数文件
vi spfileorcl1.ora
#---------------------------------------------------
db_name=orcl1
sga_target=2048M
db_create_file_dest='/u01/app/oracle/oradata/orcl1'
enable_pluggable_database=true
#---------------------------------------------------

# 创建所需目录
mkdir -p /u01/app/oracle/oradata/orcl1
```

<!-- more -->

### 手工创建数据库
``` perl
# OMF方式创建数据库
export ORACLE_SID=orcl1
sqlplus / as sysdba
startup nomount
CREATE DATABASE orcl1
    USER SYS IDENTIFIED BY oracle
    USER SYSTEM IDENTIFIED BY oracle
    EXTENT MANAGEMENT LOCAL
    DEFAULT TABLESPACE users
    DEFAULT TEMPORARY TABLESPACE temp
    UNDO TABLESPACE undotbs1
    ENABLE PLUGGABLE DATABASE
    SEED
        SYSTEM DATAFILES SIZE 125M AUTOEXTEND ON NEXT 10M MAXSIZE UNLIMITED
        SYSAUX DATAFILES SIZE 100M;

show con_name

CON_NAME
------------------------------
CDB$ROOT

show pdbs;

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO

# 非OMF方式创建数据库
CREATE DATABASE newcdb
  USER SYS IDENTIFIED BY sys_password
  USER SYSTEM IDENTIFIED BY system_password
  LOGFILE GROUP 1 ('/u01/logs/my/redo01a.log','/u02/logs/my/redo01b.log')
             SIZE 100M BLOCKSIZE 512,
          GROUP 2 ('/u01/logs/my/redo02a.log','/u02/logs/my/redo02b.log')
             SIZE 100M BLOCKSIZE 512,
          GROUP 3 ('/u01/logs/my/redo03a.log','/u02/logs/my/redo03b.log')
             SIZE 100M BLOCKSIZE 512
  MAXLOGHISTORY 1
  MAXLOGFILES 16
  MAXLOGMEMBERS 3
  MAXDATAFILES 1024
  CHARACTER SET AL32UTF8
  NATIONAL CHARACTER SET AL16UTF16
  EXTENT MANAGEMENT LOCAL
  DATAFILE '/u01/app/oracle/oradata/newcdb/system01.dbf'
    SIZE 700M REUSE AUTOEXTEND ON NEXT 10240K MAXSIZE UNLIMITED
  SYSAUX DATAFILE '/u01/app/oracle/oradata/newcdb/sysaux01.dbf'
    SIZE 550M REUSE AUTOEXTEND ON NEXT 10240K MAXSIZE UNLIMITED
  DEFAULT TABLESPACE deftbs
    DATAFILE '/u01/app/oracle/oradata/newcdb/deftbs01.dbf'
    SIZE 500M REUSE AUTOEXTEND ON MAXSIZE UNLIMITED
  DEFAULT TEMPORARY TABLESPACE tempts1
    TEMPFILE '/u01/app/oracle/oradata/newcdb/temp01.dbf'
    SIZE 20M REUSE AUTOEXTEND ON NEXT 640K MAXSIZE UNLIMITED
  UNDO TABLESPACE undotbs1
    DATAFILE '/u01/app/oracle/oradata/newcdb/undotbs01.dbf'
    SIZE 200M REUSE AUTOEXTEND ON NEXT 5120K MAXSIZE UNLIMITED
  ENABLE PLUGGABLE DATABASE
    SEED
    FILE_NAME_CONVERT = ('/u01/app/oracle/oradata/newcdb/',
                         '/u01/app/oracle/oradata/pdbseed/')
    SYSTEM DATAFILES SIZE 125M AUTOEXTEND ON NEXT 10M MAXSIZE UNLIMITED
    SYSAUX DATAFILES SIZE 100M
  USER_DATA TABLESPACE usertbs
    DATAFILE '/u01/app/oracle/oradata/pdbseed/usertbs01.dbf'
    SIZE 200M REUSE AUTOEXTEND ON MAXSIZE UNLIMITED;
```

### 创建后的补充工作
``` perl
cat $ORACLE_HOME/rdbms/admin/catcdb.sql
vi /etc/oratab
#-----------------------------------------------------------------------------------------
orcl1:/u01/app/oracle/product/12.2.0/db_1:Y
#-----------------------------------------------------------------------------------------
export ORACLE_SID=orcl1
export PERL5LIB=$ORACLE_HOME/rdbms/admin:$PERL5LIB
export PATH=$ORACLE_HOME/bin:$ORACLE_HOME/perl/bin:$PATH
cd $ORACLE_HOME/perl/lib/5.22.0/x86_64-linux-thread-multi/Hash
# 建软链接，否则会出现找不到文件提示
ln -s Util.pm util.pm
sqlplus / as sysdba
@?/rdbms/admin/catcdb.sql
#-----------------------------------------------------------------------------------------
......
SQL> host perl -I &&rdbms_admin &&rdbms_admin_catcdb --logDirectory &&1 --logFilename &&2
Enter value for 1: /home/oracle           # 输入日志目录
Enter value for 2: /home/oracle/a.log     # 输入日志文件名字
Enter new password for SYS: oracle
Enter new password for SYSTEM: oracle
Enter temporary tablespace name: temp
No options to container mapping specified, no options will be installed in any containers
catcon: ALL catcon-related output will be written to [/home/oracle/catalog_catcon_339.lst]
catcon: See [/home/oracle/catalog*.log] files for output generated by scripts
catcon: See [/home/oracle/catalog_*.lst] files for spool files, if any
...
catcon: ALL catcon-related output will be written to [/home/oracle/utlrp_catcon_2672.lst]
catcon: See [/home/oracle/utlrp*.log] files for output generated by scripts
catcon: See [/home/oracle/utlrp_*.lst] files for spool files, if any
catcon.pl: completed successfully
#-----------------------------------------------------------------------------------------

create spfile from pfile;
```

## DBCA创建CDB
``` perl
dbca -silent -createDatabase -templateName $ORACLE_HOME/assistants/dbca/templates/General_Purpose.dbc \
-gdbname orcl2 -sid orcl2 -characterSet ZHS16GBK -sysPassword oracle -systemPassword oracle \
-createAsContainerDatabase true
#-----------------------------------------------------------------------------------------
[WARNING] [DBT-06208] The 'SYS' password entered does not conform to the Oracle recommended standards.
   CAUSE: 
a. Oracle recommends that the password entered should be at least 8 characters in length, contain at least 1 uppercase character, 1 lower case character and 1 digit [0-9].
b.The password entered is a keyword that Oracle does not recommend to be used as password
   ACTION: Specify a strong password. If required refer Oracle documentation for guidelines.
[WARNING] [DBT-06208] The 'SYSTEM' password entered does not conform to the Oracle recommended standards.
   CAUSE: 
a. Oracle recommends that the password entered should be at least 8 characters in length, contain at least 1 uppercase character, 1 lower case character and 1 digit [0-9].
b.The password entered is a keyword that Oracle does not recommend to be used as password
   ACTION: Specify a strong password. If required refer Oracle documentation for guidelines.
Copying database files
1% complete
2% complete
18% complete
33% complete
Creating and starting Oracle instance
35% complete
40% complete
41% complete
42% complete
46% complete
51% complete
52% complete
53% complete
55% complete
Completing Database Creation
56% complete
58% complete
59% complete
62% complete
65% complete
66% complete
Executing Post Configuration Actions
100% complete
Look at the log file "/u01/app/oracle/cfgtoollogs/dbca/orcl2/orcl2.log" for further details
#-----------------------------------------------------------------------------------------
```

## 创建3个PDB，用多种方法实现
### 通过种子容器创建pdb
``` perl
export ORACLE_SID=orcl2
sqlplus / as sysdba
# 需要指定文件存放位置，可通过如下两种方式
create pluggable database pdb1 admin user pdb1 identified by oracle
 file_name_convert=('/u01/app/oracle/oradata/orcl2/pdbseed','/u01/app/oracle/oradata/orcl2/pdb1');

# 或者修改pdb_file_name_convert参数
alter session set pdb_file_name_convert='/u01/app/oracle/oradata/orcl2/pdbseed','/u01/app/oracle/oradata/orcl2/pdb2';
create pluggable database pdb2 admin user pdb2 identified by oracle;
```

### 克隆一个本地PDB
``` perl
alter pluggable database pdb2 open;
create pluggable database pdb3 from pdb2
 file_name_convert=('/u01/app/oracle/oradata/orcl2/pdb2','/u01/app/oracle/oradata/orcl2/pdb3');

#查看PDB
show pdbs;

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 PDB1                           READ WRITE NO
         4 PDB2                           READ WRITE NO
         6 PDB3                           MOUNTED
```

### 使用DBCA图形化方式创建PDB
``` perl
dbca
```

![](http://oligvdnzp.bkt.clouddn.com/0519_oracle12c_createpdb_01.png)
![](http://oligvdnzp.bkt.clouddn.com/0519_oracle12c_createpdb_02.png)
![](http://oligvdnzp.bkt.clouddn.com/0519_oracle12c_createpdb_03.png)
![](http://oligvdnzp.bkt.clouddn.com/0519_oracle12c_createpdb_04.png)
![](http://oligvdnzp.bkt.clouddn.com/0519_oracle12c_createpdb_05.png)
![](http://oligvdnzp.bkt.clouddn.com/0519_oracle12c_createpdb_06.png)
![](http://oligvdnzp.bkt.clouddn.com/0519_oracle12c_createpdb_07.png)
![](http://oligvdnzp.bkt.clouddn.com/0519_oracle12c_createpdb_08.png)
![](http://oligvdnzp.bkt.clouddn.com/0519_oracle12c_createpdb_09.png)

## 熟练掌握PDB的基本管理
### 容器查看和切换
``` perl
select name, decode(cdb, 'YES', 'Multitenant Option enabled', 'Regular 12c Database: ') "Multitenant Option", 
 open_mode, con_id from v$database;

NAME      Multitenant Option         OPEN_MODE                CON_ID
--------- -------------------------- -------------------- ----------
ORCL2     Multitenant Option enabled READ WRITE                    0

show pdbs;

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 PDB1                           READ WRITE NO
         4 PDB2                           READ WRITE NO
         5 PDB4                           READ WRITE NO
         6 PDB3                           READ WRITE NO

alter session set container=pdb1;
```

### 启停PDB数据库
``` perl
# 以下以PDB3为例
alter pluggable database pdb3 open;
alter pluggable database pdb3 close;

# 或切换到pdb3内执行
alter session set container=pdb3;
show pdbs

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         6 PDB3                           MOUNTED

alter database open;
shutdown immediate
```

### 删除PDB数据库
``` perl
conn / as sysdba
alter pluggable database pdb4 close;
drop pluggable database pdb4 including datafiles;
```

## 完成CDB,PDB的网络监听配置
### 配置LISTENER
``` perl
cd $ORACLE_HOME/network/admin
vi listener.ora
#-----------------------------------------------------------------------------------------
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = orcl12)(PORT = 1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
    )
  )

SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = pdb1)
      (ORACLE_HOME = /u01/app/oracle/product/12.2.0/db_1)
      (SID_NAME = orcl2)
    )
    (SID_DESC =
      (GLOBAL_DBNAME = pdb2)
      (ORACLE_HOME = /u01/app/oracle/product/12.2.0/db_1)
      (SID_NAME = orcl2)
    )
    (SID_DESC =
      (GLOBAL_DBNAME = pdb3)
      (ORACLE_HOME = /u01/app/oracle/product/12.2.0/db_1)
      (SID_NAME = orcl2)
    )
)
#-----------------------------------------------------------------------------------------
```

### 配置tnsnames
``` perl
cat /etc/hosts
#-----------------------------------------------------------------------------------------
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.245.231.205 orcl12
#-----------------------------------------------------------------------------------------

vi tnsnames.ora
#-----------------------------------------------------------------------------------------
ORCL2 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 10.245.231.205)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = orcl2)
    )
  )

PDB1 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 10.245.231.205)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = pdb1)
    )
  )

PDB2 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 10.245.231.205)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = pdb2)
    )
  )

PDB3 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 10.245.231.205)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = pdb3)
    )
  )
#-----------------------------------------------------------------------------------------

# 验证配置
tnsping pdb1
#-----------------------------------------------------------------------------------------
TNS Ping Utility for Linux: Version 12.2.0.1.0 - Production on 19-MAY-2017 16:41:53

Copyright (c) 1997, 2016, Oracle.  All rights reserved.

Used parameter files:


Used TNSNAMES adapter to resolve the alias
Attempting to contact (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = 10.245.231.205)(PORT = 1521)) (CONNECT_DATA = (SERVER = DEDICATED) (SERVICE_NAME = pdb1)))
OK (0 msec)
#-----------------------------------------------------------------------------------------

sqlplus pdb1/oracle@pdb1
#-----------------------------------------------------------------------------------------
SQL*Plus: Release 12.2.0.1.0 Production on Fri May 19 16:42:08 2017

Copyright (c) 1982, 2016, Oracle.  All rights reserved.

Last Successful login time: Fri May 19 2017 16:40:02 +08:00

Connected to:
Oracle Database 12c Enterprise Edition Release 12.2.0.1.0 - 64bit Production

SQL> show con_name

CON_NAME
------------------------------
PDB1
#-----------------------------------------------------------------------------------------
```

## 杂记
### 手工模拟DBCA创建CDB数据库
``` perl
# 只生成建库脚本
dbca -silent -templateName $ORACLE_HOME/assistants/dbca/templates/General_Purpose.dbc -gdbname orcl2 -sid orcl2 \
-characterSet ZHS16GBK -sysPassword oracle -systemPassword oracle -createAsContainerDatabase true -generateScripts
#-----------------------------------------------------------------------------------------
Database creation script generation
1% complete
2% complete
18% complete
33% complete
35% complete
40% complete
41% complete
42% complete
46% complete
51% complete
52% complete
55% complete
56% complete
60% complete
63% complete
66% complete
100% complete
Look at the log file "/u01/app/oracle/admin/orcl2/scripts/orcl2.log" for further details.
#-----------------------------------------------------------------------------------------

# 查看分析建库脚本
cd /u01/app/oracle/admin/orcl2/scripts
ll
#-----------------------------------------------------------------------------------------
total 18348
-rw-r----- 1 oracle oinstall     1891 May 19 08:59 cloneDBCreation.sql
-rw-r----- 1 oracle oinstall      833 May 19 08:59 CloneRmanRestore.sql
-rw-r----- 1 oracle oinstall     2227 May 19 08:59 init.ora
-rw-r----- 1 oracle oinstall     2238 May 19 08:59 initorcl2TempOMF.ora
-rw-r----- 1 oracle oinstall     2343 May 19 08:59 initorcl2Temp.ora
-rw-r----- 1 oracle oinstall      548 May 19 08:59 lockAccount.sql
-rw-r----- 1 oracle oinstall      521 May 19 08:59 orcl2.log
-rwxr-xr-x 1 oracle oinstall      785 May 19 08:59 orcl2.sh
-rwxr-xr-x 1 oracle oinstall      612 May 19 08:59 orcl2.sql
-rw-r----- 1 oracle oinstall     3635 May 19 08:59 plug_PDBSeed.sql
-rw-r----- 1 oracle oinstall      810 May 19 08:59 postDBCreation.sql
-rw-r----- 1 oracle oinstall     1852 May 19 08:59 postScripts.sql
-rw-r----- 1 oracle oinstall      101 May 19 08:59 rmanPDBCleanUpDatafiles.sql
-rw-r----- 1 oracle oinstall      377 May 19 08:59 rmanPDBRestoreDatafiles.sql
-rwxr-xr-x 1 oracle oinstall      560 May 19 08:59 rmanRestoreDatafiles.sql
-rw-r----- 1 oracle oinstall 18726912 May 19 08:59 tempControl.ctl
#-----------------------------------------------------------------------------------------

cat orcl2.sh
#-----------------------------------------------------------------------------------------
#!/bin/sh

OLD_UMASK=`umask`
umask 0027
mkdir -p /u01/app/oracle
mkdir -p /u01/app/oracle/admin/orcl2/adump
mkdir -p /u01/app/oracle/admin/orcl2/dpdump
mkdir -p /u01/app/oracle/admin/orcl2/pfile
mkdir -p /u01/app/oracle/audit
mkdir -p /u01/app/oracle/cfgtoollogs/dbca/orcl2
mkdir -p /u01/app/oracle/oradata/orcl2
mkdir -p /u01/app/oracle/oradata/orcl2/pdbseed
mkdir -p /u01/app/oracle/product/12.2.0/db_1/dbs
umask ${OLD_UMASK}
PERL5LIB=$ORACLE_HOME/rdbms/admin:$PERL5LIB; export PERL5LIB
ORACLE_SID=orcl2; export ORACLE_SID
PATH=$ORACLE_HOME/bin:$ORACLE_HOME/perl/bin:$PATH; export PATH
echo You should Add this entry in the /etc/oratab: orcl2:/u01/app/oracle/product/12.2.0/db_1:Y
/u01/app/oracle/product/12.2.0/db_1/bin/sqlplus /nolog @/u01/app/oracle/admin/orcl2/scripts/orcl2.sql
#-----------------------------------------------------------------------------------------

cat orcl2.sql
#-----------------------------------------------------------------------------------------
set verify off
ACCEPT sysPassword CHAR PROMPT 'Enter new password for SYS: ' HIDE
ACCEPT systemPassword CHAR PROMPT 'Enter new password for SYSTEM: ' HIDE
host /u01/app/oracle/product/12.2.0/db_1/bin/orapwd file=/u01/app/oracle/product/12.2.0/db_1/dbs/orapworcl2 force=y format=12
@/u01/app/oracle/admin/orcl2/scripts/CloneRmanRestore.sql
@/u01/app/oracle/admin/orcl2/scripts/cloneDBCreation.sql
@/u01/app/oracle/admin/orcl2/scripts/plug_PDBSeed.sql
@/u01/app/oracle/admin/orcl2/scripts/postScripts.sql
@/u01/app/oracle/admin/orcl2/scripts/lockAccount.sql
@/u01/app/oracle/admin/orcl2/scripts/postDBCreation.sql
#-----------------------------------------------------------------------------------------
```

### 通过控制文件查看cdb的结构
``` sql
alter database backup controlfile to trace;
col VALUE for a80
select INST_ID,VALUE from v$diag_info where name='Default Trace File';

   INST_ID VALUE
---------- --------------------------------------------------------------------------------
         1 /u01/app/oracle/diag/rdbms/orcl2/orcl2/trace/orcl2_ora_16951.trc
```
查看生成的Trace File
``` perl
cat /u01/app/oracle/diag/rdbms/orcl2/orcl2/trace/orcl2_ora_16951.trc
#-----------------------------------------------------------------------------------------
Trace file /u01/app/oracle/diag/rdbms/orcl2/orcl2/trace/orcl2_ora_16951.trc
Oracle Database 12c Enterprise Edition Release 12.2.0.1.0 - 64bit Production
Build label:    RDBMS_12.2.0.1.0_LINUX.X64_170125
ORACLE_HOME:    /u01/app/oracle/product/12.2.0/db_1
System name:    Linux
Node name:      orcl12
Release:        4.1.12-61.1.18.el7uek.x86_64
Version:        #2 SMP Fri Nov 4 15:48:30 PDT 2016
Machine:        x86_64
Instance name: orcl2
Redo thread mounted by this instance: 1
Oracle process number: 27
Unix process pid: 16951, image: oracle@orcl12 (TNS V1-V3)


*** 2017-05-19T16:19:27.267587+08:00 (PDB4(5))
*** SESSION ID:(397.57468) 2017-05-19T16:19:27.267655+08:00
*** CLIENT ID:() 2017-05-19T16:19:27.267666+08:00
*** SERVICE NAME:(SYS$USERS) 2017-05-19T16:19:27.267675+08:00
*** MODULE NAME:(sqlplus@orcl12 (TNS V1-V3)) 2017-05-19T16:19:27.267685+08:00
*** ACTION NAME:() 2017-05-19T16:19:27.267694+08:00
*** CLIENT DRIVER:(SQL*PLUS) 2017-05-19T16:19:27.267703+08:00
*** CONTAINER ID:(5) 2017-05-19T16:19:27.267712+08:00
 
JIT: pid 16951 requesting stop

*** 2017-05-19T16:21:04.923126+08:00 (CDB$ROOT(1))
-- The following are current System-scope REDO Log Archival related
-- parameters and can be included in the database initialization file.
--
-- LOG_ARCHIVE_DEST=''
-- LOG_ARCHIVE_DUPLEX_DEST=''
--
-- LOG_ARCHIVE_FORMAT=%t_%s_%r.dbf
--
-- DB_UNIQUE_NAME="orcl2"
--
-- LOG_ARCHIVE_CONFIG='SEND, RECEIVE, NODG_CONFIG'
-- LOG_ARCHIVE_MAX_PROCESSES=4
-- STANDBY_FILE_MANAGEMENT=MANUAL
-- STANDBY_ARCHIVE_DEST=?#/dbs/arch
-- FAL_CLIENT=''
-- FAL_SERVER=''
--
-- LOG_ARCHIVE_DEST_1='LOCATION=/u01/app/oracle/product/12.2.0/db_1/dbs/arch'
-- LOG_ARCHIVE_DEST_1='MANDATORY NOREOPEN NODELAY'
-- LOG_ARCHIVE_DEST_1='ARCH NOAFFIRM NOVERIFY SYNC'
-- LOG_ARCHIVE_DEST_1='NOREGISTER NOALTERNATE NODEPENDENCY'
-- LOG_ARCHIVE_DEST_1='NOMAX_FAILURE NOQUOTA_SIZE NOQUOTA_USED NODB_UNIQUE_NAME'
-- LOG_ARCHIVE_DEST_1='VALID_FOR=(PRIMARY_ROLE,ONLINE_LOGFILES)'
-- LOG_ARCHIVE_DEST_STATE_1=ENABLE
--
-- Below are two sets of SQL statements, each of which creates a new
-- control file and uses it to open the database. The first set opens
-- the database with the NORESETLOGS option and should be used only if
-- the current versions of all online logs are available. The second
-- set opens the database with the RESETLOGS option and should be used
-- if online logs are unavailable.
-- The appropriate set of statements can be copied from the trace into
-- a script file, edited as necessary, and executed when there is a
-- need to re-create the control file.
--
--     Set #1. NORESETLOGS case
--
-- The following commands will create a new control file and use it
-- to open the database.
-- Data used by Recovery Manager will be lost.
-- Additional logs may be required for media recovery of offline
-- Use this only if the current versions of all online logs are
-- available.
-- After mounting the created controlfile, the following SQL
-- statement will place the database in the appropriate
-- protection mode:
--  ALTER DATABASE SET STANDBY DATABASE TO MAXIMIZE PERFORMANCE
STARTUP NOMOUNT
CREATE CONTROLFILE REUSE DATABASE "ORCL2" NORESETLOGS  NOARCHIVELOG
    MAXLOGFILES 16
    MAXLOGMEMBERS 3
    MAXDATAFILES 1024
    MAXINSTANCES 8
    MAXLOGHISTORY 292
LOGFILE
  GROUP 1 '/u01/app/oracle/oradata/orcl2/redo01.log'  SIZE 200M BLOCKSIZE 512,
  GROUP 2 '/u01/app/oracle/oradata/orcl2/redo02.log'  SIZE 200M BLOCKSIZE 512,
  GROUP 3 '/u01/app/oracle/oradata/orcl2/redo03.log'  SIZE 200M BLOCKSIZE 512
-- STANDBY LOGFILE
DATAFILE
  '/u01/app/oracle/oradata/orcl2/system01.dbf',
  '/u01/app/oracle/oradata/orcl2/sysaux01.dbf',
  '/u01/app/oracle/oradata/orcl2/undotbs01.dbf',
  '/u01/app/oracle/oradata/orcl2/pdbseed/system01.dbf',
  '/u01/app/oracle/oradata/orcl2/pdbseed/sysaux01.dbf',
  '/u01/app/oracle/oradata/orcl2/users01.dbf',
  '/u01/app/oracle/oradata/orcl2/pdbseed/undotbs01.dbf',
  '/u01/app/oracle/oradata/orcl2/pdb1/system01.dbf',
  '/u01/app/oracle/oradata/orcl2/pdb1/sysaux01.dbf',
  '/u01/app/oracle/oradata/orcl2/pdb1/undotbs01.dbf',
  '/u01/app/oracle/oradata/orcl2/pdb2/system01.dbf',
  '/u01/app/oracle/oradata/orcl2/pdb2/sysaux01.dbf',
  '/u01/app/oracle/oradata/orcl2/pdb2/undotbs01.dbf',
  '/u01/app/oracle/oradata/orcl2/pdb3/system01.dbf',
  '/u01/app/oracle/oradata/orcl2/pdb3/sysaux01.dbf',
  '/u01/app/oracle/oradata/orcl2/pdb3/undotbs01.dbf'
CHARACTER SET ZHS16GBK
;
-- Commands to re-create incarnation table
-- Below log names MUST be changed to existing filenames on
-- disk. Any one log file from each branch can be used to
-- re-create incarnation records.
-- ALTER DATABASE REGISTER LOGFILE '/u01/app/oracle/product/12.2.0/db_1/dbs/arch1_1_934293149.dbf';
-- ALTER DATABASE REGISTER LOGFILE '/u01/app/oracle/product/12.2.0/db_1/dbs/arch1_1_944407156.dbf';
-- Recovery is required if any of the datafiles are restored backups,
-- or if the last shutdown was not normal or immediate.
RECOVER DATABASE
-- Database can now be opened normally.
ALTER DATABASE OPEN;
-- Open all the PDBs.
ALTER PLUGGABLE DATABASE ALL OPEN;
-- Commands to add tempfiles to temporary tablespaces.
-- Online tempfiles have complete space information.
-- Other tempfiles may require adjustment.
ALTER TABLESPACE TEMP ADD TEMPFILE '/u01/app/oracle/oradata/orcl2/temp01.dbf'
     SIZE 34603008  REUSE AUTOEXTEND ON NEXT 655360  MAXSIZE 32767M;
ALTER SESSION SET CONTAINER = PDB$SEED;
ALTER TABLESPACE TEMP ADD TEMPFILE '/u01/app/oracle/oradata/orcl2/pdbseed/temp012017-05-19_15-19-57-248-PM.dbf'
     SIZE 67108864  REUSE AUTOEXTEND ON NEXT 655360  MAXSIZE 32767M;
ALTER SESSION SET CONTAINER = PDB1;
ALTER TABLESPACE TEMP ADD TEMPFILE '/u01/app/oracle/oradata/orcl2/pdb1/temp012017-05-19_15-19-57-248-PM.dbf'
     SIZE 67108864  REUSE AUTOEXTEND ON NEXT 655360  MAXSIZE 32767M;
ALTER SESSION SET CONTAINER = PDB2;
ALTER TABLESPACE TEMP ADD TEMPFILE '/u01/app/oracle/oradata/orcl2/pdb2/temp012017-05-19_15-19-57-248-PM.dbf' REUSE;
ALTER SESSION SET CONTAINER = PDB3;
ALTER TABLESPACE TEMP ADD TEMPFILE '/u01/app/oracle/oradata/orcl2/pdb3/temp012017-05-19_15-19-57-248-PM.dbf'
     SIZE 67108864  REUSE AUTOEXTEND ON NEXT 655360  MAXSIZE 32767M;
ALTER SESSION SET CONTAINER = CDB$ROOT;
-- End of tempfile additions.
--
--     Set #2. RESETLOGS case
--
-- The following commands will create a new control file and use it
-- to open the database.
-- Data used by Recovery Manager will be lost.
-- The contents of online logs will be lost and all backups will
-- be invalidated. Use this only if online logs are damaged.
-- After mounting the created controlfile, the following SQL
-- statement will place the database in the appropriate
-- protection mode:
--  ALTER DATABASE SET STANDBY DATABASE TO MAXIMIZE PERFORMANCE
STARTUP NOMOUNT
CREATE CONTROLFILE REUSE DATABASE "ORCL2" RESETLOGS  NOARCHIVELOG
    MAXLOGFILES 16
    MAXLOGMEMBERS 3
    MAXDATAFILES 1024
    MAXINSTANCES 8
    MAXLOGHISTORY 292
LOGFILE
  GROUP 1 '/u01/app/oracle/oradata/orcl2/redo01.log'  SIZE 200M BLOCKSIZE 512,
  GROUP 2 '/u01/app/oracle/oradata/orcl2/redo02.log'  SIZE 200M BLOCKSIZE 512,
  GROUP 3 '/u01/app/oracle/oradata/orcl2/redo03.log'  SIZE 200M BLOCKSIZE 512
-- STANDBY LOGFILE
DATAFILE
  '/u01/app/oracle/oradata/orcl2/system01.dbf',
  '/u01/app/oracle/oradata/orcl2/sysaux01.dbf',
  '/u01/app/oracle/oradata/orcl2/undotbs01.dbf',
  '/u01/app/oracle/oradata/orcl2/pdbseed/system01.dbf',
  '/u01/app/oracle/oradata/orcl2/pdbseed/sysaux01.dbf',
  '/u01/app/oracle/oradata/orcl2/users01.dbf',
  '/u01/app/oracle/oradata/orcl2/pdbseed/undotbs01.dbf',
  '/u01/app/oracle/oradata/orcl2/pdb1/system01.dbf',
  '/u01/app/oracle/oradata/orcl2/pdb1/sysaux01.dbf',
  '/u01/app/oracle/oradata/orcl2/pdb1/undotbs01.dbf',
  '/u01/app/oracle/oradata/orcl2/pdb2/system01.dbf',
  '/u01/app/oracle/oradata/orcl2/pdb2/sysaux01.dbf',
  '/u01/app/oracle/oradata/orcl2/pdb2/undotbs01.dbf',
  '/u01/app/oracle/oradata/orcl2/pdb3/system01.dbf',
  '/u01/app/oracle/oradata/orcl2/pdb3/sysaux01.dbf',
  '/u01/app/oracle/oradata/orcl2/pdb3/undotbs01.dbf'
CHARACTER SET ZHS16GBK
;
-- Commands to re-create incarnation table
-- Below log names MUST be changed to existing filenames on
-- disk. Any one log file from each branch can be used to
-- re-create incarnation records.
-- ALTER DATABASE REGISTER LOGFILE '/u01/app/oracle/product/12.2.0/db_1/dbs/arch1_1_934293149.dbf';
-- ALTER DATABASE REGISTER LOGFILE '/u01/app/oracle/product/12.2.0/db_1/dbs/arch1_1_944407156.dbf';
-- Recovery is required if any of the datafiles are restored backups,
-- or if the last shutdown was not normal or immediate.
RECOVER DATABASE USING BACKUP CONTROLFILE
-- Database can now be opened zeroing the online logs.
ALTER DATABASE OPEN RESETLOGS;
-- Open all the PDBs.
ALTER PLUGGABLE DATABASE ALL OPEN;
-- Commands to add tempfiles to temporary tablespaces.
-- Online tempfiles have complete space information.
-- Other tempfiles may require adjustment.
ALTER TABLESPACE TEMP ADD TEMPFILE '/u01/app/oracle/oradata/orcl2/temp01.dbf'
     SIZE 34603008  REUSE AUTOEXTEND ON NEXT 655360  MAXSIZE 32767M;
ALTER SESSION SET CONTAINER = PDB$SEED;
ALTER TABLESPACE TEMP ADD TEMPFILE '/u01/app/oracle/oradata/orcl2/pdbseed/temp012017-05-19_15-19-57-248-PM.dbf'
     SIZE 67108864  REUSE AUTOEXTEND ON NEXT 655360  MAXSIZE 32767M;
ALTER SESSION SET CONTAINER = PDB1;
ALTER TABLESPACE TEMP ADD TEMPFILE '/u01/app/oracle/oradata/orcl2/pdb1/temp012017-05-19_15-19-57-248-PM.dbf'
     SIZE 67108864  REUSE AUTOEXTEND ON NEXT 655360  MAXSIZE 32767M;
ALTER SESSION SET CONTAINER = PDB2;
ALTER TABLESPACE TEMP ADD TEMPFILE '/u01/app/oracle/oradata/orcl2/pdb2/temp012017-05-19_15-19-57-248-PM.dbf' REUSE;
ALTER SESSION SET CONTAINER = PDB3;
ALTER TABLESPACE TEMP ADD TEMPFILE '/u01/app/oracle/oradata/orcl2/pdb3/temp012017-05-19_15-19-57-248-PM.dbf'
     SIZE 67108864  REUSE AUTOEXTEND ON NEXT 655360  MAXSIZE 32767M;
ALTER SESSION SET CONTAINER = CDB$ROOT;
-- End of tempfile additions.
--
#-----------------------------------------------------------------------------------------
```