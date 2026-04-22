# 🧾 Centralized Logging with Grafana + Loki + Promtail

## 📌 Overview

ระบบนี้ใช้สำหรับรวม logs จากหลาย server มาไว้ที่ศูนย์กลาง เพื่อให้สามารถค้นหา วิเคราะห์ และ debug ได้ง่ายขึ้น

### 🧩 Architecture

```
App Servers (หลายเครื่อง)
   ↓
Promtail (แต่ละเครื่อง)
   ↓
Loki (ศูนย์กลาง)
   ↓
Grafana (Dashboard / Query)
```

---

## 🚀 Stack ที่ใช้

* Grafana → Dashboard / Visualization
* Grafana Loki → Log Storage
* Promtail → Log Collector

---

## ⚙️ การติดตั้ง (Central Server)

### 1. สร้างไฟล์ `docker-compose.yml`

```yaml
version: "3.8"

services:
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana

  loki:
    image: grafana/loki:2.9.0
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - ./loki-config.yaml:/etc/loki/local-config.yaml

volumes:
  grafana-data:
```

---

### 2. สร้างไฟล์ `loki-config.yaml`

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h
```

---

### 3. รันระบบ

```bash
docker compose up -d
```

---

## 🖥️ การติดตั้ง (App Servers ทุกเครื่อง)

### 1. ติดตั้ง Promtail

สร้างไฟล์ `promtail-config.yaml`

```yaml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://<LOKI_IP>:3100/loki/api/v1/push

scrape_configs:
  - job_name: app
    static_configs:
      - targets:
          - localhost
        labels:
          job: app
          service: payment
          server: server-a
          tenant_id: shop_123
          __path__: /var/log/app.log
```

---

### 2. รัน Promtail

```bash
docker run -d \
  -v /var/log:/var/log \
  -v $(pwd)/promtail-config.yaml:/etc/promtail/config.yaml \
  grafana/promtail:2.9.0 \
  -config.file=/etc/promtail/config.yaml
```

---

## 🔗 เชื่อม Loki กับ Grafana

1. เข้า http://localhost:3000
2. ไปที่ **Settings → Data Sources**
3. Add data source → Loki
4. URL:

```
http://loki:3100
```

5. กด Save & Test

---

## 🔍 การใช้งาน (Query Logs)

### ดู log ทั้งหมด

```
{job="app"}
```

### หา error

```
{job="app"} |= "error"
```

### แยกตาม service

```
{service="payment"}
```

### แยกตาม tenant (สำคัญสำหรับ SaaS)

```
{tenant_id="shop_123"} |= "error"
```

---

## 🧠 Best Practices

### ✅ 1. ใช้ Structured Logging (JSON)

```json
{
  "level": "error",
  "message": "payment failed",
  "order_id": 123,
  "tenant_id": "shop_123"
}
```

---

### ✅ 2. ตั้ง Labels ให้เหมาะสม

ใช้:

* tenant_id
* service
* server
* env

หลีกเลี่ยง:

* order_id ❌ (cardinality สูง)

---

### ✅ 3. แยก Logs ตาม Service

```
auth-service
payment-service
order-service
```

---

### ✅ 4. รองรับ Multi-Server

ทุก server:

* ติดตั้ง Promtail
* ส่ง logs ไป Loki กลาง

---

## ⚡ Advanced (Production)

### 🔹 ใช้ S3 Storage

* ลด disk usage
* scale ได้ไม่จำกัด

### 🔹 Loki Cluster

* รองรับ traffic สูง

### 🔹 Alerting

* แจ้งเตือนผ่าน Telegram / Slack

---

## 🎯 เหมาะกับ Use Case

* SaaS Multi-tenant
* POS / Inventory System
* Microservices Architecture
* High traffic system

---

## 📦 สรุป

| Component | หน้าที่            |
| --------- | ------------------ |
| Promtail  | Collect logs       |
| Loki      | Store + Query logs |
| Grafana   | Visualize          |

---

## 🚀 Next Steps

* เพิ่ม Dashboard (Error Rate / Slow API)
* ทำ Alert แจ้งเตือน
* Integrate กับ Golang Middleware
* แยก environment (dev / staging / prod)

---

**💡 Tip:**
ถ้าระบบคุณเริ่มใหญ่ ให้เพิ่ม label `env=production` และแยก cluster จะช่วย debug ง่ายขึ้นมาก

---
