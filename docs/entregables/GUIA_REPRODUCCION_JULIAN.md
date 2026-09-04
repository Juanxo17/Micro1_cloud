# GUÍA DE REPRODUCCIÓN — Levantar el proyecto completo desde cero (para Julian)

> **Objetivo:** reconstruir ÍNTEGRamente el microproyecto 1 (Cluster Consul + HAProxy + Artillery) en un equipo que no lo ha tocado nunca. Al terminar esta guía tenés 3 VMs corriendo, Qubo respondiendo detrás del balanceador, un clúster Consul de 3 nodos y las pruebas de carga documentadas.
>
> **Tiempo estimado:** 30–45 min la primera vez (descargas de box + paquetes). Lo demás es copy-paste.

---

## 1. Qué vas a obtener al final

| Recurso | Dónde queda | Para qué sirve |
|---------|-------------|----------------|
| web1 — 192.168.56.10 | Linux VM | Consul **server** + Qubo (3 réplicas: 8080, 8081, 8082) |
| web2 — 192.168.56.20 | Linux VM | Consul **server** + Qubo (3 réplicas: 8080, 8081, 8082) |
| lb — 192.168.56.30 | Linux VM | Consul **client** + **HAProxy** (balanceador + stats GUI) |

Todo el tráfico entra por **HAProxy** (`192.168.56.30:80`). Nada entra directo a los servidores web.

---

## 2. Requisitos previos (en tu máquina Windows)

Instalado y funcional:

- **VirtualBox** (7.x para arriba) → https://www.virtualbox.org/
- **Vagrant** (2.4 o superior) → https://developer.hashicorp.com/vagrant/install
- **Git** → https://git-scm.com/
- Un equipo con al menos **8 GB de RAM libre** (3 VMs × 1 GB + host) y **~25 GB de disco** (VMs + caja base).

Verificá en PowerShell/CMD:
```bash
virtualbox --version   # algo como 7.0.x
vagrant --version      # algo como 2.4.x
git --version
```

> ⚠️ **Golpe de la experiencia (truco importante):** si abrís un `vagrant up` y al arrancar la VM la **ventana de VirtualBox no queda visible al frente**, el arranque da timeout. Dejá abierta la ventana de cada máquina mientras Vagrant la levanta.

---

## 3. Clonar los dos repositorios

```bash
# 1) El proyecto de infraestructura (Vagrantfile + Ansible + pruebas)
git clone https://github.com/Juanxo17/Micro1_cloud.git

# 2) La aplicación Qubo (backend Node.js + frontend React)
git clone https://github.com/Juanxo17/Qubo.git
```

> En `Qubo` hay que usar la rama `cloud/health-and-whoami` (contiene el health check mejorado y el frontend apuntando al balanceador):
> ```bash
> cd Qubo
> git checkout cloud/health-and-whoami
> ```

---

## 4. El ÚNICO cambio manual requerido

El archivo `Micro1_cloud/vagrant/Vagrantfile` está hecho **hardcodeando la ruta del backend de Qubo** tal y como existe en la máquina que lo desarrolló. En tu equipo cambia la línea ~26:

```ruby
# ANTES (ruta de la máquina original):
qubo_backend_host = "C:/Users/perxa/Desktop/.../estadoDelArte/Qubo/Backend"

# DESPUÉS (tu ruta local al backend):
qubo_backend_host = "C:/TU/RUTA/A/Qubo/Backend"
```

