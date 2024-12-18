# วิธีติดตั้ง Redis Server บน Ubuntu 23.10 (x86_64)

## หัวข้อการติดตั้ง
- ตรวจสอบความพร้อมของระบบ
- ติดตั้ง Redis
- เปิดใช้งานและเริ่มการใช้งาน
- ตั้งค่าความปลอดภัยให้กับระบบ
- กำหนดค่าการเก็บของมูล

## ตรวจสอบความพร้อมของระบบ
ตรวจสอบให้แน่ใจว่าระบบได้ทำการ Allow การทำงานของ OpenSSH แล้วหรือยัง หากยังให้ทำตามขั้นตอนต่อไปนี้

1. ทำการ Allow OpenSSH ด้วยคำสั่ง
```
sudo ufw allow OpenSSH
```

2. เปิดการใช้งาน Firewall
```
sudo ufw enable
```

3. ตรวจสอบให้แน่ใจว่าว่า Firewall ทำการ Allow ให้กับ Protocol อะไรบ้าง โดยเฉพาะ OpenSSH ด้วยคำสั่ง
```
sudo ufw status
```
ระบบจะแสดงผลดังต่อไปนี้
```
 To                         Action      From
 --                         ------      ----
 OpenSSH                    ALLOW       Anywhere
 OpenSSH (v6)               ALLOW       Anywhere (v6)
 ```

1. ติดตั้ง Redis
เพิ่ม Redis repository ใน Ubuntu source repositories
```
sudo add-apt-repository ppa:redislabs/redis
```

2. ทำการ Update ubuntu
```
 sudo apt update
```

3. ทำการติดตั้ง Redis
```
sudo apt install redis-server
```

## เปิดใช้งานและเริ่มการใช้งาน
1. แก้ไขไฟล์ `/etc/redis/redis.conf`
```
sudo nano /etc/redis/redis.conf
```
ค้นหาคำว่า `supervised` และเปิดคอมเม้นโดยการเอา # ออก แล้วใส่ค่า `systemd` 
```
...
supervised systemd
...
```
เปลี่ยน `bind 127.0.0.1` เป็น `bind 0.0.0.0` เพื่อให้เชื่อมต่อด้วยการ Remote จากภายนอกได้

2. ทำการ Restart serve
```
sudo systemctl restart redis-server
```

3. ทดสอบด้วยคำสั่ง
```
redis-cli
```

4. พิมพ์คำว่า ping เพื่อทดสอบ
```
PING
```
ระบบจะแสดงตัวอย่างดังต่อไปนี้
```
PONG
```

5. ใช้คำสั่ง `SET` เพื่อสร้าง `key-value` ระบบจะตอบกลับเป็นคำว่า `OK` แสดงว่าเรียบร้อย
```
SET server:name "foo"
```

6. ตรวจสอบว่า `key-value` ที่สร้างใช้ได้หรือไม่ด้วยคำสั่ง
```
GET server:name
```
ระบบจะตอบกลับตัวอย่าง
```
foo
```

## ตั้งค่าความปลอดภัยให้กับระบบ

### สร้าง Password ในการเชื่อมต่อระบบ
1. แก้ไขไฟล์ `/etc/redis/redis.conf`
```
sudo nano /etc/redis/redis.conf
```
ค้นหาคำว่า `requirepass` แล้วใส่ password ที่คุณต้องการ
```
...
requirepass [yourpassword]
...
```
จากนั้นทำการบันทึกและออกจากการแก้ไขไฟล์

2. ทำการ Restart service ของ Redis
```
sudo systemctl restart redis-server
```

3. เมื่อทดลองใช้คำสั่ง `redis-cli` และพิมพ์คำสั่ง `SET server:name "foo2"` ระบบจะแสดงตัวอย่างดังนี้
```
(error) NOAUTH Authentication required.
```
แสดงการตั้งค่าสำเร็จ

4. กรณีต้องการเชื่อมต่อด้วย password ให้ใช้คำสั่ง `redis-cli`
```
AUTH [yourpassword]
```
แล้วลองใช้คำสั่งเดิม `SET server:name "foo2"` ระบบจะตอบกลับ `OK` แสดงว่าสำเร็จ

### สร้าง User และให้สิทธิ์การใช้งาน
1. ทำการเรียกดู User ภายในระบบด้วยคำสั่ง
```
redis-cli
```
จากนั้น
```
ACL LIST
```
ระบบจะแสดงรายการ user default ที่ 1 ตามตัวอย่างดังต่อไปนี้
```
127.0.0.1:6379> ACL LIST
1) "user default on #0bdc51691af3caf87e06159759a60efc86ed676bddc1740c2d711510cfb31f15 ~* &* +@all"
```

2. การสร้าง user เพิ่มเติมให้ทำการเปิดไฟล์ `/etc/redis/redis.conf` ตามตัวอย่างต่อไปนี้
```
sudo nano /etc/redis/redis.conf
```
จากนั้นเพิ่มคำสั่งสร้าง user และกำหนด password ตามจำนวนที่ต้องการ ในไฟล์ตามตัวอย่าง
```
...
user [YOUR USERNAME 2] +@all allkeys on >[YOUR PASSWORD 2]
user [YOUR USERNAME 3] +@all -SET allkeys on >[YOUR PASSWORD 3]
...
```

3. ทำการ Restart service
```
sudo systemctl restart redis.service
```

4. ทดลองเชื่อมต่อด้วย username/password ที่สร้างใหม่
```
redis-cli
> AUTH user2 [YOUR PASSWORD 2]
> SET server:name "foo2"
```

## กำหนดค่าการเก็บของมูล

1. แก้ไขไฟล์ `/etc/redis/redis.conf` ด้วยคำสั่งต่อไปนี้

```
...
save 900 1
save 300 10
save 60 10000
...
appendonly yes
...
appendfsync everysec
...
```

2. ทำการ Restart service

```
sudo systemctl restart redis.service
```

Credit: <a href="https://www.linode.com/docs/guides/install-redis-ubuntu/">Install and Configure Redis on Ubuntu 20.04</a>
