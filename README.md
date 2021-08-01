# HISGateway-docker


1.ติดตั้ง Docker

```markdown
[https://docs.docker.com/engine/install/centos](https://docs.docker.com/engine/install/centos)
[https://docs.docker.com/engine/install/debian](https://docs.docker.com/engine/install/debian)
[https://docs.docker.com/engine/install/fedora](https://docs.docker.com/engine/install/fedora)
[https://docs.docker.com/engine/install/ubuntu](https://docs.docker.com/engine/install/ubuntu)
```

2.ติดตั้ง Docker-compose

```markdown
[https://docs.docker.com/compose/install/](https://docs.docker.com/compose/install/)
```



ตั้งค่า Database 

Postgres

```shell
(1) Install plugin
CentOS: sudo yum install wal2json13
Ubuntu: sudo apt-get install postgresql-13-wal2json
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

Mysql

```shell
(1) Configuration options in my.cnf/my.ini	
server_id=10001
log_bin=gwhis
binlog_format=row
binlog_do_db=ชื่อฐานข้อมูล

log_slave_updates=on    ##กรณีตั้งค่าที่เครื่อง slave โดยใช้ของ mysql   ถ้าเป็น slaveโดยใช้ tools hosxp ไม่ต้องใส่

(1)ทดสอบ Binlog โดยการเข้าไป Query ในฐานข้อมูลใช้คำสั่ง
SHOW BINARY LOGS;
```