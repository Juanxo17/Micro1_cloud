# Constitution — Microproyecto 1: Computación en la Nube (UAO)

> Documento maestro de contexto del proyecto. Toda decisión, plan o artefacto derivado de este microproyecto debe ser consistente con lo que sigue.

---

## 1. Contexto del Proyecto

**Nombre:** Cluster Consul + Balanceador de Carga usando HAProxy + Artillery en ambiente Vagrant

**Materia:** Computación en la Nube — Universidad Autónoma de Occidente (UAO)

**Objetivo:** Implementar un service mesh con HashiCorp Consul. El clúster corre agentes Consul en al menos dos nodos, cada uno alojando una aplicación web (Qubo). Se implementa balanceo de carga con HAProxy: los clientes envían peticiones al balanceador HAProxy y obtienen respuesta desde los servidores web. Se ejecutan pruebas de carga con Artillery.

**Estado actual:** No hay Vagrantfile, no hay configuración de Consul ni HAProxy, no hay scripts de provisioning. La app Qubo está clonada localmente. El repo FinalTelematicos sirve como referencia conceptual.

---

## 2. Rúbrica de Evaluación

| Módulo | Valor | Descripción |
|--------|-------|-------------|
| Pregunta 1 — Cluster Consul | 1.5 pts | Funcionamiento del clúster Consul con agentes en VMs |
| Pregunta 2 — Aprovisionamiento | 1.5 pts | Aprovisionamiento automático de VMs vía Vagrant |
| Pregunta 3 — Disponibilidad, balanceo y Artillery | 1.0 pt | HAProxy, escalabilidad, página de fallback, pruebas de carga |
| Sustentación individual | 1.0 pt | Demo en vivo + respuestas conceptuales |
| **Total** | **5.0 pts** | |

---

## 3. Requerimientos No Negociables

1. **Cliente → HAProxy → servidores web.** Las peticiones NUNCA llegan directo a los servidores — HAProxy decide qué servidor procesa cada petición.
2. **Cada servidor web corre un solo proceso NodeJS.** El balanceador corre HAProxy.
3. **GUI/stats de HAProxy accesible desde la máquina anfitriona** (host), mostrando estado y estadísticas detalladas de los servidores web.
4. **Cada VM que aloja un servidor web corre también un agente Consul.**
5. **Aprovisionamiento automático** de todas las VMs vía Vagrant (shell, Ansible u otro provisionador).
6. **Escalabilidad demostrable:** instanciar varias réplicas del servidor web en las mismas VMs y observar el cambio en las estadísticas de HAProxy.
7. **Página personalizada de HAProxy** cuando ningún servidor esté disponible (custom errorfile).
8. **Pruebas de carga con Artillery** en varios escenarios de demanda de tráfico.

---

## 4. Reglas Administrativas

- Trabajo grupal (3 personas) pero calificación **individual**.
- Sustentación individual, horario fijo, sin excepciones por retraso.
- Sin asesorías externas (trabajo independiente).
- Subir scripts a GitHub y Classroom **antes** de la sustentación.
- Llegar con todo terminado — no se trabaja en pendientes durante la franja de sustentación.
- Preguntas conceptuales y cambios "en caliente" durante la sustentación.

---

## 5. Equipo

| Persona | Rol | Notas |
|---------|-----|-------|
| **Juan** | Líder técnico, backend/AWS | Ejecuta el trabajo técnico junto con Claude |
| **Equipo (resto)** | Colaboradores | Colaboradores con roles complementarios — preparar documentación clara y de fácil acceso para que cualquier integrante pueda levantar el entorno. |
| **bigpickle (opencode)** | Planificación y guía paso a paso | No construye (salvo tareas de GitHub que Juan le indique). MCP de GitHub disponible. |
| **Claude** | Construcción real paso a paso | Scripts, configuración, explicación técnica — siempre consultando antes de crear artefactos. |

**Entorno objetivo:** Vagrant + VirtualBox, VMs Ubuntu, sin Vagrantfile aún.

---

## 6. Checklist por Módulos

### Módulo 1 — Cluster Consul (1.5 pts)

- [ ] Definir topología del clúster Consul (cuántos nodos server/agent, o modo dev para simplicidad justificable en sustentación)
- [ ] Instalar y configurar Consul en cada VM que aloja un servidor web (mínimo 2 nodos con agente Consul)
- [ ] Verificar que los agentes se ven entre sí (`consul members`) y que el clúster está sano (`consul info` / UI de Consul)
- [ ] Registrar el servicio web (Qubo) en Consul con health check apuntando al endpoint mejorado de health
- [ ] Documentar comandos y decisiones para la sustentación

