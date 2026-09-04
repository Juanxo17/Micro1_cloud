# Sustentación / Recapitulación — Fase 3 — HAProxy, Escalabilidad y Artillery

> Documento de sustentación individual de la **Fase 3** del microproyecto **Cluster Consul + HAProxy + Artillery**.
> Nivel: explicado **para bobos**, desde cero. Estructura: primero las herramientas y conceptos a fondo, luego el paso a paso con los comandos exactos y los resultados de las pruebas.

---

## PARTE 1 — Teoría y herramientas a fondo

### 1. Los 3 pilares que se demuestran en esta fase

| Pilar | Qué demuestra | Cómo se nota en la práctica |
|-------|---------------|-----------------------------|
| **Balanceo de carga** | El tráfico se reparte entre las 6 réplicas | Header `X-Qubo-Served-By: server-N` va rotando 1→2→…→6 |
| **Tolerancia a fallos** | Si TODOS los servers caen, el sistema avisa lindo | `curl` devuelve HTTP **503** con página custom "Qubo no disponible" |
| **Escalabilidad** | Se agregan réplicas en caliente con solo cambiar una variable | `replicas: 2 → 3` → stats pasa de 4 a 6 servers **UP** |

---

### 2. Tolerancia a fallos y el errorfile 503 custom

**El problema:** por defecto, cuando HAProxy no tiene NINGÚN server disponible, responde un error 503 genérico y feo del navegador. Eso no da información y rompe la experiencia de usuario.

**La solución:** se configuró HAProxy con una **página de error personalizada** (un archivo `qubo_down.http` en Jinja2) que se sirve cuando no hay servidores:
```
HAProxy: <errorfile 503 /etc/haproxy/errors/qubo_down.http>
```
Cuando hicimos la prueba (deteniendo todas las réplicas de web1 y web2 con `pm2 stop`), desde el host esto respondió:
```bash
curl -i http://localhost/health
# HTTP/1.1 503 Service Unavailable
# --- página custom "Qubo no disponible" ---
# x-qubo-served-by: NINGUNO
```
El header extra `X-Qubo-Served-By: NINGUNO` aparece porque no había ningún server que lo atendiera — evidencia explícita de que el balanceador **sí sabía** que no había backends y por eso sirvió el error.

**Rol en esta fase:** demuestra que ante caída total el sistema **se comporta de forma controlada y comunicativa** (no es un error silencioso).

---

### 3. Escalabilidad horizontal con una variable

**Escala horizontal** = agregar más instancias (réplicas) para absorber más carga, en vez de hacer cada máquina más poderosa (escala vertical).

En nuestra infraestructura, el número de réplicas es una **variable de Ansible**:
```yaml
# roles/qubo-app/defaults/main.yml y roles/haproxy/defaults/main.yml
replicas: 3
```
El rol `qubo-app` lanza N réplicas (`qubo-<vm>-8080`, `-8081`, `-8082`, ...) y el rol `haproxy` genera un backend con el mismo rango de servidores. **Ambos deben tener el mismo número** (sino HAProxy apuntaría a puertos donde no hay app).

**Qué hicimos para escalar:** cambiamos `replicas: 2 → 3` en ambos defaults y reprovisionamos las 3 VMs:
```bash
vagrant provision web1   # agrega la réplica 8082 a web1
vagrant provision web2   # agrega la réplica 8082 a web2
vagrant provision lb     # regenera el backend de HAProxy con 6 servers
```
Resultado visible:
- Stats GUI (`http://localhost:7000`): **6 servers** (web1-8080/8081/8082, web2-8080/8081/8082) todos **UP**.
- Header rotando: `server-1 … server-6`.

**Rol en esta fase:** el requisito #6 de la rúbrica ("instanciar varias réplicas y observar el cambio en las estadísticas") queda cumplido y documentado.

---

### 4. Artillery — El generador de carga

