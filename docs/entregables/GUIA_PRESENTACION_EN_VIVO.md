# GUÍA DE SUSTENTACIÓN EN VIVO — Presentación página por página (Microproyecto 1)

> **Cómo usar este documento:** es el guion de lo que vas a **mostrar en pantalla y decir** mientras hablas. Cada "escena" dice: qué mostrar, qué decís, y qué comando tenés a mano. Orden pensado para que el profesor siga la historia: arquitectura → Qubo → aprovisionamiento → Consul → HAProxy → escalabilidad → tolerancia a fallos → Artillery → cierre.
>
> **Preparación (15 min antes):**
> ```bash
> cd Micro1_cloud/vagrant
> vagrant status                          # 3 VMs running
> curl -s http://192.168.56.30/health     # responde
> ```
> Y tener abiertas de antemano (pestañas): stats HAProxy `:7000`, Consul `:18500` y `:28500`, y el diagrama `docs/arquitectura.html`.

---

## ESCENA 0 — Apertura (1 min)

**Mostrar:** el diagrama de arquitectura (`docs/arquitectura.html` o la imagen del enunciado).

**Decís:**
> "Este es el proyecto: un clúster de Consul como service mesh, una aplicación web (Qubo) replicada en dos servidores, y un balanceador HAProxy que recibe todo el tráfico. Se levanta íntegramente con Vagrant + Ansible desde cero, y se evalúa con Artillery."

**Marcar a medida que hablás (checklist):**
- ☐ RG1 (cliente → balanceador, nunca directo) — está en el diagrama
- ☐ RG2 (Node en los servers, HAProxy en el balancer)
- ☐ RG3 (GUI del balancer en el host)
- ☐ RG4 (agente Consul en cada server web)

---

## ESCENA 1 — Las 3 VMs (1 min)

**Mostrar y ejecutar:**
```bash
cd Micro1_cloud/vagrant
vagrant status
```

**Decís:**
> "Tres máquinas virtuales: web1 (192.168.56.10), web2 (192.168.56.20) y el balanceador lb (192.168.56.30). Cada web VM corre Consul en modo server + Qubo con 3 réplicas; el balanceador corre Consul como client + HAProxy."

**Checklist:** ☐ P1.1 (agentes en VMs), ☐ P2.1/P2.2 (aprovisionamiento automático → esto es el resultado).

---

## ESCENA 2 — Qubo: la app que vamos a balancear (2 min)

**Mostrar:** la estructura de Qubo (carpetas `Backend/` y `anuel/`).

**Decís:**
> "Qubo es una red social. Frontend en React (carpeta anuel), backend en Node.js + Express con MongoDB Atlas. Es una app stateless: no guarda sesiones en memoria, todo el estado vive en la base. **Por eso se puede balancear**: cualquier réplica puede atender cualquier petición sin romper nada."
>
> "El frontend NO apunta a un servidor concreto: está configurado para hablar con la IP del balanceador."

**Mostrar el health check que usa todo el sistema:**
```bash
# Una réplica sin pasar por el balancer:
curl http://192.168.56.10:8080/health
# (o desde el host directamente)
```
Salida: `{"node":"web1","instance":"web1-8080","status":"ok","db":"connected"}`.

**Decís:**
> "Este endpoint dice qué réplica respondió y que la base responde. Es el que usan Consul y HAProxy para saber quién está vivo."

**Checklist:** ☐ P2.3 (Node + app instalada en las VMs), sienta las bases de P1.4/P3.1.

---

## ESCENA 3 — El aprovisionamiento (el "cómo se armó todo") (3 min)

**Mostrar** (ventana del editor, con estos archivos abiertos en pestañas):
1. `vagrant/Vagrantfile` → las 3 VMs, IPs, puertos reenviados, `ansible_local`.
2. `vagrant/ansible/site.yml` → roles por hostname.
3. Un rol representativo, p.ej. `roles/qubo-app/tasks/main.yml`.

