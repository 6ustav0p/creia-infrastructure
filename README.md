# 🏗️ WhatsApp Sessions Service - Diagramas de Infraestructura

## 📊 Vista General de Auto-Scaling

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         INTERNET / USUARIOS                              │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 │ HTTPS
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    DigitalOcean Load Balancer                            │
│                  (Round Robin / Health Checks)                           │
└────────────┬────────────┬────────────┬────────────┬─────────────────────┘
             │            │            │            │
             │            │            │            │ (Auto-scaled instances)
    ┌────────▼───┐  ┌─────▼────┐  ┌───▼──────┐  ┌─▼────────┐
    │ Droplet #1 │  │Droplet #2│  │Droplet #3│  │Droplet #N│
    │ IP: x.x.1  │  │IP: x.x.2 │  │IP: x.x.3 │  │IP: x.x.N │
    ├────────────┤  ├──────────┤  ├──────────┤  ├──────────┤
    │   Docker   │  │  Docker  │  │  Docker  │  │  Docker  │
    │  WhatsApp  │  │ WhatsApp │  │ WhatsApp │  │ WhatsApp │
    │  Sessions  │  │ Sessions │  │ Sessions │  │ Sessions │
    │            │  │          │  │          │  │          │
    │ 50/50 ses. │  │ 45/50    │  │ 30/50    │  │  5/50    │
    │ 🔴 FULL    │  │ 🟡 HIGH  │  │ 🟢 OK    │  │ 🟢 OK    │
    └────┬───────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘
         │               │              │             │
         └───────────────┴──────────────┴─────────────┘
                         │
                         │ MongoDB Protocol
                         ▼
         ┌───────────────────────────────────────┐
         │  DigitalOcean Managed MongoDB Cluster │
         │  - baileys_creds (credenciales WA)   │
         │  - baileys_keys (claves E2E)         │
         │  - businesses (negocios)             │
         │  - sessions (auth compartida)        │
         └───────────────┬───────────────────────┘
                         │
                         │
         ┌───────────────▼───────────────────────┐
         │        Orchestrator Service            │
         │  (Monitorea cada 5 minutos)           │
         │                                        │
         │  ┌──────────────────────────────┐    │
         │  │ while(true) {                │    │
         │  │   checkAllInstances()        │    │
         │  │   if (capacity >= 90%) {     │    │
         │  │     createNewDroplet()       │    │
         │  │   }                          │    │
         │  │   sleep(5min)                │    │
         │  │ }                            │    │
         │  └──────────────────────────────┘    │
         └───────────────────────────────────────┘
                         │
                         │ DigitalOcean API
                         ▼
         ┌───────────────────────────────────────┐
         │      DigitalOcean Control Plane       │
         │  - Create Droplets                    │
         │  - Container Registry                 │
         │  - Monitoring                         │
         └───────────────────────────────────────┘
```

---

## 🔄 Flujo de Auto-Scaling Detallado

```
START
  │
  ▼
┌────────────────────────────────────────┐
│ Orchestrator inicia                    │
│ - Descubre instancias existentes      │
│ - Registra en pool de monitoreo       │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│ Ciclo cada 5 minutos                   │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│ Para cada instancia en pool:           │
│   GET http://ip:3001/metrics/capacity  │
│   Header: X-Shared-Secret              │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│ Recibe respuesta:                      │
│ {                                      │
│   capacity: {                          │
│     max: 50,                           │
│     active: 47,                        │
│     utilizationPercent: 94             │
│   },                                   │
│   autoscaling: {                       │
│     shouldScale: true,                 │
│     recommendation: "CREATE_NEW"       │
│   }                                    │
│ }                                      │
└────────────────┬───────────────────────┘
                 │
                 ▼
        ┌────────────────┐
        │ Utilización    │
        │   >= 90%?      │
        └────┬───────┬───┘
             │       │
         NO  │       │ SI
             │       │
             ▼       ▼
    ┌─────────┐  ┌──────────────────────────┐
    │Continue │  │ ¿En cooldown period?     │
    │Monitor  │  │ (10 min desde última     │
    └─────────┘  │  creación)               │
                 └────┬───────────┬──────────┘
                      │           │
                  SI  │           │ NO
                      │           │
                      ▼           ▼
              ┌─────────┐  ┌──────────────────┐
              │  WAIT   │  │ Crear Nuevo      │
              │Continue │  │ Droplet          │
              │Monitor  │  └────────┬─────────┘
              └─────────┘           │
                                    ▼
                    ┌───────────────────────────────┐
                    │ POST /v2/droplets             │
                    │ {                             │
                    │   name: "whatsapp-sess-xxx",  │
                    │   region: "nyc3",             │
                    │   size: "s-2vcpu-4gb",        │
                    │   image: "docker-20-04",      │
                    │   user_data: cloud_init_script│
                    │ }                             │
                    └───────────────┬───────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │ Cloud-Init ejecuta:           │
                    │ 1. Instala Docker             │
                    │ 2. Login a DO Registry        │
                    │ 3. Pull imagen                │
                    │ 4. Run container con env vars │
                    │ 5. Configura firewall (UFW)   │
                    └───────────────┬───────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │ Esperar droplet activo        │
                    │ (Polling cada 10s, max 5min)  │
                    └───────────────┬───────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │ Registrar en pool             │
                    │ instances.set(ip, {...})      │
                    └───────────────┬───────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │ Notificar éxito               │
                    │ - Log                         │
                    │ - Webhook (opcional)          │
                    │ - Slack (opcional)            │
                    └───────────────┬───────────────┘
                                    │
                                    ▼
                            ┌───────────────┐
                            │  Cooldown     │
                            │  10 minutos   │
                            └───────┬───────┘
                                    │
                                    ▼
                            [Volver a ciclo]
