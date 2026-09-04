# GUION FINAL DE SUSTENTACIÓN PARA JULIAN — Contexto completo del proyecto

> **A quién va dirigido:** Julian. No conoce el proyecto de nada. Este documento le da TODO el contexto para poder **sustentar** con soltura: qué se construyó, por qué, cómo funciona cada pieza, y qué responder si el profesor pregunta.
>
> **Nota:** la sustentación es **individual** (la rúbrica lo dice). Este guion cubre la parte de infraestructura/reproducción — la que Julian validó levantándola en su máquina.

---

## 1. El proyecto en 30 segundos (discurso de apertura)

> "Construimos un **cluster de Consul** (service discovery) en tres máquinas virtuales, desplegamos la aplicación **Qubo** (una red social en Node.js + MongoDB) en dos de ellas con **varias réplicas** por máquina, pusimos un **balanceador de carga (HAProxy)** que recibe todas las peticiones y las reparte entre las réplicas, y **probamos con Artillery** cuánto aguanta el sistema balanceado vs. sin balancear. Todo se levanta automáticamente con **Vagrant + Ansible**: `vagrant up` y la infraestructura aparece sola."

**Las 3 VMs:**
| VM | IP | Rol | Qué corre |
|----|-----|-----|-----------|
| web1 | 192.168.56.10 | Servidor web | Consul server + Qubo (3 réplicas: 8080, 8081, 8082) |
| web2 | 192.168.56.20 | Servidor web | Consul server + Qubo (3 réplicas: 8080, 8081, 8082) |
| lb | 192.168.56.30 | Balanceador | Consul client + HAProxy (:80) + stats GUI (:7000) |

---

## 2. Glosario mínimo (para no quedar en blanco)

| Término | Qué es (para decirlo fácil) |
|---------|------------------------------|
| **Service mesh / service discovery** | Capa que permite a los servicios enterarse y encontrarse: qué servicios existen, dónde están, y si están sanos |
| **Consul** | Herramienta de HashiCorp que implementa service discovery con un **catálogo** central y **health checks** |
| **Agente Consul** | El proceso de Consul corriendo en cada máquina. Puede ser **server** (guarda el catálogo, participa del quórum) o **client** (solo consulta) |
| **Quórum / líder** | Con dos servers, Consul elige un **líder** (Raft). El `bootstrap_expect = 2` hace que esperen verse antes de elegir |
| **HAProxy** | Balanceador de carga HTTP: reparte peticiones entre varios servidores |
| **Round-robin** | Algoritmo de reparto cíclico: server-1, server-2, ..., server-6, server-1... |
| **Réplica** | Copia del mismo proceso (3 puertos por VM: la misma app en 8080/8081/8082) |
| **PM2** | Gestor de procesos de Node.js: mantiene las réplicas vivas y las reinicia si se caen |
| **Ansible** | Herramienta de aprovisionamiento (IaC): describe el "estado deseado" y lo aplica idempotentemente |
| **Vagrant** | Orquesta las máquinas virtuales de VirtualBox desde código (Vagrantfile) |
| **Artillery** | Generador de pruebas de carga (usuarios virtuales golpeando la API) |
| **Health check** | Consul/HAProxy preguntan a cada réplica GET /health cada pocos segundos para saber si sigue viva |
| **Éxito de una prueba** | Peticiones que terminaron en HTTP 200 (respuesta correcta) |

---

## 3. Arquitectura (para dibujarla / describirla)

```
Cliente (host / navegador / Artillery)
        │  SÓLO habla con el balanceador
        ▼
   HAProxy (lb :80)  ── round-robin ──▶  web1: 8080/8081/8082
        │                                   web2: 8080/8081/8082
        └──▶ Repartición según health checks (UP/DOWN)
 web1 y web2 corren tb agentes Consul SERVER (clúster dc1)
 lb corre agent Consul CLIENT (consulta catálogo)
 BD compartida: MongoDB Atlas (nube)
```

**Puntos que hay que poder decir en voz alta:**
1. "El cliente **nunca** pega directo a los servers; todo entra por HAProxy." → *requisito RG1 del enunciado.*
2. "Cada web VM tiene su agente Consul" (RG4) y "en el balanceador corre HAProxy" (RG2).
3. "La GUI del balanceador está en `:7000` y es accesible desde el host" (RG3).
4. "Para escalar solo cambio `replicas` y reprovisiono; las stats lo reflejan" (P3.1).

---

## 4. ¿Cómo funciona Qubo? (para poder explicarla)