**¿Qué es?** Herramienta de pruebas de carga para APIs HTTP. Simula "usuarios virtuales" (VUs) que golpean el sistema a un ritmo configurable, y mide: tiempo de respuesta, throughput, errores.

**Estructura de un escenario** (`baseline-sin-lb.yml` y `balanceado-6replicas.yml`):
```yaml
config:
  target: 'http://192.168.56.30'        # a quién golpea
  phases:
    - duration: 30
      arrivalRate: 5                     # 5 usuarios nuevos/segundo (calentamiento)
      name: "Calentamiento"
    - duration: 90
      arrivalRate: 15                    # carga principal: 15 usuarios/seg
      name: "Prueba"
    - duration: 30
      arrivalRate: 5                     # enfriamiento
      name: "Enfriamiento"
scenarios:
  - name: "Login real"
    weight: 50                           # 50% del tráfico
    flow:
      - post:
          url: "/api/login"
          json: { email: "test@artillery.qubo", password: "Test1234" }
          capture:
            - json: "$.token"
              as: "authToken"
  - name: "Feed autenticado"
    weight: 20
    flow:
      - post:
          url: "/api/login" ...
      - get:
          url: "/api/posts"
          headers:
            Authorization: "Bearer {{ authToken }}"
```

**Los `capture`** guardan el token devuelto por el login y lo reutilizan en la siguiente petición (`{{ authToken }}`) — simula un usuario que de verdad navega.

**Cómo se corre (desde el host, Artillery ya instalado):**
```bash
cd load_balancer/load-tests
artillery run baseline-sin-lb.yml --output baseline-sin-lb.json
artillery run balanceado-6replicas.yml --output balanceado-6replicas.json
```

**Cómo se interpreta el JSON** (cada archivo de salida tiene `counters` y `summaries`):
```js
counters['vusers.created']      // usuarios lanzados
counters['vusers.failed']       // fallidos (por ejemplo ETIMEDOUT)
counters['http.codes.200']      // respuestas exitosas
summaries['http.response_time'] // latencias: p50, p95, p99, min, max
```

---

## PARTE 2 — Paso a paso de la Fase 3 (con comandos y resultados)

### Paso 1 — Dejar el backend probado end-to-end
Se creó un usuario de prueba con su perfil y posts (para que el login y el feed autenticado devuelvan datos reales):
```bash
POST /api/register  {email: test@artillery.qubo, password: Test1234}
POST /api/profile   {nombre: "Artillery Test", nombreUsuario: "artillerytest"}
POST /api/posts     {contenido: "Post de prueba..."}
```

### Paso 2 — Escalar de 4 a 6 réplicas
1. `replicas: 2 → 3` en `roles/qubo-app/defaults/main.yml` **y** `roles/haproxy/defaults/main.yml`.
2. Reprovisionar web1, web2, lb.
3. Verificar:
```bash
curl -sI http://192.168.56.30/health | grep -i x-qubo-served-by   # rotan 1..6
# GUI: http://localhost:7000 → 6 servers UP
```

### Paso 3 — Preparar los escenarios de prueba
Dos lecturas comparables con **la misma carga** (1650 VUs en ~2.5 min):

| Escenario | Target | Qué representa |
|-----------|--------|----------------|
| `baseline-sin-lb.yml` | `http://192.168.56.10:8080` | 1 sola réplica, **sin** balanceador |
| `balanceado-6replicas.yml` | `http://192.168.56.30` | 6 réplicas vía **HAProxy** |

### Paso 4 — Correr Artillery
```bash
nohup artillery run baseline-sin-lb.yml --output baseline-sin-lb.json &
nohup artillery run balanceado-6replicas.yml --output balanceado-6replicas.json &
```

### Paso 5 — Resultados obtenidos (misma carga en ambos)

#### Baseline — 1 réplica (sin balanceador)
| Métrica | Valor |
|---|---|
| VUs creados / completados | 1650 / 215 (13%) |
| Peticiones exitosas (HTTP 200) | 256 |
| Fallidas (ETIMEDOUT) | 1435 (86.9%) |
| Latencia p50 / p95 / p99 | 1436 / 7865 / 9047 ms |