```

---

## 🏢 Arquitectura de una Instancia Individual

```
┌─────────────────────────────────────────────────────────────┐
│                    DROPLET (Ubuntu + Docker)                 │
│  IP Pública: xxx.xxx.xxx.xxx                                │
│  Hostname: whatsapp-sessions-1234567890                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │         Docker Container: whatsapp-sessions        │    │
│  │         Image: registry.do.com/xxx:latest          │    │
│  ├────────────────────────────────────────────────────┤    │
│  │                                                     │    │
│  │  ┌──────────────────────────────────────────┐    │    │
│  │  │      Node.js 20 (Alpine Linux)           │    │    │
│  │  │  Port: 3001                              │    │    │
│  │  ├──────────────────────────────────────────┤    │    │
│  │  │                                           │    │    │
│  │  │  📦 Express Server                       │    │    │
│  │  │  ├─ Health Checks (/health, /ready)     │    │    │
│  │  │  ├─ Capacity Metrics (/metrics/capacity)│    │    │
│  │  │  └─ Session API                          │    │    │
│  │  │                                           │    │    │
│  │  │  🔌 Session Manager                      │    │    │
│  │  │  ├─ Bootstrap (reconecta sesiones)      │    │    │
│  │  │  ├─ Socket Factory (crea sockets)       │    │    │
│  │  │  └─ Health Monitor (cada 5min)          │    │    │
│  │  │                                           │    │    │
│  │  │  📱 WhatsApp Sockets (Baileys)          │    │    │
│  │  │  ┌─────────────────────────────────┐   │    │    │
│  │  │  │ Session 1  │ Session 2  │ ...   │   │    │    │
│  │  │  │ Negocio A  │ Negocio B  │ 50max │   │    │    │
│  │  │  │ Connected  │ Connected  │       │   │    │    │
│  │  │  └─────────────────────────────────┘   │    │    │
│  │  │                                           │    │    │
│  │  │  🔐 MongoDB Client                       │    │    │
│  │  │  ├─ Auth State (creds + keys)           │    │    │
│  │  │  ├─ Business Collection                 │    │    │
│  │  │  └─ Session Store (Passport)            │    │    │
│  │  │                                           │    │    │
│  │  │  🔔 Webhook Client (Axios)              │    │    │
│  │  │  └─ POST eventos a chatbot              │    │    │
│  │  │                                           │    │    │
│  │  └───────────────────────────────────────────┘    │    │
│  │                                                     │    │
│  │  Environment Variables:                            │    │
│  │  - MONGODB_URI                                     │    │
│  │  - MAX_SESSIONS=50                                 │    │
│  │  - SESSION_SECRET                                  │    │
│  │  - SHARED_SECRET                                   │    │
│  │  - CHATBOT_WEBHOOK_URL                            │    │
│  │                                                     │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  🔥 UFW Firewall                                            │
│  ├─ Port 22 (SSH) ✅                                       │
│  ├─ Port 3001 (API) ✅                                     │
│  └─ Default: Deny incoming ❌                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
         │                                    │
         │ TCP 3001                           │ MongoDB Protocol
         ▼                                    ▼
   Load Balancer                     MongoDB Cluster
