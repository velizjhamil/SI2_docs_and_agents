## Purpose

Proporciona el esqueleto web de la plataforma: pantalla de login, layout con navegación por roles, protección de rutas e integración del token JWT con el backend.

## ADDED Requirements

### Requirement: Pantalla de login

El sistema SHALL mostrar una pantalla de login con formulario de correo y contraseña que consume `POST /api/v1/auth/login`, gestiona errores y redirige al dashboard tras autenticarse.

#### Scenario: Login exitoso
- **WHEN** el usuario ingresa credenciales válidas y envía el formulario
- **THEN** el sistema almacena el JWT y redirige al dashboard correspondiente

#### Scenario: Credenciales inválidas
- **WHEN** el usuario ingresa credenciales inválidas y envía el formulario
- **THEN** el sistema muestra un mensaje de error claro sin redirigir

#### Scenario: Errores de red o servidor
- **WHEN** el backend no responde o retorna un error de servidor
- **THEN** el sistema muestra un mensaje de error indicando el problema

### Requirement: Layout principal con sidebar según rol

Tras iniciar sesión, el sistema SHALL mostrar un layout con un sidebar de navegación cuyos items dependan del rol del usuario autenticado. Administrador: Dashboard, Socios, Créditos, Caja, Reportes, Configuración. Asesor: Dashboard, Socios, Créditos, Caja. Socio: Dashboard, Mi Cuenta, Mis Créditos.

#### Scenario: Sidebar de administrador
- **WHEN** un usuario con rol administrador inicia sesión
- **THEN** el sidebar muestra Dashboard, Socios, Créditos, Caja, Reportes y Configuración

#### Scenario: Sidebar de asesor
- **WHEN** un usuario con rol asesor inicia sesión
- **THEN** el sidebar muestra Dashboard, Socios, Créditos y Caja

#### Scenario: Sidebar de socio
- **WHEN** un usuario con rol socio inicia sesión
- **THEN** el sidebar muestra Dashboard, Mi Cuenta y Mis Créditos

### Requirement: Guard de rutas

El sistema SHALL proteger las rutas autenticadas mediante un guard que redirige al login cuando no existe un token válido.

#### Scenario: Acceso autenticado a ruta protegida
- **WHEN** un usuario con JWT navega a una ruta protegida
- **THEN** el sistema permite el acceso

#### Scenario: Acceso sin token a ruta protegida
- **WHEN** un usuario sin JWT intenta acceder a una ruta protegida
- **THEN** el sistema redirige a la pantalla de login

#### Scenario: Acceso a login estando autenticado
- **WHEN** un usuario autenticado intenta acceder a la pantalla de login
- **THEN** el sistema redirige al dashboard

### Requirement: Interceptor HTTP de autenticación

El sistema SHALL adjuntar el JWT como encabezado de autorización en cada solicitud HTTP saliente mediante un interceptor HTTP.

#### Scenario: Inyección del token en cada request
- **WHEN** el frontend realiza una solicitud HTTP autenticada
- **THEN** la solicitud incluye el encabezado de autorización con el JWT

#### Scenario: Respuesta 401
- **WHEN** el backend retorna 401 en una respuesta
- **THEN** el sistema limpia el token almacenado y redirige al login

### Requirement: Dashboard básico

El sistema SHALL mostrar una página de Dashboard con un mensaje de bienvenida y los datos del usuario autenticado.

#### Scenario: Visualización del dashboard
- **WHEN** un usuario autenticado accede al Dashboard
- **THEN** el sistema muestra un mensaje de bienvenida y los datos del usuario (nombre, correo y rol)