#### Balanceado — 6 réplicas vía HAProxy
| Métrica | Valor |
|---|---|
| VUs creados / completados | 1650 / 635 (38%) |
| Peticiones exitosas (HTTP 200) | 747 |
| Fallidas (ETIMEDOUT) | 1015 (61.5%) |
| Latencia p50 / p95 / p99 | 1978 / 8520 / 9607 ms |

#### Comparativa
| Métrica | Baseline | Balanceado | Mejora |
|---|---|---|---|
| Peticiones exitosas (200) | 256 | 747 | **×2.9** |
| VUs completados | 215 | 635 | **×2.95** |
| Tasa de fallo | 86.9% | 61.5% | −25.4 p.p. |

**Lectura honesta de los resultados:**
- La **ganancia es en throughput/éxito**, no en latencia: el cuello de botella es la CPU (bcrypt del login es deliberadamente costoso) y las VMs de 1GB RAM/2CPU, así que p95/p99 se mantienen similares porque las colas se llenan igual.
- Los ETIMEDOUT masivos significan que el servidor no aceptó la conexión dentro del tiempo límite: es la firma clásica de **una sola instancia saturada** vs. un pool que la distribuye.
- Índice de éxito **3× mayor** en el balanceado = la hipótesis H1 del plan de pruebas (**confirmada**).

### Paso 6 — Prueba de tolerancia a fallos (503 custom)
1. Se detuvieron todas las réplicas con PM2 en web1 y web2.
2. Desde el host:
```bash
curl -i http://localhost/health
# HTTP/1.1 503 Service Unavailable  → página custom "Qubo no disponible"
# x-qubo-served-by: NINGUNO
```
3. Se volvieron a arrancar las réplicas (`pm2 restart ...`) y el balanceo volvió a repartir (`/health` → ok en las 6).

### Paso 7 — Problemas resueltos sobre la marcha

| Problema | Causa | Solución |
|----------|-------|----------|
| Artillery se quedaba "tildado" en la shell | El proceso bloqueba la terminal del host | Correr con `nohup ... &` (background) y monitorear el JSON |
| Baseline con ETIMEDOUT masivo | 1 réplica no aguanta 15 usuarios/s (bcrypt en login) | Es el resultado esperado del baseline: se documentó como tal |
| Login 404 / posts sin contenido | El usuario de prueba no tenía perfil en la DB | Crear perfil + 2 posts antes de las pruebas |
| Error al arrancar servidor era mentira | Guest Additions desactualizada | Ignorar el warning; los montajes funcionan |

---

## PARTE 3 — Decisiones tomadas y alternativas descartadas

| Decisión | Por qué | Alternativa descartada |
|----------|---------|------------------------|
| Round-robin | Justo para réplicas idénticas; simple de demostrar | leastconn / ip-hash (complejidad sin beneficio aquí) |
| Error 503 custom + header `NINGUNO` | Comportamiento controlado y comunicativo ante caída total | 503 por defecto de HAProxy |
| Artillery con **la misma carga** en ambos escenarios | Aisla la variable (balanceador) y permite comparación válida | Cargas distintas por escenario (no comparables) |
| Baseline directo a web1:8080 (sin LB) | Mide el "antes" real: una réplica que satura | Baseline "con LB pero 1 server" (mezcla conceptos) |
| Documentar ETIMEDOUT como evidencia | Es el comportamiento real del colapso, no un error a maquillar | Ajustar la carga hasta que todo pase (engañoso) |
| Correr Artillery en background | Evita bloquear la terminal/shell del host | Correr en foreground (la shell "se tildea") |

---

**Fin de la Fase 3 — HAProxy, Escalabilidad y Artillery.** Aprobado: balanceo round-robin verificado (rotan server-1..6), escalado 4→6 réplicas visible en stats, 503 custom probado en caída total, y pruebas de carga documentadas (3× más éxitos con balancear) ✅