**Reglas de esa ruta:**
- Usá **barras normales `/`** (no `\`) o esa línea rompe.
- Apuntá a la carpeta `Backend` de Qubo (la que tiene `package.json`).
- **NO debe existir `node_modules` en esa carpeta** del host: las dependencias se instalan dentro de cada VM (es deliberado, ver §6). Si hay `node_modules`, borralo antes:
  ```bash
  cd Qubo/Backend
  rmdir /s node_modules
  ```

---

## 5. Configuración de la base de datos (MongoDB Atlas)

Ninguna. En serio, ninguna:

- La app usa una BD en la nube (**MongoDB Atlas**).
- Las credenciales ya están dentro del rol de Ansible (`roles/qubo-app/defaults/main.yml`); Ansible genera el `.env` de cada VM automáticamente en `/srv/qubo/src/.env`.
- La whitelist de IPs de Atlas ya está **abierta** (`0.0.0.0`), así que conecta desde cualquier equipo.
- Si algún día conecta y Atlas rechaza la IP, en la consola de Atlas → Network Access → permitir `0.0.0.0/0`.

---

## 6. Levantar todo de una sola tanda

```bash
cd Micro1_cloud/vagrant
vagrant up
```

Qué va a pasar (automático, sin intervención):

1. Descarga la caja base **bento/ubuntu-20.04** (la primera vez, ~10 min).
2. Levanta **web1**, **web2** y **lb** con sus IPs privadas.
3. En cada web VM: prueba una shell que fija DNS e instala Ansible, y después corre el playbook `ansible/site.yml` que:
   - Instala **Consul** (modo server) + registra el servicio `qubo`.
   - Instala **Node.js 20** + **PM2**, copia el backend vía `rsync` a `/srv/qubo`, instala dependencias DENTRO de la VM, genera `.env`, y lanza **3 réplicas** (8080/8081/8082).
4. En `lb`: instala **Consul** (modo client) + **HAProxy** con su configuración (roundrobin, stats en :7000, error 503 personalizado).

> ⏳ Los `npm install` de las 3 réplicas por VM son lo más lento (~5 min por VM).

---

## 7. Si algo falla a mitad de camino

Nunca hace falta borrar todo. Con las máquinas creadas, **re-ejecutar solo el aprovisionamiento** (idempotente: no rompe lo que ya está):

```bash
cd Micro1_cloud/vagrant
vagrant provision          # re-aplica Ansible en las 3 VMs
# o una sola:
vagrant provision web1
```

Problemas conocidos y su solución:

| Síntoma | Causa | Solución |
|---------|-------|----------|
| `Failed to mount folders` al arrancar | warning temporal de Guest Additions | reintentar; en la práctica desaparece solo |
| Timeout al arrancar | ventana de VirtualBox no visible | dejar la ventana al frente durante el boot |
| DNS/apt falla dentro de la VM | resolver del host Windows | ya está parcheado por el provisioner (Fija 8.8.8.8) |
| La app corre pero `db: disconnected` | Atlas rechaza la IP | abrir whitelist `0.0.0.0/0` en Atlas |

---

## 8. Verificación rápida de que TODO funciona

Desde tu máquina (host), sin entrar a las VMs:

```bash
# 1. Las 3 VMs deben figurar running
vagrant status

# 2. El balanceador responde y rota entre las 6 réplicas (web1 y web2 ×3)
curl -s http://192.168.56.30/health

# 3. Ver qué servidor atendió cada petición (debe alternar server-1..6)
for i in $(seq 1 6); do curl -sI http://192.168.56.30/health | grep -i x-qubo-served-by; done

# 4. Stats GUI del balanceador (ver los 6 servers en UP / color verde)
#    abrir en el navegador: http://localhost:7000

# 5. UI de Consul (ver los 3 nodos y el servicio qubo)
#    abrir en el navegador: http://localhost:18500  (web1)
#    abrir en el navegador: http://localhost:28500  (web2)
```

Resultado esperado de #3:
```
x-qubo-served-by: server-1
x-qubo-served-by: server-2
x-qubo-served-by: server-3
x-qubo-served-by: server-4
x-qubo-served-by: server-5
x-qubo-served-by: server-6
```

---

## 9. (Opcional) Correr el frontend de Qubo contra el balanceador

Si querés la experiencia completa (la app de usuario real pegándole al balanceador):

```bash
cd Qubo/anuel
npm install        # la primera vez, tarda 2-4 min
npm run dev
# abrir http://localhost:5173
```

El frontend ya está configurado para hablar con `http://192.168.56.30:80/api` (el balanceador). Registrate, creá perfil, publicá — todo ese tráfico pasa por HAProxy y se reparte.

---

## 10. Copia de seguridad: si algún día hay que regenerar TODO

```bash
cd Micro1_cloud/vagrant
vagrant destroy -f    # apaga y borra las 3 VMs
vagrant up            # las recrea y provisiona solas
```

Esto es **la prueba de que todo es reproducible**: no hay un solo paso manual en todo el proceso (fuera de cambiar la ruta del backend en el Vagrantfile).

---

## 11. Referencias del proyecto

| Archivo | Qué contiene |
|---------|--------------|
| `Micro1_cloud/vagrant/Vagrantfile` | Definición de las 3 VMs, IPs, puertos reenviados, provisioners |
| `Micro1_cloud/vagrant/ansible/site.yml` | Qué roles va a cada VM según su hostname |
| `Micro1_cloud/vagrant/ansible/roles/consul-server/` | Consul modo server (web1/web2) |
| `Micro1_cloud/vagrant/ansible/roles/consul-client/` | Consul modo client (lb) |
| `Micro1_cloud/vagrant/ansible/roles/qubo-app/` | Instala Node, PM2, copia backend, réplicas |
| `Micro1_cloud/vagrant/ansible/roles/haproxy/` | HAProxy: roundrobin, stats, error 503 |
| `Micro1_cloud/load-tests/` | Escenarios Artillery + resultados JSON + informe de pruebas |
| `Micro1_cloud/docs/` | Constitution, plan de arquitectura, sustentaciones por fase |