- **Frontend:** React + Vite (carpeta `anuel`). Corre en el host de desarrollo (`npm run dev`, puerto 5173) o se sirve estático. Está configurado para hablar con `http://192.168.56.30:80/api` — **el balanceador**, no un servidor concreto.
- **Backend:** Node.js + Express + **MongoDB Atlas** (BD en la nube). Autenticación con JWT.
- **Health check:** `GET /health` devuelve `{"node":"web1","instance":"web1-8080","status":"ok","db":"connected"}` — identifica qué réplica respondió y confirma que la BD responde. Ese endpoint lo usan Consul y HAProxy para saber si la réplica está viva.
- **Flujo real de un login:** frontend → POST al balanceador → HAProxy elige réplica → la réplica valida en Atlas → devuelve token JWT.

**Por qué se puede balancear:** la app es *stateless* (sin sesiones en memoria); el estado vive en MongoDB. Cualquier réplica puede atender cualquier petición → el balanceo funciona sin romper nada.

---

## 5. Los 4 archivos que "guardan el proyecto" (dónde están y qué hacen)

| Archivo | Función |
|---------|---------|
| `Micro1_cloud/vagrant/Vagrantfile` | Define las 3 VMs, IPs, puertos reenviados (80, 7000, 18500, 28500) y el provisioner Ansible |
| `Micro1_cloud/vagrant/ansible/site.yml` | Decide qué roles van a cada VM según su hostname |
| `Micro1_cloud/vagrant/ansible/roles/*` | Roles Ansible: `consul-server`, `consul-client`, `qubo-app`, `haproxy` |
| `Micro1_cloud/load-tests/*` | Escenarios Artillery + resultados (baseline y balanceado) + informe |

---

## 6. Comandos CORE que Julian tiene que saberse de memoria

```bash
# Clientela: estado y acceso
cd Micro1_cloud/vagrant
vagrant status                    # 3 VMs running
vagrant ssh web1                  # entrar a un servidor web
vagrant ssh lb                    # entrar al balanceador

# Dentro de una VM web:
consul members                    # los 3 agentes, vivos
consul catalog services           # servicios registrados (consul, qubo)
sudo pm2 list                     # las 3 réplicas online

# Dentro de lb:
service haproxy status            # activo
cat /etc/haproxy/haproxy.cfg      # la config del balanceador

# Desde el host:
curl -s http://192.168.56.30/health                       # responde alguna réplica
for i in $(seq 1 6); do curl -sI http://192.168.56.30/health | grep -i x-qubo-served-by; done  # rota server-1..6

# Matar/reactivar UNA réplica (demo de caída):
vagrant ssh web1 -c "sudo pm2 stop qubo-web1-8082"
vagrant ssh web1 -c "sudo pm2 start qubo-web1-8082"

# Matar/reactivar TODO (demo de página 503):
vagrant ssh web1 -c "sudo pm2 stop all"
vagrant ssh web2 -c "sudo pm2 stop all"
curl -i http://192.168.56.30/health     # → 503 con la página personalizada
vagrant ssh web1 -c "sudo pm2 restart all"
vagrant ssh web2 -c "sudo pm2 restart all"

# Artillery (resultados ya guardados; para reejecutar):
cd Micro1_cloud/load-tests
artillery run baseline-sin-lb.yml --output baseline-sin-lb.json
artillery run balanceado-6replicas.yml --output balanceado-6replicas.json
```

---

## 7. Respuestas preparadas a las preguntas más probables del profesor

### "¿Por qué dos servers y un client? ¿Por qué no 3 servers?"
> "La topología mínima correcta para mostrar un clúster es 2 servers + 1 client. Con 2 servers hay quórum real (se elige líder con Raft, y lo vemos con `consul members` y la UI). El tercer nodo (lb) como client demuestra que el catálogo se **consume** desde otro lugar — que es exactamente lo que necesita un balanceador."

### "¿Qué es `bootstrap_expect = 2`?"
> "Le dice a Consul: no elijas líder hasta que haya 2 servers presentes. Así evitamos que los dos se crean líderes a la vez (split-brain). Cuando se ven, Consul elige uno por Raft."

### "¿Por qué round-robin y no least-connections o ip-hash?"
> "Porque todas las réplicas son un server web idéntico de la misma app (stateless): no hay sesiones que conservar ni diferencias de carga previas. Round-robin es simple, predecible y suficiente — y se demuestra con el header rotando `server-1..6`."

