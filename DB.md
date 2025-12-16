# DB
---
## ลำดับที่ต้องทำ  

- Role Management (Auth & Authorization)
    - ต้องทำ ก่อนอื่นเลย เพราะต้องกำหนดว่าใครเข้าถึงข้อมูลอะไร

- Row/Column-Level Security

    - ต้องมี Roles ก่อน ถึงจะกำหนดว่าใครอ่าน/เขียน row หรือ column ไหนได้

- Materialized Views

    - ขึ้นกับว่า Security ถูกตั้งเรียบร้อยแล้ว เพราะ views ต้อง respect สิทธิ์ผู้ใช้

- Audit Logs + Monitor

    - ติดตั้งหลัง Security และ Views เพื่อบันทึกกิจกรรมจริง

- Backup / Recovery

    - ทำหลังระบบ Security + Audit เพราะเราต้อง backup ข้อมูลที่ปลอดภัยและสมบูรณ์

- Tested Restore

    - ทดสอบการ restore backup ว่าข้อมูลถูกต้อง

- Secrets Management

    - จัดการ password, keys, tokens เพื่อให้ระบบปลอดภัย

- Replication

    - ทำสุดท้าย เพื่อให้ระบบ มีสำเนา (replica) รองรับความล้มเหลว
```
                 +----------------------+
                 |   Role Management    |
                 | (Auth & Authorization)|
                 +----------+-----------+
                            |
                            v
                 +----------------------+
                 | Row/Column-Level     |
                 | Security             |
                 +----------+-----------+
                            |
                            v
                 +----------------------+
                 | Materialized Views   |
                 | (Optional, for perf)|
                 +----------+-----------+
                            |
                            v
                 +----------------------+
                 | Audit Logs + Monitor |
                 +----------+-----------+
                            |
                            v
                 +----------------------+
                 | Backup / Recovery    |
                 +----------+-----------+
                            |
                            v
                 +----------------------+
                 | Tested Restore       |
                 +----------+-----------+
                            |
                            v
                 +----------------------+
                 | Secrets Management   |
                 +----------+-----------+
                            |
                            v
                 +----------------------+
                 | Replication          |
                 +----------------------+

```
## setup project

- โครงสร้างคร่าวๆ
```
mysql-replication-lab/
├── docker-compose.yml
├── .env
├── master/
│   └── my.cnf
└── slave/
    └── my.cnf

```

- .env
```
MYSQL_ROOT_PASSWORD= #root pass
MYSQL_DATABASE= #db name
MYSQL_USER= #user master
MYSQL_PASSWORD= #pass master
REPLICATION_USER= #user repli
REPLICATION_PASSWORD= #pass repli
```

- ไฟล์ config master/my.cnf
```
[mysqld]
# ตั้งค่า Server ID ให้ไม่ซ้ำกัน (Master ควรเป็น 1)
server-id=1

# เปิดใช้งาน Binary Log
log-bin=mysql-bin

# กำหนดฐานข้อมูลที่จะให้ Replication
binlog-do-db=mydb
```

- slave/my.cnf
```
[mysqld]
# ตั้งค่า Server ID ให้ไม่ซ้ำกับ Master (Slave ควรเป็น 2 ขึ้นไป)
server-id=2

# เปิดใช้งาน Relay Log (ที่ Slave จะเก็บ log จาก Master)
relay-log=relay-bin

# ทำให้ Slave เป็นแบบอ่านอย่างเดียวเพื่อความปลอดภัย
read_only=1
# กำหนดว่าจะ replicate เฉพาะ database นี้เท่านั้น
replicate-do-db=mydb
```

- docker-compose.yaml
```
services:
  mysql-master:
    image: mysql:8.0
    container_name: mysql-master
    restart: always
    env_file: .env
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    ports:
      - "3307:3306"
    volumes:
      - ./master/my.cnf:/etc/mysql/conf.d/my.cnf # map ไฟล์ config ของ master
      - master_data:/var/lib/mysql # เก็บข้อมูล database ไว้ที่ volume (ชื่อ volume):(ที่เก็บข้อมูลใน container)
    networks:
      - mysql-network

  mysql-slave:
    image: mysql:8.0
    container_name: mysql-slave
    restart: always
    env_file: .env
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      # Slave จะ sync database มาจาก master อัตโนมัติ
      MYSQL_DATABASE: ${MYSQL_DATABASE}
    ports:
      - "3308:3306" # เปิดพอร์ต 3308 เพื่อไม่ให้ชนกับ master
    volumes:
      - ./slave/my.cnf:/etc/mysql/conf.d/my.cnf # map ไฟล์ config ของ slave
      - slave_data:/var/lib/mysql # เก็บข้อมูล database ไว้ที่ volume (ชื่อ volume):(ที่เก็บข้อมูลใน container)
    depends_on:
      - mysql-master
    networks:
      - mysql-network

volumes:
  master_data: 
  slave_data:

networks:
  mysql-network:
    driver: bridge
```

