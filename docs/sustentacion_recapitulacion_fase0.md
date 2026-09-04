# Sustentación / Recapitulación — Fase 0 — Debug de Qubo + endpoints `/health` y `/whoami`

> Documento de sustentación individual de la **Fase 0** del microproyecto **Cluster Consul + HAProxy + Artillery**.
> Nivel: explicado **para bobos**, desde cero. Estructura: primero las herramientas y conceptos a fondo, luego el paso a paso con los comandos exactos.

---

## PARTE 1 — Teoría y herramientas a fondo

### 1. Node.js y npm

**¿Qué es Node.js?**
Node.js es un entorno de ejecución de JavaScript. Hasta hace unos años, JavaScript solo se ejecutaba dentro del navegador (Chrome, Firefox, etc.). Node.js sacó a JavaScript del navegador y lo dejó correr directamente en una computadora/server, igual que Python o Java. Esto permitió crear **servidores web en JavaScript** sin necesidad de un navegador.

**¿Qué es npm?**
npm (Node Package Manager) es el **gestor de paquetes** de Node. Un "paquete" es una biblioteca de código que alguien más escribió y publicó, lista para reutilizar. npm hace tres cosas básicas:
1. **Instalar** bibliotecas (ej. `npm install express`).
2. **Gestionar dependencias**: lleva un registro de todos los paquetes que tu proyecto necesita.
3. **Ejecutar scripts** definidos en el `package.json` (ej. `npm run dev`).

**¿Qué es `package.json`?**
Es el archivo de "identidad" de un proyecto Node. Contiene:
- El nombre del proyecto, versión, scripts (`start`, `dev`, etc.).
- La lista de **dependencias** (paquetes de producción) y **devDependencies** (paquetes solo para desarrollo).
- El campo `"type": "module"` que indica que el proyecto usa módulos modernos (ESM, `import`/`export`) en vez del estilo antiguo (`require`).

**¿Qué es `package-lock.json`?**
Es un archivo **generado automáticamente** por npm que "congela" las versiones EXACTAS de todos los paquetes instalados (incluyendo las dependencias de las dependencias). Sin este archivo, dos personas que corren `npm install` en momentos distintos podrían terminar con versiones ligeramente diferentes y que la app se comporte distinto. Con él, **el entorno es 100% reproducible**: todos instalan exactamente lo mismo.

> **Rol en esta fase:** lo usamos para levantar el backend de Qubo en local, verificar que las dependencias se instalan, y generar el `package-lock.json` (que en el repo estaba en `.gitignore`, es decir, no existía) para que el despliegue en las VMs sea reproducible.

---

### 2. Express y cómo corre un servidor web

**¿Qué es Express?**
Express es el framework de servidor web más popular de Node. "Framework" = un esqueleto que te da la estructura y herramientas para no escribir todo desde cero. Con Express creas **rutas** (endpoints) que responden a peticiones HTTP (GET, POST, etc.).

Ejemplo conceptual:
```js
const express = require('express');
const app = express();
app.get('/saludo', (req, res) => res.json({ mensaje: 'hola' }));
app.listen(8080);
```
Eso levanta un servidor en el puerto 8080; cuando alguien hace `GET /saludo`, recibe `{ mensaje: "hola" }`.

**¿Cómo corre el backend de Qubo?**
El punto de entrada es `Backend/src/index.js`. Los pasos son:
1. `dotenv.config()` → carga las variables del archivo `.env`.
2. Se crea la app Express.
3. `connectDB()` → conecta a MongoDB (ATLAS, cloud).
4. Se configuran CORS, cookie parser, body parsers.
5. Se montan las rutas bajo `/api`.
6. Health check en `GET /`.
7. `app.listen(PORT)` → **abre el puerto para recibir peticiones**.

**El bug de `NODE_ENV` (crucial):**
En el `src/index.js` original, el `app.listen(PORT)` estaba envuelto en algo como:
```js
if (process.env.NODE_ENV !== 'production') {
  app.listen(PORT, ...);
}
```
Es decir: **si el ambiente era "production", el servidor NO abría ningún puerto**. Eso tiene sentido para plataformas serverless (Vercel), donde el framework levanta el servidor por vos. PERO en nuestras VMs con Node corriendo normal, queremos que **SIEMPRE** abra el puerto. Lo corregimos para que el `app.listen` corra siempre.

