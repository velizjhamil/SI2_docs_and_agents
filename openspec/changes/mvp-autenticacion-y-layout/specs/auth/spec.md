## Purpose

Autentica usuarios de la cooperativa por correo y contraseña, emite tokens JWT con expiración y expone el usuario autenticado con su rol para proteger el acceso a la API.

## ADDED Requirements

### Requirement: Autenticación por correo y contraseña

El sistema SHALL autenticar a los usuarios mediante su correo y contraseña a través de `POST /api/v1/auth/login`, retornando un JWT. La contraseña SHALL almacenarse únicamente como hash con bcrypt y nunca en texto plano. El correo SHALL ser único por usuario.

#### Scenario: Login exitoso
- **WHEN** un usuario envía credenciales válidas (correo y contraseña correctos) a `POST /api/v1/auth/login`
- **THEN** el sistema retorna un JWT con expiración de 8 horas y los datos básicos del usuario

#### Scenario: Login con contraseña incorrecta
- **WHEN** un usuario envía una contraseña incorrecta a `POST /api/v1/auth/login`
- **THEN** el sistema retorna un error de autenticación (401) sin revelar si el correo existe

#### Scenario: Login con usuario inexistente
- **WHEN** un usuario envía un correo que no existe a `POST /api/v1/auth/login`
- **THEN** el sistema retorna un error de autenticación (401) sin revelar si el correo existe

#### Scenario: Usuario inactivo
- **WHEN** un usuario con estado inactivo intenta iniciar sesión
- **THEN** el sistema rechaza el login con un error de autenticación (401)

### Requirement: Cierre de sesión

El sistema SHALL permitir cerrar la sesión a través de `POST /api/v1/auth/logout`, invalidando el token JWT del usuario autenticado.

#### Scenario: Logout con token válido
- **WHEN** un usuario autenticado envía su token a `POST /api/v1/auth/logout`
- **THEN** el sistema invalida el token y retorna confirmación de cierre de sesión

#### Scenario: Logout sin token válido
- **WHEN** una solicitud sin token o con token inválido llega a `POST /api/v1/auth/logout`
- **THEN** el sistema retorna un error de autenticación (401)

### Requirement: Obtener usuario autenticado

El sistema SHALL exponer `GET /api/v1/auth/me` que retorna el usuario autenticado junto con su rol.

#### Scenario: Consulta con token válido
- **WHEN** un usuario autenticado envía su token a `GET /api/v1/auth/me`
- **THEN** el sistema retorna los datos del usuario, su rol y sus permisos

#### Scenario: Consulta sin token válido
- **WHEN** una solicitud sin token o con token inválido o expirado llega a `GET /api/v1/auth/me`
- **THEN** el sistema retorna un error de autenticación (401)

### Requirement: Protección de endpoints autenticados

Los endpoints del sistema SHALL exigir un JWT válido y no expirado. El JWT SHALL tener una expiración máxima de 8 horas desde su emisión.

#### Scenario: Token expirado
- **WHEN** un usuario envía un JWT que supera las 8 horas de vigencia a un endpoint protegido
- **THEN** el sistema rechaza la solicitud con un error de autenticación (401)

#### Scenario: Token con firma inválida
- **WHEN** un usuario envía un JWT con firma inválida o alterada a un endpoint protegido
- **THEN** el sistema rechaza la solicitud con un error de autenticación (401)

### Requirement: Modelo de datos de usuarios y roles

El sistema SHALL persistir en PostgreSQL los usuarios y los roles con sus permisos. La relación SHALL ser muchos a muchos entre usuarios y roles, y entre roles y permisos, mediante las tablas `USUARIO`, `ROL`, `ROL_PERMISO` y `PERMISO`. El usuario SHALL tener: id, nombre, contraseña (hash), correo, estado y fecha_creacion.

#### Scenario: Creación de migración de autenticación
- **WHEN** se aplican las migraciones de base de datos
- **THEN** se crean las tablas `USUARIO`, `ROL`, `ROL_PERMISO` y `PERMISO` con sus relaciones y claves

#### Scenario: Carga de roles iniciales
- **WHEN** se inicializa la base de datos
- **THEN** existen los roles administrador, asesor y socio con sus permisos asignados
