# PLAN DE PRUEBAS DE LOAD BALANCING - MICROPROYECTO 1 (QUBO + HAProxy)

**Universidad**: UAO | **Materia**: Especialización en IA | **Fecha**: Septiembre 2026

---

## OBJETIVOS DE LA INVESTIGACIÓN

### Objetivo General
Evaluar y comparar el rendimiento de Qubo **sin balanceador** (1 réplica directa) vs **balanceado por HAProxy** (6 réplicas con round-robin), en la infraestructura de 3 VMs (Consul + HAProxy), utilizando métricas cuantificables de performance.

### Objetivos Específicos
1. Establecer **baseline** de rendimiento sin load balancing (1 réplica).
2. Medir el beneficio de **escalabilidad horizontal** (réplicas: 2 -> 3 por VM) a través de HAProxy.
3. Documentar Tolerancia a Fallos: header `X-Qubo-Served-By`, errorfile 503 custom, health checks Consul `passing`.

### Hipótesis de Trabajo
- **H1**: HAProxy (6 réplicas) responderá al menos 2x más peticiones exitosas que una réplica única bajo la misma carga.
- **H2**: La latencia p99 bajo carga será similar (el cuello de botella es CPU/bcrypt), pero el número de éxitos será claramente mayor en el balanceado.

---

## INFRAESTRUCTURA DE PRUEBAS

### Componentes del Sistema
- **Aplicación**: Qubo (Node.js + Express + MongoDB Atlas)
- **Balanceador**: HAProxy (192.168.56.30, roundrobin, 6 backend servers)
- **Servidores web**: web1 (192.168.56.10) y web2 (192.168.56.20), 3 réplicas PM2 c/u (8080, 8081, 8082)
- **Service Discovery**: Consul (clúster 3 nodos, health checks `qubo` passing)
- **Testing Tool**: Artillery 2 (instalado en el host)
- **Monitoreo**: HAProxy stats GUI en `:7000`, PM2, Consul UI

### Datos de prueba
- Usuario: `test@artillery.qubo` / `Test1234` (perfil creado + 2 posts para el feed autenticado)
- Endpoints: `/health`, `/whoami`, `/api/login`, `/api/posts` (autenticado con Bearer)

---

## ESCENARIOS DE PRUEBA

## ESCENARIO 1: BASELINE - SERVIDOR ÚNICO (sin load balancing)
**Objetivo**: Establecer métricas de referencia apuntando DIRECTO a una réplica (web1:8080), sin pasar por HAProxy.

### Configuración
- Target: `http://192.168.56.10:8080` (1 sola réplica, la cuarta parte de la infraestructura)
- Fases: 30s warmup (5 rps) + 90s prueba (15 rps) + 30s cooldown (5 rps)
- Escenarios: health 20%, whoami 10%, login 50%, feed autenticado 20%

### Comando
`artillery run baseline-sin-lb.yml --output baseline-sin-lb.json`

### Resultados Obtenidos
| Métrica | Valor |
|---|---|
| VUs creados / completados | 1650 / 215 (13%) |
| Peticiones HTTP totales | 1691 |
| Respuestas HTTP 200 | 256 |
| Peticiones fallidas | 1435 (86.9%) — **ETIMEDOUT** por saturación de la réplica |
| Latencia p50 | 1436.8 ms |
| Latencia p95 | 7865.6 ms |
| Latencia p99 | 9047.6 ms |
| Latencia máx | 9364 ms |

**Análisis**: Una sola réplica + bcrypt saturen la cola de conexiones de Node/Puerto 8080; el 87% de los virtuales ni siquiera logra completar (ETIMEDOUT = no establece conexión TCP dentro del timeout). Este es el clásico colapso de un único punto.

---

## ESCENARIO 2: BALANCEADO - HAProxy (6 réplicas, round-robin)
**Objetivo**: Comparar el mismo tráfico contra el balanceador con las 6 réplicas.

### Configuración
- Target: `http://192.168.56.30` (HAProxy, rota entre server-1..6)
- Mismas fases, misma carga, mismos escenarios (repetibilidad).

### Comando
`artillery run balanceado-6replicas.yml --output balanceado-6replicas.json`

### Resultados Obtenidos
| Métrica | Valor |
|---|---|
| VUs creados / completados | 1650 / 635 (38%) |
| Peticiones HTTP totales | 1764 |
| Respuestas HTTP 200 | 747 |
| Respuestas HTTP 503 | 8 (durante pico) |
| Peticiones fallidas | 1015 (61.5%) — ETIMEDOUT |
| Latencia p50 | 1978.7 ms |
| Latencia p95 | 8520.7 ms |
| Latencia p99 | 9607.1 ms |
| Latencia máx | 9880 ms |

**Análisis**: Con las 6 réplicas el sistema absorbe **~3x más peticiones exitosas** (747 vs 256) y completa ~3x más VUs. La latencia p95/p99 no mejora (el cuello de botella es CPU bcrypt + disco, que se comparte entre las 3 réplicas de cada VM de 1GB RAM); el beneficio medible es **throughput y tasa de éxito** (H1 CONFIRMADA). Los 8 errores 503 son conexiones que HAProxy rechazó en el pico de saturación (todas las réplicas ocupadas), lo que valida que el errorfile 503 custom se sirve en producción.

---

## COMPARATIVA FINAL

| Métrica | Baseline (1 réplica) | Balanceado (6 réplicas) | Mejora |
|---|---|---|---|
| Peticiones exitosas (200) | 256 | 747 | **2.92x** |
| VUs completados | 215 | 635 | **2.95x** |
| Tasa de fallo | 86.9% | 61.5% | -25.4 p.p. |
| Latencia p50 | 1436.8 ms | 1978.7 ms | peor (colas iguales) |
| Latencia p95 | 7865.6 ms | 8520.7 ms | similar |

### Conclusión
- **Escalabilidad (requisito #6)**: subir `replicas` de 2 a 3 y reprovisionar con Ansible agregó 2 réplicas "en caliente", visibles en stats GUI (`:7000`) y header rotando `server-1..6`.
- **Balanceo (requisito #1)**: HAProxy round-robin reparte uniformemente; el mismo tráfico logra ~3x más éxito con 6 réplicas.
- **Tolerancia a fallos (requisito #7)**: con todas las réplicas detenidas, HAProxy devuelve el errorfile 503 custom + header `X-Qubo-Served-By: NINGUNO`.
- **Service discovery (Módulo 1)**: Consul mantiene los health checks `qubo` charados y registrados; el clúster queda `passing`.

### Limitaciones del experimento
- Carga limitada por la capacidad de la máquina host y las VMs de 1GB RAM / 2 CPU. No se mide realismo de usuarios sino comparación relativa bajo carga idéntica.
- bcrypt (login) es deliberadamente costoso en CPU; satura primero, por eso la latencia no mejora.