# VERIFICACIÓN END-TO-END — Microproyecto 1 (Cluster Consul + HAProxy + Artillery)

> **Objetivo:** dejar de ver la infraestructura como "caja negra". Este documento permite **entrar a cada VM**, verificar qué debe tener, **entender el contenido y la función de cada archivo** que creó Ansible, y comprobar — punto por punto — que se cumple **cada** requerimiento de la práctica.
>
> Cómo se usa: primero **Parte A** (checlist completa de la práctica), después **Parte C** (el inventario de archivos por VM para entender qué mira uno) y al final **Parte D** (la verificación paso a paso; cada paso dice explícitamente **qué parte de la checklist cumplió**).
>
> Todo se hace desde el directorio `Micro1_cloud/vagrant/`. Para entrar a una VM: `vagrant ssh <vm>`.

---

# PARTE A — CHECKLIST COMPLETA DE LA PRÁCTICA

## A.1 Requerimientos Generales

| # | Requerimiento general | Estado |
|---|----------------------|--------|
| RG1 | Las peticiones NO van directo a los servidores web; el **balanceador decide** qué servidor procesa cada petición | ☐ |
| RG2 | Los servidores web corren la app Node.js y en el balanceador corre **HAProxy** | ☐ |
| RG3 | **GUI del balanceador** accesible desde la máquina anfitriona (estado/estadísticas detalladas) | ☐ |
| RG4 | Cada VM que corre un servidor web también corre un **agente Consul** | ☐ |

## A.2 Pregunta 1 — Cluster Consul (1.5 pts)

| # | Ítem | Estado |
|---|------|--------|
| P1.1 | Clúster Consul implementado con agentes corriendo en las VMs | ☐ |
| P1.2 | Al menos dos nodos con agente Consul (`consul members` muestra los 3 sanos) | ☐ |
| P1.3 | Service discovery en funcionamiento (`consul catalog services` lista `qubo`) | ☐ |
| P1.4 | Health check del servicio registrado (`consul health checks` OK) | ☐ |

## A.3 Pregunta 2 — Aprovisionamiento (1.5 pts)

| # | Ítem | Estado |
|---|------|--------|
| P2.1 | Aprovisionamiento **automático** con un aprovisionador de preferencia en Vagrant (Ansible) | ☐ |
| P2.2 | `vagrant up` reproduce TODO desde cero, sin pasos manuales (salvo la ruta del backend en el Vagrantfile) | ☐ |
| P2.3 | Cada VM queda con: Node.js, Qubo (app), Consul, y en lb además HAProxy | ☐ |

## A.4 Pregunta 3 — Disponibilidad, balanceo y pruebas de carga (1.0 pt)

| # | Ítem | Estado |
|---|------|--------|
| P3.1 | **Escalabilidad:** varias réplicas del servidor web en las mismas VMs y **cambio visible en las stats de HAProxy** | ☐ |
| P3.2 | **Caso sin ningún servidor:** HAProxy despliega la **página personalizada** disculpándose | ☐ |
| P3.3 | **Artillery:** pruebas de carga con **diferentes escenarios** que caracterizan la respuesta del sistema a distintas demandas de tráfico | ☐ |

## A.5 Sustentación individual (1.0 pt)

| # | Ítem | Estado |
|---|------|--------|
| S1 | Demo en vivo (clúster, balanceo, fallback 503, Artillery) | ☐ |
| S2 | Respuestas a preguntas conceptuales / cambios en caliente | ☐ |
| S3 | Scripts subidos a Classroom y GitHub antes de la sustentación | ☐ |

---

# PARTE B — MAPA DE LA INFRAESTRUCTURA (3 VMs)

