# Café Aurora · Plataforma de Eventos e Inscripciones

Proyecto de **Backend II** de CoderHouse: una API REST con Express organizada por capas y
persistencia en MongoDB, que crece entrega a entrega hacia una plataforma completa de eventos
con autenticación, roles, inscripciones y control de cupos.

**Pre-entrega 4:** autenticación centralizada con **Passport.js**. El registro, el login y la
consulta del usuario actual pasan ahora por estrategias de Passport (`register`, `login` y
`current`) definidas en `src/config/passport.config.js`. El contrato externo de la API es el
mismo de la entrega anterior: rutas, códigos, mensajes y cookie `currentUser` no cambian; lo que
mejora es la organización interna, y el sistema queda preparado para sumar providers externos
(Google, GitHub) sin tocar `app.js`.

## Temática elegida

**Café Aurora** es una cafetería de especialidad (marca que acompaña todos mis proyectos de la
carrera Fullstack) que organiza eventos para su comunidad: catas de café, talleres de barismo,
charlas con productores y noches de música en vivo. La plataforma permitirá publicar esos eventos
y gestionar las inscripciones de los clientes.

## Tecnologías

| Tecnología | Uso |
|------------|-----|
| Node.js (>= 20.19) | Entorno de ejecución, módulos ESM |
| Express 5 | Servidor HTTP y enrutamiento |
| MongoDB + Mongoose | Persistencia y modelos (`User`, `Event`) |
| Passport (`passport`, `passport-local`, `passport-jwt`) | Estrategias de autenticación `register`, `login` y `current` |
| bcrypt | Hash y comparación de contraseñas |
| jsonwebtoken | Firma del JWT (la verificación la hace `passport-jwt`) |
| cookie-parser | Lectura de la cookie de sesión |
| dotenv | Variables de entorno |
| node:test + supertest + mongodb-memory-server | Tests de integración con base en memoria |
| pnpm | Gestor de paquetes |

## Instalación

```bash
git clone https://github.com/mcornejov/coderhouse-backend2-preentrega4.git
cd coderhouse-backend2-preentrega4
pnpm install
```

> Si no usas pnpm, `npm install` también funciona.

## Configuración de variables de entorno

Copia el archivo de ejemplo y ajusta los valores:

```bash
cp .env.example .env
```

| Variable | Descripción | Obligatoria |
|----------|-------------|-------------|
| `PORT` | Puerto donde escucha el servidor | Sí |
| `NODE_ENV` | Entorno: `development`, `production` o `test`. En `production` la cookie se marca `secure` | Sí |
| `MONGO_URL` | Cadena de conexión a MongoDB (Atlas o local). Sin ella el servidor no arranca | Sí |
| `JWT_SECRET` | Secreto con el que se firman (controller) y verifican (estrategia `current`) los JWT. Usa un valor largo y aleatorio | Sí |
| `JWT_EXPIRES_IN` | Expiración del JWT: número de segundos (`3600`) o duración con unidad (`15m`, `1h`, `7d`). Por defecto `1h` | No |
| `BCRYPT_SALT_ROUNDS` | Costo del hash de bcrypt, entre 10 y 14 (por defecto 10) | No |

El servidor valida la configuración al arrancar (Fail-Fast): si falta una variable obligatoria o
`PORT`/`JWT_EXPIRES_IN` tienen un valor inválido, se detiene con un mensaje claro en vez de
servir una API rota. El secreto **nunca está en el código**: solo se lee de `process.env`.

El archivo `.env` está excluido del repositorio mediante `.gitignore`; nunca se versionan
credenciales.

## Cómo ejecutar

```bash
pnpm start      # producción: node src/server.js
pnpm dev        # desarrollo: reinicia automáticamente al guardar cambios
pnpm test       # suite de tests (usa MongoDB en memoria, no necesita base instalada)
```

Con la configuración por defecto el servidor queda disponible en `http://localhost:8080`.

## Estrategias de Passport

Todas las estrategias viven en `src/config/passport.config.js`. `app.js` solo las registra con
`configurarPassport()` e inicializa Passport con `passport.initialize()`; no usa
`passport.session()` porque la sesión viaja en el JWT, no en el servidor.

