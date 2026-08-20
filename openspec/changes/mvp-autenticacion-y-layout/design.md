## Context

La plataforma SI2 parte de cero: no existen specs previas en `openspec/specs/` ni código base de autenticación en `SI2_backend_api` ni `SI2_frontend_web`. El stack es FastAPI + PostgreSQL (backend) y Angular (frontend). Ver `proposal.md` para motivación; los requisitos de comportamiento están en `specs/auth/spec.md` y `specs/web/layout/spec.md`.

## Goals / Non-Goals

**Goals:**
- Autenticación stateless con JWT en el backend FastAPI.
- Persistencia de usuarios, roles y permisos en PostgreSQL con migraciones versionadas.
- Esqueleto Angular navegable con login, sidebar por rol, guard e interceptor.
- Documentar explícitamente qué archivos se crean/modifican en cada repo.

**Non-Goals:**
- Módulos funcionales (socios, créditos, caja, reportes): solo se crean las rutas de navegación, no su funcionalidad.
- Refresh tokens: se usa un único JWT con expiración de 8 horas.
- App móvil (Flutter): fuera del alcance del MVP.
- BCRYPT/seguridad avanzada de sesión revocable: logout será stateless (ver Decisión 2).

## Decisions

### 1. Almacenamiento de contraseñas con bcrypt
Se usa `passlib[bcrypt]` para el hash y verificación de contraseñas. El hash se guarda en la columna `contraseña` de `USUARIO`.
- Alternativa descartada: `sha256` plano (inseguro) y `argon2` (más fuerte pero más costo de cómputo; bcrypt es estándar y suficiente para el MVP).

### 2. Sesiones stateless: JWT de acceso único, logout por invalidación en lista negra (opcional) o stateless
Se emite un JWT firmado con `python-jose` (HS256), con `exp = now + 8h` e incluyendo `sub` (id de usuario) y `role`. Para `logout`:
- MVP simple: el frontend descarta el token; el backend responde 200 y no valida estado (stateless).
- Opcional robusto: tabla en memoria/DB de tokens revocados validada en cada request. Se documenta como mejora, no como requisito del MVP.
- Alternativa descartada: refresh tokens + OAuth2 flow, innecesario para el alcance inicial.

### 3. Modelo de datos relacional
Tablas `USUARIO`, `ROL`, `PERMISO`, `ROL_PERMISO` con SQLAlchemy (ORM) y migraciones con Alembic. Relaciones:
- `USUARIO.rol_id` → `ROL.id` (un rol por usuario en el MVP; la relación usuarios-roles se modela como N:1, suficiente para los 3 roles del negocio).
- `ROL_PERMISO` (id_rol, id_permiso) puente N:M entre `ROL` y `PERMISO`.
- Seed inicial de roles: administrador, asesor, socio.

### 4. Estructura de la API (backend)
Rutas bajo `/api/v1/auth/`: `login`, `logout`, `me`. Dependencia de autenticación FastAPI (`HTTPBearer`) que valida el JWT y se reutiliza en endpoints protegidos. `me` retorna usuario + rol (+ permisos).

### 5. Frontend Angular
- Servicio `AuthService` (login, logout, estado, almacenamiento del token en `localStorage`).
- `AuthGuard` para rutas protegidas y para redirigir login→dashboard.
- `Interceptor` HTTP que añade `Authorization: Bearer <token>` y maneja 401 (limpia token + redirige a login).
- Componentes: `login`, `layout` (con `sidebar` dinámico por rol), `dashboard`.
- `SidebarService` o pipe de roles para decidir los items visibles según el rol.
- Router con rutas hijas bajo el layout.

### 6. Archivos por repositorio
Ver `tasks.md` para el desglose; a alto nivel:
- **SI2_backend_api**: `app/models/` (usuario, rol, permiso), `app/schemas/`, `app/routers/auth.py`, `app/security.py`, `app/deps.py`, migración Alembic + seed.
- **SI2_frontend_web**: `src/app/auth/` (login, service, guard, interceptor), `src/app/layout/` (layout, sidebar), `src/app/pages/dashboard/`, config de rutas y módulo.

## Risks / Trade-offs

- **Logout stateless no revoca el token en el servidor** → El token sigue válido hasta expirar (8h). Mitigación: el frontend elimina el token; para requisitos de seguridad estrictos añadir lista de revocación (documentado como mejora).
- **Rol único por usuario limita multi-rol** → Suficiente para los 3 roles del negocio; migrar a tabla puente usuarios-roles si surge la necesidad.
- **JWT en localStorage es vulnerable a XSS** → Aceptado para el MVP; considerar `httpOnly` cookies + refresh si se endurece la seguridad.
- **bcrypt costo/rendimiento en registros masivos** → Impacto bajo en el MVP; ajustable con el factor de costo.
- **Sync de roles/permisos entre backend y sidebar** → El sidebar se deriva del rol devuelto por `me`; mantener contrato de roles consistente entre ambos repos.