```
              HOST (máquina anfitriona)
   ┌────────────┼───────────────────────────────┐
   │            │                               │
   │  Consul UI │  HAProxy GUI                 │  Frontend Qubo
   │  :18500    │  :7000                        │  (localhost:5173)
   │  :28500    │                               │
   └────────────┼───────────────────────────────┘
                │  HTTP
                ▼
        ┌────────────────┐       192.168.56.30
        │   lb (VM 3)    │  HAProxy :80  +  Consul CLIENT  +  stats :7000
        └───────┬────────┘
    ┌───────────┴───────────┐
    ▼                       ▼
┌───────────┐           ┌───────────┐
│ web1 (VM1)│           │ web2 (VM2)│  192.168.56.10 / .20
│ Consul SERVER          Consul SERVER
│   8080 8081 8082       │   8080 8081 8082   (3 réplicas PM2 c/u)
└───────────┘           └───────────┘
    └──────────▶  MongoDB Atlas (BD en la nube, compartida)  ◀──────────┘
```

**Lectura de la imagen:**
- El cliente (host / frontend) **solo** habla con `lb`. Nunca llega directo a web1/web2. → cumplimiento de **RG1**.
- web1 y web2 corren: app Qubo (Node.js) + agente Consul **server**. → cumplimiento de **RG2** y **RG4**.
- `lb` corre: HAProxy (una sola app en el balanceador) + agente Consul **client**. → cumplimiento de **RG2**.
- La GUI de HAProxy (`:7000`) está reenviada al host → **RG3**.

---

# PARTE C — INVENTARIO DE ARCHIVOS POR VM (para entender qué se está mirando)

## C.1 En cada server web: web1 (192.168.56.10) y web2 (192.168.56.20)

### Archivo 1 — Consul server config: `/etc/consul.d/server.hcl`

Este archivo convierte a la VM en un agente Consul **server** (participa del quórum y almacena el catálogo). Campo por campo:

| Campo | Valor en web1/web2 | Qué significa |
|-------|--------------------|---------------|
| `datacenter` | `"dc1"` | Nombre del datacenter lógico del clúster |
| `node_name` | `"web1"` / `"web2"` | Identidad del nodo en el clúster (la da `inventory_hostname`) |
| `server` | `true` | Modo server (participa del quórum) |
| `bootstrap_expect` | `2` | Espera a que haya 2 servers para iniciar el quórum y elegir líder |
| `bind_addr` | `192.168.56.10` / `192.168.56.20` | IP de la red privada en la que habla con el clúster (NO la NAT 10.0.2.x) |
| `client_addr` | `0.0.0.0` | Acepta consultas de la API/UI desde cualquier interfaz |
| `ui` | `true` | Habilita la interfaz web de Consul |
| `retry_join` | apunta al otro server (`web2` o `web1`) | Se intenta unir al otro server automáticamente |

### Archivo 2 — Registro del servicio: `/etc/consul.d/qubo.json`

Registra Qubo en el catálogo de Consul y le asocia un health check:

| Campo | Valor | Qué significa |
|-------|-------|---------------|
| `service.name` | `"qubo"` | Nombre del servicio en el catálogo |
| `service.tags` | `["web","nodejs"]` | Etiquetas de clasificación |
| `service.port` | `8080` | Puerto principal del servicio |
| `check.http` | `http://192.168.56.1X:8080/health` | Consul hace GET a `/health` para medicar salud |
| `check.interval` | `5s` / `timeout 3s` | Cada 5 s con timeout de 3 s |

### Archivo 3 — Variables de la app: `/srv/qubo/src/.env`

Ansible lo genera por cada VM (con `cd /srv/qubo/src` en el lanzamiento de PM2, porque la librería `dotenv` lee el `.env` del **directorio de trabajo**). Contiene la conexión a MongoDB Atlas, el secreto JWT (idéntico en todas las réplicas para validar los mismos tokens) y datos de nodo:

```bash
DATABASE_URL=mongodb+srv://...  # conexión a Atlas (mismo en ambas VMs)
DB_NAME=qubo_cloud
JWT_SECRET=...                  # MÚSMO en las 6 réplicas (los tokens deben validarse igual)
NODE_ID=web1                    # o web2 — lo usa /health para identificarse
INSTANCE_ID=web1-8080           # se sobreescribe con cada réplica
```