- รัน docker
```
docker compose up -d
```

- เข้า exec master
```
docker exec -it mysql-master mysql -u root -p

bash---
CREATE USER '(repliUser)'@'(ip network)' IDENTIFIED WITH mysql_native_password BY '(รหัส)';
GRANT REPLICATION SLAVE ON *.* TO '(repliUser)'@'(ip network)';
FLUSH PRIVILEGES;

show master status\G
----------
# เช็ค user
SELECT User, Host FROM mysql.user;
----------
# ip network มาจาก 
docker network ls # รายชื่อ network
docker network inspect (ชื่อ network)
```

- เข้า slave
```
change master to
master_host='mysql-master',
master_user='(userRepli)',
master_password='(password)',
master_log_file='(bin จาก status)',
master_log_pos=position จาก status ;

start slave;

# เช็ค connection
show slave status\G
```

- ทดสอบ สร้าง table ใน master
```
create table products (id int auto_increment primary key,name varchar(255) not null,price decimel(10,2));

insert into products (name,price) values ('labtop',30000.00)
---
slave
---
select * from products;

# ส่วนนี้ต้องเห็น
```

## Role (Authentication & Authorization)

- สร้าง Role สำหรับ "อ่านข้อมูลเท่านั้น"
```
CREATE ROLE 'product_reader';
GRANT SELECT ON mydb.products TO 'product_reader';
```

- สร้าง User ใหม่และมอบ Role นี้ให้
```
CREATE USER 'report_user'@'%' IDENTIFIED BY 'reportP@ssw0rd';
GRANT 'product_reader' TO 'report_user';

มอบสิทธิ์พื้นฐานให้สามารถเชื่อมต่อกับฐานข้อมูล mydb ได้
---
GRANT USAGE ON mydb.* TO 'report_user'@'%';


ตั้งค่าให้ Role ทั้งหมดที่มอบให้ report_user ทำงานโดยอัตโนมัติทุกครั้งที่ล็อกอิน
---
ALTER USER 'report_user'@'%' DEFAULT ROLE ALL;

รีเฟรชสิทธิ์ให้ MySQL
---
FLUSH PRIVILEGES;
```

## Row-level / Column-level Security

- จำลองโดยการสร้าง table บน master
```
CREATE TABLE employees (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    department VARCHAR(50),
    salary DECIMAL(10, 2)
);

INSERT INTO employees (name, department, salary) VALUES ('Alice', 'HR', 60000), ('Bob', 'IT', 80000);

View สำหรับ Manager (ไม่เห็น salary)
---
CREATE VIEW v_employees_public AS
SELECT id, name, department FROM employees;

View สำหรับ HR (เห็นทุกอย่าง)
---
CREATE VIEW v_employees_hr AS
SELECT id, name, department, salary FROM employees;

สร้าง User และให้สิทธิ์
---
CREATE USER 'manager_user'@'%' IDENTIFIED BY 'manager123';
GRANT SELECT ON mydb.v_employees_public TO 'manager_user';

CREATE USER 'hr_user'@'%' IDENTIFIED BY 'hr123';
GRANT SELECT ON mydb.v_employees_hr TO 'hr_user';


ทดสอบ
---
docker exec -it mysql-master mysql -u manager_user -pmanager123
SELECT * FROM mydb.v_employees_public;

docker exec -it mysql-master mysql -u hr_user -phr123
SELECT * FROM mydb.v_employees_hr;

```

## Materialized Views

- สร้างตารางทดสอบ
```
CREATE TABLE mv_product_summary (
  total_products INT NOT NULL,
  average_price DECIMAL(10,2) NOT NULL,
  last_refresh DATETIME NOT NULL
);
```

- สร้าง Stored Procedure แบบ Atomic
```
DELIMITER $$

CREATE PROCEDURE refresh_mv_product_summary()
BEGIN
  CREATE TABLE mv_product_summary_tmp LIKE mv_product_summary;

  INSERT INTO mv_product_summary_tmp
  SELECT 
    COUNT(*) AS total_products,
    AVG(price) AS average_price,
    NOW() AS last_refresh
  FROM products;

  RENAME TABLE 
    mv_product_summary TO mv_product_summary_old,
    mv_product_summary_tmp TO mv_product_summary;

  DROP TABLE mv_product_summary_old;
END$$

DELIMITER ;

```

- Event Scheduler
```
CREATE EVENT ev_refresh_mv_product_summary
ON SCHEDULE EVERY 5 MINUTE
DO CALL refresh_mv_product_summary();
```