### "¿Cómo sabe Consul que una réplica sigue viva?" / "¿Qué es el health check?"
> "Registramos el servicio `qubo` en el catálogo con un health check que hace `GET /health` cada 5 segundos. Consul lo corre y marca el servicio passing/critical. HAProxy **también** hace su propio `check` por réplica (httpchk a /health), así el balanceador sabe a quién mandar tráfico."

### "El enunciado dice que cada servidor web corre SOLO UN server web Node.js, pero veo 3 réplicas por VM..."
> "Correcto, y es intencional: la **parte 3 del enunciado pide la escalabilidad** — 'instancie varias réplicas de los servidores web configurados y observe el cambio en las stats de HAProxy'. Empezamos con una réplica por VM y luego escalamos a 3 (y se puede a 4, 5, ...) para demostrar esa característica. Es literalmente el requisito de escalabilidad."

### "¿Por qué las 3 réplicas comparten el `.env`?"
> "Son la misma app: el JWT se firma con el mismo secreto para que cualquier réplica valide los tokens de las demás, y la BD es la misma (Atlas). Por eso el balanceo es transparente."

### "¿Qué significa ese header `X-Qubo-Served-By`?"
> "Es un encabezado que HAProxy agrega a cada respuesta indicando qué server (réplica) la atendió usando `%[srv_id]`. Nos sirve para demostrar el reparto: haciendo curl en un loop se ve rotar server-1..6, y si apago una réplica deja de aparecer."

### ¿Y si piden "máteme un servidor y demostréme que ya no responde"?
> "Apago el 8082 de web1 (`pm2 stop`), vuelvo a hacer el loop del header y server-3 ya no aparece. Si apago TODAS las réplicas, el balanceador devuelve la página 503 personalizada. Y en la GUI de `:7000` el server queda rojo (DOWN). Al reactivarla vuelve sola al green."

### "¿Cómo se aprovisiona todo?" 
> "Con Ansible vía Vagrant: el Vagrantfile corre `ansible_local` que ejecuta `site.yml` dentro de cada VM; según el hostname aplica los roles (Consul server, Qubo app, Consul client, HAProxy). Es idempotente: si ya está hecho, no rompe nada; el código fuente de Qubo se copia con `rsync` (excluyendo node_modules) y las dependencias se instalan dentro de la VM."

### "¿Qué resultados dieron las pruebas de carga?"
> "Bajo la misma carga, una sola réplica (baseline) logró 256 respuestas correctas y se cayó (86.9% de timeouts por saturación); el sistema balanceado de 6 réplicas logró **747 (≈3× más)** con 61.5% de fallos. Conclusión: balancear **triplica la capacidad de atender** aunque la latencia p95 sea similar (el cuello de botella es CPU/bcrypt de una VM de 1 GB)."

### "¿Cómo se destruye y se reconstruye todo?"
> "`vagrant destroy -f && vagrant up`. La VM se elimina y se recrea + provisiona sola. La única configuración manual es la ruta al backend de Qubo en el Vagrantfile, que es específica de cada máquina del desarrollador."

---

## 8. Guion de la demo propuesta (si Julian tiene que mostrar algo)

1. `vagrant status` → 3 VMs running.
2. `consul members` (entrando a web1) → 3 agentes vivos con server/client.
3. `curl http://192.168.56.30/health` + loop del header → el balanceador reparte (server-1..6).
4. Abrir stats GUI `http://localhost:7000` → las 6 réplicas en UP.
5. Abrir Consul UI `http://localhost:18500` → el catálogo y el clúster.
6. Matar 8082 de web1 → el loop ya no muestra server-3 y en :7000 queda DOWN → reactivar.
7. Apagar TODO → `curl -i` devuelve la **página 503 personalizada** → reactivar todo.
8. Mostrar `load-tests/*.json` o `artillery report` → baseline 256 vs balanceado 747.

---

## 9. Datos que TAMBIÉN hay que poder soltar rápido

| Pregunta | Respuesta |
|----------|-----------|
| ¿Box? | `bento/ubuntu-20.04`, 1024 MB RAM, 2 CPU por VM |
| ¿Puertos públicos? | HAProxy :80 y stats :7000; Consul UIs 18500 (web1) y 28500 (web2) |
| ¿Cuántas réplicas en total? | 6 (3 en web1 + 3 en web2) |
| ¿Qué algoritmo? | round-robin |
| ¿Dónde está el health endpoint? | `GET /health` en cada réplica |
| ¿La BD? | MongoDB Atlas (nube), misma para todas las réplicas |
| ¿Total de puntos del microproyecto? | 5.0 (1.5 Consul + 1.5 Aprovisionamiento + 1.0 Disponibilidad/Artillery + 1.0 Sustentación) |