```

---

## 🔐 Flujo de Autenticación y Sesiones

```
┌──────────────┐
│   Usuario    │
│  (Frontend)  │
└──────┬───────┘
       │
       │ 1. Login con credenciales
       │ POST /auth/login
       ▼
┌─────────────────────────────────────────┐
│  Proyecto Principal (Otro Servicio)     │
│  - Valida credenciales                  │
│  - Crea cookie de sesión                │
│  - Guarda en MongoDB                    │
└──────┬──────────────────────────────────┘
       │
       │ 2. Cookie establecida
       │ Set-Cookie: connect.sid=xxx
       ▼
┌──────────────┐
│   Usuario    │
│  con Cookie  │
└──────┬───────┘
       │
       │ 3. Request a WhatsApp Service
       │ POST /session/123/start
       │ Cookie: connect.sid=xxx
       ▼
┌─────────────────────────────────────────┐
│  WhatsApp Sessions Service (Droplet)    │
│                                          │
│  ┌────────────────────────────────┐    │
│  │  Express Session Middleware    │    │
│  │  - Lee cookie                  │    │
│  │  - Busca en MongoDB            │    │
│  │  - Deserializa usuario         │    │
│  └────────┬───────────────────────┘    │
│           │                             │
│           ▼                             │
│  ┌────────────────────────────────┐    │
│  │  Passport Middleware           │    │
│  │  - Valida sessionVersion       │    │
│  │  - Verifica usuario activo     │    │
│  └────────┬───────────────────────┘    │
│           │                             │
│           ▼                             │
│  ┌────────────────────────────────┐    │
│  │  Mixed Auth Middleware         │    │
│  │                                 │    │
│  │  Opción A: Cookie ✅           │    │
│  │  req.user = {id, username}     │    │
│  │                                 │    │
│  │  Opción B: X-Shared-Secret     │    │
│  │  (para otros servicios)        │    │
│  └────────┬───────────────────────┘    │
│           │                             │
│           ▼                             │
│  ┌────────────────────────────────┐    │
│  │  Session Controller            │    │
│  │  - Procesa request             │    │
│  │  - Inicia sesión WhatsApp      │    │
│  └────────────────────────────────┘    │
│                                          │
└──────────────────────────────────────────┘
```

---

## 📊 Flujo de Datos: Mensaje WhatsApp

```
WhatsApp          Droplet #2           MongoDB          Webhook
Servidor       (Docker Container)      Cluster          Destino
   │                  │                   │                │
   │                  │                   │                │
   │ 1. Mensaje       │                   │                │
   │  entrante        │                   │                │
   ├─────────────────>│                   │                │
   │                  │                   │                │
   │                  │ 2. Desencriptar   │                │
   │                  │    (E2E con keys) │                │
   │                  ├──────────────────>│                │
   │                  │    GET keys       │                │
   │                  │<──────────────────┤                │
   │                  │    Keys           │                │
   │                  │                   │                │
   │                  │ 3. Procesar msg   │                │
   │                  │    (logger, stats)│                │
   │                  │                   │                │
   │                  │ 4. Emitir evento  │                │
   │                  │    webhook        │                │
   │                  ├───────────────────────────────────>│
   │                  │    POST /whatsapp/events           │
   │                  │    {                               │
   │                  │      type: "message",              │
   │                  │      data: {                       │
   │                  │        cobroId: "123",             │
   │                  │        from: "+1234567890",        │
   │                  │        text: "Hola",               │
   │                  │        timestamp: ...              │
   │                  │      }                             │
   │                  │    }                               │
   │                  │    Header: X-Signature (HMAC)     │
   │                  │                   │                │
   │                  │                   │ 5. Chatbot     │
   │                  │                   │    procesa     │
   │                  │                   │    mensaje     │
   │                  │<───────────────────────────────────┤
   │                  │    HTTP 200 OK    │                │
   │                  │                   │                │
   │                  │ 6. Log exitoso    │                │
   │                  │                   │                │