> **Rol en esta fase:** entender este patrón fue clave para que la app pudiera arrancar en nuestras VMs como un proceso Node normal.

---

### 3. MongoDB Atlas (base de datos en la nube)

**¿Qué es MongoDB?**
MongoDB es una base de datos **NoSQL** orientada a documentos. En vez de tablas con filas y columnas (como SQL/PostgreSQL), guarda **documentos JSON** en colecciones. Qubo usa Mongoose como ODM (Object Document Mapper), que traduce entre objetos JavaScript y documentos de MongoDB.

**¿Qué es MongoDB Atlas?**
Es la versión **en la nube (SaaS)** de MongoDB. En vez de instalar y mantener una base de datos en un servidor propio, Mongo te da un clúster gestionado. Te conectás por internet con una cadena de conexión.

**¿Qué es la cadena `mongodb+srv://...`?**
```
mongodb+srv://juanplata:C6hUFffc8vHdwCWd@cluster0.kpaarhg.mongodb.net/?retryWrites=true&w=majority
```
Partes:
- `mongodb+srv://` → esquema que usa **DNS SRV records** para descubrir todas las direcciones del cluster automáticamente (útil para clusters con varias réplicas).
- `juanplata` → usuario.
- `C6hUFff...` → contraseña.
- `@cluster0.kpaarhg.mongodb.net` → nombre y ubicación del cluster.
- `?retryWrites=true&w=majority` → parámetros de conexión (reintento de escrituras, mayoría confirmada).

> **Rol en esta fase / decisión de arquitectura:** decidimos que la base de datos es **MongoDB Atlas en la nube** (el cluster oficial de Qubo, reactivado), y que **NO** habrá una BD dedicada dentro de una VM. Esto simplifica infraestructura y es defendible en sustentación: el requisito es "cada servidor web corre un solo proceso Node", no que cada uno lleve su BD.

**Nombre de base por nodo:** hicimos configurable `dbName` (antes estaba hardcoded como `'Qubo'`) vía variable `DB_NAME`, pero en la práctica usamos `qubo_cloud`.

---

### 4. El bug del DNS en Windows (el dolor de cabeza)

**¿Qué es el DNS?**
DNS (Domain Name System) es la "agenda telefónica" de internet: convierte nombres como `cluster0.kpaarhg.mongodb.net` en direcciones IP. Cuando Node quiere conectarse a Atlas, primero pregunta al DNS "¿qué IP tiene ese nombre?".

**¿Qué pasaba?**
El backend fallaba al conectar a MongoDB con `ECONNREFUSED` (conexión rechazada). La causa: en Windows, Node estaba usando como servidor DNS la dirección `192.168.80.1` (un adaptador de red virtual, de VirtualBox u otra herramienta). Ese DNS **no resolvía correctamente los registros SRV** de `.mongodb.net`, entonces la conexión nunca encontraba el destino y se rechazaba.

**El workaround (SOLO local, NO va a producción):**
Creamos un **precargador** temporal `dns-fix.cjs` que fuerza a Node a usar DNS públicos:
```js
const dns = require('dns');
dns.setServers(['8.8.8.8', '8.8.4.4']); // DNS de Google
```
Y lo pasamos a Node al arrancar. Esto era un truco de **debug en Windows**, no un cambio de código de producción. En las VMs Ubuntu no hará falta porque el DNS ahí funciona normal.

> **Rol en esta fase:** resolvió el bloqueo para poder probar la app en local y validar los endpoints antes de llevarla a las VMs.

---

### 5. Firebase graceful degradation

**¿Qué es Firebase?**
Firebase es un conjunto de servicios de Google (base de datos en tiempo real, autenticación, notificaciones push, etc.). Qubo usa:
- **Firebase Realtime Database** → solo para notificaciones.
- **Firebase Admin SDK** → para autenticación con Google (`verifyIdToken`), que requiere un archivo de service account (`firebase-service-account.json`).

