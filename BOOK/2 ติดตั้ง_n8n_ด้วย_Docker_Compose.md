# คู่มือติดตั้งระบบ n8n for AIoT ด้วย Docker Compose (Localhost)

## ขั้นตอนที่ 3 การติดตั้งและรัน n8n, Node-RED, MQTT Broker และ Ngrok

**ระบบปฏิบัติการ:** Windows 10 22H2 (สามารถประยุกต์ใช้กับ Windows 11 ได้)

> หมายเหตุ: ขั้นตอนนี้ต่อจาก [1 ติดตั้ง_DockerDesktop_และ_NodeJS.md](1%20ติดตั้ง_DockerDesktop_และ_NodeJS.md)
> ต้องติดตั้ง Docker Desktop และเปิดใช้งาน WSL2 ให้เรียบร้อยก่อน

------------------------------------------------------------------------

## วัตถุประสงค์

รันระบบ AIoT ครบชุดผ่าน Docker Compose แบบ Localhost ประกอบด้วย 4 Service

1. **n8n** — Workflow Automation
2. **Node-RED** — Flow-based Programming สำหรับเชื่อมต่ออุปกรณ์ IoT
3. **MQTT Broker (Eclipse Mosquitto)** — จุดกลางรับ-ส่งข้อมูลระหว่างอุปกรณ์
4. **Ngrok** — เปิด Tunnel จาก Internet เข้าสู่ n8n ที่รันบน Localhost
   เพื่อให้บริการภายนอก (เช่น LINE Notify, GitHub Webhook, อุปกรณ์ IoT
   จากนอกเครือข่าย) เรียก Webhook ของ n8n ได้จริง

รูปแบบการทำงาน: อุปกรณ์ IoT / Node-RED Publish ข้อมูลเข้า Mosquitto แล้ว
n8n หรือ Node-RED Subscribe ต่อผ่าน MQTT เพื่อประมวลผลหรือ Automate ต่อ
ส่วน Webhook จากภายนอกจะวิ่งผ่าน Ngrok เข้ามาที่ n8n โดยตรง

------------------------------------------------------------------------

## โครงสร้างไฟล์

```
AIoTs101/
├── docker-compose.yml
├── .env.example
├── .env                        (สร้างเอง ไม่ commit ขึ้น Git)
└── mosquitto/
    └── config/
        └── mosquitto.conf
```

------------------------------------------------------------------------

## ไฟล์ docker-compose.yml

``` yaml
services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=${NGROK_DOMAIN}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=https://${NGROK_DOMAIN}/
      - GENERIC_TIMEZONE=Asia/Bangkok
      - TZ=Asia/Bangkok
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - mosquitto

  nodered:
    image: nodered/node-red:latest
    container_name: nodered
    restart: unless-stopped
    ports:
      - "1880:1880"
    environment:
      - TZ=Asia/Bangkok
    volumes:
      - nodered_data:/data
    depends_on:
      - mosquitto

  mosquitto:
    image: eclipse-mosquitto:latest
    container_name: mosquitto
    restart: unless-stopped
    ports:
      - "1883:1883"
      - "9001:9001"
    volumes:
      - ./mosquitto/config:/mosquitto/config
      - mosquitto_data:/mosquitto/data
      - mosquitto_log:/mosquitto/log

  ngrok:
    image: ngrok/ngrok:latest
    container_name: ngrok
    restart: unless-stopped
    environment:
      - NGROK_AUTHTOKEN=${NGROK_AUTHTOKEN}
    command:
      - "http"
      - "n8n:5678"
      - "--domain=${NGROK_DOMAIN}"
    ports:
      - "4040:4040"
    depends_on:
      - n8n

volumes:
  n8n_data:
  nodered_data:
  mosquitto_data:
  mosquitto_log:
```

ข้อมูล Workflow/Credentials ของ n8n เก็บใน Volume `n8n_data` (SQLite
ในตัว) และข้อมูล Flow ของ Node-RED เก็บใน Volume `nodered_data`
ทำให้ข้อมูลไม่หายแม้ลบ Container แล้วสร้างใหม่

> `N8N_HOST` / `WEBHOOK_URL` ถูกตั้งเป็นโดเมนของ Ngrok (ไม่ใช่
> `localhost`) เพื่อให้ Webhook Node ของ n8n สร้าง URL ที่เรียกจาก
> ภายนอกได้จริง ส่วนหน้า Editor ยังเข้าผ่าน `http://localhost:5678`
> ได้ตามปกติจากเครื่องตัวเอง

------------------------------------------------------------------------

## ไฟล์ mosquitto/config/mosquitto.conf

``` text
listener 1883
allow_anonymous true

listener 9001
protocol websockets
allow_anonymous true

persistence true
persistence_location /mosquitto/data/

log_dest file /mosquitto/log/mosquitto.log
log_dest stdout
```

  Port    โปรโตคอล            การใช้งาน
  ------- ------------------ -----------------------------------
  1883    MQTT (TCP)         อุปกรณ์ IoT / n8n MQTT Node เชื่อมต่อ
  9001    MQTT over WebSocket สำหรับ Client บนเว็บเบราว์เซอร์