**Decís:**
> "Todo se aprovisiona con Ansible a través de Vagrant. El Vagrantfile declara las VMs; el provisioner corre `site.yml` dentro de cada VM; según el hostname, Ansible aplica los roles: consol-server + qubo-app en web1/web2, client + haproxy en lb."
> "El rol qubo-app instala Node 20, PM2, copia el código desde la carpeta sincronizada con `rsync` (excluyendo node_modules), instala dependencias DENTRO de la VM, genera el `.env`, y lanza 3 réplicas en 8080/8081/8082."
> "Es idempotente y reproducible: `vagrant destroy -f && vagrant up` y vuelve a quedar todo igual."

**Si el profe pide un cambio en caliente** ("póngame otra réplica"): subir `replicas` en `roles/qubo-app/defaults/main.yml` y `roles/haproxy/defaults/main.yml`, y:
```bash
vagrant provision web1
vagrant provision web2
vagrant provision lb
```
Luego ir a la GUI `:7000` → ahora hay 8 servers. (Explicar: esto es exactamente el requisito de escalabilidad.)

**Checklist:** ☐ P2.1, ☐ P2.2, ☐ P2.3.

---

## ESCENA 4 — Consul: el service mesh (3 min)

**Mostrar:**
1. Entrar a web1 y correr:
```bash
vagrant ssh web1
consul members
consul catalog services
sudo ls /etc/consul.d/
sudo cat /etc/consul.d/server.hcl    # mostrar: server=true, bootstrap_expect=2, retry_join
```
2. Abrir la **UI de Consul en el navegador**: `http://localhost:18500` (web1) y `http://localhost:28500` (web2).

**Decís:**
> "Consul es el service mesh. web1 y web2 son agents en modo server: almacenan el catálogo y participan en el quórum con Raft (acá se ve el líder). lb es un agent client: consulta el catálogo, no guarda nada."
> "En `server.hcl` lo importante: `server = true`, `bootstrap_expect = 2` (no elige líder hasta tener 2 servers), y `retry_join` apuntando al otro server."
> "Registramos el servicio `qubo` con su health check: Consul hace GET /health cada 5 segundos. Vean la pestaña Services: el servicio está registrado."

**Verificar salud explícita:**
```bash
curl -s localhost:8500/v1/health/checks/qubo
```
→ `"Status": "passing"` en web1 y web2.

**Checklist:** ☐ P1.1 (agentes), ☐ P1.2 (`consul members` con 3 sanos), ☐ P1.3 (catálogo), ☐ P1.4 (health checks passing), ☐ RG4.

---

## ESCENA 5 — HAProxy: el balanceador (3 min)

**Mostrar:**
```bash
vagrant ssh lb
sudo cat /etc/haproxy/haproxy.cfg
```
Irá señalando con el cursor:
- `frontend qstats` (bind 7000) → la GUI.
- `frontend qfront` (bind 80) → `http-response set-header X-Qubo-Served-By server-%[srv_id]`.
- `backend qubo_servers` → `balance roundrobin` + las 6 líneas `server ... check`.

**Demostrar la rotación** (desde el host):
```bash
for i in $(seq 1 6); do curl -sI http://192.168.56.30/health | grep -i x-qubo-served-by; done
```
Salida: `server-1 ... server-6`.

**Decís:**
> "El header `X-Qubo-Served-By` dice qué réplica atendió cada petición. Recorriendo un loop se ve rotar de server-1 a server-6: round-robin repartiendo. Además cada server tiene su `check` definido con `httpchk GET /health`, así HAProxy mismo sabe quién está UP."

**Mostrar la GUI:** `http://localhost:7000` → las 6 réplicas en UP, contadores que suben al hacer los curls.

**Checklist:** ☐ RG1 (el reparto lo decide el balancer), ☐ RG2 (HAProxy en lb), ☐ RG3 (GUI accesible desde el host).

---

## ESCENA 6 — Escalabilidad: réplicas visibles en las stats (1 min)

**Esta escena dobla de la Escena 3 si el profe pidió el cambio en caliente; si no, hacela igual:**

**Mostrar/decir:**
> "La escalabilidad es un cambio de variable más reprovisionar."

```bash
# (desde el host)
vagrant provision lb             # regenera el backend de HAProxy con las réplicas configuradas
# y ver en :7000 cómo cambia la cantidad de servers
```

**Checklist:** ☐ **P3.1** (escalabilidad: instancie varias réplicas y observe el cambio en estadísticas de HAProxy).