### Procesos — PM2 (3 réplicas por VM)

```
sudo pm2 list
```
Resultado esperado en cada VM (puertos 8080, 8081, 8082):

| # | Proceso | Puerto | Estado |
|---|---------|--------|--------|
| 0 | qubo-web1-8080 | 8080 | online |
| 1 | qubo-web1-8081 | 8081 | online |
| 2 | qubo-web1-8082 | 8082 | online |

Cada réplica es el **mismo** `index.js` corriendo en un puerto distinto → son réplicas horizontales del mismo server web.

## C.2 En el balanceador: lb (192.168.56.30)

### Archivo 1 — Consul client: `/etc/consul.d/client.hcl`

| Campo | Valor | Qué significa |
|-------|-------|---------------|
| `server` | `false` | Agente **client**: consulta el catálogo pero NO guarda datos ni participa en el quórum |
| `bind_addr` | `192.168.56.30` | IP de la red privada para el clúster |
| `retry_join` | `["192.168.56.10","192.168.56.20"]` | Se une a los 2 servers |

### Archivo 2 — HAProxy: `/etc/haproxy/haproxy.cfg`

Es el corazón del balanceo. Secciones:

**`frontend qstats`** → la GUI de monitoreo:
```haproxy
bind *:7000
stats enable
stats uri /
stats refresh 5s
```

**`frontend qfront`** → la puerta pública:
```haproxy
bind *:80
http-response set-header X-Qubo-Served-By server-%[srv_id]
default_backend qubo_servers
```
El header `X-Qubo-Served-By` dice qué server atendió cada petición (de ahí sale `server-1..6` cuando hacemos curl).

**`backend qubo_servers`** → los 6 destinos:
```haproxy
balance roundrobin
option httpchk GET /health
http-check expect status 200
server web1-8080 192.168.56.10:8080 check
server web1-8081 192.168.56.10:8081 check
server web1-8082 192.168.56.10:8082 check
server web2-8080 192.168.56.20:8080 check
server web2-8081 192.168.56.20:8081 check
server web2-8082 192.168.56.20:8082 check
```
- `roundrobin` → reparte en orden ciclico (1,2,3,4,5,6,1,2,...).
- `check` + `httpchk GET /health` → HAProxy mismo mira el health de cada server y lo marca UP/DOWN.

### Archivo 3 — Página de error personalizada: `/etc/haproxy/errors/qubo_down.http`

Cuando **ningún** server está disponible, HAProxy responde 503 con esta página custom (en vez del error genérico del navegador). Incluye el header `X-Qubo-Served-By: NINGUNO` para que se vea que no había backends.

---

# PARTE D — VERIFICACIÓN PUNTO POR PUNTO

> Formato de cada paso: **qué verifico** → **comando(s)** → **salida esperada** → **qué parte de la checklist se cumplió aquí**.

## D1 — Las 3 VMs están `running` y con sus IPs

```bash
cd Micro1_cloud/vagrant
vagrant status
```
Salida esperada: `web1 running`, `web2 running`, `lb running`.

> **Checklist:** confirma la base de **P2.2** (reproducción) y el escenario de **P1.1** (los agentes existen en las VMs).

## D2 — El clúster Consul está sano (los 3 se ven)

Entrar a **web1**:
```bash
vagrant ssh web1
consul members
```
Salida esperada (3 nodos `alive`, 2 servers + 1 client):
```
Node  Address             Status  Type    Build   Protocol  DC   Partition  Segment
web1  192.168.56.10:8301  alive   server  1.18.1  2         dc1  default    <all>
web2  192.168.56.20:8301  alive   server  1.18.1  2         dc1  default    <all>
lb    192.168.56.30:8301  alive   client  1.18.1  2         dc1  default    <default>
```
Verificar líder/quórum:
```bash
consul operator raft list-peers
```
Salida: web1 **leader** (o web2) + el otro como *Voter*.