> ตั้งค่า `allow_anonymous true` เพื่อความสะดวกในการทดสอบบน Localhost
> เท่านั้น หากนำไปใช้งานจริงหรือเปิดพอร์ตออกนอกเครื่อง
> ควรเปลี่ยนไปใช้ Username/Password หรือ TLS Certificate

------------------------------------------------------------------------

## เตรียมบัญชี Ngrok

1. สมัครสมาชิกฟรีที่ <https://dashboard.ngrok.com/signup>
2. คัดลอก Authtoken จาก
   <https://dashboard.ngrok.com/get-started/your-authtoken>
3. สร้าง Static Domain ฟรี (1 โดเมนต่อบัญชี) ที่
   <https://dashboard.ngrok.com/domains> จะได้โดเมนรูปแบบ
   `xxxxx.ngrok-free.app`

> ต้องใช้ **Static Domain** แทน Domain แบบสุ่ม เพราะ Domain แบบสุ่ม
> จะเปลี่ยนทุกครั้งที่ Container รีสตาร์ต ทำให้ Webhook URL ที่บันทึกไว้
> ใน Workflow ของ n8n ใช้งานไม่ได้อีก

------------------------------------------------------------------------

## ขั้นตอนการติดตั้ง

### 1. คัดลอกไฟล์ตัวอย่าง .env

``` powershell
Copy-Item .env.example .env
```

### 2. แก้ไขไฟล์ .env

``` env
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=changeme
N8N_ENCRYPTION_KEY=replace_with_a_long_random_string
NGROK_AUTHTOKEN=<authtoken จาก Ngrok Dashboard>
NGROK_DOMAIN=<static domain จาก Ngrok เช่น xxxxx.ngrok-free.app>
```

> ควรเปลี่ยน `N8N_BASIC_AUTH_PASSWORD` และ `N8N_ENCRYPTION_KEY`
> เป็นค่าที่คาดเดายาก โดยเฉพาะ `N8N_ENCRYPTION_KEY` ห้ามเปลี่ยนภายหลัง
> เพราะจะทำให้ Credentials ที่เข้ารหัสไว้เดิมใช้งานไม่ได้

### 3. สั่งรัน Container

``` powershell
docker compose up -d
```

### 4. ตรวจสอบสถานะ

``` powershell
docker compose ps
```

ผลลัพธ์ที่ควรเห็น

``` text
NAME        IMAGE                    STATUS
n8n         n8nio/n8n:latest         Up (healthy)
nodered     nodered/node-red:latest  Up
mosquitto   eclipse-mosquitto:latest Up
ngrok       ngrok/ngrok:latest       Up
```

### 5. เปิดใช้งาน n8n และ Node-RED

เปิดเบราว์เซอร์ไปที่

``` text
n8n (จากในเครื่อง):     http://localhost:5678
n8n (จากภายนอกผ่าน Ngrok): https://<NGROK_DOMAIN>
Node-RED:               http://localhost:1880
```

ใส่ Username / Password ของ n8n ตามที่ตั้งไว้ใน `.env`
(Node-RED ยังไม่ได้ตั้ง Login เริ่มต้น เข้าใช้งานได้ทันที)

### 6. ตรวจสอบ Ngrok Tunnel

เปิด Ngrok Web Inspector เพื่อดูสถานะ Tunnel และ Request ที่วิ่งผ่านเข้ามา

``` text
http://localhost:4040
```

ถ้าเชื่อมต่อสำเร็จจะเห็น Status เป็น `online` และ URL ตรงกับ
`NGROK_DOMAIN` ที่ตั้งไว้ ทดสอบ Webhook จากภายนอกได้ด้วยการสร้าง
Webhook Node ใน n8n แล้วเรียกผ่าน `https://<NGROK_DOMAIN>/webhook/...`

### 7. ทดสอบ MQTT Broker

ทดสอบ Publish/Subscribe ด้วย mosquitto client ที่รันในตัว Container เอง
(ไม่ต้องติดตั้งอะไรเพิ่มบนเครื่อง)

``` powershell
# Terminal ที่ 1: subscribe รอรับข้อความ
docker exec -it mosquitto mosquitto_sub -h localhost -t "test/topic"

# Terminal ที่ 2: publish ข้อความทดสอบ
docker exec -it mosquitto mosquitto_pub -h localhost -t "test/topic" -m "Hello n8n"
```

ถ้า Terminal ที่ 1 แสดงข้อความ `Hello n8n` แปลว่า Mosquitto ทำงานถูกต้อง