### Módulo 2 — Aprovisionamiento (1.5 pts)

- [ ] Elegir aprovisionador (shell script vs. Ansible) y justificar la elección
- [ ] Definir Vagrantfile con las VMs necesarias (servidores web + HAProxy), IP estática en red privada, synced folders
- [ ] Script(s) de aprovisionamiento que instalen automáticamente: Node.js, MongoDB (o cliente hacia una instancia central), Consul, dependencias de Qubo, HAProxy (en la VM correspondiente)
- [ ] Provisionamiento debe ser reproducible con `vagrant up` desde cero, sin pasos manuales
- [ ] Documentar cómo cualquier integrante del equipo levanta el entorno completo con la menor cantidad de pasos posible

### Módulo 3 — Disponibilidad, Balanceo y Pruebas de Carga (1.0 pt)

- [ ] Instalar y configurar HAProxy en su VM dedicada, apuntando a los servidores web (vía Consul service discovery o config estática inicial + luego integración con Consul)
- [ ] Habilitar la GUI/stats de HAProxy accesible desde la máquina anfitriona (mapeo de puertos en Vagrant)
- [ ] Configurar página de error personalizada para "ningún servidor disponible" (custom errorfile en HAProxy)
- [ ] Probar escalabilidad: levantar réplicas adicionales del servidor web en las mismas VMs y verificar cambios en las stats de HAProxy
- [ ] Instalar Artillery y definir varios escenarios de prueba de carga (baseline sin balanceo, con balanceo, distintos niveles de tráfico)
- [ ] Ejecutar las pruebas, capturar resultados (RT, throughput, error rate) y documentarlos con el mismo rigor que `PLAN_PRUEBAS_LOAD_BALANCING.md`
- [ ] *(Nice to have)* Explorar un mecanismo de balanceo/routing propio como valor agregado sobre HAProxy+Consul

### Módulo 4 — Sustentación Individual (1.0 pt)

- [ ] Preparar demo en vivo: cluster Consul funcionando, HAProxy balanceando, página de fallback, resultados de Artillery
- [ ] Preparar respuestas a posibles preguntas conceptuales (service mesh, health checking de Consul, routing de HAProxy, resultados de pruebas de carga)
- [ ] Confirmar horario de sustentación (excel de horarios) y subir repo a GitHub/Classroom antes de esa hora

---

## 7. Datos Técnicos de Qubo (App a Desplegar)

### 7.1 Stack

| Componente | Valor |
|------------|-------|
| Runtime | Node.js (sin restricción de versión explícita) |
| Framework | **Express 5** (`^5.1.0`) — ESM (`"type": "module"`) |
| Gestor de paquetes | npm (lock files gitignorados — hay que generarlos) |
| Frontend | React + Vite (subcarpeta `anuel/`) |
| Process manager (producción previa) | Vercel serverless |

### 7.2 Dependencias de Base de Datos

**MongoDB (principal, obligatoria):**
- ODM: Mongoose (`^8.14.1`)
- Conexión: `mongoose.connect(process.env.DATABASE_URL, {...})`
- Nombre de BD: `'Qubo'` (hardcoded en `Backend/config/db.js:12`)
- Sin la variable `DATABASE_URL`, el proceso hace `process.exit(1)`
- 8 modelos Mongoose activos: Account, Profile, Publicacion, Comentario, MeGusta, Seguidor, Notificacion, ConfiguracionUsuario

**Firebase Realtime Database (opcional, solo notificaciones):**
- URL hardcoded: `https://qubo-db982-default-rtdb.firebaseio.com` en `Services/NotificationService.js:26`
- Se puede deshabilitar para el demo sin afectar funcionalidad core

**Firebase Admin SDK (autenticación):**
- Requiere archivo de service account en `Backend/firebase/firebase-service-account.json` (gitignorado)
- Usado para `adminAuth.verifyIdToken()` en login con Google

**Prisma + PostgreSQL (abandonado):**
- Existe schema completo en `Backend/src/generated/prisma/schema.prisma` pero NO se usa en el código corriente — es basura, ignorar.

### 7.3 Arranque

- **Comando:** `npm run dev` → ejecuta `nodemon src/index.js`
- **Puerto por defecto:** `8080` (configurable vía `PORT`)
- **Entrada:** `Backend/src/index.js`