## Audit Logs
```
# 1. สร้างตารางสำหรับเก็บ Log
---
docker exec -it mysql-master mysql -uroot -p${MYSQL_ROOT_PASSWORD} mydb
---
CREATE TABLE products_audit (
    audit_id INT AUTO_INCREMENT PRIMARY KEY,
    product_id INT,
    old_name VARCHAR(255),
    new_name VARCHAR(255),
    old_price DECIMAL(10,2),
    new_price DECIMAL(10,2),
    action_type VARCHAR(10), -- 'INSERT', 'UPDATE', 'DELETE'
    action_user VARCHAR(100),
    action_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);


# 2. สร้าง Trigger สำหรับการ INSERT
---
DELIMITER $$ 
CREATE TRIGGER products_after_insert
AFTER INSERT ON products
FOR EACH ROW
BEGIN
    INSERT INTO products_audit (product_id, new_name, new_price, action_type, action_user)
    VALUES (NEW.id, NEW.name, NEW.price, 'INSERT', USER());
END$$ 
DELIMITER ;


# 3. ทดสอบ: เพิ่มสินค้าใหม่แล้วตรวจสอบตาราง audit
---
INSERT INTO products (name, price) VALUES ('Monitor', 300.00);

SELECT * FROM products_audit ORDER BY audit_id DESC LIMIT 1;
```

## backup
- docker
```
# สำรอง เฉพาะ database
docker exec -i mysql-master mysqldump -u root -p --single-transaction --routines --triggers --database mydb > backup_$(date +%F).sql

# สำรองทั้งหมด (รวมถึง users, grants)
docker exec -i mysql-master mysqldump -uroot -p --all-databases --single-transaction --routines --triggers > full_backup_$(date +%F).sql


** ยังไม่ได้ลอง กรณีไฟล์ ขนาดใหญ่
docker exec -i mysql-master mysqldump -uroot -p --single-transaction --routines --triggers mydb | gzip > backup_$(date +%F).sql.gz

```  
- ถ้าไม่ใช่ docker
```
---
# สำรอง เฉพาะ database
mysqldump -u root -p --single-transaction --routines --triggers --database mydb > backup_$(date +%F).sql

# สำรองทั้งหมด (รวมถึง users, grants)
mysqldump -u root -p --all-databases --single-transaction --routines --triggers > full_backup_$(date +%F).sql

** ยังไม่ได้ลอง กรณีไฟล์ ขนาดใหญ่
mysqldump -u root -p --single-transaction --routines --triggers mydb | gzip > backup_$(date +%F).sql.gz


```

## restore
- docker
```
docker cp backup_$(date +%F).sql mysql-slave:/tmp/backup.sql
**(ชื่อ container):(ที่อยู่ไฟล์ชั่วคราว)

docker exec -it mysql-slave mysql -u root -p
use (ชื่อ db)
SOURCE /tmp/backup.sql;

.gz
---
# จาก container
gunzip -c /tmp/backup.sql.gz | mysql -u root -p

# จาก host ไม่ copy
gunzip -c backup_$(date +%F).sql.gz | docker exec -i mysql-slave mysql -u root -p
```

- ไม่ใช้ docker
```
mysql -u root -p < backup_2025-12-15.sql

.gz
---
gunzip -c backup_2025-12-15.sql.gz | mysql -u root -p

```

## Secrets Management

- เพิ่ม secrets แก้ไฟล์ compose
```
services:
  mysql-master:
    image: mysql:8.0
    container_name: mysql-master
    restart: always

    secrets:
      - mysql_root_password
      - mysql_user_password
      - mysql_repl_password

    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/mysql_root_password
      MYSQL_DATABASE: mydb
      MYSQL_USER: appuser
      MYSQL_PASSWORD_FILE: /run/secrets/mysql_user_password

    ports:
      - "3307:3306"

    volumes:
      - ./master/my.cnf:/etc/mysql/conf.d/my.cnf
      - master_data:/var/lib/mysql

    networks:
      - mysql-network

  mysql-slave:
    image: mysql:8.0
    container_name: mysql-slave
    restart: always

    secrets:
      - mysql_root_password
      - mysql_repl_password

    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/mysql_root_password

    ports:
      - "3308:3306"

    volumes:
      - ./slave/my.cnf:/etc/mysql/conf.d/my.cnf
      - slave_data:/var/lib/mysql

    depends_on:
      - mysql-master

    networks:
      - mysql-network

volumes:
  master_data: 
  slave_data:

networks:
  mysql-network:
    driver: bridge

secrets:
  mysql_root_password:
    file: ./secrets/mysql_root_password.txt
  mysql_user_password:
    file: ./secrets/mysql_user_password.txt
  mysql_repl_password:
    file: ./secrets/mysql_repl_password.txt

```
<!-- #### Role (Authentication & Authorization)
#### Row-level / Column-level Security
#### Materialized Views
#### Audit Logs + Monitor
#### Backup/Recovery
#### Tested Restore
#### Secrets Management
#### Replication -->