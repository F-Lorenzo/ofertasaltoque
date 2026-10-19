# Ofertas al Toque

Plataforma argentina de descubrimiento, comparación y publicación de **ofertas verificadas**. Monetiza con el programa de afiliados de Mercado Libre Argentina.

> **Ofertas reales. Ahorros verificados.** — Comparamos precios para encontrar oportunidades que valen la pena.

Este repositorio contiene la **Fase 1: web funcional**. El resto del negocio (motor de precios, carruseles, distribución, monitoreo) está diseñado en el esquema y la arquitectura, pero **todavía no está implementado**: ver [docs/FASES.md](docs/FASES.md).

## Qué hace hoy (Fase 1)

**Web pública** — catálogo de oportunidades: portada, categorías, buscador (nombre, marca, modelo, categoría o código `OF023`), filtros, orden, paginación, fichas de producto, redirección de afiliado con registro de clics y páginas informativas (metodología, privacidad, términos, aviso de afiliados).

**Panel `/admin`** (Supabase Auth + roles + RLS) — alta y edición de productos, imágenes, enlaces de afiliado, publicar/despublicar/archivar, estado de validación, categorías, vista de afiliados y configuración de umbrales. Los módulos de las fases siguientes aparecen deshabilitados y rotulados con su fase.

**Reglas de honestidad que el sistema impone (en la base de datos, no solo en la interfaz):**

- Se distingue siempre el **descuento anunciado por Mercado Libre** (dato informativo, gris) del **ahorro real** (calculado por nosotros, verde).
- El ahorro real solo existe si el producto tiene una validación vigente. **Nadie del panel puede marcar un producto como validado**: lo impide un trigger. Hoy no hay motor de validación, así que todas las ofertas se muestran como *“Ahorro real sin verificar”*.
- No se publica una oferta sin imagen, precio verificado, disponibilidad confirmada y enlace de afiliado activo (trigger `publish_requirements`).
- No se emiten datos estructurados (JSON-LD) con precio para ofertas no verificadas.
- No hay datos de demostración en producción: los datos ficticios existen solo en la pila local de pruebas (`.stack/`).

## Stack

Next.js 16 (App Router) · React 19 · TypeScript estricto · Tailwind CSS v4 · Radix UI + componentes propios al estilo shadcn/ui · Lucide · Supabase (PostgreSQL, Auth, Storage, RLS) · Zod · Vitest · PGlite · Playwright.

## Puesta en marcha

### 1. Proyecto de Supabase

Creá un proyecto gratuito en [supabase.com](https://supabase.com) y anotá la **URL** y la clave **anon/publishable** (Project Settings → API).

### 2. Aplicar las migraciones

Las migraciones están en `supabase/migrations/` (orden por nombre). Con la CLI de Supabase:

```bash
npx supabase login
npx supabase link --project-ref <tu-project-ref>
npx supabase db push
```

o pegando cada archivo, en orden, en el **SQL Editor** de Supabase. Crean el esquema completo (25 tablas), triggers, vistas públicas, políticas RLS, el bucket `product-images` y los datos de referencia (categorías, roles, umbrales y comercios candidatos).

### 3. Desactivar el registro público

Authentication → Sign In / Providers → Email → desactivá **“Allow new users to sign up”**. Los administradores nunca se registran solos: se crean con el script del paso 5.

### 4. Variables de entorno

```bash
cp .env.example .env.local
```

Completá `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `NEXT_PUBLIC_SITE_URL`, `CLICK_HASH_SALT` (cadena larga y aleatoria) y, opcionalmente, `NEXT_PUBLIC_CONTACT_EMAIL`. Para el paso 5 también `SUPABASE_SERVICE_ROLE_KEY`.

> La `service_role` **solo** se usa en scripts de operación y workers. La aplicación web no la usa: toda su seguridad descansa en RLS.

### 5. Crear el primer administrador

```bash
npm install
npm run admin:create -- --email vos@tudominio.com --role admin
```

Genera una contraseña aleatoria y la muestra una sola vez (o pasá `--password` con 12+ caracteres). Después podés quitar `SUPABASE_SERVICE_ROLE_KEY` de `.env.local`.

### 6. Correr

```bash
npm run dev
```

- Web: http://localhost:3000
- Panel: http://localhost:3000/admin

### Primer flujo de trabajo

1. `/admin/productos/nuevo`: cargá el producto (nombre, categoría, precio, URL de Mercado Libre…).
2. En su ficha: subí imágenes, pegá el **enlace de afiliado** (se guarda exactamente como lo pegues) y marcá “Verifiqué el precio y la disponibilidad ahora”.
3. Cuando el panel de publicación muestre todos los requisitos en verde: **Publicar**. Aparece en la web y en el sitemap.

## Despliegue (Vercel)

Variables: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `NEXT_PUBLIC_SITE_URL`, `CLICK_HASH_SALT`, `NEXT_PUBLIC_CONTACT_EMAIL`. **No** definas la `service_role` en Vercel. Las páginas públicas se regeneran cada 5 minutos (ISR) y el panel invalida la caché al publicar o editar.

## Scripts

| Comando | Qué hace |
| --- | --- |
| `npm run dev` | Servidor de desarrollo |
| `npm run check` | Typecheck + lint + tests (unitarios y de base de datos) |
| `npm run test:unit` / `test:db` | Pruebas unitarias / migraciones y RLS sobre Postgres real (PGlite) |
| `npm run test:e2e` | Pruebas end-to-end con Playwright sobre la pila local |
| `npm run admin:create` | Alta de administradores/editores (usa `service_role`) |
| `npm run stack:setup` / `stack:demo` / `dev:stack` | Pila local sin Docker para pruebas y desarrollo ([docs/PRUEBAS.md](docs/PRUEBAS.md)) |

## Estructura

```
supabase/migrations/   esquema, triggers, RLS, vistas públicas, storage, datos de referencia
src/app/(public)/      web pública (/, /ofertas, /categoria, /oferta/[slug], páginas informativas)
src/app/go/[slug]/     redirección de afiliado con registro de clic
src/app/admin/         login y panel
src/components/{ui,site,admin}/
src/domain/            cálculo determinista de ahorro (puro, compartido con los futuros workers)
src/lib/               clientes de Supabase, datos, utilidades, lógica del panel
tests/{unit,db,e2e}/   Vitest, PGlite (SQL + RLS), Playwright
scripts/               alta de administradores y pila local de pruebas
workers/               reservado para scraping y automatización (Fase 2+)
docs/                  arquitectura, pruebas y roadmap
```

## Documentación

- [docs/ARQUITECTURA.md](docs/ARQUITECTURA.md) — arquitectura, modelo de datos, flujo, diseño y decisiones.
- [docs/PRUEBAS.md](docs/PRUEBAS.md) — estrategia de pruebas y pila local.
- [docs/FASES.md](docs/FASES.md) — qué falta y con qué contratos se construye.

## Aviso

Los textos legales (privacidad, términos) describen el funcionamiento real del sitio pero **deben ser revisados por un profesional** antes de operar comercialmente.
# ofertasaltoque
