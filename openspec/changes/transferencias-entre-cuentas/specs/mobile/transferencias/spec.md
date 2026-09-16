## Purpose

Habilita en la app móvil un flujo de autoservicio para transferir saldo entre cuentas propias del socio, consumiendo `POST /api/v1/ahorros/transferencias` con el mismo patrón de servicio, inyección y manejo de errores que `cuentas_service.dart`.

## ADDED Requirements

### Requirement: Servicio de transferencias

El sistema SHALL implementar `transferencias_service.dart` con un cliente HTTP inyectable, adjuntando el JWT desde `AuthService.tokenJWT`, y SHALL devolver siempre un resultado tipado (never-throw) con mensajes de error en español, siguiendo el patrón de `CuentasService`.

#### Scenario: Transferencia exitosa
- **GIVEN** una sesión activa con JWT válido
- **WHEN** el servicio invoca `POST /api/v1/ahorros/transferencias` con datos válidos y el backend responde éxito
- **THEN** el servicio retorna un resultado `success` con los datos de la transferencia

#### Scenario: Error de backend o red
- **GIVEN** una sesión activa
- **WHEN** el backend responde con error (401, 400, 404) o la conexión falla
- **THEN** el servicio retorna un resultado `failure` con un mensaje en español, sin lanzar excepciones

### Requirement: Pantalla de transferencia entre cuentas propias

El sistema SHALL mostrar una pantalla de transferencia, accesible desde dashboard/portfolio, que permite seleccionar cuenta origen y destino entre las cuentas reales del socio (obtenidas de `CuentasService`), ingresar un monto y confirmar antes de enviar la solicitud.

#### Scenario: Transferencia confirmada exitosamente
- **GIVEN** el socio con al menos dos cuentas propias activas
- **WHEN** selecciona origen, destino y monto válidos, y confirma
- **THEN** la pantalla muestra el resultado exitoso y los saldos actualizados de ambas cuentas

#### Scenario: Envío bloqueado por datos incompletos
- **GIVEN** el formulario de transferencia
- **WHEN** el socio intenta enviar sin seleccionar cuenta destino o sin ingresar un monto válido
- **THEN** la pantalla bloquea el envío y muestra una indicación de los datos faltantes

### Requirement: Manejo de rechazos de negocio en la UI

El sistema SHALL mostrar mensajes claros en español cuando el backend rechaza la transferencia por reglas de negocio (monedas distintas, saldo insuficiente, cuenta inactiva o ajena), sin dejar la pantalla en un estado inconsistente.

#### Scenario: Rechazo por moneda distinta
- **WHEN** el backend rechaza la transferencia por monedas distintas entre las cuentas
- **THEN** la pantalla muestra un mensaje claro y permite corregir la selección

#### Scenario: Rechazo por saldo insuficiente
- **WHEN** el backend rechaza la transferencia por saldo insuficiente
- **THEN** la pantalla muestra un mensaje claro y no descuenta el monto localmente
