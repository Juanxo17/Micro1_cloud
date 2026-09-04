# Sustentación / Recapitulación — Fase 1 — Cluster Consul

> Documento de sustentación individual de la **Fase 1** del microproyecto **Cluster Consul + HAProxy + Artillery**.
> Nivel: explicado **para bobos**, desde cero. Estructura: primero las herramientas y conceptos a fondo, luego el paso a paso con los comandos exactos.

---

## PARTE 1 — Teoría y herramientas a fondo

### 1. Qué es un "service mesh" y por qué importa

Un **microservicio** es una app partida en servicios pequeños que corren por separado y se comunican por red. Cuando tenés varios servidores iguales (réplicas de Qubo), aparece un problema: **¿cómo sabe el sistema qué réplicas existen, cuáles están vivas y cuáles sanas?** Eso es el *service discovery* (descubrimiento de servicios).

Un **service mesh** es una capa de infraestructura que resuelve eso de forma centralizada: registra servicios, hace health checks, y entrega esa info a quien la necesite (balanceadores, etc.). En este proyecto, **Consul** es el service mesh / service discovery.

**Rol en esta fase:** montamos el service discovery que conectará luego con HAProxy (Fase 3). Sin esto, el balanceador no sabría a qué servidores mandar tráfico.

---

### 2. HashiCorp Consul

**¿Qué es?**
Consul (de HashiCorp) es una herramienta de **service discovery y configuración** distribuida. Guarda un **catálogo** de servicios (lista de qué servicios hay, en qué IP/puerto, y su estado de salud) y permite consultarlo.

**Conceptos clave de Consul:**

| Concepto | Qué es |
|----------|--------|
| **Datacenter** | Agrupación lógica de nodos (nosotros: `dc1`) |
| **Agent** | El proceso de Consul corriendo en cada máquina |
| **Server / Client** | Dos modos de agente (ver abajo) |
| **Catálogo** | La base de datos central de servicios disponibles |
| **Gossip protocol** | Protocolo de comunicación entre agentes (se "susurran" estado entre sí) |
| **Raft** | Algoritmo de consenso usado por los servers para elegir líder |
| **Quórum** | Cantidad mínima de servers que deben estar de acuerdo para que el clúster funcione |
| **Líder** | El server que coordina el clúster en un momento dado |

**Modos de agente:**
- **Server:** almacena el catálogo y participa en el **quórum** (toma decisiones). Es el "cerebro" del clúster.
- **Client:** consulta el catálogo (para saber a dónde mandar tráfico) pero **no participa en el quórum** ni almacena de forma autoritativa.

**Topología de nuestro proyecto:**
- `web1` y `web2` → agentes **server** (forman el clúster, 2 servers).
- `lb` → agente **client** (consulta el catálogo, no participa en decisiones).

**Justificación sustentable:** 2 servers + 1 client es la topología mínima correcta de un service mesh con service discovery distribuido. No hay quórum perfecto (ideal serían 3 servers), pero es correcto y defendible para el alcance académico de 2 nodos: lo importante es demostrar que los agentes **forman clúster y se ven entre sí** (`consul members`).

**Archivo de configuración HCL:**
Consul se configura con archivos **HCL** (HashiCorp Configuration Language), un formato declarativo. Ej. `server.hcl` / `client.hcl`.

**Comandos principales de Consul (CLI):**
```bash
consul members                    # lista los miembros del clúster y su estado
consul operator raft list-peers   # muestra el líder y los votantes del quórum
consul catalog services           # lista los servicios registrados
consul catalog nodes -service=X   # en qué nodos está el servicio X
```

**Consul UI:**
Consul trae una interfaz web integrada (`ui = true`). Se accede al puerto 8500 de cada agente; con port-forward de Vagrant se abre desde el host.

**Rol en esta fase:** instalar/configurar los agentes, formar el clúster de 2 servers + 1 client, y registrar el servicio Qubo.

---

### 3. El Puerto 8500 y las IPs del clúster

- `8500` → API HTTP + UI de Consul en cada agente.
- `8300` → Raft (comunicación server↔server para el consenso).
- `8301` → gossip LAN (los miembros se comunican entre sí por acá).

**IPs del clúster (red host-only de Vagrant):**
| VM | IP host-only | Rol |
|----|--------------|-----|
| web1 | 192.168.56.10 | server |
| web2 | 192.168.56.20 | server |
| lb   | 192.168.56.30 | client |

**Detalle importante que descubrimos:** Consul debe bindearse a la IP de la **red host-only** (192.168.56.x), NO a la IP NAT interna (10.0.2.x). Los agentes se comunican y se unen por la host-only; por eso el `retry_join` apunta a las 192.168.56.x.