> **Checklist:** cumple **P1.1**, **P1.2**, **RG4** (cada VM web tiene su agente).

## D3 — Service discovery: el servicio `qubo` está en el catálogo

Desde web1:
```bash
consul catalog services
consul catalog nodes -service=qubo
```
Salida esperada: lista los servicios `consul` y `qubo`; y `qubo` está en `web1` y `web2`.

> **Checklist:** cumple **P1.3** (service discovery funcionando).

## D4 — Los health checks de Consul están `passing`

Desde web1:
```bash
curl -s localhost:8500/v1/health/checks/qubo
```
Salida esperada: dos entradas con `"Status": "passing"` (una por cada nodo).

> **Checklist:** cumple **P1.4** (health check del servicio registrado y sano). Es el mismo estatus que se ve coloreado de verde en la UI de Consul.

## D5 — Qubo corre en ambas VMs con sus 3 réplicas (PM2)

En web1 y web2:
```bash
sudo pm2 list
```
Salida esperada: 3 procesos `online` (qubo-<vm>-8080, -8081, -8082).

> **Checklist:** cumple **P2.3** (Node.js + app en las VMs) y deja la base para **P3.1** (las réplicas ya están instanciadas).

## D6 — Cada réplica responde su `/health` (con nodo e instancia)

Desde el host (o desde la VM):
```bash
curl http://192.168.56.10:8080/health
curl http://192.168.56.10:8081/health
curl http://192.168.56.10:8082/health
curl http://192.168.56.20:8080/health
curl http://192.168.56.20:8081/health
curl http://192.168.56.20:8082/health
```
Salida esperada (varía el nodo/instancia):
```json
{"node":"web1","instance":"web1-8080","status":"ok","db":"connected"}
```
`db: connected` demuestra que MongoDB Atlas responde.

> **Checklist:** cumple **P2.3** + confirma que todo el stack corre. Se usa como *before* para las pruebas de caída (si apago una réplica, esta URL deja de responder).

## D7 — Las peticiones del host pasan SÍ o SÍ por HAProxy (nunca directo)

Desde el host:
```bash
# Por el balanceador: funciona y rota
curl -s http://192.168.56.30/health
# Parar la réplica 8082 de web1 (el balanceador la quita de la rotación):
vagrant ssh web1 -c "sudo pm2 stop qubo-web1-8082"
# Ahora el header deja de mostrar server-3:
for i in $(seq 1 6); do curl -sI http://192.168.56.30/health | grep -i x-qubo-served-by; done
# Restaurar:
vagrant ssh web1 -c "sudo pm2 start qubo-web1-8082"
```
Resultado esperado mientras 8082 está detenida: rota `server-1,2,4,5,6` (nunca server-3). Con todo levantado rota `server-1..6`.

> **Checklist:** este es el ejemplo vivo de **RG1** — el tráfico no va directo a las réplicas: va al balanceador y **él decide** (y hasta cambia la decisión cuando una réplica cae).

## D8 — GUI/statísticas de HAProxy accesible desde el host

En el navegador del host:
```
http://localhost:7000
```
Salida esperada: tabla con las 6 réplicas (`web1-8080`, `web1-8081`, `web1-8082`, `web2-8080`, ...) en estado `UP` (verde) y contadores de tráfico que suben al golpearlas con curl.

> **Checklist:** cumple **RG3** (GUI desde la máquina anfitriona) y es donde se **observa el cambio de stats de P3.1** (subir/parar réplicas y ver UP/DOWN + conexiones).

## D9 — Escalabilidad: agregar réplicas y ver el cambio en las stats

1. Editar `roles/qubo-app/defaults/main.yml` y `roles/haproxy/defaults/main.yml`: `replicas: 3` → `replicas: 4`.
2. Reprovisionar:
```bash
vagrant provision web1
vagrant provision web2
vagrant provision lb
```
3. Ver el cambio en la GUI (`http://localhost:7000`): ahora hay **8 servers** (web1/web2 × 8080-8083) en UP, y el header rota `server-1..8`.

