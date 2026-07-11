# AIoTs101

ระบบ **AIoT (AI + IoT) Automation** ที่รันทั้งหมดบนเครื่องตัวเอง (Localhost)
ผ่าน **Docker Compose** ใช้สำหรับเชื่อมต่ออุปกรณ์ IoT เข้ากับ Workflow
Automation และเปิดให้บริการภายนอกเรียกเข้ามาได้จริงผ่าน Ngrok Tunnel

------------------------------------------------------------------------

## 1. ภาพรวมระบบ

ระบบประกอบด้วย 4 Container ที่ทำงานร่วมกันบน Docker Network เดียวกัน

| # | Service                        | หน้าที่ |
|---|---------------------------------|---------|
| 1 | **n8n**                         | Workflow Automation — สร้างเงื่อนไข/Automation, รับ Webhook, เชื่อมต่อ API ภายนอก |
| 2 | **Node-RED**                    | Flow-based Programming — เชื่อมต่อ/ประมวลผลข้อมูลจากอุปกรณ์ IoT |
| 3 | **MQTT Broker (Eclipse Mosquitto)** | จุดกลางรับ-ส่งข้อมูล (Pub/Sub) ระหว่างอุปกรณ์ IoT, Node-RED และ n8n |
| 4 | **Ngrok**                       | เปิด Tunnel จาก Internet เข้าสู่ n8n ที่รันบน Localhost เพื่อให้ Webhook จากภายนอกเรียกเข้ามาได้จริง |

### แผนผังการทำงาน

```
อุปกรณ์ IoT ──┐
              ├─► Mosquitto (MQTT Broker) ◄──► Node-RED ──► n8n (Automation)
Node-RED ─────┘                                                 │
                                                                  ▼
                                              Ngrok Tunnel ◄──── n8n Webhook
                                                    │
                                                    ▼
                                     บริการภายนอก (LINE Notify, GitHub
                                     Webhook, อุปกรณ์ IoT นอกเครือข่าย ฯลฯ)
```

- อุปกรณ์ IoT หรือ Node-RED **Publish** ข้อมูลเข้า Mosquitto
- n8n หรือ Node-RED **Subscribe** ข้อมูลจาก Mosquitto เพื่อนำไปประมวลผลหรือ Automate ต่อ
- คำขอ (Request) จากภายนอกอินเทอร์เน็ตวิ่งผ่าน **Ngrok** เข้ามาที่ n8n โดยตรง (เช่น Webhook)

------------------------------------------------------------------------

## 2. โครงสร้างไฟล์ในโปรเจกต์

```
AIoTs101/
├── docker-compose.yml          # นิยาม Container ทั้ง 4 ตัว
├── .env.example                # ตัวอย่างค่า Environment Variable
├── .env                        # สร้างเอง ไม่ commit ขึ้น Git
├── mosquitto/
│   └── config/
│       └── mosquitto.conf      # ค่าตั้งต้นของ MQTT Broker
└── BOOK/                       # เอกสารคู่มือแบบละเอียดทีละขั้นตอน
    ├── 1 ติดตั้ง_DockerDesktop_และ_NodeJS.md
    ├── 2 ติดตั้ง_n8n_ด้วย_Docker_Compose.md
    └── เครื่องมือ.md            # โปรแกรม/Extension ที่ใช้พัฒนา
```

------------------------------------------------------------------------

## 3. สิ่งที่ต้องเตรียมก่อนติดตั้ง

- **Windows 10 22H2** ขึ้นไป (ใช้กับ Windows 11 ได้เช่นกัน)
- **Docker Desktop** (ใช้ WSL2 เป็น Backend)
- บัญชี **Ngrok** ฟรี พร้อม Static Domain (สำหรับเปิด Webhook ให้ภายนอกเรียกได้)

> รายละเอียดการติดตั้ง Docker Desktop และแก้ปัญหา WSL2 ดูได้ที่
> [1 ติดตั้ง_DockerDesktop_และ_NodeJS.md](BOOK/1%20ติดตั้ง_DockerDesktop_และ_NodeJS.md)

------------------------------------------------------------------------

## 4. ขั้นตอนการติดตั้งระบบลงบน Docker Desktop

### ขั้นที่ 1 — ตรวจสอบว่า Docker Desktop พร้อมใช้งาน

