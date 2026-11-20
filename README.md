# FilaCero — PWA del paciente

Aplicación Web Progresiva para pre-registro, triaje **demostrativo**, emisión de turnos QR verificables y seguimiento de fila en tiempo real. Basada en el documento académico *Avance 1 – FilaCero*, Universidad Tecnológica del Sur de Sonora, 10 de noviembre de 2026.

> ⚠️ **SOLO PARA DEMOSTRACIÓN CON DATOS FICTICIOS.** No es un sistema médico ni una plataforma apta para conservar documentación real de pacientes. El triaje no está aprobado clínicamente.

## Inicio rápido (funciona sin instalar paquetes)

Necesitas **Node.js 22 o superior**.

```bash
cd FilaCero
npm start
```

Luego abre:

- **Paciente:** http://localhost:3000
- **Simulador técnico de recepción:** http://localhost:3000/#/admin
- **API de salud:** http://localhost:3000/api/health

El simulador técnico usa la clave de demostración `demo-fila-2026`. En este modo los datos están sólo en memoria y desaparecen al detener el servidor. Usa siempre datos ficticios.

### Flujo de prueba

1. Ve a **Pre-registro** e introduce nombre ficticio, NSS de prueba de 11 dígitos y una de las clínicas (el documento opcional admite PDF/JPG/PNG hasta 1 MB).
2. En **Cuestionario**, selecciona respuestas de prueba; si marcas dolor torácico agudo, verás la prioridad roja de la prueba UT-TRIAGE-01.
3. Continúa a **Mi turno** y presiona **Generar mi turno QR**.
4. Abre `/#/admin` en otra pestaña **del mismo navegador** (o en una sesión diferente) y usa la clave; cambia el estado a **Llamado a consultorio** y escribe `Consultorio 3`.
5. Regresa a la PWA: recibirás el cambio de estado por WebSocket, sin recarga. Si el teléfono lo permite, vibrará.
6. Para probar el QR offline, abre **Mi turno** en un navegador compatible después de cargar la PWA, luego desconéctalo de internet y recarga. Verás el último QR guardado (no habrá cambios de fila en vivo).

**Nota:** en localhost la cookie de sesión funciona; para pruebas desde un teléfono distinto al servidor utiliza un dominio **HTTPS**. Service Workers normalmente no funcionan en direcciones IP privadas con HTTP.

## Arquitectura y tecnologías

| Capa | Implementación |
|---|---|
| PWA paciente | HTML5 + CSS + JavaScript modular (responsive, instalable) |
| Offline | Service Worker + IndexedDB (sólo pase ya emitido) |
| QR | SVG local de QR versión 4-L; contenido firmado HMAC-SHA256 en el backend |
| REST | Node.js HTTP nativo, rutas `/api/*` |
| Tiempo real | WebSocket Gateway `/ws/queues` con eventos individuales autenticados |
| Datos | Repositorio en memoria (demo) o PostgreSQL 16 (`pg`) |
| Triaje | Servicio separado con reglas de prueba, pendiente de validación clínica |
| Calidad | `node:test`, integración REST/WS, GitHub Actions, guiones k6/JMeter |

Se eligió JavaScript modular para que el modo demo sea ejecutable **sin `npm install`**. El documento no impone frameworks o lenguaje; su arquitectura de componentes y tecnologías PWA, IndexedDB, WebSocket y PostgreSQL se respeta.

## Ejecutar con PostgreSQL

Para un ambiente de **pruebas con datos ficticios**:

1. `docker compose up -d` inicia PostgreSQL local (opcional si ya tienes una instancia).
2. Ejecuta `npm install` para instalar el cliente de PostgreSQL.
3. Copia `.env.example` como `.env`, cambia `DEMO_MODE=false` y configura:

   ```env
   PORT=3000
   DEMO_MODE=false
   DATABASE_URL=postgresql://filacero:filacero_local@localhost:5432/filacero
   SESSION_SECRET=una_clave_aleatoria_muy_larga_diferente_para_sesion
   QR_SECRET=otra_clave_aleatoria_muy_larga_diferente_para_qr
   ADMIN_DEMO_KEY=clave_de_pruebas_diferente_con_32_caracteres
   ```

4. `npm run db:migrate` crea las tablas.
5. `npm start` ejecuta la misma PWA y ahora persiste las entidades en PostgreSQL.

Asegúrate de que todos los secretos sean aleatorios y tengan **al menos 32 caracteres**; no compartas `.env` ni lo subas a Git. El modo PostgreSQL no hace que el sistema sea apto para producción clínica.

## Estructura de archivos

```text
FilaCero/
├── web/
│   ├── index.html, style.css, app.js     # pantallas y navegación
│   ├── qr.js                             # generador de QR SVG offline
│   ├── db.js, sw.js                      # IndexedDB, Service Worker
│   ├── manifest.webmanifest, icons/     # instalación de PWA
├── server/
│   ├── index.mjs                         # API, estáticos, sesiones, rate limits
│   ├── validation.mjs, triage.mjs        # validaciones y triaje de prueba
│   ├── auth.mjs, ws.mjs                  # tokens firmados y WebSocket
│   ├── repository.mjs, migrate.mjs      # persistencia y migración
├── db/schema.sql                        # PostgreSQL
├── tests/                               # 18 tests de unidades e integración
├── performance/                         # k6 y JMeter
├── docs/                                # arquitectura, pruebas, Scrum, seguridad
├── .github/workflows/ci.yml             # pruebas CI
├── docker-compose.yml
└── .env.example
```

## Pruebas

```bash
npm run check
npm test
```

La suite automatizada verifica validaciones, reglas de triaje, firma y lectura estructural del QR, emisión del turno, verificación en API y un evento WebSocket. Las metas del documento de **menos de 500 ms, FCP menor o igual a 1.5 s, API menor de 200 ms, 2000 req/s y SUS > 80** no se presentan como cumplidas sin las mediciones correspondientes.

## Documentación de trabajo

- [`docs/ARQUITECTURA.md`](docs/ARQUITECTURA.md): vista lógica, APIs, persistencia y decisiones.
- [`docs/PLAN_SCRUM.md`](docs/PLAN_SCRUM.md): 5 sprints, backlog, roles propuestos, ramas y versiones.
- [`docs/PRUEBAS.md`](docs/PRUEBAS.md): matriz del plan original y estado de cada caso.
- [`docs/PRIVACIDAD_Y_LIMITES.md`](docs/PRIVACIDAD_Y_LIMITES.md): riesgos, seguridad y pendientes de validación.

## Alcance importante

La **PWA paciente** es el producto principal de este repositorio. El archivo del proyecto especifica que administración de salas, escaneo de QR en ventanilla y cierre de atención competen a sistemas externos. Para permitir una demostración integral, se incluye un **simulador de recepción** `/#/admin`; no debe confundirse con el panel administrativo clínico terminado.

**Siguiente paso propuesto:** conectar una interfaz administrativa autenticada de la clínica y validar sus reglas de triaje, documentos y permisos antes de crear una versión `v1.0.0` de uso real.


## Practica de Entrega Continua

Se implemento un pipeline de Entrega Continua
utilizando GitHub y CircleCI.

El sistema automatiza:
- Instalacion de dependencias.
- Verificacion del codigo.
- Ejecucion de pruebas.
- Generacion de artefactos.

Herramienta utilizada: CircleCI.