| Estrategia | Tipo | Qué hace | Entrega en `req.user` | Si falla |
|------------|------|----------|------------------------|----------|
| `register` | `passport-local` (`usernameField: 'email'`, `passReqToCallback`) | Valida campos obligatorios, formato de email y largo de contraseña; normaliza el email; verifica unicidad; hashea con bcrypt; crea el usuario con rol `user` (el rol del body se ignora) | Usuario creado sin contraseña | `400` / `409` según la regla incumplida |
| `login` | `passport-local` | Busca el usuario por email normalizado y compara la contraseña con bcrypt (contra un hash señuelo si el email no existe). **No firma el JWT ni setea la cookie** | `{ id, email, role }` | `401` `Credenciales inválidas` (genérico) · `400` si falta email o password |
| `current` | `passport-jwt` | Extrae el JWT de la cookie `currentUser` (o del header `Authorization: Bearer`), verifica firma (`HS256`, `JWT_SECRET`) y vigencia | `{ id, email, role }` del payload | `401` `No autenticado` |

Las rutas delegan en `passport.authenticate(...)` a través de `autenticarCon(estrategia)`
(`src/middlewares/auth.middleware.js`), un envoltorio que convierte los fallos de Passport al
formato JSON de la API en vez del `Unauthorized` en texto plano por defecto. Tras una estrategia
exitosa, el **controller** es quien responde: en el login firma el JWT (`utils/jwt.js`) y lo
guarda en la cookie; en `current` devuelve `req.user`; el logout no pasa por Passport.

### Preparado para providers externos

`passport.config.js` construye las estrategias en una sola función (`crearEstrategias`) que
devuelve un objeto `{ nombre: estrategia }` y las registra en un bucle. Para agregar Google o
GitHub basta con instalar la estrategia correspondiente (`passport-google-oauth20`,
`passport-github2`), sumar una entrada más a ese objeto con su `clientID`/`clientSecret` leídos de
`.env`, y crear las dos rutas del flujo OAuth (`/api/sessions/google` y su `callback`). Ni
`app.js`, ni las rutas existentes, ni el controller cambian: la nueva estrategia entrega el
usuario en `req.user` y el controller emite la misma cookie `currentUser` de siempre.

## Rutas disponibles

| Método | Ruta | Descripción | Estrategia | Respuesta |
|--------|------|-------------|------------|-----------|
| GET | `/api/health` | Estado del servidor | — | `200` `{ "status": "ok", "message": "Servidor activo" }` |
| GET | `/api/events` | Lista de eventos (vacía por ahora) | — | `200` `{ "status": "success", "payload": [] }` |
| GET | `/api/events/:eid` | Detalle de un evento | — | `200` con el evento, o `404` si no existe |
| POST | `/api/sessions/register` | Registro de usuario | `register` | `201` con el usuario creado · `400` · `409` |
| POST | `/api/sessions/login` | Inicio de sesión: emite el JWT en la cookie `currentUser` | `login` | `200` `Login correcto` · `400` · `401` |
| GET | `/api/sessions/current` | Usuario autenticado (lee la cookie) | `current` | `200` `{ id, email, role }` · `401` |
| POST | `/api/sessions/logout` | Cierre de sesión: elimina la cookie | — | `200` `Sesión cerrada` |

Formato de respuesta: éxito `{ "status": "success", "payload": ... }` o
`{ "status": "success", "message": "..." }`; error `{ "status": "error", "message": "..." }`.
Es el mismo contrato de la Pre-entrega 3.

A continuación, el detalle de cada ruta de sesión con ejemplos de request y response.

### `POST /api/sessions/register`

Request:

```json
{ "first_name": "Ana", "last_name": "Pérez", "email": "Ana@Mail.com ", "password": "Secreta123" }
```

| Campo | Reglas |
|-------|--------|
| `first_name`, `last_name` | Obligatorios |
| `email` | Obligatorio, formato válido, único. Se normaliza con `trim` + minúsculas |
| `password` | Obligatoria, mínimo 8 caracteres y máximo 72 bytes (límite de bcrypt). Se guarda hasheada con bcrypt |

El campo `role` **no se acepta desde el body**: todo usuario nuevo se crea con el rol `user`
(menor privilegio). Los roles posibles son `user`, `organizer` y `admin`.

Response `201`:

```json
{ "status": "success", "payload": { "id": "665f2a...", "first_name": "Ana", "last_name": "Pérez", "email": "ana@mail.com", "role": "user" } }
```

