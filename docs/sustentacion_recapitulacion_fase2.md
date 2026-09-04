# Sustentación / Recapitulación — Fase 2 — Aprovisionamiento de Qubo + HAProxy

> Documento de sustentación individual de la **Fase 2** del microproyecto **Cluster Consul + HAProxy + Artillery**.
> Nivel: explicado **para bobos**, desde cero. Estructura: primero las herramientas y conceptos a fondo, luego el paso a paso con los comandos exactos.

---

## PARTE 1 — Teoría y herramientas a fondo

### 1. Qué es "aprovisionamiento" y por qué importa

**Aprovisionamiento** (provisioning) es dejar una máquina lista para su trabajo: sistema operativo actualizado, paquetes instalados, servicios configurados y corriendo, archivos de configuración desplegados. En proyectos de infraestructura se hace **como código** (Infraestructura como Código, **IaC**): en lugar de instalar a mano, se describe el estado deseado en archivos y una herramienta lo aplica.

**Rol en esta fase:** dejar Qubo (la app a balancear) corriendo en las VMs web1 y web2, y dejar HAProxy listo como balanceador de carga — todo automatizado con Ansible desde el `Vagrantfile`.

**Por qué importa:** + el entorno queda **reproducible** (destruir y recrear da lo mismo), + la **documentación del estado** queda en archivos (nadie instala "de memoria"), y + se **escala con un cambio de variable** (ej. `replicas: 3`).

---

### 2. Node.js, npm y PM2

#### Node.js
Runtime de JavaScript del lado del servidor. Qubo (backend) corre sobre Node. En la fase instalamos Node 20 (LTS) con el instalador oficial de NodeSource.

#### npm
Gestor de paquetes de Node. `npm install` descarga e instala las dependencias que declara `package.json` de Qubo. En nuestro caso se ejecuta **dentro de la VM** (nunca en el host Windows), porque instalar dependencias en Windows producía montajes rotos y lentitud. Por eso quitamos `node_modules` del host: se genera en cada VM.

#### PM2
Gestor de procesos de Node.js. En lugar de correr Qubo con `node index.js` a mano (que muere al cerrar la terminal), PM2:
- mantiene el proceso vivo (lo reinicia si se cae),
- lo **clona** para correr N réplicas (ej. `replicas: 3` → 3 procesos en puertos 8080/8081/8082),
- corre como **servicio** (daemon) y aguanta reinicios de la VM.

**La clave de su configuración:** el `.env` de Qubo debe estar en el **mismo directorio desde el que se lanza el proceso** (`/srv/qubo/src/`) porque la librería `dotenv` lee el `.env` del directorio de trabajo actual (cwd). Si PM2 lanza desde otro directorio, la app no encuentra las variables (ej. la URL de MongoDB) y falla. Por eso el lanzamiento real fue:
```bash
cd /srv/qubo/src && pm2 start index.js --name qubo-web1-8080
```

---

### 3. MongoDB Atlas y el `.env`

Qubo guarda datos en **MongoDB Atlas** (DB en la nube). La conexión se configura con variables de entorno en un archivo `.env`:

```
database_url=mongodb+srv://<usuario>:<password>@<cluster>.mongodb.net
db_name=qubo_cloud
jwt_secret=<clave para firmar tokens>
```

**Detalles importantes descubiertos en la práctica:**
1. **Ubicación del `.env`:** dotenv lee el archivo desde el **directorio de trabajo** del proceso. La solución correcta fue ubicarlo en `/srv/qubo/src/.env` (junto a `index.js`) y lanzar PM2 con `cd /srv/qubo/src`.
2. **Whitelist de Atlas:** MongoDB Atlas bloquea conexiones por IP. La única forma de que las VMs (IP de salida de internet) conectaran fue **abrir la whitelist** (con `0.0.0.0`) desde la consola de Atlas. Sin eso, la app conecta a la DB y el health check reporta `db: disconnected`.

**Verificación del estado completo** (`GET /health`):
```json
{"node":"web1","instance":"web1-8080","status":"ok","db":"connected"}
```
- `status: ok` → la app responde.
- `db: connected` → MongoDB Atlas accesible.

---

### 4. Rsync, montajes (synced folders) y rutas en la VM

#### Synced folder de Vagrant
El `Vagrantfile` monta el código del backend del host en la VM con `n.vm.synced_folder`. La ruta montada es `/opt/qubo/backend`. Esto permite **escribir código en el host y ejecutarlo sincronizado en la VM** — pero forzar la instalación de `node_modules` ahí (a través del montaje VirtualBox) era lento y rompía.