---

## ESCENA 7 — Tolerancia a fallos: matar uno, matar todo (3 min)

**Parte A — matar UNA réplica** (el profe de "demuéstreme que se cayó"):

```bash
vagrant ssh web1 -c "sudo pm2 stop qubo-web1-8082"
# Loop del header — ya no debe aparecer server-3:
for i in $(seq 1 6); do curl -sI http://192.168.56.30/health | grep -i x-qubo-served-by; done
# Y en :7000 el server web1-8082 quedó DOWN (rojo).
# Reactivar:
vagrant ssh web1 -c "sudo pm2 start qubo-web1-8082"
```

**Decís:**
> "Apagué una réplica. El balanceador la saca de la rotación al instante — el header ya no la muestra — y en la GUI queda en rojo. Al reactivarla, HAProxy le hace su health check de nuevo y vuelve a repartirle tráfico."

**Parte B — matar TODO (página personalizada 503):**

```bash
vagrant ssh web1 -c "sudo pm2 stop all"
vagrant ssh web2 -c "sudo pm2 stop all"
curl -i http://192.168.56.30/health
```
Salida esperada:
```http
HTTP/1.1 503 Service Unavailable
X-Qubo-Served-By: NINGUNO (todos fuera de servicio)
... 503 - Qubo no disponible ...
```
Recuperar:
```bash
vagrant ssh web1 -c "sudo pm2 restart all"
vagrant ssh web2 -c "sudo pm2 restart all"
curl -s http://192.168.56.30/health
```

**Decís:**
> "Con todas las réplicas caídas, HAProxy ya no puede repartir y despliega la página personalizada que configuramos (errorfile 503). Fíjense en el detalle del header: 'NINGUNO', porque no había backends. Es la evidencia de que el balanceador sabe qué está pasando. Al reiniciar todo, el servicio vuelve solo."

**Checklist:** ☐ **P3.2** (ningún servidor disponible → página personalizada), refuerza RG1 y P1.4.

---

## ESCENA 8 — Pruebas de carga con Artillery: ejecución y resultados (4 min)

> ⚠ **Aclaración para el sustento:** Artillery es **CLI** (no tiene GUI de escritorio). La "UI" aquí son dos cosas: la **statística de HAProxy en :7000** (que se puede mostrar mientras corre la prueba, viendo subir conexiones por server) y el **reporte HTML** que Artillery genera al final (`artillery report`). En vivo mostrás: la consola compilando + el reporte comparativo.

**Paso 1 — mostrar los escenarios:**
```bash
cd Micro1_cloud/load-tests
cat baseline-sin-lb.yml
cat balanceado-6replicas.yml
```

**Decís:**
> "Dos escenarios con la **misma carga** (1650 usuarios virtuales en ~2,5 min: calentamiento 5 rps, prueba 15 rps, enfriamiento 5 rps). La diferencia es el destino: el baseline pega DIFECTO a una sola réplica (192.168.56.10:8080, sin balanceador); el balanceado pega a HAProxy que reparte entre las 6 réplicas. Así la única variable es el balanceo."

**Paso 2 — correr en vivo (o mostrar resultados ya medidos):**
```bash
# Desde el host (misma máquina donde está instalado artillery):
artillery run baseline-sin-lb.yml --output baseline-sin-lb.json
artillery run balanceado-6replicas.yml --output balanceado-6replicas.json

# Ver reporte comparativo legible:
artillery report baseline-sin-lb.json
# abre un HTML. Ídem con balanceado.
```

**Paso 3 — presentar la comparación (ya hecha con los datos del proyecto):**

| Métrica | **Baseline** (1 réplica, sin LB) | **Balanceado** (6 réplicas, LB) | **Lectura** |
|---|---|---|---|
| Respuestas HTTP 200 (éxitos) | **256** | **747** | **≈3× más** con solo balancear |
| Usuarios completados | 215 (13%) | 635 (38%) | 3× más pierden el proceso completo |
| Fallos (ETIMEDOUT: no se estableció conexión) | 1435 (86.9%) | 1015 (61.5%) | −25 puntos porcentuales |
| Latencia p50 | 1436 ms | 1978 ms | similar (misma cola de CPU) |
| Latencia p95 | 7865 ms | 8520 ms | similar |
| Latencia p99 | 9047 ms | 9607 ms | similar |
| HTTP 503 | 0 | 8 (pico) | HAProxy rechaza conexiones cuando satura (aplica el errorfile) |