ทั้ง n8n และ Node-RED เชื่อมต่อ MQTT โดยตั้งค่า Host เป็นชื่อ Service
`mosquitto` (คนละชื่อ Container ใน Docker Network เดียวกัน) พอร์ต `1883`

  Service   วิธีเชื่อมต่อ MQTT
  --------- ----------------------------------------------------
  n8n       เพิ่ม **MQTT Trigger Node** / **MQTT Node** ตั้ง
            Credential Host = `mosquitto`, Port = `1883`
  Node-RED  ลาก Node **mqtt in** / **mqtt out** จาก Palette
            ตั้งค่า Server = `mosquitto:1883`

------------------------------------------------------------------------

## คำสั่งที่ใช้บ่อย

  คำสั่ง                              ความหมาย
  ----------------------------------- ----------------------------------------------------
  `docker compose up -d`              รัน n8n, nodered, mosquitto, ngrok แบบ Background
  `docker compose down`               หยุดและลบ Container (Volume คงอยู่)
  `docker compose logs -f n8n`        ดู Log ของ n8n แบบ Real-time
  `docker compose logs -f nodered`    ดู Log ของ Node-RED แบบ Real-time
  `docker compose logs -f mosquitto`  ดู Log ของ Mosquitto แบบ Real-time
  `docker compose logs -f ngrok`      ดู Log ของ Ngrok Tunnel แบบ Real-time
  `docker compose restart n8n`        รีสตาร์ต n8n
  `docker compose pull`               ดึง Image เวอร์ชันล่าสุด

------------------------------------------------------------------------

## ปัญหาที่พบบ่อย

  ------------------------------------------------------------------------
  ปัญหา                          สาเหตุ                    วิธีแก้
  ------------------------------ ------------------------ ------------------
  เข้า localhost:5678 ไม่ได้        Docker Desktop ยังไม่รัน     เปิด Docker Desktop
                                                            รอจน Engine Running

  Container ขึ้น Exit ทันที         ไม่มีไฟล์ .env             สร้างไฟล์ .env
                                                            ตามขั้นตอนที่ 1-2

  Port 5678 ถูกใช้งานอยู่           มีโปรแกรมอื่นใช้ Port      แก้ ports เป็น
                                    เดียวกัน                  `"5679:5678"`
                                                            แล้วเข้า
                                                            localhost:5679

  ลืม Password Basic Auth         -                        ลบ Container แล้ว
                                                            แก้ .env ใหม่
                                                            (ข้อมูล Workflow
                                                            ไม่หาย เพราะอยู่
                                                            ใน Volume)

  n8n ต่อ MQTT ไม่ได้              ใช้ `localhost` เป็น Host        ใช้ชื่อ Service
                                    ใน MQTT Node (คนละ            `mosquitto` แทน
                                    Network Namespace กับ         เพราะอยู่ใน
                                    Container mosquitto)          Docker Network
                                                                  เดียวกัน

  Ngrok ขึ้น `ERR_NGROK_...`      ไม่ได้ตั้ง                     ตรวจสอบ `.env`
  หรือ Container Exit ทันที        `NGROK_AUTHTOKEN`               ว่ามีค่า
                                    หรือค่าไม่ถูกต้อง               `NGROK_AUTHTOKEN`
                                                                    ถูกต้อง

  Ngrok Domain ใช้ไม่ได้           ยังไม่ได้สร้าง Static Domain      สร้าง Static
  (`domain not found`)             ในบัญชี Ngrok หรือพิมพ์          Domain ที่
                                    `NGROK_DOMAIN` ผิด               dashboard.ngrok
                                                                    .com/domains

  Webhook ใน n8n เป็น              ตั้ง `WEBHOOK_URL` /            ตรวจสอบว่า
  `http://localhost:5678/...`      `N8N_HOST` ผิด หรือยังไม่ได้     `.env` มีค่า
  ไม่ใช่ Ngrok Domain               รีสตาร์ต Container n8n          `NGROK_DOMAIN`
                                    หลังแก้ .env                    แล้วรัน
                                                                    `docker compose
                                                                    up -d --force-
                                                                    recreate n8n`
  ------------------------------------------------------------------------

------------------------------------------------------------------------

## สรุปสถานะหลังติดตั้ง

  องค์ประกอบ                          สถานะ
  ------------------------------------ -------
  Docker Desktop                       ✅
  docker-compose.yml                   ✅
  ไฟล์ .env                             ✅
  mosquitto.conf                       ✅
  บัญชี Ngrok + Static Domain            ✅
  n8n Container                        ✅
  Node-RED Container                   ✅
  mosquitto Container                  ✅
  ngrok Container                      ✅
  เข้าถึง n8n ผ่าน localhost:5678        ✅
  เข้าถึง n8n ผ่าน Ngrok Domain (Webhook) ✅
  เข้าถึง Node-RED ผ่าน localhost:1880   ✅
  ทดสอบ MQTT Pub/Sub                     ✅

------------------------------------------------------------------------