**Secuencia de arranque:**
1. `dotenv.config()` — carga `.env`
2. Crear app Express
3. `connectDB()` — conecta a MongoDB (async, no awaited al nivel superior)
4. Configurar CORS, cookie parser, body parsers
5. Montar rutas en `/api`
6. Health check en `GET /`
7. `app.listen(PORT)` — **SOLO si `NODE_ENV !== 'production'`** (crítico: en production asume serverless y NO abre puerto)

### 7.4 Variables de Entorno

| Variable | Obligatoria | Default | Nota |
|----------|-------------|---------|------|
| `DATABASE_URL` | **SÍ** | — | URI de MongoDB. Sin esto, `process.exit(1)` |
| `JWT_SECRET` | **SÍ** | — | Secret para firmar tokens JWT |
| `JWT_EXPIRES_IN` | **SÍ** | — | TTL de tokens (ej: `1h`, `7d`) |
| `PORT` | No | `8080` | Puerto de escucha |
| `FRONTEND_URL` | No | — | Para configuración CORS |
| `NODE_ENV` | No | — | Si es `production`, NO abre puerto |
| `CLOUDINARY_NAME` | No | `'demo'` | Tiene defaults dummy pero no funciona |
| `CLOUDINARY_API` | No | `'123456789012345'` | |
| `CLOUDINARY_SECRET` | No | `'abcdefghijklmnopqrstuvwxyz12'` | |

**No existe `.env.example`** — hay que crearlo.

### 7.5 Health Check

- **Endpoint existente:** `GET /` → `{ "message": "Backend funcionando correctamente!" }`
- **Ubicación:** `Backend/src/index.js:34-36`
- **Autenticación:** No requerida
- **Verificación de DB:** No realiza — solo retorna un JSON estático
- **Identificador de instancia:** No retorna hostname, PID ni node ID

**Mejora necesaria para Consul/HAProxy:** Crear un health check que retorne algo como `{ node: "web-N", status: "ok", db: "connected" }` y que internamente verifique la conexión a MongoDB.

### 7.6 Endpoints Principales (37 rutas totales)

| Grupo | Rutas | Métodos |
|-------|-------|---------|
| Auth | `/api/login`, `/register`, `/logout`, `/firebase-login` | POST |
| Profile | `/api/profile`, `/check-profile`, `/profile/me`, `/profile/username/:username`, `/profile/update`, `/profile/:id` | GET/POST/PUT/DELETE |
| Imágenes | `/api/profile/image`, `/api/posts/image` | POST (Cloudinary) |
| Posts | `/api/posts`, `/api/posts/my`, `/api/posts/:id`, `/api/profile/:profileId/posts` | GET/POST/PUT/DELETE |
| Comentarios | `/api/posts/:id/comments`, `/api/comments/:id` | GET/POST/PUT/DELETE |
| Likes | `/api/posts/:postId/like`, `/api/posts/:postId/likes` | GET/POST |
| Seguidores | `/api/profile/:profileId/follow`, `/followers`, `/following` | GET/POST |
| Notificaciones | `/api/notifications`, `/notifications/unread`, `/:notificationId/read`, `/read-all` | GET/PUT/DELETE |
| Health | `GET /` | GET |

### 7.7 Estado en Memoria / Multi-Instancia

**La app es esencialmente stateless — segura para balanceo:**
- JWT-based auth (sin sesiones express activas — `express-session` está instalado pero nunca se importa ni configura)
- Sin WebSockets
- Sin caches en memoria
- `multer.memoryStorage()` para uploads: buffers temporales por-request (5MB max), no compartidos entre requests
- `NotificationService.js` tiene singleton de Firebase Admin a nivel módulo, pero es cliente HTTP stateless

**Conclusión:** Las únicas dependencias de estado externo son MongoDB (compartido) y Firebase RTDB (compartido). Los JWTs se validan con un secret que debe ser idéntico en todas las réplicas.

### 7.8 Peso y Arranque

- ~14 dependencias de producción, 1 dev (nodemon) — muy ligeras
- Sin migraciones, seeds, ni scripts de setup (MongoDB crea collections automáticamente vía Mongoose)
- Lock files gitignorados — hay que generarlos y commitearlos para builds reproducibles
- Arranque limpio: solo `connectDB()` + `app.listen()`, sin jobs pesados

### 7.9 Ajustes Necesarios Antes del Despliegue