**¿Qué es "graceful degradation"?**
Significa que una app **funciona aunque una parte opcional falle**, en lugar de romperse toda. En lugar de "si no hay Firebase, el servidor no arranca", el sistema degrada: sigue sirviendo lo esencial y simplemente desactiva las funciones que dependen de Firebase.

**Qué hicimos:**
- `config/firebase.js` → si no encuentra el service account, el objeto adminAuth queda en `null` (no rompe la importación).
- `controllers/AuthFirebase.js` → si `adminAuth` es `null`, responde 503 (no disponible) en vez de tirar un error.
- `Services/NotificationService.js` → si Firebase no está, las notificaciones se saltan pero **MongoDB sigue funcionando**.

> **Rol en esta fase:** garantiza que en las VMs la app arranque limpio aunque no haya service account de Firebase. Firebase solo es notificaciones (no es core), así que no debe bloquear el arranque.

---

### 6. Los endpoints `/health` y `/whoami`

**¿Por qué los necesitamos?**
En un cluster con balanceador y service discovery (Consul), el sistema necesita saber **qué servidores están vivos y sanos** para mandarle tráfico, y **cuál respondió** para demostrar que el balanceo funciona. Dos endpoints cubren esto:

**`GET /health`** → responde algo como:
```json
{ "node": "...", "instance": "...", "status": "ok", "db": "connected" }
```
Reglas:
- Si la conexión a MongoDB está activa (`mongoose.connection.readyState === 1`) → HTTP **200** y `status: "ok"`.
- Si NO → HTTP **503** y `status` indicando fallo.

Lo usan Consul (health check) y HAProxy (health check http) para saber si una réplica está sana.

**`GET /whoami`** → responde algo como:
```json
{ "node": "...", "instance": "...", "hostname": "...", "pid": ... }
```
Identifica **qué servidor procesó la petición** (vía variables `NODE_ID`/`INSTANCE_ID` del `.env`). Junto con el header `X-Qubo-Served-By` de HAProxy, demuestra visualmente que el balanceo reparte peticiones entre servidores — clave para la demostración.

---

### 7. Git y GitHub — ramas, commit y push

**¿Qué es Git?**
Git es un sistema de **control de versiones**: lleva historial de todos los cambios de tu código y permite trabajar en paralelo en distintas versiones.

**¿Qué es una rama (branch)?**
Una rama es una línea de trabajo independiente. La rama principal suele llamarse `main`. Creamos una rama separada para no tocar el código estable mientras trabajamos en cambios experimentales:
- Nuestra rama: `cloud/health-and-whoami`.

**Comandos clave:**
```bash
git status            # ver qué cambió
git add <archivos>    # "stage" = marcar cambios para el commit
git commit -m "msg"   # guardar los cambios en el historial
git push origin <rama> # subir la rama al repositorio remoto (GitHub)
git checkout -b <rama> # crear y cambiarte a una rama nueva
```

> **Rol en esta fase:** Qubo se edita en **repo aparte (rama)**, no embebido en este repo. Así mantenemos limpio el repo de infraestructura y Qubo aparte con sus cambios versionados.

---

### 8. La skill archify (diagrama de arquitectura)

**¿Qué es?**
archify es una skill (una herramienta de IA) que convierte una descripción JSON de una arquitectura en un **diagrama visual** (HTML/SVG) con reglas de calidad de layout (sin cruces de líneas, sin superposiciones, etc.).

**Qué hicimos:**
1. Definimos la arquitectura en `arquitectura.json` (componentes, conexiones, labels).
2. Validamos con el comando `validate` hasta pasar todas las reglas de composición (`ok: true`, `checksPassed: 9/9`).
3. Generamos el diagrama con `deliver` → `arquitectura.html`.

**El diagrama representa:**
- Host anfitrión (cliente + Artillery) → HAProxy (LB, :80 + :7000) → Qubo réplicas (Web1 + Web2) → Cluster Consul → MongoDB Atlas.
- Flujo izquierda→derecha sin cruces; Atlas en la parte inferior vía `mongodb+srv`.

