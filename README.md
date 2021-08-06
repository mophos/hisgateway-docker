# คู่มือการติดตั้ง

## ติดตั้ง Docker
1.ติดตั้ง Docker
- [https://docs.docker.com/engine/install/centos](https://docs.docker.com/engine/install/centos)
- [https://docs.docker.com/engine/install/debian](https://docs.docker.com/engine/install/debian)
- [https://docs.docker.com/engine/install/fedora](https://docs.docker.com/engine/install/fedora)
- [https://docs.docker.com/engine/install/ubuntu](https://docs.docker.com/engine/install/ubuntu)


2.ติดตั้ง Docker-compose
- [https://docs.docker.com/compose/install/](https://docs.docker.com/compose/install/)



## ตั้งค่า Database 

### Postgres

```shell
(1) Install plugin
CentOS: sudo yum install wal2json<version>
Ubuntu: sudo apt-get install postgresql-<version>-wal2json
example postgres v.13 wal2json13 | postgresql-13-wal2json
#ref https://github.com/eulerto/wal2json

(2) Configuration options in postgresql.conf:
wal_level = logical;
max_replication_slots = 10;
shared_preload_libraries = 'wal2json' 

(PS) Show config path
SHOW config_file
Ubuntu /etc/postgresql/{{version}}/main/postgresql.conf
CentOS /var/lib/pgsql/{{version}}/data/postgresql.conf 
```

### Mysql

```shell
(1) Configuration options in my.cnf/my.ini	
server_id=10001
log_bin=gwhis
binlog_format=row
binlog_do_db=ชื่อฐานข้อมูล
binlog_row_image=FULL
expire_logs_days=7   บางเวอร์ชั่นใช้ binlog_expire_logs_seconds=
log_slave_updates=on    ##กรณีตั้งค่าที่เครื่อง slave โดยใช้ของ mysql   ถ้าเป็น slaveโดยใช้ tools hosxp ไม่ต้องใส่

(1)ทดสอบ Binlog โดยการเข้าไป Query ในฐานข้อมูลใช้คำสั่ง
SHOW BINARY LOGS;

(PS)
GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT
```

### SQL Server

```shell
(1) ใช้คำสั่ง Query เพื่อเปิด CDC สำหรับฐานข้อมูล
EXEC sys.sp_cdc_enable_db

(2) ใช้คำสั่ง Query เพื่อเปิด CDC ให้กับตาราง (ทีละตาราง)
EXEC sys.sp_cdc_enable_table
@source_schema = 'dbo',
@source_name = 'tableName',
@role_name = NULL,
@filegroup_name = NULL,
@supports_net_changes = 1;

(3) สร้าง function เพื่อเปิด cdc ทีเดียว
######## start copy ######## 
create procedure sp_enable_disable_cdc_all_tables(@dbname varchar(100), @enable bit)  
as  
BEGIN TRY  
DECLARE @source_name varchar(400);  
declare @sql varchar(1000)  
DECLARE the_cursor CURSOR FAST_FORWARD FOR  
SELECT table_name  
FROM INFORMATION_SCHEMA.TABLES where TABLE_CATALOG=@dbname and table_schema='dbo' and table_name != 'systranschemas'  
OPEN the_cursor  
FETCH NEXT FROM the_cursor INTO @source_name  
WHILE @@FETCH_STATUS = 0  
BEGIN  
if @enable = 1  
set @sql =' Use '+ @dbname+ ';EXEC sys.sp_cdc_enable_table  
            @source_schema = N''dbo'',@source_name = '+@source_name+'  
          , @role_name = N'''+'dbo'+''''       
else  
set @sql =' Use '+ @dbname+ ';EXEC sys.sp_cdc_disable_table  
            @source_schema = N''dbo'',@source_name = '+@source_name+',  @capture_instance =''all'''  
exec(@sql)  
  FETCH NEXT FROM the_cursor INTO @source_name  
END  
CLOSE the_cursor  
DEALLOCATE the_cursor  
SELECT 'Successful'  
END TRY  
BEGIN CATCH  
CLOSE the_cursor  
DEALLOCATE the_cursor  
    SELECT   
        ERROR_NUMBER() AS ErrorNumber  
        ,ERROR_MESSAGE() AS ErrorMessage;  
END CATCH  
######## END copy ######## 

EXEC sp_enable_disable_cdc_all_tables "database",1

(4) ใช้คำสั่ง Query เพื่อดูตารางที่เปิด CDC
SELECT t.name, t.is_tracked_by_cdc FROM sys.tables t WHERE t.is_tracked_by_cdc = 1;
```



### Oracle

```shell
ORACLE_SID=ORACLCDB dbz_oracle sqlplus /nolog

CONNECT sys/top_secret AS SYSDBA
alter system set db_recovery_file_dest_size = 10G;
alter system set db_recovery_file_dest = '/opt/oracle/oradta/recovery_area' scope=spfile;
shutdown immediate
startup mount
alter database archivelog;
alter database open;
-- Should now "Database log mode: Archive Mode"
archive log list

exit;

ref: https://debezium.io/documentation/reference/connectors/oracle.html#_preparing_the_database
```

## ตั้งค่า ไฟล์ docker-compose.yaml 
```
แก้ไข xxxxx ให้เป็น รหัสโรงพยาบาล
      - GROUP_ID=xxxxx_x
      - CONFIG_STORAGE_TOPIC=xxxxx_hisgateway_connect_configs
      - OFFSET_STORAGE_TOPIC=xxxxx_hisgateway_connect_offsets
      - STATUS_STORAGE_TOPIC=xxxxx_hisgateway_connect_statuses
      - CONNECT_SSL_TRUSTSTORE_LOCATION=/var/private/ssl/kafka.client.xxxxx.truststore.jks
      - CONNECT_PRODUCER_SSL_TRUSTSTORE_LOCATION=/var/private/ssl/kafka.client.xxxxx.truststore.jks
      - CONNECT_SSL_KEYSTORE_LOCATION=/var/private/ssl/kafka.client.xxxxx.keystore.jks
      - CONNECT_PRODUCER_SSL_KEYSTORE_LOCATION=/var/private/ssl/kafka.client.xxxxx.keystore.jks
      - HOSPCODE=xxxxx
      
 แก้ไข PPPPP ให้เป็น รหัสของ cert
      - CONNECT_PRODUCER_SSL_TRUSTSTORE_PASSWORD=PPPPP
      - CONNECT_PRODUCER_SSL_KEYSTORE_PASSWORD=PPPPP
      - CONNECT_SSL_TRUSTSTORE_PASSWORD=PPPPP
      - CONNECT_SSL_KEYSTORE_PASSWORD=PPPPP
      - CONNECT_SSL_KEY_PASSWORD=PPPPP
      - CONNECT_PRODUCER_SSL_KEY_PASSWORD=PPPPP
 ตั้งค่า port 
      จะมีให้ตั้งค่าอยู่ 2 จุด
      connect:
       ports:
            - 8083:8083
      nginx:
       port:
            - 80:80
      โดยแก้เพียงเลขทางซ้าย หาก port ชนกับ port ในเครื่องที่ติดตั้ง
 ```
 ## run docker-compose  
 ```
    คำสั่ง docker-compose up -d
    ณ ตำแหน่งเดียวกับที่มีไฟล์ docker-compose.yaml อยู่
```
## เข้าใช้งาน web
```
    เข้าผ่าน localhost:80 ของเครื่องที่ติดตั้ง
    port จะขึ้นกับ port ที่ตั้งใน nginx
```
## การสร้าง connectors
```
กด + New Connector
  tab HIS database
    เลือก HIS Type
    เลือก database connector
    ใส่ database address เป็น ip ของ database ที่ใช้งาน ตัวเดียวกับที่ทำการตั้งค่า
    ใส่ database port
    ใส่ database username
    ใส่ database password
    ใส่ database name
tab MOPH Broker
    ในส่วน keystore part และ truststore part
       แก้ไข xxxxx ให้เป็น รหัสโรงพยาบาล
    ในส่วน keystore password และ truststore password
       นำ password จากไฟล์ password_xxxxx.txt มาใส่
```
