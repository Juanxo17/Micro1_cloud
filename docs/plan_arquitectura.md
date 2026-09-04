# Plan de Arquitectura e Infraestructura — Microproyecto 1

> Documento de planeación técnica del microproyecto. Complementa a `constitution.md` (contexto/checklist) con la arquitectura de infraestructura, las decisiones de diseño y el plan de ejecución por fases.

---

## 1. Decisiones de arquitectura (consolidadas)

| # | Aspecto | Decisión |
|---|---------|----------|
| 1 | RAM/CPU de todas las VMs | **1024 MB / 2 CPU** (uniforme) |
| 2 | Base box Vagrant | **`bento/ubuntu-22.04`** |
| 3 | Provisionador | **Ansible** (estándar de industria) |
| 4 | Base de datos | **MongoDB Atlas (cloud)** — cluster oficial de Qubo reactivado. Sin BD dedicada en VM. |
| 5 | Balanceador | **HAProxy** en VM dedicada |
| 6 | Service discovery | **HashiCorp Consul** (agentes server en web1/web2, client en lb) |
| 7 | Integración HAProxy↔Consul | **consul-template** (config dinámica) como flujo principal + health checks HTTP nativos de respaldo |
| 8 | Identificación de servidor | **Opción C**: endpoint `/whoami` en Qubo + header `X-Qubo-Served-By` en HAProxy |
| 9 | Qubo | Se edita en **repo aparte (rama)**, NO se embebe en este repo. El foco es infra/balanceo/pruebas. |
| 10 | Escalabilidad | Múltiples réplicas Node por VM (puertos 8080/8081/...) + opcional VM extra en escenarios |

---

## 2. Arquitectura de red (VMs)

| VM | Rol | IP | RAM/CPU | Correrá | Puertos expuestos al host |
|----|-----|-----|------|---------|---------------------------|
| `web1` | Servidor web 1 | 192.168.56.10 | 1024 MB / 2 CPU | Qubo (Node, réplicas 8080/8081...) + **Consul server** | 8500 (Consul UI) |
| `web2` | Servidor web 2 | 192.168.56.20 | 1024 MB / 2 CPU | Qubo (Node, réplicas 8080/8081...) + **Consul server** | 8500 (Consul UI) |
| `lb`  | Balanceador | 192.168.56.30 | 1024 MB / 2 CPU | **HAProxy** + **consul-template** + **Consul client** | 80 (proxy), 7000 (stats GUI) |

**Red privada Vagrant:** `192.168.56.x` (host-only).

---

## 3. Topología Consul

- **web1, web2:** agentes Consul en modo **server** → forman el clúster y almacenan el catálogo de servicios.
- **lb:** agente Consul en modo **client** → consulta el catálogo (service discovery) sin participar en el quórum.
- Justificación sustentable: 2 servers + 1 client es la topología mínima correcta de un service mesh con service discovery distribuido. No hay quórum perfecto (ideal 3 servers) pero es correcto y defendible para el alcance académico de 2 nodos: lo importante es demostrar que los agentes forman clúster y se ven entre sí (`consul members`).

---

## 4. Flujo de datos

```
Cliente / Artillery (host)
   │  :80 (proxy)                :7000 (stats GUI)
   ▼                             ▼
LB 192.168.56.30 ── HAProxy ── consul-template (config dinámica)
   │                             ▲
   ├──► web1:8080/8081... Qubo ──► Consul server
   └──► web2:8080/8081... Qubo ──► Consul server
         │
         └────► MongoDB Atlas (cloud) — DATABASE_URL compartida
```

- Los clientes **NUNCA** acceden directo a web1/web2 vía proxy — solo HAProxy decide.
- El service discovery flujo: Consul cataloga los servicios Qubo sanos → consul-template genera el `backend` de HAProxy dinámicamente → HAProxy balancea.

---

## 5. Qubo: ajustes (rama aparte)

**Endpoints a agregar/mejorar:**
1. **`/health`** — health check mejorado: verifica conexión a MongoDB (`mongoose.connection.readyState`) y retorna `{ node, instance, status, db }`. Para Consul y HAProxy.
2. **`/whoami`** — retorna `{ node, instance, hostname, pid }`, seteados vía `NODE_ID` / `INSTANCE_ID` en `.env` de cada réplica. Para demostrar qué servidor respondió.

**Cambios de código:**
- Fix de `NODE_ENV`: asegurar que el server siempre escuche (quitar o invertir `if (process.env.NODE_ENV !== 'production')`).
- Ajustar CORS si hace falta para aceptar las IPs de las VMs / host.
- Desactivar/graceful-degrade de Firebase (solo notificaciones, no es core).

**Reproducibilidad:**
- Generar y commitear `package-lock.json`.
- Crear `.env.example` con: `DATABASE_URL`, `JWT_SECRET`, `JWT_EXPIRES_IN`, `PORT`, `NODE_ENV`, `NODE_ID`, `INSTANCE_ID`.

**Despliegue:**
- Shared folder de Vagrant monta el Backend de Qubo desde el host a `/opt/qubo` en web1/web2.
- Ansible corre `npm install`, genera `.env` por nodo, y levanta con PM2 (tantas réplicas como se pida).

---