> **Rol en esta fase / del proyecto:** es la representación visual que acompaña la sustentación conceptual del "todo".

---

## PARTE 2 — Paso a paso de la Fase 0 (con comandos)

### Paso 1 — Clonar y ubicar el repo de Qubo
Trabajamos sobre la copia local de Qubo en:
```
...\Micro1\estadoDelArte\Qubo
```
(Backend en `Backend/`, app React en `anuel/`).

### Paso 2 — Crear la rama de trabajo
```bash
git checkout -b cloud/health-and-whoami
```

### Paso 3 — Instalar dependencias y generar lock file
```bash
cd Backend
npm install
```
Esto instaló los paquetes (358) y **generó `package-lock.json`**.

### Paso 4 — Crear `.env` local y `.env.example`
Creamos un `.env` local con las variables necesarias (para probar en Windows) y un `.env.example` como plantilla:
```
DATABASE_URL=
JWT_SECRET=
JWT_EXPIRES_IN=
PORT=8080
NODE_ENV=development
NODE_ID=
INSTANCE_ID=
```

### Paso 5 — Arrancar con el fix de DNS (solo Windows)
```bash
node --require "C:\...\opencode\qubo-fix\dns-fix.cjs" src/index.js
```
Se observó: `Mongo connected succesfuly.`

### Paso 6 — Modificar código
Aplicamos los fixes:
1. **`src/index.js`** → agregar imports (`os`, `mongoose`), endpoints `/health` y `/whoami`, y hacer que `app.listen` corra siempre.
2. **`config/db.js`** → `dbName` configurable (`process.env.DB_NAME || 'Qubo'`) y quitar el `process.exit(1)`.
3. **`config/firebase.js`** → degradación gracegful (adminAuth → null si falta service account).
4. **`controllers/AuthFirebase.js`** → 503 si adminAuth es null.
5. **`Services/NotificationService.js`** → saltar Firebase, mantener Mongo.
6. **`controllers/AuthController.js`** → corregir doble conexión a Mongo (side effect del import).

### Paso 7 — Verificar endpoints
- `GET /health` → HTTP 200 `{ status: "ok", db: "connected" }`.
- `GET /whoami` → datos del nodo/instancia.
- Registro (201) y login (200 + JWT) funcionando contra Atlas.

### Paso 8 — Commit y push de la rama
```bash
git add Backend/Services/NotificationService.js Backend/config/db.js Backend/config/firebase.js Backend/controllers/AuthController.js Backend/controllers/AuthFirebase.js Backend/src/index.js Backend/.env.example
git commit -m "feat(cloud): endpoints /health y /whoami + degraded graceful de Firebase"
git push -u origin cloud/health-and-whoami
```

### Paso 9 — Generar diagrama con archify
```bash
# (desde la carpeta de la skill archify)
node bin/archify.mjs validate architecture "...\load_balancer\docs\arquitectura.json" --quality showcase --json
node bin/archify.mjs deliver architecture "...\load_balancer\docs\arquitectura.json" "...\load_balancer\docs\arquitectura.html" --quality showcase
```
Resultado: validación `ok: true` (9/9 checks) y diagrama `arquitectura.html` entregado.

---

## PARTE 3 — Decisiones tomadas y alternativas descartadas

| Decisión | Por qué | Alternativa descartada |
|----------|---------|------------------------|
| MongoDB Atlas (cloud), sin BD en VM | Simplifica infra, requisito es "1 proceso Node por servidor" | BD MongoDB local en VM |
| Rama `cloud/health-and-whoami` en repo aparte | No embarullar el repo de infra; versionar los cambios de Qubo | Embeber Qubo en el repo de infra |
| Degradación graceful de Firebase | No bloquear arranque por una parte no-core | Exigir service account en cada VM |
| DNS fix fíjico (workaround) | Desbloquea el debug en Windows | Cambiar código de producción |
| Diagrama archify | Documentación visual para sustentación | Solo diagrama en texto |

---

**Fin de la Fase 0.** Aprobado por Juan ✅ — lista para la Fase 1.