| Código | Cuándo | Cuerpo |
|--------|--------|--------|
| `400` | Faltan campos obligatorios | `{ "status": "error", "message": "Faltan campos obligatorios" }` |
| `400` | Email con formato inválido | `{ "status": "error", "message": "El email no tiene un formato válido" }` |
| `400` | Contraseña menor a 8 caracteres | `{ "status": "error", "message": "La contraseña debe tener al menos 8 caracteres" }` |
| `400` | Contraseña mayor a 72 bytes | `{ "status": "error", "message": "La contraseña no puede superar los 72 bytes" }` |
| `409` | Email ya registrado | `{ "status": "error", "message": "El email ya está registrado" }` |

La respuesta **nunca incluye la contraseña**, ni en texto plano ni hasheada.

### `POST /api/sessions/login`

Request:

```json
{ "email": "ana@mail.com", "password": "Secreta123" }
```

Response `200` (además setea la cookie `currentUser`, `HttpOnly`):

```json
{ "status": "success", "message": "Login correcto" }
```

```http
Set-Cookie: currentUser=eyJhbGciOiJIUzI1NiIs...; Max-Age=3600; Path=/; HttpOnly; SameSite=Lax
```

Response `401` (email inexistente **o** contraseña incorrecta: siempre el mismo mensaje, para no
revelar cuál de los dos falló):

```json
{ "status": "error", "message": "Credenciales inválidas" }
```

Response `400` (falta `email` o `password`):

```json
{ "status": "error", "message": "Faltan campos obligatorios" }
```

Qué hace el login:

1. La estrategia `login` valida la presencia de `email` y `password` y normaliza el email.
2. Busca el usuario y compara la contraseña con el hash almacenado (`bcrypt.compare`).
   Si el email no existe igual se ejecuta una comparación contra un hash señuelo, para que el
   tiempo de respuesta no delate qué correos están registrados. Si todo coincide, deja
   `{ id, email, role }` en `req.user`; si no, falla con el mensaje genérico.
3. El **controller** genera el JWT con ese payload mínimo (nunca la contraseña), firmado con
   `JWT_SECRET` y con expiración `JWT_EXPIRES_IN`.
4. El controller guarda el token en la cookie `currentUser` con `httpOnly: true`,
   `sameSite: 'lax'`, `maxAge: 3600000` y `secure: true` solo en producción. El token **no viaja
   en el body**.
   La cookie dura una hora; si `JWT_EXPIRES_IN` es menor, la sesión termina cuando expira el
   token (`/current` responde `401` aunque la cookie siga presente).

### `GET /api/sessions/current`

Ruta protegida por el middleware `auth`, que es la estrategia `current` de Passport: lee la
cookie `currentUser` (o, alternativamente, el header `Authorization: Bearer <token>`), verifica
la firma y la vigencia del JWT y deja `{ id, email, role }` en `req.user`.

Response `200` (con la cookie):

```json
{ "status": "success", "payload": { "id": "665f2a...", "email": "ana@mail.com", "role": "user" } }
```

Response `401` (sin cookie, token manipulado, firmado con otro secreto o expirado):

```json
{ "status": "error", "message": "No autenticado" }
```

### `POST /api/sessions/logout`

Elimina la cookie `currentUser` (la reenvía vencida con los mismos atributos con que se creó).
No requiere estar autenticado.

Response `200`:

```json
{ "status": "success", "message": "Sesión cerrada" }
```

## Cómo probarlo

Con `curl` (la opción `-c` guarda la cookie que devuelve el login y `-b` la reenvía):

```bash
# 1. Registro
curl -X POST http://localhost:8080/api/sessions/register \
  -H "Content-Type: application/json" \
  -d '{ "first_name": "Ana", "last_name": "Pérez", "email": "ana@mail.com", "password": "Secreta123" }'

# 2. Login: guarda la cookie currentUser en cookies.txt (con -i se ve el header Set-Cookie)
curl -i -c cookies.txt -X POST http://localhost:8080/api/sessions/login \
  -H "Content-Type: application/json" \
  -d '{ "email": "ana@mail.com", "password": "Secreta123" }'
# { "status": "success", "message": "Login correcto" }

# 3. Usuario actual con la cookie → 200
curl -b cookies.txt http://localhost:8080/api/sessions/current
# { "status": "success", "payload": { "id": "...", "email": "ana@mail.com", "role": "user" } }

# 4. Logout: borra la cookie
curl -b cookies.txt -c cookies.txt -X POST http://localhost:8080/api/sessions/logout
# { "status": "success", "message": "Sesión cerrada" }

# 5. Usuario actual tras el logout → 401
curl -b cookies.txt http://localhost:8080/api/sessions/current
# { "status": "error", "message": "No autenticado" }

# Login con email inexistente o contraseña incorrecta → 401 con el mismo mensaje
curl -X POST http://localhost:8080/api/sessions/login \
  -H "Content-Type: application/json" \
  -d '{ "email": "nadie@mail.com", "password": "Secreta123" }'
# { "status": "error", "message": "Credenciales inválidas" }

# /current sin cookie → 401
curl http://localhost:8080/api/sessions/current
# { "status": "error", "message": "No autenticado" }

# /current con token manipulado → 401
curl -b "currentUser=token.manipulado.xyz" http://localhost:8080/api/sessions/current
# { "status": "error", "message": "No autenticado" }
```

