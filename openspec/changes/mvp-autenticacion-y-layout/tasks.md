## 1. Backend — Modelos y seguridad (SI2_backend_api)

- [x] 1.1 Crear modelos SQLAlchemy en `app/models/models.py`: `Usuario` (USUARIO), `Rol` (ROL), `Permiso` (PERMISO) y tabla `rol_permiso` (ROL_PERMISO), alineados al esquema real de `bd.sql`
- [x] 1.2 Dependencias `bcrypt` y `python-jose` ya presentes en `requirements.txt` (sin cambios); se usa `bcrypt` directamente por incompatibilidad passlib 1.7.4 + bcrypt 5.0.0
- [x] 1.3 Crear `app/core/security.py`: hash/verificación con bcrypt y creación/decodificación de JWT (HS256, expiración 8h vía settings)
- [x] 1.4 Crear `app/api/v1/deps.py`: dependencia FastAPI `HTTPBearer` (`get_current_user`) que valida el JWT y resuelve el usuario actual
- [x] 1.5 Crear los schemas Pydantic en `app/schemas/schemas.py`: `LoginRequest`, `TokenResponse`, `UserOut` (usuario + rol + permisos), `RolOut`, `PermisoOut`

## 2. Backend — Router de autenticación y migración (SI2_backend_api)

- [x] 2.1 Crear `app/api/v1/endpoints/auth.py` con `POST /api/v1/auth/login`, `POST /api/v1/auth/logout` y `GET /api/v1/auth/me`
- [x] 2.2 Registrar el router `/api/v1/auth` en `app/api/v1/router.py` y montarlo en `main.py` con prefijo `/api/v1`
- [ ] 2.3 Crear migración Alembic que crea las tablas `USUARIO`, `ROL`, `ROL_PERMISO` y `PERMISO` con sus claves y relaciones (el esquema ya está definido en `bd.sql`; se puede arrancar sin Alembic)
- [ ] 2.4 Seed/inicialización: los roles/permisos ya se cargan en `bd.sql`, pero las contraseñas sembradas son hashes falsos; falta un usuario con hash bcrypt real

## 3. Frontend — Servicio de autenticación, guard e interceptor (SI2_frontend_web)

- [ ] 3.1 Crear `src/app/auth/auth.service.ts`: login, logout, obtención de estado y almacenamiento del JWT en `localStorage`
- [ ] 3.2 Crear el modelo/dto de usuario (`User` con rol) en `src/app/models/` y un servicio de roles para mapear items de navegación
- [ ] 3.3 Crear `src/app/auth/auth.guard.ts` (guarda rutas protegidas; redirige a login sin token y login→dashboard si autenticado)
- [ ] 3.4 Crear `src/app/auth/auth.interceptor.ts`: inyecta `Authorization: Bearer <token>` en cada request y maneja 401 (limpia token + redirige a login)

## 4. Frontend — Login, layout y dashboard (SI2_frontend_web)

- [ ] 4.1 Crear `src/app/auth/pages/login/` (formulario correo + contraseña, manejo de errores, consumo de `POST /api/v1/auth/login`, redirección al dashboard)
- [ ] 4.2 Crear `src/app/layout/` con `layout.component.ts` y `sidebar.component.ts` (sidebar dinámico según rol del usuario: administrador → Dashboard, Socios, Créditos, Caja, Reportes, Configuración; asesor → Dashboard, Socios, Créditos, Caja; socio → Dashboard, Mi Cuenta, Mis Créditos)
- [ ] 4.3 Crear `src/app/pages/dashboard/` con mensaje de bienvenida y datos del usuario autenticado
- [ ] 4.4 Configurar el router en `app-routing.module.ts` con rutas hijas bajo el layout protegidas por `AuthGuard`

## 5. Verificación

- [ ] 5.1 Backend: ejecutar migraciones y probar login/logout/me con usuarios de los tres roles (validar expiración y errores 401)
- [ ] 5.2 Frontend: compilar la app y verificar login, sidebar por rol, guard de rutas y dashboard
- [ ] 5.3 Documentar en Engram las decisiones de arquitectura tomadas (diseño de autenticación y layout)