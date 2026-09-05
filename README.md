# Nicaragua Creativa — Mapa de la Red Nacional de Ciudades Creativas

El objetivo de esta aplicacion es ayudar a facilitar optimizar y automatizar uno de los mas grandes
problemas que tienen los turistas hoy en dia en cual es no saber en donde estan, la mayoria no puede contar con un guia turistico es por eso que tras tus huellas llega al rescate, nuestra aplicacion incluye un mapa interactivo donde el usuario podra ubicr distintas zonas comerciales famosas y concurridas del pais asi podra tener variedad a la hora de elegir donde explorar, ademas agrega un sistema de afiliacion entre local y usuario para que puedan seguir en contacto y un sistema de reseñas a los locales afiliados a nuestra aplicacion. 

## Estructura del proyecto

```
ciudades-creativas/
├── index.html        página principal
├── css/style.css      estilos (paleta, tipografía, responsive)
├── js/data.js         las 10 ciudades y lugares de ejemplo
└── js/app.js          lógica del mapa, panel, formularios y reseñas
```

## Funcionalidad incluida

- Mapa interactivo (Leaflet + OpenStreetMap, gratuito, sin API key) con las
  10 ciudades creativas marcadas.
- Selector de ciudades (chips) que centra el mapa y filtra los lugares.
- Buscador de lugares por nombre.
- Pantalla de historia por ciudad
- Cuentas de usuario (Registro y inicio de sesion con correo y contraseña.
- Cierre de sesion.)
- Boton para agregar un lugar dentro del mapa (reqiuiere sesion iniciada).
- Reseñas por lugar: calificación de 1 a 5 estrellas, nombre y comentario.
  El promedio de estrellas se ve tanto en el panel como en el globo del
  mapa (popup).
- Panel lateral en computadora / hoja deslizable en móvil, con vista de
  ciudad, listado de lugares y detalle de cada lugar.

## Resumen ejecutivo

Este proyecto por el momento no tiene servidor propio. Es una aplicación *frontend-only*
que se sirve como archivos estáticos y que delega
casi todo el backend — base de datos, autenticación, autorización y API —
a Supabase, un BaaS (Backend-as-a-Service) sobre PostgreSQL.

No hay `package.json`, no hay build step, no hay framework (React/Vue/etc.),
no hay servidor Node/Express. La "API" del sistema son directamente las
llamadas del SDK `@supabase/supabase-js` contra las tablas de Postgres,
protegidas por políticas de Row Level Security (RLS).

## 2. Arquitectura general
┌─────────────────────────────────────────────────────────────┐
│                      NAVEGADOR (cliente)                     │
│                                                               │
│  index.html                                                  │
│   ├─ css/style.css            (presentación)                 │
│   └─ js/  (scripts cargados en orden, sin módulos ES / sin    │
│            bundler — todo vive en el objeto global window)   │
│       1. supabaseConfig.js  → crea supabaseClient             │
│       2. auth.js            → sesión, login/registro          │
│       3. data.js            → catálogo estático (ciudades,    │
│                                categorías) embebido en JS      │
│       4. geo.js             → geolocalización nativa          │
│       5. eventos.js         → eventos culturales + notifs     │
│       6. app.js             → orquestador: mapa, panel, CRUD  │
│                                de lugares/reseñas              │
│                                                               │
│  Librerías de terceros (vía CDN, sin instalación):            │
│   • Leaflet 1.9.4        → mapa interactivo (tiles OSM)        │
│   • @supabase/supabase-js@2 → cliente REST/Realtime/Auth       │
└───────────────────────────┬───────────────────────────────────┘
                            │ HTTPS (REST autogenerado + Auth)
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                         SUPABASE (BaaS)                      │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────────┐ │
│  │  Auth          │  │  PostgREST    │  │  Postgres          │ │
│  │  (auth.users,  │  │  (API REST    │  │  (tablas públicas: │ │
│  │  email+pass,   │  │  autogenerada │  │  perfiles,         │ │
│  │  OAuth Google) │  │  sobre las    │  │  categorias,       │ │
│  │                │  │  tablas)      │  │  ciudades,         │ │
│  └───────┬────────┘  └───────┬───────┘  │  lugares, resenas, │ │
│          │                   │          │  eventos)          │ │
│          │   trigger al      │          │  + Row Level       │ │
│          └──registrar user──▶│          │    Security (RLS)  │ │
│                               ▼          └───────────────────┘ │
│                      políticas RLS deciden qué puede           │
│                      leer/escribir cada request                │
└─────────────────────────────────────────────────────────────┘