En Postman o Thunder Client la cookie se guarda sola tras el login: basta con llamar a
`/current` después y ver el `200`; para probar el `401`, borrar la cookie `currentUser` en el
gestor de cookies (o esperar a que expire).

### Suite de tests

Los casos anteriores (y otros, como token expirado, firmado con otro secreto o con algoritmo
`none`, atributos de la cookie, payload del JWT sin contraseña) están automatizados:

```bash
pnpm test
```

41 tests con MongoDB en memoria. Los 35 de la entrega anterior corren **sin modificar** sobre la
nueva implementación, lo que demuestra que el contrato externo se mantuvo: flujo completo
registro → login → current → logout → 401, email duplicado, credenciales inválidas, `/current`
sin cookie o con token manipulado/expirado, validación de configuración al arrancar. Los 6
nuevos (`test/passport.test.js`) verifican la integración: las tres estrategias quedan
registradas con el tipo correcto, `app.js` no define estrategias, la estrategia de login no firma
el JWT (lo hace el controller), los fallos responden JSON y el login también funciona como
formulario.

## Decisiones de seguridad

- **Contraseñas**: se guardan solo como hash de bcrypt (`utils/hash.js`, funciones
  reutilizables `hashPassword` / `comparePassword`). Nunca se devuelven ni se incluyen en el JWT.
  Se rechazan las de más de 72 bytes porque bcrypt ignora el resto y compararía truncado.
- **Entrada validada en la frontera**: email y password deben ser strings; un objeto en el body
  (por ejemplo `{ "$gt": "" }`) se rechaza antes de llegar a la consulta de Mongoose.
- **Mensaje genérico en el login**: `Credenciales inválidas` tanto si el email no existe como si
  la contraseña no coincide. Además se compara siempre contra un hash (real o señuelo) para no
  revelar por tiempo de respuesta qué emails existen.
- **JWT**: firmado con `JWT_SECRET` de entorno, algoritmo fijado en el servidor
  (`algorithms: ['HS256']`, nunca tomado del header del token), payload mínimo `{ id, email, role }`
  y expiración configurable.
- **Cookie `httpOnly`**: el token no es accesible desde JavaScript del navegador (mitiga XSS);
  `sameSite: 'lax'` reduce el riesgo de CSRF; `secure` solo en producción para poder probar en
  `http://localhost`.
- **Estrategia `current` / middleware `auth`**: ante cualquier falla (sin token, firma inválida,
  manipulado, expirado) responde `401 No autenticado` sin detallar la causa.
- **Passport no firma tokens**: las estrategias solo autentican y entregan el usuario; emitir el
  JWT y manejar la cookie es responsabilidad del controller (capa HTTP).
- **Fail-Fast de configuración**: sin `JWT_SECRET` o `MONGO_URL` el servidor no arranca; un
  `JWT_EXPIRES_IN` inválido o cero también lo detiene, y un secreto de menos de 32 caracteres
  genera un aviso al iniciar.
- **Sesión sin estado**: `/current` responde desde el payload del JWT, sin consultar la base.
  Un cambio de rol o la baja del usuario se reflejan cuando el token expira (por eso la
  expiración es corta); el logout borra la cookie del navegador.

## Estructura de carpetas