---

### 4. `bootstrap_expect` y el quórum

`bootstrap_expect = 2` le dice a Consul: "el clúster debe tener 2 servers para considerarse formado". Con eso, los servers esperan a alcanzar 2 miembros y **recién entonces eligen líder** (Raft). Sin esto, cada server intentaría ser líder por su cuenta (split-brain).

**Rol en esta fase:** garantiza que los 2 servers formen quórum y elijan UN solo líder (vimos que web2 quedó como líder y web1 como follower).

---

### 5. `retry_join`

`retry_join` le dice al agente: "intentá unirte a estas direcciones, y si falla, reintentá automáticamente". En un clúster de 2 servers, cada uno hace `retry_join` hacia el otro:
- web1 → `retry_join = ["192.168.56.20"]`
- web2 → `retry_join = ["192.168.56.10"]`
- lb → `retry_join = ["192.168.56.10", "192.168.56.20"]` (client se une a ambos servers)

**Rol en esta fase:** permite que el clúster se arme solo y se autorepare si un nodo se cae.

---

### 6. El registro de un servicio y su health check

En `qubo.json` definimos un servicio y su chequeo de salud:
```json
{
  "service": {
    "name": "qubo",
    "tags": ["web", "nodejs"],
    "port": 8080,
    "check": {
      "id": "qubo-health",
      "name": "Qubo health check",
      "http": "http://192.168.56.10:8080/health",
      "interval": "5s",
      "timeout": "3s"
    }
  }
}
```
- `name: "qubo"` → cómo se llama el servicio en el catálogo.
- `port: 8080` → puerto donde Qubo escucha.
- `check.http` → Consul hace un GET HTTP a `/health` cada `5s` para saber si el servidor está sano.

**Qué vimos:** el servicio `qubo` quedó registrado en web1 y web2 (`consul catalog services` → `qubo`). Su health check aparece "critical" porque **Qubo aún no corre** (eso se resuelve en la fase de Qubo). No es un error: es Consul reportando que no puede verificar la salud de un servicio que todavía no está desplegado.

**Rol en esta fase:** deja el servicio Qubo en el catálogo con su health check, listo para que HAProxy lo use después.

---

### 7. Ansible y los Roles (repaso aplicado a esta fase)

**¿Qué es Ansible?** Herramienta de **aprovisionamiento y configuración** (IaaC, infraestructura como código): describe el estado deseado de las máquinas y lo aplica de forma **idempotente** (re-ejecutar no rompe nada).

**¿Qué es un Rol?** Una unidad reutilizable de configuración: tareas, templates, handlers, defaults. Estructura típica:
```
roles/<nombre>/
├── tasks/main.yml     # qué hace el rol
├── handlers/main.yml  # acciones gatillo (ej. reiniciar servicio)
└── templates/*.j2     # archivos de config con plantillas
```

**Nuestros roles en esta fase:**
- `consul-server` → aplica a web1, web2 (configura agente server + registra qubo).
- `consul-client` → aplica a lb (configura agente client).

**Templates (`.j2`):** archivos Jinja2 que Ansible rellena con variables por máquina. Ej. el `server.hcl.j2` escribe `node_name = "web1"` o `"web2"` según la VM.

**Rol en esta fase:** automatizó la instalación y configuración idéntica de Consul en las 3 VMs de forma reproducible.

---

### 8. Vagrant y las VMs (repaso aplicado)

**¿Qué es Vagrant?** Herramienta para definir y levantar máquinas virtuales desde código. El `Vagrantfile` describe las VMs. Comandos:
```bash
vagrant up         # crea y provisiona las VMs
vagrant provision  # re-ejecuta el aprovisionamiento SIN recrear
vagrant ssh <vm>   # entrar por SSH a una VM
vagrant status     # estado de las VMs
vagrant destroy -f # borrar las VMs
```

**Provisioner `ansible_local`:** Vagrant copia el playbook a cada VM y corre Ansible **dentro** de ella (en vez de depender de Ansible instalado en el host Windows). Reproducible con `vagrant up` desde cero.

**Rol en esta fase:** dar de alta y aprovisionar las 3 VMs que corren el clúster.

---

## PARTE 2 — Paso a paso de la Fase 1 (con comandos)

> En esta fase se construyó la base de infraestructura (Vagrant + Ansible) y sobre ella el clúster Consul (opción B: solo el rol Consul).

### Paso 1 — Crear la estructura de archivos
Crear `load_balancer/vagrant/` con:
```
vagrant/
├── Vagrantfile
└── ansible/
    ├── site.yml
    └── roles/
        ├── consul-server/{tasks,handlers,templates}
        └── consul-client/{tasks,handlers,templates}
```

