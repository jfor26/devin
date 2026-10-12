# ShipLog

Bitácora de desarrollo: registra qué construiste en cada proyecto, cuánto te costó en tokens y dólares, y míralo agrupado por semana.

- Producto, modelo de datos y fases: [`plan.md`](plan.md)
- Dirección visual: [`design-plan.md`](design-plan.md)

## Funcionalidades

- **Login sin contraseña** con magic link de Supabase Auth.
- **Proyectos**: nombre, slug, descripción, URL del repo y estado (`active`, `paused`, `done`).
- **Entradas ("ships")**: título, cuerpo, tipo (`feature`, `bugfix`, `refactor`, `other`), tags, fecha de envío, tokens consumidos y costo en USD.
- **Timeline semanal** por proyecto, agrupada por semana ISO 8601 (lunes a domingo), con el total invertido.
- **Multiusuario**: cada usuario solo ve sus proyectos, garantizado con Row Level Security (RLS) en Postgres.

Pendiente (ver fases en `plan.md`): páginas públicas compartibles `/p/[slug]`, integración con GitHub y recordatorios por email.

## Stack

| Capa | Tecnología |
|---|---|
| Framework | Next.js 16 (App Router) + React 19 + TypeScript |
| UI | Tailwind CSS 4 + componentes shadcn (`src/components/ui`) |
| Backend | Supabase: Auth, Postgres y RLS (`@supabase/ssr`) |
| Entornos | `dotenv-cli` para elegir `.env.local` o `.env.prod` |

## Requisitos

- Node 22
- Docker (para correr Supabase en local; la CLI de Supabase viene como dependencia de desarrollo)

## Desarrollo local

```bash
npm install
npm run supabase:start     # imprime API URL y publishable key
cp .env.example .env.local # pega la publishable key
npm run dev                # http://localhost:3000
```

Login por magic link: escribe un correo en `/login` y abre el enlace en Mailpit (http://127.0.0.1:54324). El seed crea `demo@shiplog.local` con un proyecto de ejemplo.

### Servicios locales de Supabase

| Servicio | URL |
|---|---|
| API | http://127.0.0.1:54321 |
| Postgres | `postgresql://postgres:postgres@127.0.0.1:54322/postgres` |
| Studio | http://127.0.0.1:54323 |
| Mailpit (correos) | http://127.0.0.1:54324 |

## Variables de entorno y perfiles

Hay dos perfiles; cada script de npm carga el suyo con `dotenv-cli`:

| Perfil | Archivo | Scripts | Backend |
|---|---|---|---|
| Local | `.env.local` | `npm run dev`, `npm run build` | Supabase local |
| Producción | `.env.prod` | `npm run dev:prod`, `npm run build:prod` | Proyecto Supabase en la nube (datos reales) |

Ambos archivos parten de [`.env.example`](.env.example) y necesitan:

- `NEXT_PUBLIC_SUPABASE_URL`
- `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY`

Las variables `NEXT_PUBLIC_*` se inlinean al compilar, así que hay que reconstruir al cambiar de perfil. Los archivos `.env*` (salvo `.env.example`) están ignorados por git.

## Scripts

| Comando | Qué hace |
|---|---|
| `npm run dev` / `npm run dev:prod` | Servidor de desarrollo contra Supabase local / producción |
| `npm run build` / `npm run build:prod` | Build de producción con el perfil local / producción |
| `npm run start` | Sirve el build generado |
| `npm run lint` | ESLint |
| `npm run typecheck` | Chequeo de tipos con `tsc --noEmit` |
| `npm run supabase:start` / `supabase:stop` | Levanta / detiene Supabase local |
| `npm run supabase:reset` | Recrea la base local aplicando migraciones y `supabase/seed.sql` |
| `npm run db:types` | Regenera `src/lib/database.types.ts` desde el esquema local |

## Estructura del proyecto

```
src/
  app/
    (app)/                 # rutas autenticadas
      page.tsx             # /                     lista de proyectos
      projects/new/        # /projects/new         crear proyecto
      projects/[slug]/     # /projects/[slug]      detalle + timeline + nueva entrada
        edit/              # /projects/[slug]/edit editar proyecto
        entries/[id]/      # editar entrada
    actions/               # server actions: auth, projects, entries
    auth/callback/         # intercambio del código del magic link
    login/                 # formulario de magic link
  components/              # formularios, timeline, footer y ui/ (shadcn)
  lib/
    supabase/              # clientes de Supabase (browser, server, proxy)
    week.ts                # cálculo y agrupación por semana ISO
    constants.ts           # estados de proyecto y tipos de entrada
    database.types.ts      # tipos generados (npm run db:types)
  proxy.ts                 # refresca la sesión y redirige a /login si no hay usuario
supabase/
  migrations/              # esquema SQL, RLS y grants
  seed.sql                 # datos demo para local
  seed.prod.sql            # seed idempotente para producción (se corre a mano)
  config.toml              # configuración de Supabase local
```

## Base de datos

Dos tablas en `public`, definidas en `supabase/migrations/`:

- **`projects`**: pertenece a un usuario (`user_id`); `slug` único por usuario; `is_public` reservado para las páginas públicas.
- **`entries`**: pertenece a un proyecto (borrado en cascada); incluye `kind`, `tags`, `shipped_at`, `tokens` y `cost_usd`.

RLS: cada usuario tiene acceso total solo a sus filas (`user_id = auth.uid()`). La lectura anónima se permite únicamente para proyectos con `is_public = true` y sus entradas.

Para cambiar el esquema: crea una nueva migración en `supabase/migrations/`, corre `npm run supabase:reset` y luego `npm run db:types`.

## Autenticación

1. El usuario envía su correo desde `/login`.
2. Supabase envía el magic link (en local, llega a Mailpit).
3. `/auth/callback` intercambia el código por una sesión y redirige a la ruta original (`next`).
4. `src/proxy.ts` refresca la sesión en cada request y manda a `/login` a quien no esté autenticado.

Los redirects permitidos están en `supabase/config.toml` (`site_url` y `additional_redirect_urls`). En producción, el Site URL y los redirect URLs del proyecto remoto deben incluir el dominio/puerto en uso.

## Antes de abrir un PR

```bash
npm run lint
npm run typecheck
npm run build
```

> Este proyecto usa una versión de Next.js con cambios incompatibles respecto a versiones anteriores. Consulta `AGENTS.md` y la documentación en `node_modules/next/dist/docs/` antes de modificar código.