```
coderhouse-backend2-preentrega4/
├── src/
│   ├── app.js                        # configura Express (json, cookie-parser, routers); no levanta el server
│   ├── server.js                     # punto de entrada: conecta MongoDB y levanta el servidor
│   ├── config/
│   │   ├── env.config.js             # carga y valida variables de entorno (Fail-Fast), JWT_EXPIRES_IN
│   │   ├── db.config.js              # conexión a MongoDB con Mongoose
│   │   └── passport.config.js        # estrategias 'register', 'login' y 'current' centralizadas
│   ├── routes/
│   │   ├── index.js                  # router principal: monta los recursos bajo /api
│   │   ├── health.router.js
│   │   ├── events.router.js
│   │   └── sessions.router.js        # delega en passport.authenticate(...) vía autenticarCon
│   ├── controllers/
│   │   ├── health.controller.js
│   │   ├── events.controller.js
│   │   └── sessions.controller.js    # HTTP: responde códigos, firma el JWT y setea/borra la cookie
│   ├── services/
│   │   └── events.service.js         # (la lógica de sesiones vive ahora en las estrategias de Passport)
│   ├── repositories/
│   │   ├── events.repository.js
│   │   └── users.repository.js       # acceso a datos; devuelve DTOs sin contraseña
│   ├── dao/
│   │   ├── events.dao.js
│   │   └── users.dao.js              # única capa que usa el modelo User de Mongoose
│   ├── dto/
│   │   └── user.dto.js               # toUserDTO (registro) y toSessionDTO ({ id, email, role })
│   ├── models/
│   │   ├── User.js                   # first_name, last_name, email (único), password, role
│   │   └── Event.js
│   ├── middlewares/
│   │   ├── auth.middleware.js        # autenticarCon(estrategia) envuelve passport.authenticate; auth = 'current'
│   │   ├── notFound.middleware.js
│   │   └── error.middleware.js       # traduce errores a 400 / 401 / 404 / 409 / 500
│   └── utils/
│       ├── jwt.js                    # signToken (jsonwebtoken, HS256, JWT_SECRET); la verificación la hace passport-jwt
│       ├── hash.js                   # hashPassword / comparePassword con bcrypt
│       ├── cookies.util.js           # nombre y opciones de la cookie currentUser
│       ├── validators.js             # isValidEmail, normalizeEmail, largo mínimo de contraseña
│       ├── errors.util.js            # HttpError y errores 400 / 401 / 403 / 404 / 409
│       └── responses.util.js         # helpers de respuesta uniforme
├── test/
│   ├── env.js                        # variables mínimas para la suite
│   ├── setup.js                      # MongoDB en memoria para los tests
│   ├── config.test.js                # validación de JWT_EXPIRES_IN y JWT_SECRET al arrancar
│   ├── api.test.js
│   ├── register.test.js
│   ├── auth.test.js                  # login, current, logout, flujo completo
│   └── passport.test.js              # estrategias registradas, separación app.js / config, JSON en fallos
├── .env.example
├── .gitignore
├── package.json
├── pnpm-workspace.yaml
└── README.md
```

### Flujo de la autenticación

```
POST /api/sessions/register
  → sessions.router        autenticarCon('register') → passport.authenticate('register')
  → passport.config        estrategia 'register': valida, normaliza, verifica unicidad, hashea
                           (utils/hash.js) y crea el usuario vía users.repository → users.dao
  → sessions.controller    responde 201 con req.user (DTO sin contraseña)

POST /api/sessions/login
  → sessions.router        autenticarCon('login') → passport.authenticate('login')
  → passport.config        estrategia 'login': findCredentialsByEmail + bcrypt.compare;
                           done(null, { id, email, role }) o done(null, false, 'Credenciales inválidas')
  → sessions.controller    firma el JWT (utils/jwt.js) y lo guarda en la cookie currentUser

GET /api/sessions/current
  → auth.middleware        auth = autenticarCon('current') → passport.authenticate('current')
  → passport.config        estrategia 'current' (passport-jwt): cookie currentUser o Bearer,
                           verifica firma/vigencia, done(null, { id, email, role })
  → sessions.controller    responde { id, email, role } desde req.user

POST /api/sessions/logout
  → sessions.controller    clearCookie('currentUser') y confirma (no pasa por Passport)
```

## Modelos

- **User**: `first_name`, `last_name`, `email` (único, minúsculas), `password` (hash de bcrypt),
  `role` (`user` por defecto | `organizer` | `admin`), timestamps.
- **Event**: `title`, `description`, `date`, `location`, `capacity`, `price`, `status`,
  `organizer` (referencia a `User`), `category` (referencia a `Category`, se gestiona con el
  CRUD de eventos).

## Próximos pasos

Autorización por roles (`organizer`/`admin`) sobre `req.user`, CRUD completo de eventos con
categorías e inscripciones con control de cupos, y providers externos (Google/GitHub) como
estrategias adicionales en `passport.config.js`.