#### Por eso usamos `rsync`
`rsync` copia archivos de un lugar a otro pidiendo sólo las diferencias. En el rol `qubo-app` copiamos el código **fuera** del montaje, a un directorio propio de la VM, excluyendo lo que no se necesita:
```bash
rsync -a --delete --exclude=node_modules --exclude=.env /opt/qubo/backend/ /srv/qubo/
```
- `/opt/qubo/backend/` → código fuente sincronizado desde el host (la fuente de verdad).
- `/srv/qubo/` → copia de trabajo donde la VM instala dependencias y corre.
- `--exclude=node_modules` → no copiar carpetas de dependencias (se instalan frescas en la VM).
- `--exclude=.env` → no copiar el archivo de secretos del host (el rol genera el suyo por separado).

**Rol en esta fase:** separar "código que edita el desarrollador" de "entorno instalado donde corre la app".

---

### 5. HAProxy y el balanceo round-robin

**¿Qué es HAProxy?** Balanceador de carga de código abierto que reparte tráfico HTTP entre varios servidores. Vive en la VM `lb` (192.168.56.30) y es la única puerta de entrada: nadie entra directo a web1/web2 desde el exterior.

**Algoritmo elegido: round-robin.** Reparte las peticiones en orden cíclico (server-1, server-2, ..., server-6, server-1, ...). Justo lo que queremos para un servicio idéntico (todas las réplicas de Qubo son iguales).

**Backend (los servidores a los que reparte):** las 6 réplicas = 3 por VM × 2 VMs:
```
web1-8080, web1-8081, web1-8082   (192.168.56.10)
web2-8080, web2-8081, web2-8082   (192.168.56.20)
```

**Frontends de HAProxy en esta fase:**
- `:80` (qfront) → puerto público; recibe el tráfico del host y lo reparte.
- `:7000` (qstats) → la **statística GUI**: panel que muestra el estado de los 6 servers (UP/DOWN), conexiones, y el tráfico. Se accede desde el host en `http://localhost:7000`.

**Header `X-Qubo-Served-By`:** en la respuesta HAProxy agrega un header que dice **qué server atendió** cada petición (`server-3`, etc.). Sirve para **probar que el balanceo sí rota**:
```bash
for i in $(seq 1 8); do curl -sI localhost/health | grep -i x-qubo-served-by; done
# server-1, server-2, server-3, ... server-6, server-1, ...
```

---

### 6. El rol Ansible `qubo-app` (paso a paso interno)

El rol que deja la app lista en cada VM web. Tareas principales (en orden):
1. **Instalar Node 20** vía NodeSource (repositorio oficial de Node).
2. **Instalar PM2** global (`npm i -g pm2`; queda como daemon).
3. **Copiar el código** con `rsync` desde `/opt/qubo/backend` → `/srv/qubo`.
4. **Instalar dependencias** en `/srv/qubo` (`npm install`) — dentro de la VM.
5. **Generar el `.env`** en `/srv/qubo/src/.env` (features: DB Atlas, nombre de DB, JWT), con permisos restringidos.
6. **Lanzar las réplicas** con PM2: por cada número de `replicas`, `cd /srv/qubo/src && pm2 start index.js --name qubo-<vm>-<puerto>` en puertos 8080, 8081, ...
7. **Habilitar PM2 al arranque** (`pm2 startup` + `pm2 save`) para que las réplicas sobrevivan reinicios.

**Idempotencia y escalado:** al cambiar `replicas: 2 → 3` y reprovisionar, el rol valida las réplicas existentes y agrega la nueva (8082) sin tocar las otras. Esto es lo que hicimos en la Fase 3 para **escalar** de 4 a 6 réplicas con un solo cambio de variable.

---

## PARTE 2 — Paso a paso de la Fase 2 (con comandos)

> En esta fase se montó la aplicación Qubo sobre las VMs web y el balanceador HAProxy sobre lb, todo automatizado por Ansible.

### Paso 1 — Preparar el backend en el host (sin node_modules)
El backend vive en el host (`estadoDelArte/Qubo/Backend`). Se monta a `/opt/qubo/backend` en cada VM web. Importante: **el host no debe tener `node_modules`** (se instalan dentro de la VM):
```bash
rm -rf Backend/node_modules
```

### Paso 2 — Extender el Vagrantfile
Se agregó a las VMs web el montaje del backend y a la VM lb los port-forwards del proxy y de las stats:
```ruby
# web1/web2:
n.vm.synced_folder "C:/.../Qubo/Backend", "/opt/qubo/backend"  # solo si qubo: true
# lb:
forwards: { "80" => "80", "7000" => "7000" }
```