```

---

## 🌐 Topología de Red

```
                    INTERNET
                       │
                       │ Public IP
                       ▼
        ┌──────────────────────────────┐
        │  DigitalOcean Load Balancer  │
        │  IP: xxx.xxx.xxx.xxx         │
        │  Port: 443 (HTTPS)           │
        └──────────────┬───────────────┘
                       │
        ┌──────────────┴───────────────┐
        │      VPC (Virtual Private    │
        │      Cloud - Opcional)       │
        │                              │
        │  ┌────────┬────────┬───────┐│
        │  │        │        │       ││
        ▼  ▼        ▼        ▼       ▼▼
     ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐
     │ D1 │ │ D2 │ │ D3 │ │ D4 │ │ DN │  Droplets
     │:01 │ │:01 │ │:01 │ │:01 │ │:01 │  (WhatsApp)
     └─┬──┘ └─┬──┘ └─┬──┘ └─┬──┘ └─┬──┘
       │      │      │      │      │
       │      │      │      │      │
       └──────┴──────┴──────┴──────┘
                     │
                     │ Private Network
                     │ (MongoDB Protocol)
                     │
                     ▼
        ┌────────────────────────────┐
        │   MongoDB Managed Cluster  │
        │   - Primary Node           │
        │   - Secondary Nodes (x2)   │
        │   - Automatic Backups      │
        │                            │
        │   Private IP only          │
        │   Trusted Sources:         │
        │   - Droplet IPs allowed    │
        │   - Orchestrator IP        │
        └────────────────────────────┘

Orchestrator
     │
     │ Public Internet
     │ DigitalOcean API
     │
     ▼
┌────────────────────┐
│  DO Control Plane  │
│  - Droplet API     │
│  - Registry API    │
│  - Monitoring API  │
└────────────────────┘
```

---

## 📈 Evolución de Capacidad (Timeline)

```
Tiempo →

T=0 (Inicio)
┌────────┐
│ Drop 1 │  5 sesiones
│  5/50  │  10% utilización
└────────┘

T=1 hora
┌────────┐
│ Drop 1 │  25 sesiones
│ 25/50  │  50% utilización
└────────┘

T=2 horas
┌────────┐
│ Drop 1 │  45 sesiones
│ 45/50  │  90% utilización ⚠️
└────────┘
  │
  │ Orchestrator detecta >= 90%
  │ Crea Droplet 2
  ▼

T=2h + 5min
┌────────┐  ┌────────┐
│ Drop 1 │  │ Drop 2 │
│ 48/50  │  │  2/50  │  Total: 50 sesiones
│  96%   │  │   4%   │  Capacidad: 100 slots
└────────┘  └────────┘

T=4 horas
┌────────┐  ┌────────┐
│ Drop 1 │  │ Drop 2 │
│ 50/50  │  │ 45/50  │  Total: 95 sesiones
│ 100% 🔴│  │  90% ⚠️│  Capacidad: 100 slots
└────────┘  └────────┘
                │
                │ Orchestrator detecta Drop 2 >= 90%
                │ Crea Droplet 3
                ▼

T=4h + 5min
┌────────┐  ┌────────┐  ┌────────┐
│ Drop 1 │  │ Drop 2 │  │ Drop 3 │
│ 50/50  │  │ 47/50  │  │  1/50  │  Total: 98 sesiones
│ 100% 🔴│  │  94% ⚠️│  │   2% 🟢│  Capacidad: 150 slots
└────────┘  └────────┘  └────────┘

... y así sucesivamente hasta N instancias
```

---

## 💾 Arquitectura de Persistencia (MongoDB)

```
┌────────────────────────────────────────────────────────┐
│         MongoDB Managed Cluster (DigitalOcean)         │
├────────────────────────────────────────────────────────┤
│                                                         │
│  Database: whatsapp_sessions                           │
│  ├─ Collection: baileys_creds                          │
│  │  └─ Documents:                                      │
│  │     ├─ _id: "baileys-123:creds"                     │
│  │     │  data: "{noiseKey, signedIdentityKey...}"    │
│  │     ├─ _id: "baileys-456:creds"                     │
│  │     └─ ...                                          │
│  │                                                      │
│  ├─ Collection: baileys_keys                           │
│  │  └─ Documents:                                      │
│  │     ├─ _id: "baileys-123:session:+1234567890"      │
│  │     │  data: "{sessionKey, baseKey...}"            │
│  │     ├─ _id: "baileys-123:pre-key:12345"            │
│  │     │  data: "{keyPair...}"                        │
│  │     └─ ...                                          │
│  │                                                      │
│  └─ Collection: businesses                             │
│     └─ Documents:                                      │
│        ├─ _id: ObjectId("...")                         │
│        │  cobroId: "123"                               │
│        │  name: "Negocio A"                            │
│        │  whatsappNumber: "+1234567890"               │
│        └─ ...                                          │
│                                                         │
│  Database: main_project (Autenticación)                │
│  ├─ Collection: users                                  │
│  │  └─ Usuarios del sistema                           │
│  │                                                      │
│  └─ Collection: sessions (express-session)             │
│     └─ Sesiones HTTP activas                          │
│                                                         │
│  Configuración:                                         │
│  ✅ Replica Set (3 nodos)                              │
│  ✅ Automatic Backups (daily)                          │
│  ✅ Point-in-time Recovery                             │
│  ✅ TLS/SSL habilitado                                 │
│  ✅ Authentication requerida                           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🚀 Pipeline de Deployment

