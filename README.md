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
  - [5. Auth — usuarios y configuración de URLs](#5-auth--usuarios-y-configuración-de-urls)
  - [6. EmailJS (correo de confirmación)](#6-emailjs-correo-de-confirmación)
  - [7. Conectar las claves en `index.html`](#7-conectar-las-claves-en-indexhtml)
- [Páginas y flujos](#páginas-y-flujos)
- [Panel de administración](#panel-de-administración)
- [Pagos / escrow (simulado)](#pagos--escrow-simulado)
- [Esquema de datos](#esquema-de-datos)
- [Limitaciones conocidas / TODO](#limitaciones-conocidas--todo)
- [Seguridad](#seguridad)

---

## Qué hace

Nexo es una SPA (Single Page Application) sin framework, contenida en un único `index.html`. La navegación entre secciones usa `goTo()` con transiciones CSS. Tiene 7 páginas internas:

- **`page-home`** — landing con buscador, categorías con contadores reales desde Supabase, y cards de profesionales destacados.
- **`page-clientes`** — catálogo de servicios disponibles, filtros por categoría, y botón de contacto por WhatsApp con simulación de escrow.
- **`page-pro`** — landing para profesionales, con descripción de nichos (salud / informática) y botones de inscripción.
- **`page-registro`** — flujo de inscripción de 5 pasos: datos personales → documentos → bot de validación conversacional → test de nivel (solo TI, 15 preguntas, clasifica en Junior/Mid/Senior) → resumen y envío a Supabase.
- **`page-login`** — registro e inicio de sesión de clientes y profesionales con email + contraseña (Supabase Auth). Incluye recuperación de contraseña por email.
- **`page-perfil`** — panel del usuario autenticado: historial de solicitudes, favoritos, y estado del perfil profesional (si aplica).
- **`page-admin`** — panel interno (`/#admin`) para revisar, aprobar o rechazar profesionales registrados.

## Stack

- HTML/CSS/JS vanilla (un solo archivo, sin build step)
- [Supabase](https://supabase.com) — base de datos Postgres, Auth, Storage
- [EmailJS](https://www.emailjs.com) — correo de confirmación al profesional (100% frontend, sin backend de email)
- Fuentes: Google Fonts (Syne + DM Sans)

No hay backend propio. Todo corre en el navegador y habla directo con Supabase usando la **anon/legacy key** (`eyJ...`), que está pensada para ser expuesta — la seguridad real vive en las políticas RLS de Postgres (ver [Seguridad](#seguridad)).

> ⚠️ La clave correcta es la **Legacy anon key** (pestaña "Legacy anon, service_role API keys" en el Dashboard de Supabase), que empieza con `eyJ...`. La nueva "Publishable key" (`sb_publishable_...`) no es compatible con `@supabase/supabase-js@2`.

## Estructura del repo

```
.
├── index.html               # Toda la app (HTML + CSS + JS)
├── setup_supabase.sql       # SQL v1: columnas reg_super/documentos, RLS profesionales, storage policy
├── setup_supabase_v2.sql    # SQL v2: tabla pagos, columnas extra de solicitudes, RLS pagos
├── setup_auth_v3.sql        # SQL v3: tabla favoritos, RLS actualizado con auth de usuarios
├── fix_rls_solicitudes.sql  # SQL auxiliar: diagnóstico y corrección de RLS en solicitudes
├── CONTRIBUTING.md
└── README.md
```

## Setup desde cero

### 1. Clonar y servir

No requiere build. Para desarrollo local sirve el archivo directamente (no abrir con `file://` — CORS de Supabase falla):

```bash
git clone <url-del-repo>
cd nexo
python3 -m http.server 8080
# abrir http://localhost:8080/index.html
```

Para producción, cualquier hosting estático sirve (GitHub Pages, Vercel, Netlify, Cloudflare Pages).

### 2. Crear el proyecto en Supabase

1. Crea una cuenta en [supabase.com](https://supabase.com) y un nuevo proyecto.
2. Ve a **Settings → API → Legacy anon, service_role API keys** y copia la **anon key** (`eyJ...`). La vas a necesitar en el paso 7.
3. En el Table Editor, crea las tablas base `profesionales` y `solicitudes` con al menos estas columnas:
   - `profesionales`: `id` (identity PK), `nombre`, `apellido`, `email`, `telefono`, `region`, `tipo`, `especialidad`, `nivel`, `test_score`, `estado` (default `'pendiente'`)
   - `solicitudes`: `id` (identity PK), `servicio`, `categoria`, `descripcion`, `estado`

### 3. Tablas y políticas (SQL)

Corre los scripts SQL del repo **en orden**, en Supabase → SQL Editor:

1. **`setup_supabase.sql`** — `reg_super` y `documentos` (jsonb) en `profesionales`, RLS, policy de Storage.
2. **`setup_supabase_v2.sql`** — tabla `pagos`, columnas extra en `solicitudes` (`monto`, `cliente_nombre`, `cliente_email`, `cliente_telefono`), RLS de pagos y solicitudes.
3. **`setup_auth_v3.sql`** — tabla `favoritos` (clientes ↔ profesionales), RLS actualizado para que cada usuario vea solo sus propias solicitudes y favoritos.

Todos son **idempotentes** — se pueden correr más de una vez sin romper nada.

Si después de correrlos sigues viendo errores RLS en consola, corre `fix_rls_solicitudes.sql` para diagnosticar y reparar las políticas de `solicitudes`.

### 4. Storage (subida de CV/títulos)

1. Storage → New bucket → nombre **`documentos`** → marcar como **público**.
2. La policy de Storage para `anon` ya está en `setup_supabase.sql`. Si el bucket rechaza subidas, ve a Storage → Policies → `documentos` y confirma que existe una policy `INSERT` para `anon`.

> ⚠️ Bucket público: cualquiera con la URL directa puede ver los archivos. Como se suben CVs, títulos y números de registro de Superintendencia de Salud, evalúa pasar a bucket **privado con URLs firmadas** antes de tener usuarios reales.

### 5. Auth — usuarios y configuración de URLs

**Usuario admin (panel interno):**
1. Authentication → Users → **Add user**.
2. Ingresa correo y contraseña del admin.
3. Marca **Auto Confirm User**.

**Usuarios clientes/profesionales** se crean solos desde `page-login` del sitio. Reciben un email de confirmación automático de Supabase antes de poder ingresar.

**Configuración de URLs — paso crítico:**

Sin esto, los links de confirmación de email y recuperación de contraseña redirigen a `localhost` y el usuario termina en una página en blanco.

1. Authentication → **URL Configuration**
2. **Site URL**: `https://biukun.github.io/nexo/`
3. **Redirect URLs**: agregar `https://biukun.github.io/nexo/`

Si cambias de dominio en el futuro, actualiza estos dos valores.

**Confirmar email requerido (recomendado):**
Authentication → Providers → Email → **Confirm email: ON**

### 6. EmailJS (correo de confirmación al profesional)

EmailJS envía un correo cuando un profesional completa el registro ("recibimos tu solicitud, te avisamos en 24-48h"). Esto es distinto al email de confirmación de cuenta que maneja Supabase directamente.

1. Cuenta gratis en [emailjs.com](https://www.emailjs.com) (~200 correos/mes).
2. Email Services → conecta Gmail → copia el **Service ID**.
3. Email Templates → crea template con variables `{{to_name}}`, `{{to_email}}`, `{{tipo}}` → copia el **Template ID**.
4. Account → General → copia la **Public Key**.

### 7. Conectar las claves en `index.html`

Todas las claves están juntas al inicio del `<script>` principal:

```js
// Configuración Supabase
const SUPABASE_URL = 'https://TU-PROYECTO.supabase.co';
const SUPABASE_KEY = 'eyJ...'; // Legacy anon key
const supabaseClient = window.supabase.createClient(SUPABASE_URL, SUPABASE_KEY);
const STORAGE_BUCKET = 'documentos';

// Configuración EmailJS
const EMAILJS_PUBLIC_KEY = 'tu-public-key-aqui';
const EMAILJS_SERVICE_ID = 'tu-service-id-aqui';
const EMAILJS_TEMPLATE_ID = 'tu-template-id-aqui';
```

Si dejas los placeholders de EmailJS sin cambiar, el sitio funciona pero no envía el correo de confirmación al profesional — falla en silencio.

El correo de contacto (`contact.us.nexo@gmail.com`) aparece hardcodeado en 3 lugares del HTML. Si cambia, búscalo y reemplaza los 3.

---

## Páginas y flujos

### Login / Registro (`page-login`)

- Tabs: **Iniciar sesión** / **Crear cuenta**
- Al registrarse, el usuario elige tipo: **Cliente** (busca servicios) o **Profesional** (ofrece servicios)
- Supabase envía email de confirmación automáticamente — el usuario no puede ingresar hasta confirmar
- Recuperación de contraseña vía link al correo (redirige de vuelta al sitio con el token)
- El botón de usuario (inicial del nombre o "Ingresar") aparece en las navbars de home, clientes y pro

### Perfil de usuario (`page-perfil`)

- **Historial**: solicitudes donde `cliente_email` coincide con el email del usuario autenticado
- **Favoritos**: profesionales guardados (tabla `favoritos`); se pueden quitar con ✕
- **Mi perfil profesional** (solo visible para tipo `pro`): estado de la solicitud de inscripción, datos registrados, nivel del test técnico

### Panel de administración (`page-admin`)

- Accede vía `/#admin` o el link discreto "admin" en el footer del home
- Login con el usuario admin creado en el paso 5
- Filtra por `pendiente` / `aprobado` / `rechazado` / `todos`
- Cada tarjeta muestra datos completos + links a los documentos en Storage
- Aprobar/rechazar actualiza `estado` en `profesionales` y recalcula los contadores de categorías en la home

---

## Pagos / escrow (simulado)

El botón "simular pago" en `page-clientes`:

1. Pide nombre, correo y teléfono con `prompt()`.
2. Crea una fila en `solicitudes` con estado `pendiente_pago`.
3. Crea una fila en `pagos` con estado `retenido` (monto, comisión fija $2.000 CLP, monto al profesional).
4. Si confirmas que el servicio se completó, actualiza ambas tablas a `completado` / `liberado`.

**Esto es una demo.** No hay dinero real. Antes de manejar pagos reales hay que reemplazar este flujo con un backend (Supabase Edge Function) integrado con un procesador real (Flow, Mercado Pago, Webpay). La comisión de $2.000 también es provisional — sujeta a la estructura legal definitiva de Nexo.

---

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
| test_score | int | puntaje del test (sobre 15) |
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

**`favoritos`**

| columna | tipo | notas |
|---|---|---|
| id | bigint identity | PK |
| cliente_id | uuid | FK → `auth.users(id)` |
| profesional_id | bigint | FK → `profesionales(id)` |
| unique | (cliente_id, profesional_id) | evita duplicados |

---

## Limitaciones conocidas / TODO

- Pagos simulados, sin procesador real.
- Bucket de Storage público — CVs y títulos accesibles por URL directa.
- Sin rate-limiting ni captcha en formularios públicos.
- Mapeo de categorías en home (`categoryMap`) es aproximado. Para precisión exacta, agregar columna `categoria` al formulario de registro.
- El panel admin da acceso a cualquier usuario `authenticated` — no distingue entre usuarios normales y admins. Para producción real, agregar un check de rol explícito (columna `role` en una tabla `perfiles`, o usar claims de Supabase Auth).
- Sin tests automatizados.

---

## Seguridad

- La clave en `index.html` es la **anon/legacy key** — diseñada para ser pública. La protección real son las **políticas RLS** de cada tabla.
- Sin RLS, cualquier tabla es de libre lectura/escritura con la anon key. Siempre habilita RLS al crear tablas nuevas.
- El panel admin usa Supabase Auth — cualquier usuario `authenticated` puede ingresar. Para mayor seguridad, usa claims o una tabla de roles explícita.
- La clave pública de EmailJS puede exponerse; el límite es el plan (200 correos/mes en free).
- Nunca subas al repo la **service role key** ni claves privadas. Si en algún momento se agrega una Edge Function, sus credenciales van en variables de entorno del servidor.