> **Checklist:** cumple **P3.1** literal: "instancie varias réplicas... y observe el cambio en las estadísticas de HAProxy". (Nota técnica: dejar `replicas` en 3 es la configuración de entrega; este paso es para **demostrar** el escalado.)

## D10 — Caso sin ningún servidor: página personalizada

```bash
# Apagar las 6 réplicas
vagrant ssh web1 -c "sudo pm2 stop all"
vagrant ssh web2 -c "sudo pm2 stop all"
# Pedir al balanceador:
curl -i http://192.168.56.30/health
```
Salida esperada:
```http
HTTP/1.1 503 Service Unavailable
X-Qubo-Served-By: NINGUNO (todos fuera de servicio)

<html>... 503 - Qubo no disponible ...</html>
```
Restaurar:
```bash
vagrant ssh web1 -c "sudo pm2 restart all"
vagrant ssh web2 -c "sudo pm2 restart all"
```

> **Checklist:** cumple **P3.2** — HAProxy despliega la página personalizada disculpándose por la no disponibilidad.

## D11 — Artillery: distintos escenarios de tráfico (Pregunta 3)

Desde el host, en `Micro1_cloud/load-tests/`:

```bash
# Escenario 1: baseline — una réplica directa (SIN balanceador)
artillery run baseline-sin-lb.yml --output baseline-sin-lb.json

# Escenario 2: balanceado — 6 réplicas vía HAProxy
artillery run balanceado-6replicas.yml --output balanceado-6replicas.json

# Comparar en Python/Node o mirando el informe:
artillery report baseline-sin-lb.json      # genera HTML de resultados
artillery report balanceado-6replicas.json
```
Resultados resumen (ya medidos en el proyecto, mismos escenarios):

| Métrica | Baseline (1 réplica) | Balanceado (6 réplicas) | Mejora |
|---|---|---|---|
| Peticiones 200 | 256 | 747 | **×2.9** |
| VUs completados | 215 (13%) | 635 (38%) | **×2.95** |
| Fallos (timeout por saturación) | 1435 (86.9%) | 1015 (61.5%) | −25 p.p. |
| Latencia p50 | 1436 ms | 1978 ms | similar |
| Latencia p95 | 7865 ms | 8520 ms | similar |

> **Checklist:** cumple **P3.3** — escenarios distintos (sin LB vs con LB) que **caracterizan** la respuesta del sistema a la misma demanda. El balanceador triplica los éxitos.

## D12 — UI de Consul desde el host

En el navegador del host:
```
http://localhost:18500   (Consul de web1)
http://localhost:28500   (Consul de web2)
```
Salida esperada: en Services aparece `qubo` (verde/critical según el momento), en Nodes los 3 nodos, y el gráfico del líder.

> **Checklist:** complementa **P1.x** (evidencia visual del cluster) y sirve al sustento.

---

# RESUMEN — QUÉ SE CUMPLIÓ CON QUÉ

| Paso de verificación | Checklist que se cumple ahí |
|----------------------|------------------------------|
| D1: VMs running | P2.2, P1.1 |
| D2: `consul members` (3 alive, serv+client) | P1.1, P1.2, RG4 |
| D3: catálogo con servicio `qubo` | P1.3 |
| D4: health checks passing | P1.4 |
| D5: PM2 (3 réplicas por VM) | P2.3, base de P3.1 |
| D6: `/health` de cada réplica responde | P2.3 |
| D7: header rota según réplicas vivas | **RG1** (balanceador decide) |
| D8: stats :7000 desde el host | **RG3** |
| D9: subir `replicas` y ver 4→8 servers | **P3.1** (escalabilidad visible) |
| D10: apagar todo → 503 custom | **P3.2** |
| D11: Artillery baseline vs balanceado | **P3.3** |
| D12: Consul UI | P1.x (visual) |

RG2 (HAProxy en el balanceador, app en los webservers) queda cubierto por el mapa de la Parte B y los archivos de la Parte C.