## Why

La plataforma SI2 comienza desde cero. Para que socios, asesores y administradores puedan usar cualquier módulo posterior (socios, créditos, caja, reportes), hace falta un MVP que autentique usuarios por rol y provea el esqueleto web navegable. Sin autenticación ni layout base no existe una base segura sobre la que construir el resto de la cooperativa.

## What Changes

- **SI2_backend_api**: nuevo módulo de autenticación con login/logout/me, JWT y roles.
  - Endpoint `POST /api/v1/auth/login` (correo + contraseña → JWT).
  - Endpoint `POST /api/v1/auth/logout`.
  - Endpoint `GET /api/v1/auth/me` (usuario autenticado + rol).
  - Modelos `USUARIO`, `ROL`, `PERMISO`, `ROL_PERMISO` en PostgreSQL (relación muchos a muchos).
  - Hash de contraseña con bcrypt.
  - JWT con expiración de 8 horas.
- **SI2_frontend_web**: pantalla de login, layout post-login con sidebar según rol y protección de rutas.
  - Pantalla de Login (formulario correo + contraseña, manejo de errores).
  - Layout principal con sidebar de navegación dinámico según el rol del usuario:
    - Administrador: Dashboard, Socios, Créditos, Caja, Reportes, Configuración.
    - Asesor: Dashboard, Socios, Créditos, Caja.
    - Socio: Dashboard, Mi Cuenta, Mis Créditos.
  - Guard de rutas (redirige al login sin token) e interceptor HTTP que adjunta el JWT.
  - Página de Dashboard básico (bienvenida + datos del usuario).

## Capabilities

### New Capabilities
- `auth`: Autenticación de usuarios por correo/contraseña, emisión/validación de JWT y obtención del usuario autenticado con su rol (backend FastAPI).
- `web/layout`: Esqueleto web Angular con login, layout post-login, sidebar según rol, guard de rutas, interceptor HTTP y dashboard básico.

### Modified Capabilities
- Ninguna. No existen specs previas en `openspec/specs/`.

## Impact

- **SI2_backend_api**:
  - Nuevos modelos: `USUARIO`, `ROL`, `PERMISO`, `ROL_PERMISO`.
  - Nuevos endpoints bajo `/api/v1/auth/*`.
  - Nuevas dependencias: `bcrypt`, `python-jose` (o `PyJWT`).
  - Migración de base de datos para las tablas de autenticación.
- **SI2_frontend_web**:
  - Nuevos componentes: login, layout/sidebar, dashboard.
  - Nuevos servicios Angular de autenticación y guard de rutas.
  - Interceptor HTTP para inyección del token.
  - Almacenamiento del JWT (localStorage/sessionStorage) y expiración de 8 horas.
- **SI2_mobile_app**: sin cambios (fuera de alcance del MVP).
- **SI2_docs_and_agents**: spec y memoria de decisiones en Engram.