### Paso 3 — Crear el rol `qubo-app`
Estructura:
```
roles/qubo-app/
├── tasks/main.yml      # instalar Node, PM2, copiar código, npm install, .env, réplicas
├── templates/env.j2    # el .env con las variables de Qubo
├── handlers/main.yml   # reiniciar réplicas
└── defaults/main.yml   # replicas: 3, database_url, db_name, jwt_secret
```

### Paso 4 — Crear el rol `haproxy`
```bash
apt install -y haproxy
cp templates/haproxy.cfg.j2 /etc/haproxy/haproxy.cfg
systemctl enable --now haproxy
```
El template genera el backend con las `replicas` por VM de forma dinámica (vuelta de Ansible `range`), los frontends `qfront` (:80), `qstats` (:7000), el header `X-Qubo-Served-By` y el errorfile 503 (Fase 3).

### Paso 5 — Reproducir con `vagrant provision` (web1, web2, lb)
Como web1 ya tenía la Fase 1, se provisionan las VMs una por una para aplicar los roles nuevos:
```bash
vagrant provision web1
vagrant provision web2
vagrant provision lb
```
Salida esperada por VM: `ok=N changed=K failed=0` (recap de Ansible). **`failed=0` = aprovisionamiento limpio.**

### Paso 6 — Verificar Qubo (health ends-to-end)
```bash
curl http://192.168.56.10:8080/health
curl http://192.168.56.20:8080/health
# {"node":"web1","instance":"web1-8080","status":"ok","db":"connected"}
```

### Paso 7 — Verificar el balanceador
```bash
curl http://192.168.56.30/health          # responden las 6 réplicas rotando
curl -sI http://192.168.56.30/health | grep -i x-qubo-served-by
# http://localhost:7000  → GUI de stats (6 servers UP)
```

### Paso 8 — Problemas resueltos sobre la marcha

| Problema | Causa | Solución |
|----------|-------|----------|
| La app arrancaba pero la DB no conectaba | `.env` en directorio incorrecto (dotenv lee el cwd) | Generar `.env` en `/srv/qubo/src/.env` y lanzar PM2 con `cd /srv/qubo/src` |
| MongoDB devolvía error de IP no permitida | Atlas bloquea por whitelist | Abrir la whitelist de Atlas (`0.0.0.0`) |
| `node_modules` en el host rompía los montajes | npm install sobre el synced folder VirtualBox | Borrar `node_modules` del host e instalar dentro de la VM |
| Ansible 2.9 no soporta `create_src_dir` | La opción del módulo `npm` no existe en 2.9.6 apt | Quitar la opción; lanzar réplicas con `cd + pm2 start` explícito |
| PM2 no encontraba `index.js` | Lanzado desde directorio incorrecto | Path absoluto `/srv/qubo/src/index.js` + `cd /srv/qubo/src` |
| `srv_name` en header no era válido | No es un *fetch* de HAProxy | Usar `%[srv_id]` → `server-N` |

---

## PARTE 3 — Decisiones tomadas y alternativas descartadas

| Decisión | Por qué | Alternativa descartada |
|----------|---------|------------------------|
| Instalar node_modules **dentro** de la VM | Los synced folders VirtualBox son lentos y se rompen con npm install | npm install en el host Windows |
| Copiar código con `rsync --delete` → `/srv/qubo` | Se separa el montaje del entorno instalado; reproducible | Correr desde el synced folder directo |
| `.env` en `/srv/qubo/src` con `cd` en el lanzamiento de PM2 | dotenv lee el cwd; sin esto la DB no conecta | Paths relativos/`.env` en raíz de la VM |
| HAProxy en su propia VM `lb` | Balanceador como punto único de entrada, separado de la app | HAProxy en web1 (mezcla roles) |
| Round-robin | Justo para réplicas idénticas; demostrable con el header | leastconn / ip-hash (no aportan para réplicas puras) |
| Puertos 8080/8081/8082 por VM | Replicación horizontal "parecida a producción" sin más hardware | Una sola instancia por VM (no demuestra escalado) |
| Stats en :7000 | GUI de monitoreo para la sustentación | Ninguna (monitoreo solo por CLI) |

---

**Fin de la Fase 2 — Aprovisionamiento Qubo + HAProxy.** Aprobado: Qubo corriendo en 2 VMs con 3 réplicas c/u, HAProxy balanceando round-robin, stats GUI visible, header de diagnóstico funcionando ✅