1. **`NODE_ENV=production` cierra el server:** Hay que evitar ese valor en las VMs o eliminar la condición en `src/index.js:45-48`
2. **No hay `.env.example`:** Crearlo con las variables obligatorias documentadas
3. **Lock files gitignorados:** Generar `package-lock.json` y commitearlo
4. **Health check mejorable:** Agregar verificación de DB e identificador de instancia
5. **Docker:** No existe configuración Docker — todo se instala manualmente en VMs

---

## 8. Estado del Arte: FinalTelematicos (Referencia Conceptual)

### 8.1 Qué es

Repo anterior de Juan, proyecto de la materia Servicios Telemáticos (UAO, noviembre 2025). Implementó balanceo de carga con NGINX (3 algoritmos: round robin, least connections, ip hash) sobre la misma app Qubo (o versión equivalente), con pruebas de carga documentadas en Artillery.

### 8.2 Qué se reutiliza (conceptual, no código)

**Metodología de pruebas de carga (lo más valioso):**
- Escenarios Artillery con pesos por tipo de tráfico: Homepage (40%), Login (30%), Feed autenticado (10-30%), Health check API (20%)
- Fases de prueba: Warmup → Test principal → Cooldown
- Comparación baseline (un solo backend) vs. balanceado
- Hipótesis planteadas de antemano (H1-H4)
- Métricas objetivo: RT <300ms (bueno), >1000ms (crítico); Throughput >150 req/s (bueno), >50 (crítico); Error rate <0.5% (bueno), >5% (crítico)
- Plantilla de análisis comparativo

**Arquitectura Vagrant multi-VM:**
- Roles separados: LB dedicado, webservers con app
- IP estática en red privada (192.168.50.x)
- Provisioning por rol con scripts bash
- Synced folders para desarrollo rápido
- Proceso manager: PM2 para Node.js

**Escenarios de Artillery ya validados con Qubo:**
- `baseline-vm1.yml` / `baseline-vm2.yml`: 5→10→5 req/s, 4 min
- `round-robin-2servers.yml` / `least-connections-2servers.yml` / `ip-hash-2servers.yml`: 5→15→5 req/s, 5 min
- Todos contra `http://192.168.50.30` (IP del LB)

### 8.3 Qué NO se reutiliza

- **NGINX como balanceador:** se reemplaza por HAProxy (requisito de la guía)
- **Los 3 algoritmos de NGINX:** no aplican aquí — HAProxy tiene sus propios algoritmos
- **Código de configuración de NGINX:** no copiar, solo como referencia de estructura

### 8.4 Nice-to-have: Algoritmo Propio

Juan lo propuso como valor agregado ("todo suma"), no como reemplazo del requisito. Se mantiene como posible extra si el tiempo alcanza (ej: balance por health/carga real de réplicas, routing custom sobre Consul). Sin que sea condición para cumplir la rúbrica.

---

## 9. Artefactos de Sustentación

Para cada módulo del checklist se genera un archivo markdown tipo `Sustentacion_recapitulacion_moduloX.md` con:
- **Contexto breve** del módulo (qué cubre, por qué importa)
- **Paso a paso explicado** (cada acción realizada, con justificación)
- **Comandos exactos** (copy-paste para replicar)
- **Decisiones tomadas** y alternativas descartadas

**Reglas de creación:**
- Se crean **uno a la vez**, consultando antes con Juan cuál módulo toca
- Se confirma el contenido antes de generar el siguiente
- Nunca todos de golpe ni sin luz verde de Juan
- Pensado para que Juan pueda sustentar con solidez y también poner al día al resto del equipo

---

## 10. Reglas de Trabajo con Agentes de IA

1. Ni Claude ni bigpickle crean documentos/artefactos sin consultar antes con Juan, salvo que él dé luz verde explícita para avanzar.
2. Los mds de sustentación se crean uno a la vez y consultando antes de cada uno.
3. **bigpickle:** rol de planeación y guía, no de construcción (salvo tareas de GitHub que Juan le indique explícitamente). MCP de GitHub disponible.
4. **Claude:** construcción real paso a paso — scripts, configuración, explicación técnica — siempre consultando antes de crear artefactos.
5. Todo debe estar orientado a **Vagrant + VirtualBox + VMs Ubuntu**, sin asumir Docker salvo que se decida explícitamente.
6. Mantener un tono profesional y respetuoso sobre todos los integrantes del equipo en cualquier documento, mensaje o entregable del proyecto.