```powershell
docker --version
docker compose version
```

ต้องเห็นเลขเวอร์ชันทั้งสองคำสั่ง และ Docker Desktop ต้องมีสถานะ
**Engine Running** (ไอคอนวาฬที่ System Tray)

### ขั้นที่ 2 — เตรียมบัญชี Ngrok

1. สมัครสมาชิกฟรีที่ <https://dashboard.ngrok.com/signup>
2. คัดลอก Authtoken จาก <https://dashboard.ngrok.com/get-started/your-authtoken>
3. สร้าง **Static Domain** ฟรี (1 โดเมนต่อบัญชี) ที่
   <https://dashboard.ngrok.com/domains> จะได้โดเมนรูปแบบ
   `xxxxx.ngrok-free.app`

> ต้องใช้ Static Domain เพราะโดเมนแบบสุ่มจะเปลี่ยนทุกครั้งที่ Container
> รีสตาร์ต ทำให้ Webhook URL ที่บันทึกไว้ใน n8n ใช้งานไม่ได้อีก

### ขั้นที่ 3 — คัดลอกไฟล์ตัวอย่าง .env

```powershell
Copy-Item .env.example .env
```

### ขั้นที่ 4 — แก้ไขค่าในไฟล์ .env

```env
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=changeme
N8N_ENCRYPTION_KEY=replace_with_a_long_random_string
NGROK_AUTHTOKEN=<authtoken จาก Ngrok Dashboard>
NGROK_DOMAIN=<static domain จาก Ngrok เช่น xxxxx.ngrok-free.app>
```

> ควรเปลี่ยน `N8N_BASIC_AUTH_PASSWORD` และ `N8N_ENCRYPTION_KEY` ให้เป็นค่า
> ที่คาดเดายาก โดยเฉพาะ `N8N_ENCRYPTION_KEY` **ห้ามเปลี่ยนภายหลัง**
> เพราะจะทำให้ Credentials เดิมที่เข้ารหัสไว้ใช้งานไม่ได้

### ขั้นที่ 5 — สั่งรัน Container ทั้งหมด

```powershell
docker compose up -d
```

### ขั้นที่ 6 — ตรวจสอบสถานะ Container

```powershell
docker compose ps
```

ผลลัพธ์ที่ควรเห็น

```text
NAME        IMAGE                    STATUS
n8n         n8nio/n8n:latest         Up (healthy)
nodered     nodered/node-red:latest  Up
mosquitto   eclipse-mosquitto:latest Up
ngrok       ngrok/ngrok:latest       Up
```

### ขั้นที่ 7 — เปิดใช้งานแต่ละบริการ

| บริการ | URL |
|--------|-----|
| n8n (จากในเครื่อง) | http://localhost:5678 |
| n8n (จากภายนอกผ่าน Ngrok) | https://\<NGROK_DOMAIN\> |
| Node-RED | http://localhost:1880 |
| Ngrok Web Inspector | http://localhost:4040 |

ใส่ Username/Password ของ n8n ตามที่ตั้งไว้ใน `.env`
(Node-RED ยังไม่ได้ตั้ง Login เริ่มต้น เข้าใช้งานได้ทันที)

### ขั้นที่ 8 — ทดสอบ MQTT Broker

```powershell
# Terminal ที่ 1: subscribe รอรับข้อความ
docker exec -it mosquitto mosquitto_sub -h localhost -t "test/topic"

# Terminal ที่ 2: publish ข้อความทดสอบ
docker exec -it mosquitto mosquitto_pub -h localhost -t "test/topic" -m "Hello n8n"
```

ถ้า Terminal ที่ 1 แสดงข้อความ `Hello n8n` แปลว่า Mosquitto ทำงานถูกต้อง

> ทั้ง n8n และ Node-RED เชื่อมต่อ MQTT โดยตั้งค่า **Host = `mosquitto`**
> (ชื่อ Service ใน Docker Network เดียวกัน ไม่ใช่ `localhost`) พอร์ต `1883`

------------------------------------------------------------------------

## 5. คำสั่งที่ใช้บ่อย