## 6. HAProxy

- **Frontend `qfront`** `:80` → backend `qubo_servers`.
- **Algoritmo:** `roundrobin` (default) — a definir si se prueban otros en escenarios.
- **Health checks HTTP** contra `/health` (interval ~3s, rise 2, fall 3) como respaldo; el set de servers dinámico lo provee consul-template.
- **Custom errorfile 503** `qubo_down.http` → página personalizada cuando ningún servidor está disponible (requisito #7).
- **Frontend stats `:7000`** con `stats enable`, `stats uri /`, auth básica (credentials en variable). Accesible desde el host vía port-forward (requisito #3).
- **Header `X-Qubo-Served-By`** con el id del server que respondió (para el demo de quién responde).

---

## 7. Ansible (aprovisionamiento completo, idempotente)

**Rol a cargo de cada VM:**
| VM | Roles |
|----|-------|
| web1 | `consul-server`, `qubo-app` (réplicas) |
| web2 | `consul-server`, `qubo-app` (réplicas) |
| lb | `consul-client`, `consul-template`, `haproxy` |

**Estructura (dentro de `load_balancer/`):**
```
ansible/
├── ansible.cfg
├── inventory            (generado por Vagrant, mapeo de grupos)
├── playbooks/
│   ├── site.yml
│   ├── consul.yml
│   ├── qubo.yml
│   └── haproxy.yml
└── roles/
    ├── consul-server/
    ├── consul-client/
    ├── consul-template/
    ├── qubo-app/
    └── haproxy/
```

- **Vagrantfile:** levanta 3 VMs, provisiona con el `ansible` provisioner aplicando `site.yml`.
- Cada rol deja **todo listo de una vez**: instala binarios, configura, levanta servicios, health checks, `.env` por nodo, PM2 con réplicas.
- **Idempotente:** `vagrant provision` adicional no rompe nada — clave para el demo de caída/escalabilidad.

---

## 8. Escalabilidad (requisito #6)

- **Base:** 1 réplica/VM (web1:8080, web2:8080) — cumple "cada servidor corre un solo server web en NodeJS".
- **Demo de escalabilidad:** role `qubo-app` con `replicas: N` → web1:8080/8081/..., web2:8080/8081/... → HAProxy (vía consul-template dinámico) las agrega → visible en GUI stats.
- Alternativa: VM extra opcional con Qubo (escenario horizontal) en las pruebas.

---

## 9. Pruebas de carga (Artillery) — reutilizando metodología del precedente

**Escenarios definidos (contra `http://192.168.56.30:80`):**
1. Baseline web1 solo (web2 deshabilitado) — 5→10→5 req/s
2. Baseline web2 solo — 5→10→5 req/s
3. Balanceado (2 VMs base) — 5→15→5 req/s
4. Escalado (con réplicas extra) — 5→20→5 req/s
5. Pico/Spike — variación brusca de tráfico
6. Tolerancia a caída total — se matan web1+web2, validar página 503 custom + stats

**Scenarios (pesos):** Homepage 40%, Login 30%, Feed autenticado 20%, Health 10%.

**Métricas objetivo:** RT <300ms (bueno), <500ms (aceptable), <1000ms (crítico); throughput >150 req/s (bueno), >100 (aceptable), >50 (crítico); error rate <0.5% (bueno), <2% (aceptable), <5% (crítico).

**Documentación:** metodología de hipótesis + métricas + plantilla comparativa (uso directo del rigo de FinalTelematicos).

---

## 10. Plan de ejecución por fases

> Nota: los Mds de recapitulación por módulo se generan **después** de implementar cada fase (paso a paso, con mi guía), NO antes, y siempre con aprobación previa. Cada MD incluye aclaración exhaustiva técnica y teórica de las herramientas (nivel "para bobos").

| Fase | Objetivo | Entregable |
|------|----------|-----------|
| **Fase 0** | Debug de Qubo en Windows + rama con `/health` y `/whoami` | Qubo funcionando local + rama de cambios |
| **Módulo 1** | Cluster Consul (teoría + implementación) | Consul en 3 VMs, servicio registrado, `consul members` sano |
| **Módulo 2** | Ansible + Vagrant (teoría + implementación) | Vagrantfile 3 VMs + roles, `vagrant up` reproducible |
| **Módulo 3** | HAProxy + consul-template + Artillery (teoría + implementación) | Balanceo, stats :7000, fallback 503, escalabilidad, escenarios + resultados |
| **Módulo 4** | Sustentación individual | Demo en vivo + respuestas conceptuales |

**Orden lógico de construcción:** El Vagrantfile + roles Ansible (M2) son la base sobre la que corre Consul (M1). Aunque el checklist numera Consul como M1, en la práctica se construye primero la infraestructura de aprovisionamiento y luego la lógica de Consul sobre ella.

---

## 11. Diagrama de arquitectura (archify)

Generado cuando se solicite — tipo `architecture` con boundaries por VM (web1, web2, lb, host) + MongoDB Atlas (cloud), componentes HAProxy/consul-template/agentes Consul/réplicas Qubo, y conexiones con labels de protocolo/puerto (`:80`, `:7000`, `:8500`, health `/health`, service discovery, `DATABASE_URL`).