## 3. Dependencias

### 3.1 Dependencias externas (vía CDN, sin gestor de paquetes)

| Dependencia | Versión | Uso | Requiere API key |
|---|---|---|---|
| [Leaflet](https://leafletjs.com/) | `1.9.4` | Mapa interactivo, capas de tiles, marcadores, popups | No |
| Tiles de OpenStreetMap | `{s}.tile.openstreetmap.org` | Capa base del mapa | No |
| [`@supabase/supabase-js`](https://supabase.com/docs/reference/javascript) | `v2` (UMD, `latest` de la rama 2.x) | Cliente de Auth + Base de datos (PostgREST) | Sí (`anon public key`, ver §4) |

No hay `npm`, `yarn`, `package.json` ni bundler (Webpack/Vite). Las
librerías se referencian directamente en `index.html`:

```html
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.min.js"></script>
```

### 3.2 APIs nativas del navegador (sin librería)

| API | Archivo | Uso |
|---|---|---|
| `navigator.geolocation` (`watchPosition`/`clearWatch`) | `geo.js` | Ubicación en tiempo real |
| `Notification` | `eventos.js` | Alertas de eventos cercanos |
| `fetch` (implícito, vía supabase-js) | todos | Llamadas HTTP a Supabase |

### 3.3 Backend como dependencia (Supabase)

Provisto por el usuario (cuenta gratuita), no por el código:

- **Auth** (`auth.users`, email/password + OAuth Google)
- **Postgres** con extensión `pgcrypto` (para `gen_random_uuid()`)
- **PostgREST** (API REST autogenerada sobre las tablas `public.*`)
- **RLS** como única capa de autorización

---

## 4. "Variables de entorno" y configuración

Este proyecto **no usa un archivo `.env`** ni variables de entorno en
sentido estricto (no hay proceso de build ni servidor Node que las
lea). En su lugar, la configuración es un archivo JS con constantes
que se editan a mano una sola vez: **`js/supabaseConfig.js`**.

```js
const SUPABASE_URL = 'https://TU-PROYECTO.supabase.co';
const SUPABASE_ANON_KEY = 'TU-LLAVE-ANON-PUBLICA';
const supabaseClient = window.supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
```

| "Variable" | Dónde se define | Sensible | Notas |
|---|---|---|---|
| `SUPABASE_URL` | `js/supabaseConfig.js` | No | URL pública del proyecto Supabase |
| `SUPABASE_ANON_KEY` | `js/supabaseConfig.js` | No (por diseño) | Llave pública; el control de acceso real vive en RLS, no en esta llave |
| Credenciales OAuth de Google (`Client ID`/`Client Secret`) | **Panel de Supabase** (Authentication → Providers → Google), no en el código | Sí | Nunca tocan el repositorio; se configuran fuera del código fuente |
| `RADIO_NOTIFICACION_KM` | Constante al inicio de `js/eventos.js` (valor `1.5`) | No | Radio en km para disparar notificaciones de proximidad a eventos |

## 5. Estructura modular de scripts

ciudades-creativas/
├── index.html                       Único punto de entrada (SPA sin router)
├── css/
│   └── style.css                    Variables de tema (:root { --primary, --accent, ... }), layout responsive
├── js/
│   ├── supabaseConfig.js            Config + instancia global `supabaseClient`
│   ├── auth.js                      Módulo de autenticación
│   ├── data.js                      Catálogo estático (CIUDADES, CATEGORIAS)
│   ├── geo.js                       Geolocalización en tiempo real
│   ├── eventos.js                   Eventos culturales + notificaciones de cercanía
│   └── app.js                       Orquestador principal (mapa, panel, CRUD lugares/reseñas)
├── sql/
│   ├── schema.sql                   DDL completo + RLS + datos semilla (ejecutar una vez)
│   ├── migracion_eventos.sql        Migración incremental: agrega tabla `eventos`
│   └── migracion_login_google.sql   Migración incremental: reconoce nombre desde Google OAuth
└── docs/
    └── diagrama-base-de-datos.md    ERD en Mermaid/markdown

### 5.1 Responsabilidad de cada script

**`supabaseConfig.js`**
Punto único de conexión. Expone `supabaseClient` a todo el resto de la
app vía `window`. Sin este archivo, ningún otro script funciona.

**`auth.js`**
- Objeto de estado `auth = { user, perfil }`.
- `initAuth()`: recupera sesión existente y se suscribe a
  `onAuthStateChange` (login/logout/expiración de token).
- `registrarUsuario()`, `iniciarSesion()`, `iniciarSesionGoogle()`,
  `cerrarSesion()`: envoltorios directos sobre `supabaseClient.auth.*`.
- Sincroniza el perfil público (`perfiles.nombre`) tras cada cambio de
  sesión.
- Renderiza el modal de login/registro y el chip de usuario en la
  cabecera.
- Expone `estaAutenticado()` y `nombreUsuarioActual()`, usadas como
  guardas en `app.js`/`eventos.js` antes de cualquier escritura.

**`data.js`**
Datos 100% estáticos, sin llamadas a red: el diccionario `CATEGORIAS`
(color/ícono por categoría) y el arreglo `CIUDADES` (10 ciudades con
coordenadas, historia y ámbito). Sirve como "caché" de lectura rápida
para el mapa y los selects — la fuente de verdad relacional real está
en las tablas `ciudades`/`categorias` de Postgres.

**`geo.js`**
- Objeto `geo = { activo, watchId, lat, lng, marker }`.
- `distanciaKm()` (fórmula de Haversine) y `formatoDistancia()`,
  reutilizadas por `app.js` y `eventos.js` para calcular cercanía.
- `activarGeo()`/`desactivarGeo()`: envuelven
  `navigator.geolocation.watchPosition`/`clearWatch`.
- Pinta el punto azul de ubicación del usuario en el mapa Leaflet.
- Dispara `verificarEventosCercanos()` (definida en `eventos.js`) en
  cada actualización de posición.

**`eventos.js`**
- Objeto `eventos = { lista, notificados, permisoPedido }`.
- `cargarEventos()`: única función que hace `SELECT` sobre la tabla
  `eventos`.
- Helpers de estado temporal: `eventoEnCurso()`, `eventoProximo()`,
  `eventoVigente()`.
- Render de marcadores/pines de eventos en el mapa y su tarjeta dentro
  del panel lateral.
- `guardarEvento()`: `INSERT` en la tabla `eventos` (requiere sesión).
- Sistema de notificaciones de proximidad vía `Notification` API,
  usando el radio `RADIO_NOTIFICACION_KM`.

**`app.js`** (el más grande, ~640 líneas — orquestador central)
- Estado global `state` (ciudad activa, lugar activo, modo "colocar en
  mapa", término de búsqueda, etc.) y referencias Leaflet (`map`,
  `cityMarkers`, `placeMarkers`).
- `cargarLugares()`: único punto que trae `lugares` + sus `resenas`
  anidadas (join implícito vía PostgREST).
- Inicialización del mapa (`initMap`), capas de tiles, marcadores de
  ciudad y de lugar.
- Render del panel lateral con 4 vistas condicionales: detalle de
  lugar, ciudad seleccionada, "cerca de ti" (geolocalización activa) y
  listado general/búsqueda.
- CRUD de lugares (`guardarLugar`) y reseñas (`guardarResena`) contra
  Supabase.
- Manejo de los modales (agregar lugar, agregar evento, reseña, auth,
  acerca de) y de la "pantalla completa" de historia por ciudad.
- `DOMContentLoaded`: es el único orquestador de arranque — llama
  en orden a `initAuth → initMap → cargarLugares → cargarEventos →
  renders iniciales → binding de todos los listeners`.

### 5.2 Orden de carga (crítico)

```html
<script src="js/supabaseConfig.js"></script>  <!-- 1: crea supabaseClient -->
<script src="js/auth.js"></script>            <!-- 2: usa supabaseClient -->
<script src="js/data.js"></script>            <!-- 3: datos estáticos -->
<script src="js/geo.js"></script>             <!-- 4: usa CIUDADES indirectamente vía app.js -->
<script src="js/eventos.js"></script>         <!-- 5: usa supabaseClient, CATEGORIAS -->
<script src="js/app.js"></script>             <!-- 6: usa TODO lo anterior + dispara DOMContentLoaded -->

### 6. Modelo de datos (Supabase / PostgreSQL)

| Tabla | PK | Relaciones | Quién puede INSERT | Quién puede UPDATE/DELETE | Quién puede SELECT |
|---|---|---|---|---|---|
| `perfiles` | `id` (= `auth.users.id`) | 1:1 con `auth.users` | (automático, vía trigger) | — | público |
| `categorias` | `id` (text) | — | — | — | público |
| `ciudades` | `id` (text) | — | — | — | público |
| `lugares` | `id` (uuid) | `ciudad_id → ciudades`, `categoria_id → categorias`, `creado_por → auth.users` | usuario autenticado, solo `creado_por = auth.uid()` | solo el creador | público |
| `resenas` | `id` (uuid) | `lugar_id → lugares` (cascade delete), `usuario_id → auth.users` | usuario autenticado, solo `usuario_id = auth.uid()` | solo el autor | público |
| `eventos` | `id` (uuid) | `ciudad_id → ciudades`, `categoria_id → categorias`, `creado_por → auth.users` | usuario autenticado, solo `creado_por = auth.uid()` | solo el creador | público |

**Trigger automático:** `al_crear_usuario` (`after insert on
auth.users`) ejecuta `manejar_nuevo_usuario()`, que crea la fila
correspondiente en `perfiles`, tomando el nombre desde (en orden de
prioridad): el metadato `nombre` del registro por correo → `full_name`
u `name` entregados por Google OAuth → el prefijo del email como
respaldo.

**Migraciones incrementales** (para proyectos Supabase ya existentes):
- `migracion_eventos.sql`: agrega la tabla `eventos` completa (si el
  proyecto se configuró antes de que existiera esta funcionalidad).
- `migracion_login_google.sql`: reemplaza la función del trigger para
  que también reconozca nombres provenientes de Google.

Ver también `docs/diagrama-base-de-datos.md` para el ERD visual.

## 7. Flujo de autenticación

1. **Registro por correo:** `signUp({ email, password, options: {
   data: { nombre } } })` → Supabase crea el usuario en `auth.users` →
   el trigger crea automáticamente su fila en `perfiles`.
2. **Login por correo:** `signInWithPassword()`.
3. **Login con Google:** `signInWithOAuth({ provider: 'google',
   redirectTo: <url actual sin hash> })` → redirección a Google →
   Supabase completa la sesión al volver → `onAuthStateChange` la
   detecta sola.
4. **Persistencia de sesión:** gestionada internamente por
   `supabase-js` (localStorage); `initAuth()` la recupera al cargar la
   página con `getSession()`.
5. **Guardas de escritura:** cada acción que escribe datos (agregar
   lugar, agregar evento, escribir reseña) verifica `estaAutenticado()`
   primero; si no hay sesión, abre el modal de login en vez de
   continuar. La app **nunca confía solo en el frontend**: aunque se
   evadiera esta guarda, el `INSERT` sería rechazado por RLS si
   `creado_por`/`usuario_id` no coincide con `auth.uid()`.

---

## 8. Ejemplos de "endpoints"

No existen rutas HTTP propias (`/api/...`). Lo que cumple el rol de
"endpoints" son las llamadas del SDK de Supabase, que internamente se
traducen a peticiones REST contra PostgREST.

### 8.1 Autenticación

```
POST  supabaseClient.auth.signUp
Body: { email, password, options: { data: { nombre } } }
Auth: pública (no requiere sesión)
Uso: js/auth.js → registrarUsuario()

POST  supabaseClient.auth.signInWithPassword
Body: { email, password }
Auth: pública
Uso: js/auth.js → iniciarSesion()
```

```
POST  supabaseClient.auth.signInWithOAuth
Body: { provider: 'google', options: { redirectTo } }
Auth: pública
Uso: js/auth.js → iniciarSesionGoogle()
```

```
POST  supabaseClient.auth.signOut
Auth: sesión activa
Uso: js/auth.js → cerrarSesion()
```

### 8.2 Lectura de datos

```
GET  supabaseClient.from('perfiles').select('id, nombre').eq('id', <uid>).single()
Auth: pública (RLS: lectura para todos)
Uso: js/auth.js → actualizarSesion()
Respuesta: { id: 'uuid', nombre: 'María Pérez' }

GET  supabaseClient.from('lugares')
     .select('*, resenas(*, perfiles(nombre))')
     .order('creado_en', { ascending: true })
Auth: pública
Uso: js/app.js → cargarLugares()
Respuesta (ejemplo):
[
  {
    "id": "b3f1...",
    "ciudad_id": "masaya",
    "categoria_id": "artesania",
    "nombre": "Mercado de Artesanías de Masaya",
    "lat": 11.9745, "lng": -86.0965,
    "descripcion": "El mercado artesanal más reconocido del país.",
    "creado_por": "a1c2...",
    "creado_en": "2026-01-10T14:32:00Z",
    "resenas": [
      { "id": "7d2e...", "calificacion": 5, "texto": "Excelente variedad",
        "usuario_id": "a1c2...", "fecha": "2026-02-01T10:00:00Z",
        "perfiles": { "nombre": "Ana" } }
    ]
  }
]
```

```
GET  supabaseClient.from('eventos').select('*').order('fecha_inicio', { ascending: true })
Auth: pública
Uso: js/eventos.js → cargarEventos()
```

### 8.3 Escritura de datos (requieren sesión iniciada)

```
POST  supabaseClient.from('lugares').insert({
        ciudad_id, categoria_id, nombre, lat, lng,
        descripcion, creado_por: auth.user.id
      }).select().single()
Auth: requerida — RLS exige creado_por = auth.uid()
Uso: js/app.js → guardarLugar()
Errores posibles: violación de RLS (403 implícito vía `error`),
  violación de FK si ciudad_id/categoria_id no existen.
```

```
POST  supabaseClient.from('eventos').insert({
        ciudad_id, categoria_id, nombre, descripcion, lat, lng,
        fecha_inicio, fecha_fin, creado_por: auth.user.id
      }).select().single()
Auth: requerida — RLS exige creado_por = auth.uid()
Uso: js/eventos.js → guardarEvento()
Validación en cliente: fecha_fin > fecha_inicio (antes de enviar)
```

```
POST  supabaseClient.from('resenas').insert({
        lugar_id, usuario_id: auth.user.id, calificacion, texto
      }).select('*, perfiles(nombre)').single()
Auth: requerida — RLS exige usuario_id = auth.uid()
Uso: js/app.js → guardarResena()
Validación: calificacion entre 1 y 5 (constraint a nivel de BD)
```

### 8.4 Resumen tabular de operaciones

| Operación | Tabla | Método PostgREST | Sesión requerida | Archivo |
|---|---|---|---|---|
| Ver perfil propio | `perfiles` | `select` | No (pero sin sesión no aplica) | `auth.js` |
| Listar lugares + reseñas | `lugares` | `select` (con join anidado) | No | `app.js` |
| Listar eventos | `eventos` | `select` | No | `eventos.js` |
| Crear lugar | `lugares` | `insert` | Sí | `app.js` |
| Crear evento | `eventos` | `insert` | Sí | `eventos.js` |
| Crear reseña | `resenas` | `insert` (con join de retorno) | Sí | `app.js` |
| Registro | `auth.users` (vía Auth API) | `signUp` | No | `auth.js` |
| Login correo | `auth.users` (vía Auth API) | `signInWithPassword` | No | `auth.js` |
| Login Google | `auth.users` (vía Auth API) | `signInWithOAuth` | No | `auth.js` |
| Cerrar sesión | — | `signOut` | Sí | `auth.js` |
