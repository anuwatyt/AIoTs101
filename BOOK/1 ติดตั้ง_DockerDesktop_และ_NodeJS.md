# คู่มือสรุปการแก้ปัญหา

## ขั้นตอนที่ 1 ติดตั้ง Docker Desktop และขั้นตอนที่ 2 ติดตั้ง Node.js

**ระบบปฏิบัติการ:** Windows 10 22H2 (สามารถประยุกต์ใช้กับ Windows 11 ได้)

------------------------------------------------------------------------

# ขั้นตอนที่ 1 การติดตั้ง Docker Desktop

## วัตถุประสงค์

เตรียมสภาพแวดล้อมสำหรับใช้งาน Docker Compose และ n8n แบบ Localhost

------------------------------------------------------------------------

## ปัญหาที่พบ

หลังติดตั้ง Docker Desktop แล้วไม่สามารถใช้งานได้

ตรวจสอบ

``` powershell
wsl --status
```

พบข้อความ

``` text
Virtual Machine Platform is not enabled.
```

ตรวจสอบ Windows Features

``` powershell
dism /online /get-features /format:table
```

พบว่า

``` text
Microsoft-Windows-Subsystem-Linux   Disabled
VirtualMachinePlatform              Disabled
HypervisorPlatform                  Disabled
```

------------------------------------------------------------------------

## วิเคราะห์สาเหตุ

Docker Desktop ใช้ **WSL2** เป็น Backend

จึงต้องเปิดใช้งาน

-   Microsoft-Windows-Subsystem-Linux
-   VirtualMachinePlatform

ส่วน **HypervisorPlatform** ไม่จำเป็นสำหรับการใช้งาน Docker Desktop ด้วย WSL2

------------------------------------------------------------------------

## วิธีแก้ไข

### 1. เปิด PowerShell แบบ Administrator

### 2. เปิด WSL

``` powershell
dism /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all
```

### 3. เปิด Virtual Machine Platform

``` powershell
dism /online /enable-feature /featurename:VirtualMachinePlatform /all
```

### 4. Restart Windows

------------------------------------------------------------------------

## ปัญหาที่พบต่อ

``` powershell
wsl --status
```

พบข้อความ

``` text
The WSL 2 kernel file is not found.
```

### สาเหตุ

ยังไม่มี WSL2 Kernel

### วิธีแก้

``` powershell
wsl --update
```

หากดาวน์โหลดไม่ได้

``` powershell
wsl --update --web-download
```

------------------------------------------------------------------------

## ตรวจสอบ

``` powershell
wsl --version
```

ตัวอย่าง

``` text
WSL version: 2.7.10.0
Kernel version: 6.18.33.2-2
```

------------------------------------------------------------------------

## ตรวจสอบ Docker

``` powershell
docker --version
```

ผลลัพธ์

``` text
Docker version 29.6.1
```

ตรวจสอบ Docker Compose

``` powershell
docker compose version
```

ผลลัพธ์

``` text
Docker Compose version v5.2.0
```

------------------------------------------------------------------------

## สรุปการแก้ปัญหา Docker Desktop

  ------------------------------------------------------------------------------------
  ปัญหา                                สาเหตุ                  วิธีแก้
  ----------------------------------- ---------------------- -------------------------
  Virtual Machine Platform is not     Virtual Machine        เปิดฟีเจอร์ด้วย DISM
  enabled                             Platform ปิด            

  Microsoft-Windows-Subsystem-Linux   WSL ปิด                 เปิดฟีเจอร์ WSL
  Disabled                                                   

  WSL2 kernel file is not found       ยังไม่มี Kernel           `wsl --update`

  Docker เปิดไม่ได้                      ยังไม่ได้ Restart         รีสตาร์ตเครื่อง

  Docker ใช้งานไม่ได้                    WSL ยังไม่สมบูรณ์          ตรวจสอบ `wsl --version`
  ------------------------------------------------------------------------------------

------------------------------------------------------------------------

# ขั้นตอนที่ 2 การติดตั้ง Node.js

## วัตถุประสงค์

ติดตั้ง Node.js สำหรับพัฒนา JavaScript และใช้งาน npm, npx
รวมถึงเครื่องมือสำหรับการพัฒนา

> หมายเหตุ: n8n ที่รันใน Docker ไม่ได้ใช้ Node.js บน Windows โดยตรง แต่การติดตั้ง
> Node.js มีประโยชน์สำหรับงานพัฒนา

------------------------------------------------------------------------

## การติดตั้ง

ดาวน์โหลดจาก

https://nodejs.org

แนะนำเลือก

-   **LTS (Long Term Support)**

ในการติดตั้งให้เลือก

-   Node.js Runtime
-   npm Package Manager
-   Add to PATH

------------------------------------------------------------------------

## ตรวจสอบการติดตั้ง

``` powershell
node --version
npm --version
npx --version
where node
where npm
```

ผลลัพธ์ตัวอย่าง

``` text
node --version
v24.18.0

npm --version
11.16.0

npx --version
11.16.0

where node
C:\Program Files\nodejs\node.exe

where npm
C:\Program Files\nodejs\npm
C:\Program Files\nodejs\npm.cmd
```

------------------------------------------------------------------------

## วิเคราะห์ผล

  รายการ    สถานะ
  --------- -----------
  Node.js   ติดตั้งสำเร็จ
  npm       พร้อมใช้งาน
  npx       พร้อมใช้งาน
  PATH      ถูกต้อง

------------------------------------------------------------------------

## ปัญหาที่พบบ่อย

  -----------------------------------------------------------------------------
  ปัญหา                         สาเหตุ                  วิธีแก้
  ---------------------------- ---------------------- -------------------------
  `'node' is not recognized`   PATH ไม่ถูกต้อง           เปิด Command Prompt ใหม่
                                                      หรือตรวจสอบ PATH

  `'npm' is not recognized`    ติดตั้งไม่สมบูรณ์            ติดตั้ง Node.js ใหม่

  ติดตั้งไม่ได้                     สิทธิ์ไม่พอ                Run as administrator
  -----------------------------------------------------------------------------

------------------------------------------------------------------------

# สรุปสถานะเครื่องหลังติดตั้ง

  องค์ประกอบ        สถานะ
  ---------------- -------
  Windows          ✅
  WSL2             ✅
  Docker Desktop   ✅
  Docker Engine    ✅
  Docker Compose   ✅
  Node.js          ✅
  npm              ✅
  npx              ✅

------------------------------------------------------------------------

# ขั้นตอนถัดไป

1.  ติดตั้ง Visual Studio Code
2.  สร้างโฟลเดอร์โปรเจกต์ n8n
3.  เขียน `docker-compose.yml`
4.  ติดตั้ง n8n แบบ Localhost
5.  เพิ่ม PostgreSQL
6.  เพิ่ม pgAdmin
7.  เพิ่ม Redis
8.  เชื่อมต่อ AI และ IoT
