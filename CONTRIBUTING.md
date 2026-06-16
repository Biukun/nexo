# Contribuir a Nexo

Este repositorio es **privado y de uso restringido**. Si tienes acceso, es porque fuiste invitado explícitamente al proyecto. No redistribuyas el código ni lo uses fuera del contexto de Nexo.

---

## Antes de tocar algo

1. Lee el `README.md` completo — especialmente las secciones de [Setup desde cero](README.md#setup-desde-cero) y [Seguridad](README.md#seguridad).
2. Confirma que tienes las variables de entorno correctas (Supabase URL + anon key, EmailJS). **Nunca subas claves al repo**, ni siquiera en un commit "temporal".
3. Si vas a trabajar en algo que afecta la base de datos (nuevas tablas, cambios de esquema, políticas RLS), coordina antes — un SQL mal aplicado en producción no tiene "undo" fácil.

---

## Flujo de trabajo

```
main          → rama de producción (lo que está en biukun.github.io/nexo)
dev           → rama de desarrollo activo
feature/xxx   → ramas para funcionalidades nuevas
fix/xxx       → ramas para correcciones puntuales
```

1. **Nunca hagas push directo a `main`** — abre un Pull Request desde `dev` o desde tu rama.
2. Crea tu rama desde `dev`, no desde `main`:
   ```bash
   git checkout dev
   git pull
   git checkout -b feature/nombre-descriptivo
   ```
3. Commits en español, cortos y descriptivos:
   ```
   ✅ Bien:   "fix: corrección de política RLS en tabla solicitudes"
   ✅ Bien:   "feat: contador real de categorías desde Supabase"
   ❌ Mal:    "arreglos"
   ❌ Mal:    "update index.html"
   ```
4. Abre un Pull Request hacia `dev` con una descripción breve de qué cambió y por qué.

---

## Qué no tocar sin coordinación previa

- **`SUPABASE_KEY`** en `index.html` — si la clave cambia, hay que actualizarla en todos los entornos a la vez.
- **Estructura de tablas en Supabase** — cualquier `ALTER TABLE` o `DROP` va en un archivo `.sql` nuevo en el repo, documentado, antes de aplicarse en producción.
- **El bucket `documentos`** — tiene CVs y documentos de profesionales reales. No cambies permisos del bucket sin revisar las implicancias de privacidad.
- **El flujo de pagos** (`simularPago`) — está marcado explícitamente como simulación. No conectes un procesador real sin antes implementar el backend correspondiente (Edge Function).

---

## Reportar un bug

Abre un Issue en GitHub con:
- **Qué esperabas** que pasara
- **Qué pasó** (incluye el error de consola si hay uno)
- **Cómo reproducirlo** (pasos exactos)
- **Entorno**: navegador, dispositivo, si es mobile o desktop

---

## Preguntas

Contacto directo con el equipo: [contact.us.nexo@gmail.com](mailto:contact.us.nexo@gmail.com)
