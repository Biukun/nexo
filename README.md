# Nexo

Marketplace de profesionales verificados en Chile — conecta clientes con **kinesiólogos** y **técnicos en informática/soporte TI** que pasaron por un proceso de verificación de identidad y documentos antes de poder recibir solicitudes.

> Estado: proyecto en desarrollo / pruebas. El flujo de pagos está **simulado** (ver sección [Pagos](#pagos--escrow-simulado) más abajo) — no procesa dinero real todavía.

---

## Tabla de contenidos

- [Qué hace](#qué-hace)
- [Stack](#stack)
- [Estructura del repo](#estructura-del-repo)
- [Setup desde cero](#setup-desde-cero)
  - [1. Clonar y servir](#1-clonar-y-servir)
  - [2. Crear el proyecto en Supabase](#2-crear-el-proyecto-en-supabase)
  - [3. Tablas y políticas (SQL)](#3-tablas-y-políticas-sql)
  - [4. Storage (subida de CV/títulos)](#4-storage-subida-de-cvtítulos)
  - [5. Auth — usuario admin](#5-auth--usuario-admin)
  - [6. EmailJS (correo de confirmación)](#6-emailjs-correo-de-confirmación)
  - [7. Conectar las claves en `index.html`](#7-conectar-las-claves-en-indexhtml)
- [Panel de administración](#panel-de-administración)
- [Pagos / escrow (simulado)](#pagos--escrow-simulado)
- [Esquema de datos](#esquema-de-datos)
- [Limitaciones conocidas / TODO](#limitaciones-conocidas--todo)
- [Seguridad](#seguridad)

---

## Qué hace

Nexo tiene tres flujos principales, todos dentro de un único `index.html` (SPA sin framework, navegación entre "páginas" con `goTo()`):

- **Clientes**: buscan un servicio (salud o tecnología), ven categorías con contadores reales de profesionales disponibles, y contactan directo por WhatsApp.
- **Profesionales**: se registran en un flujo de varios pasos — datos personales, subida de documentos (CV, título, certificado/registro de Superintendencia de Salud), un bot de validación conversacional, y para el área técnica un test de 15 preguntas que los clasifica en Junior/Mid/Senior. Quedan en estado `pendiente` hasta revisión.
- **Admin**: panel interno (`/#admin`) con login real de Supabase Auth para revisar, aprobar o rechazar profesionales registrados, y ver los documentos que subieron.

## Stack

- HTML/CSS/JS vanilla (un solo archivo, sin build step)
- [Supabase](https://supabase.com) — base de datos Postgres, Auth, Storage
- [EmailJS](https://www.emailjs.com) — correo de confirmación al profesional (100% frontend, sin backend de email)
- Fuentes: Google Fonts (Syne + DM Sans)

No hay backend propio. Todo corre en el navegador y habla directo con Supabase usando la **clave pública (publishable/anon key)**, que está pensada para ser expuesta — la seguridad real vive en las políticas RLS de Postgres (ver sección [Seguridad](#seguridad)).

## Estructura del repo

```
.
├── index.html              # Toda la app (HTML + CSS + JS)
├── setup_supabase.sql      # SQL inicial: columnas reg_super/documentos, RLS de profesionales, storage policy
├── setup_supabase_v2.sql   # SQL adicional: tabla pagos, columnas extra de solicitudes
└── README.md
```

## Setup desde cero

### 1. Clonar y servir

No requiere build. Para desarrollo local basta con servir el archivo (no abrirlo con `file://` directamente, porque algunas APIs del navegador y CORS de Supabase se comportan distinto):

```bash
git clone <url-del-repo>
cd nexo
python3 -m http.server 8080
# abrir http://localhost:8080/index.html
```

Para producción, cualquier hosting estático sirve (GitHub Pages, Vercel, Netlify, Cloudflare Pages).

### 2. Crear el proyecto en Supabase

1. Crea una cuenta en [supabase.com](https://supabase.com) y un nuevo proyecto.
2. Anota la **Project URL** y la **anon/publishable key** (Settings → API). Las vas a necesitar en el paso 7.
3. En el Table Editor, crea las tablas base si no existen: `profesionales` y `solicitudes`. Como mínimo, `profesionales` necesita: `id` (identity/PK), `nombre`, `apellido`, `email`, `telefono`, `region`, `tipo`, `especialidad`, `nivel`, `test_score`, `estado` (default `'pendiente'`). `solicitudes` necesita: `id`, `servicio`, `categoria`, `descripcion`, `estado`.

### 3. Tablas y políticas (SQL)

Corre los dos archivos SQL del repo **en orden**, en Supabase → SQL Editor:

1. **`setup_supabase.sql`** — agrega `reg_super` y `documentos` (jsonb) a `profesionales`, habilita RLS y crea las políticas para que el formulario público pueda insertar y el panel admin (autenticado) pueda leer/actualizar.
2. **`setup_supabase_v2.sql`** — agrega columnas a `solicitudes` (`monto`, `cliente_nombre`, `cliente_email`, `cliente_telefono`), crea la tabla `pagos` con su RLS, y refuerza las políticas de `solicitudes`.

Ambos scripts son **idempotentes** (usan `IF NOT EXISTS` y manejo de `duplicate_object`), así que correrlos de nuevo no rompe nada si ya aplicaste parte.

### 4. Storage (subida de CV/títulos)

1. Storage → New bucket → nombre **`documentos`** → marcar como **público**.
2. La política de Storage para permitir subidas desde el formulario (clave anon) ya está incluida en `setup_supabase.sql`. Si por alguna razón el bucket no acepta subidas, revisa Storage → Policies → bucket `documentos` y confirma que existe una policy de `INSERT` para el rol `anon`.

> ⚠️ El bucket es público: cualquiera con la URL de un archivo puede verlo. Como ahí se suben CVs, títulos y números de registro de Superintendencia de Salud (datos personales), evalúa pasar a un bucket **privado con URLs firmadas** antes de tener usuarios reales. Ver [Seguridad](#seguridad).

### 5. Auth — usuario admin

El panel de `/#admin` usa Supabase Auth (login con correo y contraseña), no una contraseña hardcodeada.

1. Authentication → Users → **Add user**.
2. Ingresa el correo y contraseña que vas a usar para entrar al panel.
3. Marca **Auto Confirm User** para no tener que confirmar por correo.

Cualquier usuario que crees ahí puede entrar al panel admin (las políticas RLS le dan acceso de lectura/escritura a `profesionales` por ser `authenticated`). Si vas a tener varios admins, créalos todos acá.

### 6. EmailJS (correo de confirmación)

EmailJS envía un correo al profesional cuando su registro se guarda ("recibimos tu solicitud, te avisamos en 24-48h").

1. Crea una cuenta gratis en [emailjs.com](https://www.emailjs.com) (plan free: ~200 correos/mes).
2. Email Services → conecta tu cuenta de Gmail → copia el **Service ID**.
3. Email Templates → crea un template con variables `{{to_name}}`, `{{to_email}}`, `{{tipo}}` → copia el **Template ID**.
4. Account → General → copia tu **Public Key**.

### 7. Conectar las claves en `index.html`

Todas las claves están declaradas juntas cerca del inicio del `<script>` principal:

```js
// Configuración Supabase
const SUPABASE_URL = 'https://TU-PROYECTO.supabase.co';
const SUPABASE_KEY = 'tu-anon-key-aqui';
const supabaseClient = window.supabase.createClient(SUPABASE_URL, SUPABASE_KEY);
const STORAGE_BUCKET = 'documentos';

// Configuración EmailJS
const EMAILJS_PUBLIC_KEY = 'tu-public-key-aqui';
const EMAILJS_SERVICE_ID = 'tu-service-id-aqui';
const EMAILJS_TEMPLATE_ID = 'tu-template-id-aqui';
```

Reemplaza los 6 valores con los de tu proyecto de Supabase y tu cuenta de EmailJS. Si dejas los placeholders de EmailJS sin cambiar (`'TU_PUBLIC_KEY_AQUI'`), el sitio funciona igual pero no envía el correo de confirmación — falla en silencio.

El correo de contacto del equipo (`contact.us.nexo@gmail.com`) aparece hardcodeado en 3 lugares (botón de Gmail del modal de inscripción, texto visible, y la función `copiarCorreo`). Búscalo y reemplázalo si cambia.

---

## Panel de administración

- Accede vía `tusitio.cl/index.html#admin` o desde el link discreto "admin" en el footer del home.
- Login con el usuario creado en el paso 5.
- Filtra profesionales por `pendiente` / `aprobado` / `rechazado` / `todos`.
- Cada tarjeta muestra los datos del registro, el documento(s) adjunto(s) (links al bucket de Storage), y botones para aprobar/rechazar.
- Aprobar/rechazar actualiza `estado` en `profesionales` y recalcula los contadores de categorías en la home.

## Pagos / escrow (simulado)

El botón "simular pago" en la página de clientes:

1. Pide nombre, correo y teléfono con `prompt()`.
2. Crea una fila en `solicitudes` con estado `pendiente_pago`.
3. Crea una fila en `pagos` con estado `retenido` (monto, comisión fija de $2.000 CLP, monto al profesional).
4. Si confirmas que el servicio se completó, actualiza `solicitudes` a `completado` y `pagos` a `liberado`.

**Esto es una demo, no un sistema de pagos.** No hay dinero real moviéndose — es solo para mostrar el flujo de escrow. Antes de manejar pagos reales hay que reemplazar este flujo por un backend (Supabase Edge Function o similar) integrado con un procesador real (Flow, Mercado Pago, Webpay), que sea el único capaz de escribir en `pagos`.

## Esquema de datos

**`profesionales`**

| columna | tipo | notas |
|---|---|---|
| id | bigint identity | PK |
| nombre, apellido, email, telefono, region | text | |
| tipo | text | `'salud'` \| `'tech'` |
| especialidad | text | área específica dentro de `tipo` |
| reg_super | text | nº de registro Superintendencia de Salud (solo `salud`) |
| documentos | jsonb | `{ "cv-tech": "https://...", "doc-tech": "https://..." }` |
| nivel | text | `'junior'` \| `'mid'` \| `'senior'` (solo `tech`, según test) |
| test_score | int | puntaje del test de nivel (sobre 15) |
| estado | text | `'pendiente'` \| `'aprobado'` \| `'rechazado'` |

**`solicitudes`**

| columna | tipo | notas |
|---|---|---|
| id | bigint identity | PK |
| servicio, categoria, descripcion | text | |
| estado | text | `'nueva'` \| `'pendiente_pago'` \| `'completado'` |
| monto | numeric | solo si vino de "simular pago" |
| cliente_nombre, cliente_email, cliente_telefono | text | solo si vino de "simular pago" |

**`pagos`**

| columna | tipo | notas |
|---|---|---|
| id | bigint identity | PK |
| solicitud_id | bigint | FK → `solicitudes.id` |
| monto, comision, monto_profesional | numeric | |
| estado | text | `'retenido'` \| `'liberado'` |

## Limitaciones conocidas / TODO

- Pagos simulados, sin procesador real (ver arriba).
- Bucket de Storage público — datos personales (CVs, títulos, RUTs) accesibles por URL directa si alguien la consigue.
- No hay rate-limiting ni captcha en el formulario público — alguien podría hacer spam de registros/uploads.
- El mapeo de categorías de la home (`categoryMap`, en el script) hacia `especialidad` es aproximado, no 1:1. Si se necesita precisión exacta, conviene agregar una columna `categoria` que el profesional elija directamente al registrarse.
- Sin tests automatizados.

## Seguridad

- La clave de Supabase en `index.html` es la **anon/publishable key** — está diseñada para ser pública. La protección real son las políticas de **Row Level Security (RLS)** definidas en los scripts SQL: el rol `anon` solo puede insertar (no leer/listar todo), y solo usuarios `authenticated` (admins) pueden leer y actualizar.
- Si modificas las tablas o agregas otras nuevas, recuerda habilitar RLS y escribir las políticas correspondientes — sin RLS, una tabla con la anon key es de libre lectura/escritura para cualquiera.
- La clave pública de EmailJS también está pensada para exponerse en frontend; el límite real lo da tu plan (200 correos/mes en el free tier).
- No subas al repo ninguna clave que **no** sea explícitamente pública (service role key de Supabase, claves privadas, etc.). Si en algún momento se agrega una Edge Function o un backend, esas credenciales van en variables de entorno del lado servidor, nunca en este archivo.