**Decís (la conclusión):**
> "Bajo la misma carga, la réplica única se satura: el 87% de las conexiones ni se establecen (timeout). Con el balanceador de 6 réplicas, los **éxitos triplican**. La latencia no baja (p95/p99 similar) porque el cuello de botella es la CPU —el bcrypt del login es deliberadamente costoso— y las VMs son de 1 GB. Pero la capacidad de **atender** se multiplica por 3: ese es el valor del balanceo."

**Checklist:** ☐ **P3.3** (varios escenarios que caracterizan la respuesta ante distintas demandas), refuerza P3.1.

---

## ESCENA 9 — Cierre / evidencia de reproducibilidad (1 min)

**Mostrar:**
```bash
cd Micro1_cloud/vagrant
git log --oneline -3                 # commits del proyecto
```
o directamente decir: "El repo está en GitHub y subido a Classroom".

**Decís:**
> "Todo queda documentado en el repo `Micro1_cloud`: Vagrantfile, roles Ansible, escenarios y resultados de Artillery en `load-tests/`, y los documentos de sustentación por fase en `docs/`. La prueba final de reproducibilidad la re-verificamos con `vagrant destroy -f && vagrant up`."

**Checklist:** ☐ **S1** (demo en vivo completa), ☐ **S3** (entregado a GitHub/Classroom).

---

## ANEXO — Los "trucos del profesor" en un vistazo

| Si el profe pide... | Hacés... | Y se ve... |
|---|---|---|
| "Muéstreme que reparte" | `for i in $(seq 1 6); do curl -sI http://192.168.56.30/health \| grep -i x-qubo-served-by; done` | server-1..6 rotando |
| "Máteme un server y demuéstreme" | `vagrant ssh web1 -c "sudo pm2 stop qubo-web1-8082"` + el mismo loop | server-3 desaparece + rojo en :7000 |
| "Demuéstreme que ya no responde" | cortarle a la réplica y `curl http://192.168.56.10:8082/health` | timeout/falla |
| "Tráigame todo el sitio abajo" | `pm2 stop all` en web1 **y** web2 | `curl -i .../health` → **503** con página custom |
| "Deme otra réplica en caliente" | `replicas: 4` en ambos defaults + `vagrant provision web1 web2 lb` | en :7000 pasan de 6 a 8 servers |
| "Qué salud tiene el servicio" | `consul health` / `curl -s localhost:8500/v1/health/checks/qubo` (dentro de web1) | passing en web1 y web2 |
| "Pruebe carga y muéstreme resultados" | `artillery run ... --output ...json` + `artillery report ...` | tabla baseline 256 vs balanceado 747 |
| "¿Escaló de verdad?" | stats GUI `:7000` muestran más servers UP | evidencia visual directa |

---

## ANEXO B — Checklist global a marcar al final de la demo

**Requerimientos Generales**
- ☐ RG1 — peticiones solo al balanceador
- ☐ RG2 — Node en servers / HAProxy en balancer
- ☐ RG3 — GUI del balancer accesible desde el host
- ☐ RG4 — agente Consul en cada server web

**Pregunta 1 — Consul (1.5)**
- ☐ P1.1 clúster implementado ☐ P1.2 3 nodos vivos ☐ P1.3 catálogo con qubo ☐ P1.4 health checks passing

**Pregunta 2 — Aprovisionamiento (1.5)**
- ☐ P2.1 automático con Ansible ☐ P2.2 reproducible con `vagrant up` ☐ P2.3 Node+Qubo+Consul+HAProxy instalados

**Pregunta 3 — Disponibilidad/balanceo/Artillery (1.0)**
- ☐ P3.1 escalabilidad visible en stats
- ☐ P3.2 página 503 personalizada sin servidores
- ☐ P3.3 Artillery con varios escenarios + resultados

**Sustentación (1.0)**
- ☐ S1 demo en vivo ☐ S2 preguntas conceptuales ☐ S3 entregado en GitHub/Classroom