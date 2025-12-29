# Airflow Upgrade Guide
คู่มืออัปเกรด Airflow และ PostgreSQL ด้วย Docker

**Airflow:** 3.0.1 → 3.1.5  
**PostgreSQL:** 13 → 16  

---
## 1. Create `.env` (if not exist)
สร้างไฟล์ `.env` เพื่อกำหนด UID ให้ container ทำงานตรงกับ user บนเครื่อง

```sh
echo -e "AIRFLOW_UID=$(id -u)" > .env
```

## 2. Edit docker-compose-new.yaml
แก้ไข docker-compose สำหรับเวอร์ชันใหม่

### 2.1 Switch from Image → Build
เปลี่ยนจากการใช้ image สำเร็จรูป มาเป็นการ build เอง

- comment บรรทัดนี้
```     
image: ${AIRFLOW_IMAGE_NAME:-apache/airflow:...} --> 
# image: ${AIRFLOW_IMAGE_NAME:-apache/airflow:...}
```
- uncomment บรรทัดนี้
```   
# build: . -> build: .
```
### 2.2 Rename Named Volume
เปลี่ยนชื่อ volume เพื่อไม่ให้ชนกับ PostgreSQL เดิม
(ตัวอย่าง: postgres-db-volume-pg16)

แก้ไขในตำแหน่ง:
- services/postgres/volumes
- volumes

### 2.3 Add Extra Volumes to Postgres
เพิ่ม volume สำหรับ restore และ backup
- ./initdb → ใช้ restore database ตอน container เริ่ม

- ./backup → เก็บไฟล์ backup

docker-compose
```
services:
  postgres:
    volumes:
      - ./initdb:/docker-entrypoint-initdb.d
      - ./backup:/backup


```

## 3. Edit Dockerfile_new (Airflow Version)
อัปเดตเวอร์ชัน Airflow ที่ต้องการใช้งาน
```dockerfile
FROM apache/airflow:3.1.5
COPY requirements.txt /
RUN pip install --no-cache-dir "apache-airflow==${AIRFLOW_VERSION}" -r /requirements.txt
```

## 4. List Existing Docker Volumes
ตรวจสอบ volume ที่มีอยู่ในระบบ
```
docker volume ls
```

## 5. Navigate to Project Directory
เข้า directory ที่มี Dockerfile และ docker-compose.yaml  
- 5.1 Stop All Containers
หยุด container ทั้งหมดก่อนเริ่ม backup
```sh
docker compose stop
```

- 5.2 Backup PostgreSQL Volume (Physical Backup)  
สำรองข้อมูลระดับ volume เพื่อความปลอดภัย
```sh
docker run --rm --volumes-from airflow-2025-postgres-1 -v .:/backup ubuntu tar cvf /backup/backup.tar /var/lib/postgresql/data
#                              ^~~~~~~~~~~~~~~~~~~~~~~ postgres container name
# will create backup.tar in the current working dir
```

- กรณี ยังไม่มี ./backup mount  
ถ้ายังไม่ได้ mount volume backup ให้ทำตามขั้นตอนนี้ก่อน
```
docker compose down
```
เพิ่ม ./backup:/backup ใน docker-compose.yaml (services/postgres/volumes)
```
./backup:/backup
```
รัน
```
docker compose build postgres
```

- 5.3 Stop All Containers (Again)  
หยุด container ทั้งหมดอีกรอบเพื่อเตรียม backup แบบ SQL

- 5.4 Start Postgres Only
```
docker compose up -d postgres
```
- 5.5 Backup Database (Logical Backup)  
สำรอง database ออกมาเป็นไฟล์ SQL
```sh
docker compose exec postgres sh -c 'pg_dumpall -U airflow > /backup/backup.sql'
# will create backup.sql in ./backup
```
- 5.6 Stop & Remove Containers
หยุดและลบ container เดิม
```
docker compose stop postgres
docker compose down
```

- 5.7 Move SQL File to initdb  
ย้ายไฟล์ backup เพื่อใช้ restore ตอน PostgreSQL 16 start  
```
mv ./backup/backup.sql ./initdb/backup.sql

```

- 5.8 Edit backup.sql  
คอมเมนต์บรรทัดต่อไปนี้ออก

```sql
-- CREATE ROLE airflow;
-- ALTER ROLE airflow WITH SUPERUSER INHERIT CREATEROLE CREATEDB LOGIN REPLICATION BYPASSRLS PASSWORD ...;

-- CREATE DATABASE airflow WITH TEMPLATE = template0 ENCODING = 'UTF8' LOCALE = 'en_US.utf8';
```
- 5.9 Switch Docker Files  
สลับไฟล์เก่า → ใหม่
```
Dockerfile              → Dockerfile_old
docker-compose.yaml     → docker-compose-old.yaml

Dockerfile_new          → Dockerfile
docker-compose-new.yaml → docker-compose.yaml

```
- 5.10 Build New Images  
build image ใหม่ทั้งหมด

```
docker compose build
```

- 5.11 Start New Stack  
รันระบบใหม่ทั้งหมด

```
docker compose up -d
```

- 5.12 Check PostgreSQL Logs  
ตรวจสอบว่า restore สำเร็จหรือไม่

```sh
docker compose logs postgres | less
```
Log ที่ควรพบ  

starts with: (begin data restore)

```
/usr/local/bin/docker-entrypoint.sh: running /docker-entrypoint-initdb.d/backup.sql
```

followed by many lines of CREATE/ALTER TABLEs etc.

ends with 5 SETs, select set_config and another 4 SETs.

```
SET
SET
SET
SET
SET
 set_config 
------------
 
(1 row)

SET
SET
SET
SET
```

## done

-----------------------------------------------------------------------------------------------------
