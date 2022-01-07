# ขั้นตอนการติดตั้ง และขอใช้งาน
## สิทธิ์เข้าใช้งาน และ Certificate
1. ส่งแบบฟอร์มขอใช้บริการ([download](https://moph.cc/_aHErjLjJ)) ผ่านทางเมล saraban0212@moph.go.th และ cc: standard@moph.mail.go.th
2. เมื่อส่วนกลางอนุมัติเรียบร้อยแล้ว ไอทีรพ. สามารถกดขอ cert ในหน้าเว็บ https://hisgateway.moph.go.th/homepage (ปุ่ม <button>Download Certificate</button>)
---
<br>

## ติดตั้ง Docker
<details><summary>แสดงวิธี</summary>
<p>

1. ติดตั้ง Docker
    - centos: [https://docs.docker.com/engine/install/centos](https://docs.docker.com/engine/install/centos)
    - debian: [https://docs.docker.com/engine/install/debian](https://docs.docker.com/engine/install/debian)
    - fedora: [https://docs.docker.com/engine/install/fedora](https://docs.docker.com/engine/install/fedora)
    - ubuntu: [https://docs.docker.com/engine/install/ubuntu](https://docs.docker.com/engine/install/ubuntu)

2. ติดตั้ง Docker-compose
    - [https://docs.docker.com/compose/install/](https://docs.docker.com/compose/install/)
3. run คำสั่ง sudo systemctl enable docker เพื่อให้ service docker start โดยอัตโนมัติ
</p>
</details>

---
<br>

## ตั้งค่า Database
<details><summary>แสดงวิธี</summary>
<p>

### Postgres
<details>
  <summary>แสดงวิธี</summary>
  <p>

   1. Install plugin
   	- CentOS:
           ```
           sudo yum install wal2json<version>
           ```
   	- Ubuntu:
     	    ```
           sudo apt-get install postgresql-<version>-wal2json
           ```

       **example** Postgres V.13: `wal2json13` | `postgresql-13-wal2json`

       ***ref:*** [htps://github.com/eulerto/wal2json](htps://github.com/eulerto/wal2json)
   2. Configuration options in postgresql.conf:
       ```
       wal_level = logical;
       max_replication_slots = 10;
       shared_preload_libraries = 'wal2json'
       ```
   3. Restert service postgres
   - ***P.S.*** Show config path
       ```
       SHOW config_file
       ```
       Ubuntu: `/etc/postgresql/{{version}}/main/postgresql.conf`

       CentOS: `/var/lib/pgsql/{{version}}/data/postgresql.conf`

  </p>
</details>

---

### Mysql
      
<details><summary>แสดงวิธี</summary>
<p>

1. Configuration options in my.cnf/my.ini วางใต้ `[mysqld]`
    ```
    server_id=10001
    log_bin=gwhis
    binlog_format=row
    binlog_do_db=ชื่อฐานข้อมูล

    ;บางเวอร์ชั่นใช้ binlog_expire_logs_seconds=
    expire_logs_days=7

    ;กรณีตั้งค่าที่เครื่อง slave โดยใช้ของ mysql ถ้าเป็น slave โดยใช้ tools hosxp ไม่ต้องใส่
    log_slave_updates=on
    ```
2. restart service mysql
3. ทดสอบ Binlog โดยการเข้าไป Query ในฐานข้อมูลใช้คำสั่ง `SHOW BINARY LOGS;`
- ***P.S.*** `GRANT LOCK, SELECT, RELOAD, REPLICATION SLAVE, REPLICATION CLIENT`
</p>
</details>

---
### SQL Server
<details><summary>แสดงวิธี</summary>
<p>

> #### **IMPORTANT**
> **Change data capture (CDC) is only available in the Enterprise, Developer, and Enterprise Evaluation editions**

ถ้าหากใช้ Standard Edition จะรันคำสั่งนี้ไม่ได้ `EXEC sys.sp_cdc_enable_db` และจะติด Error ดังนี้

> Msg 22988, Level 16, State 1, Server NAME, Procedure sp_cdc_enable_db,
This instance of SQL Server is the Standard Edition (64-bit). Change data capture is only available in the Enterprise, Developer, and Enterprise Evaluation editions.
> [42000] [Microsoft][ODBC Driver 17 for SQL Server][SQL Server]This instance of SQL Server is the Standard Edition (64-bit). Change data capture is only available in the Enterprise, Developer, and Enterprise Evaluation editions. (22988)

1. ใช้คำสั่ง Query เพื่อเปิด CDC สำหรับฐานข้อมูล
    ```
    EXEC sys.sp_cdc_enable_db
    ```

2. ใช้คำสั่ง Query เพื่อเปิด CDC ให้กับตาราง

    - ทีละตาราง
        ```
        EXEC sys.sp_cdc_enable_table
            @source_schema = 'dbo',
            @source_name = 'tableName',
            @role_name = NULL,
            @filegroup_name = NULL,
            @supports_net_changes = 1;
        ```
    ---
    - สร้าง function เพื่อเปิด cdc ทีเดียว
        ```
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
        ```
      ```
      EXEC sp_enable_disable_cdc_all_tables "database",1
      ```
3. ใช้คำสั่ง Query เพื่อดูตารางที่เปิด CDC
    ```
    SELECT t.name, t.is_tracked_by_cdc FROM sys.tables t WHERE t.is_tracked_by_cdc = 1;
    ```

</p>
</details>

---
### Oracle
<details><summary>แสดงวิธี</summary>
<p>

  ```shell
  ORACLE_SID=ORACLCDB dbz_oracle sqlplus /nolog
  ```
  ```
  CONNECT sys/top_secret AS SYSDBA
  alter system set db_recovery_file_dest_size = 10G;
  alter system set db_recovery_file_dest = '/opt/oracle/oradta/recovery_area' scope=spfile;
  shutdown immediate
  startup mount
  alter database archivelog;
  alter database open;
  ```
  Should now "Database log mode: Archive Mode"
  ```
  archive log list

  exit;
  ```
  ***ref:*** https://debezium.io/documentation/reference/connectors/oracle.html#_preparing_the_database

</p>
</details>

</p>
</details>

---
<br>

## Download Hisgateway Project
    1.สร้าง folder hisgateway 
    2.Download 
  git clone (Recommend)  หากไม่มี git ให้ install git ก่อน
  ```
  git clone https://github.com/mophos/hisgateway-docker.git
  ```
    3.เมื่อ download เรียบร้อยให้สร้างโฟวเดอร์ certในโฟวเดอร์ hisgateway  (ด้านนอกโฟวเดอร์ hisgateway-docker)

  นำไฟล์ Certificate ทั้งหมดที่ได้จาก [hisgateway.moph.go.th](https://hisgateway.moph.go.th/tutorial/homepage) วางใน folder `cert`

---
<br>

## คัดลอกไฟล์ env.text เป็น .env  และแก้ไขไฟล์ .env  (มี . นำหน้า และไม่มีสกุล.text)

แก้ไข `xxxxx` ให้เป็น รหัสโรงพยาบาล
```
HOSCODE=xxxxx
```
    
`GROUP=1` ใช้ค่า =1 เปลี่ยนเมื่อโรงพยาบาลมีมากกว่า 1 db ในการส่งข้อมูล
```
GROUP=1
```
    
แก้ไข `PPPPP` ให้เป็น รหัสของ cert (ในไฟล์ `password_xxxxx.txt`)
```
PASSWORD=PPPPP
```
    
ตั้งค่า `port ที่จะเปิดเว็บ` 
```
PORT=80
```
    
ตั้งค่า `SECRET_KEY เป็นอะไรก็ได้` 
```
SECRET_KEY=12345
```
---
<br>

 ## run docker-compose
คำสั่ง (ใช้ ณ ตำแหน่งเดียวกับที่มีไฟล์ docker-compose.yaml อยู่)
  ```
  docker-compose up -d
  ```
---
## เข้าใช้งาน web
- เข้าผ่าน `localhost:80` ของเครื่องที่ติดตั้ง (port จะใช้ตามไฟล์ .env PORT=)
---
## การสร้าง connectors
1. กด <button>+ New Connector</button>
2. tab HIS database
     - เลือก HIS Type
     - เลือก database connector
     - ใส่ database address เป็น ip ของ database ที่ใช้งาน ตัวเดียวกับที่ทำการตั้งค่า
     - ใส่ database port
     - ใส่ database username
     - ใส่ database password
     - ใส่ database name
3. tab MOPH Broker
   - ในส่วน keystore part และ truststore part
       แก้ไข `xxxxx` ให้เป็น รหัสโรงพยาบาล
   - ในส่วน keystore password และ truststore password
       นำ password จากในไฟล์ `password_xxxxx.txt` มาใส่
---
## วิดีโอสอนติดตั้งในส่วนต่างๆ
    - Install docker cent(https://youtu.be/7RBvP7jhhSk)
    - Install docker ubuntu(https://youtu.be/if_P8VtBFms)
    - Install Hisgateway(https://youtu.be/DJKZLkmRWhs)
    - Setting database mysql(https://youtu.be/raVVZ0bWmjE)
    - Add Connector(https://youtu.be/0UAA4l4sHUc)
---

1. ติดตั้ง Docker
    - centos: [https://docs.docker.com/engine/install/centos](https://docs.docker.com/engine/install/centos)
    - debian: [https://docs.docker.com/engine/install/debian](https://docs.docker.com/engine/install/debian)
    - fedora: [https://docs.docker.com/engine/install/fedora](https://docs.docker.com/engine/install/fedora)
    - ubuntu: [https://docs.docker.com/engine/install/ubuntu](https://docs.docker.com/engine/install/ubuntu)
---
 ## การอัพเดท
1. เข้าไปในโฟลเดอร์ hisgateway-docker
2. docker-compose down
3. git pull origin main หรือ git pull
4. run ไฟล์ update.sh
5. docker-compose up -d 
---
# ติดต่อสอบถาม line @hisgateway
![QR](https://qr-official.line.me/sid/M/992qwkma.png)

Domain (IP) -
kafka1.moph.go.th (203.157.100.45)
mqtt.h4u.moph.go.th (203.157.103.140)
Port ขาออก -
9093
19093
31888
31990-32000 
---
ทดสอบการเชื่อมต่อ broker ด้วยคำสั่ง nc -vz kafka1.moph.go.th 31888 
หรือ docker run -it --rm appropriate/nc -vz kafka1.moph.go.th 31888