### Paso 2 — Escribir el Vagrantfile (3 VMs)
Definimos 3 VMs con `bento/ubuntu-20.04`, 1024MB/2CPU, IPs host-only, y provisioner `ansible_local`:
```ruby
config.vm.box = "bento/ubuntu-20.04"
# web1 -> 192.168.56.10, web2 -> 192.168.56.20, lb -> 192.168.56.30
# vb.customize modifyvm --natdnshostresolver1 on / --natdnsproxy1 on
# provision "ansible_local" playbook "ansible/site.yml"
```

### Paso 3 — Escribir `site.yml`
```yaml
- hosts: all
  become: true
  roles:
    - role: consul-server
      when: ansible_hostname == "web1" or ansible_hostname == "web2"
    - role: consul-client
      when: ansible_hostname == "lb"
```
(Selecciona el rol según el hostname real de cada VM.)

### Paso 4 — Escribir los roles Consul
El rol `consul-server` instala Consul (unzip de `releases.hashicorp.com`), crea usuario/directorios, despliega `server.hcl.j2`, el unit systemd, y `qubo.json.j2`. El `consul-client` hace lo mismo pero con `client.hcl.j2` y sin el servicio qubo.

### Paso 5 — Levantar las VMs
```bash
cd load_balancer/vagrant
vagrant up
```
Descarga la box `bento/ubuntu-20.04` (la primera vez tarda), crea las VMs y ejecuta Ansible en cada una.

### Paso 6 — Problemas resueltos sobre la marcha

| Problema | Causa | Solución |
|----------|-------|----------|
| Fallo de DNS al instalar | El resolver de la VM no resolvía `launchpad.net` | `--natdnsproxy1 on` + fijar `/etc/resolv.conf` con `8.8.8.8` |
| PPA de Ansible no existe en 20.04 | `ppa:ansible/ansible` está deprecado | Instalar Ansible vía `apt` en un shell provisioner previo |
| pip no instala en Python 3.8 | bootstrap moderno exige ≥3.10 | Instalar Ansible por `apt` (evita el bootstrap de pip) |
| `invalid config key config_dir` | `config_dir` es flag de CLI, no key de config HCL | Quitar la línea del template |
| Servidores bindeados a IP NAT (10.0.2.x) | `ansible_default_ipv4` devolvía la NAT | Fijar `bind_addr` explícito por hostname (192.168.56.x) |
| Consul no tomaba la config nueva | El proceso no se reiniciaba | `sudo systemctl restart consul` |

### Paso 7 — Validar el clúster
Desde cada VM (o desde el host con `vagrant ssh`):
```bash
consul members
```
Resultado esperado:
```
Node  Address             Status  Type    Build   Protocol  DC   Partition  Segment
web1  192.168.56.10:8301  alive   server  1.18.1  2         dc1  default    <all>
web2  192.168.56.20:8301  alive   server  1.18.1  2         dc1  default    <all>
lb    192.168.56.30:8301  alive   client  1.18.1  2         dc1  default    <default>
```

### Paso 8 — Verificar el líder (quórum)
```bash
consul operator raft list-peers
```
Resultado esperado: web2 `leader`, web1 `follower`, ambos `Voter`.

### Paso 9 — Verificar el catálogo y el servicio Qubo
```bash
consul catalog services
consul catalog nodes -service=qubo
```
Resultado: servicios `consul` y `qubo`; el `qubo` presente en web1 y web2.

### Paso 10 — Revisar la UI de Consul
En el navegador del host:
- `http://localhost:18500` → UI de web1
- `http://localhost:28500` → UI de web2
Se ven los 3 nodos, el estado del clúster y el servicio qubo.

---

## PARTE 3 — Decisiones tomadas y alternativas descartadas

| Decisión | Por qué | Alternativa descartada |
|----------|---------|------------------------|
| 2 servers + 1 client (Consul) | Topología mínima correcta demostrable con `consul members` | 1 solo server (sin quórum real) |
| `bento/ubuntu-20.04` | Misma box usada en FinalTelematicos, resolver DNS ya conocido | `ubuntu-22.04` (nos generó problema de DNS/pip) |
| `ansible_local` | evita instalar Ansible en Windows; reproducible con vagrant up | Ansible remoto desde host |
| bind a IP host-only explícita | El clúster se comunica por la red privada y matchea el retry_join | `ansible_default_ipv4` (mia la NAT) |
| Registrar servicio `qubo` ya (con health check) | Deja listo el service discovery para HAProxy | Registrarlo más tarde |

---

**Fin de la Fase 1 — Cluster Consul.** Aprobado: clúster de 3 nodos sano, líder elegido, servicio qubo en catálogo ✅