```
┌─────────────────┐
│  Desarrollador  │
│  Local          │
└────────┬────────┘
         │
         │ 1. git push
         ▼
┌─────────────────────────────┐
│  GitHub Repository          │
│  (Source Code)              │
└────────┬────────────────────┘
         │
         │ 2. GitHub Actions (opcional)
         │    - Run tests
         │    - Lint code
         ▼
┌─────────────────────────────┐
│  Build Docker Image         │
│  - Multi-stage Dockerfile   │
│  - Optimize layers          │
└────────┬────────────────────┘
         │
         │ 3. docker build
         ▼
┌─────────────────────────────┐
│  Local Docker Image         │
│  whatsapp-sessions:latest   │
└────────┬────────────────────┘
         │
         │ 4. docker tag + push
         ▼
┌─────────────────────────────┐
│  DigitalOcean Registry      │
│  registry.do.com/xxx/wa:v1  │
└────────┬────────────────────┘
         │
         │ 5. Orchestrator detecta necesidad
         │    de nueva instancia
         ▼
┌─────────────────────────────┐
│  Orchestrator               │
│  - POST /v2/droplets        │
│  - Cloud-init script        │
└────────┬────────────────────┘
         │
         │ 6. DigitalOcean API
         ▼
┌─────────────────────────────┐
│  Nuevo Droplet Creado       │
│  - Ubuntu + Docker          │
│  - Cloud-init ejecutando    │
└────────┬────────────────────┘
         │
         │ 7. Cloud-init:
         │    - docker pull
         │    - docker run
         ▼
┌─────────────────────────────┐
│  Container Running          │
│  - Health checks OK         │
│  - Ready para tráfico       │
└─────────────────────────────┘
```

---

## 🔍 Monitoreo y Observabilidad

```
┌─────────────────────────────────────────────────────────┐
│                     Capa de Monitoreo                    │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  Orchestrator (cada 5min)                               │
│  ├─ GET /metrics/capacity → Todas las instancias       │
│  └─ Decisión de scaling                                 │
│                                                          │
│  DigitalOcean Monitoring (incluido)                     │
│  ├─ CPU Usage                                           │
│  ├─ Memory Usage                                        │
│  ├─ Disk I/O                                            │
│  ├─ Network In/Out                                      │
│  └─ Alertas vía email                                   │
│                                                          │
│  Application Logs (Pino)                                │
│  ├─ Structured JSON logs                                │
│  ├─ Log levels: debug, info, warn, error               │
│  └─ Rotation automática                                 │
│                                                          │
│  Health Checks                                           │
│  ├─ /health (completo)                                  │
│  ├─ /healthz (liveness)                                 │
│  └─ /ready (readiness)                                  │
│                                                          │
│  Session Health Monitor (cada 5min)                     │
│  ├─ Detecta sesiones logged out                        │
│  ├─ Detecta inactividad                                │
│  └─ Emite alertas via webhook                          │
│                                                          │
│  Webhooks Outbound                                      │
│  ├─ Eventos: connected, disconnected, message          │
│  ├─ Retry automático (3 intentos)                      │
│  └─ Firma HMAC para seguridad                          │
│                                                          │
│  [Opcional - Futura implementación]                     │
│  ├─ Prometheus + Grafana                                │
│  ├─ Alertas a Slack/PagerDuty                          │
│  └─ Distributed Tracing                                 │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## 💡 Leyenda de Estados

```
🔴 FULL      - Instancia al 100% (50/50 sesiones)
🟡 HIGH      - Instancia >= 90% (45+ sesiones)
🟢 OK        - Instancia < 90% (< 45 sesiones)
⚠️  WARNING  - Cerca del límite, preparar scaling
✅ HEALTHY   - Sistema funcionando correctamente
❌ ERROR     - Sistema con problemas
```

Este conjunto de diagramas proporciona una visión completa de tu infraestructura actual y cómo escala automáticamente. ¿Te gustaría que profundice en algún diagrama específico o agregue más detalles sobre algún componente?