| คำสั่ง | ความหมาย |
|--------|----------|
| `docker compose up -d` | รันทุก Service แบบ Background |
| `docker compose down` | หยุดและลบ Container (ข้อมูลใน Volume ยังอยู่) |
| `docker compose ps` | ดูสถานะ Container |
| `docker compose logs -f n8n` | ดู Log ของ n8n แบบ Real-time |
| `docker compose logs -f nodered` | ดู Log ของ Node-RED แบบ Real-time |
| `docker compose logs -f mosquitto` | ดู Log ของ Mosquitto แบบ Real-time |
| `docker compose logs -f ngrok` | ดู Log ของ Ngrok Tunnel แบบ Real-time |
| `docker compose restart n8n` | รีสตาร์ต n8n |
| `docker compose pull` | ดึง Image เวอร์ชันล่าสุด |

------------------------------------------------------------------------

## 6. ปัญหาที่พบบ่อย

| ปัญหา | สาเหตุ | วิธีแก้ |
|-------|--------|---------|
| เข้า `localhost:5678` ไม่ได้ | Docker Desktop ยังไม่รัน | เปิด Docker Desktop รอจน Engine Running |
| Container ขึ้น Exit ทันที | ไม่มีไฟล์ `.env` | สร้างไฟล์ `.env` ตามขั้นที่ 3-4 |
| Port 5678 ถูกใช้งานอยู่ | มีโปรแกรมอื่นใช้ Port เดียวกัน | แก้ `ports` เป็น `"5679:5678"` แล้วเข้า `localhost:5679` |
| ลืม Password Basic Auth | - | ลบ Container แล้วแก้ `.env` ใหม่ (ข้อมูล Workflow ไม่หาย เพราะอยู่ใน Volume) |
| n8n ต่อ MQTT ไม่ได้ | ใช้ `localhost` เป็น Host ใน MQTT Node | ใช้ชื่อ Service `mosquitto` แทน |
| Ngrok ขึ้น `ERR_NGROK_...` หรือ Container Exit ทันที | ไม่ได้ตั้ง `NGROK_AUTHTOKEN` หรือค่าไม่ถูกต้อง | ตรวจสอบค่า `NGROK_AUTHTOKEN` ใน `.env` |
| Ngrok Domain ใช้ไม่ได้ (`domain not found`) | ยังไม่ได้สร้าง Static Domain หรือพิมพ์ `NGROK_DOMAIN` ผิด | สร้าง Static Domain ที่ dashboard.ngrok.com/domains |
| Webhook ใน n8n ยังเป็น `http://localhost:5678/...` | `.env` ยังไม่ได้ตั้งหรือยังไม่รีสตาร์ต n8n | แก้ `.env` แล้วรัน `docker compose up -d --force-recreate n8n` |

------------------------------------------------------------------------

## 7. ข้อมูลจะหายไหมถ้าลบ Container

ไม่หาย เพราะข้อมูลถูกเก็บแยกไว้ใน **Docker Volume**

| Volume | เก็บอะไร |
|--------|----------|
| `n8n_data` | Workflow และ Credentials ของ n8n (SQLite ในตัว) |
| `nodered_data` | Flow ของ Node-RED |
| `mosquitto_data` / `mosquitto_log` | ข้อมูลและ Log ของ MQTT Broker |

การรัน `docker compose down` แล้ว `docker compose up -d` ใหม่จะไม่ทำให้ข้อมูลหาย
ต้องใช้ `docker compose down -v` เท่านั้นถึงจะลบ Volume ทิ้งไปด้วย

------------------------------------------------------------------------

## 8. เอกสารเพิ่มเติม

- [BOOK/1 ติดตั้ง_DockerDesktop_และ_NodeJS.md](BOOK/1%20ติดตั้ง_DockerDesktop_และ_NodeJS.md) — วิธีติดตั้ง Docker Desktop, แก้ปัญหา WSL2 และติดตั้ง Node.js
- [BOOK/2 ติดตั้ง_n8n_ด้วย_Docker_Compose.md](BOOK/2%20ติดตั้ง_n8n_ด้วย_Docker_Compose.md) — รายละเอียดเชิงลึกของ docker-compose.yml, mosquitto.conf และการตั้งค่าทั้งหมด
- [BOOK/เครื่องมือ.md](BOOK/เครื่องมือ.md) — โปรแกรมและ Extension ที่ใช้ในการพัฒนา
