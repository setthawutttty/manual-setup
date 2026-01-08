# backup portgres windowserver

- หาที่อยู่ portgres เข้าไปเช็ค db
```command line
psgl.exe -p 4432 -U (user เข้า db) -d postgres
```

- backup
```
pg_dump.exe -p 4432 -U (user เข้า db) -v -b -f t -f (ชื่อไฟล์.tar) QSR
```