## Purpose

Permite a un socio autenticado transferir saldo entre dos cuentas de ahorro propias, sin comisión ni límites en v1, con locking determinístico de ambas cuentas dentro de una única transacción atómica y auditoría de las dos patas del movimiento.

## ADDED Requirements

### Requirement: Endpoint de transferencia entre cuentas propias

El sistema SHALL exponer `POST /api/v1/ahorros/transferencias` (cuenta origen, cuenta destino, monto, glosa opcional), protegido por la dependencia de autoservicio de `autoservicio-socio`, que exige que **ambas** cuentas tengan `socio_id` igual al del socio autenticado. Cuenta ajena (origen o destino) SHALL responder 404 sin revelar existencia.

#### Scenario: Transferencia exitosa entre cuentas propias
- **GIVEN** un socio autenticado con dos cuentas propias `ACTIVA`, misma moneda, con saldo suficiente en la cuenta origen
- **WHEN** solicita `POST /api/v1/ahorros/transferencias` con monto válido
- **THEN** el sistema descuenta el monto de la cuenta origen y lo acredita en la cuenta destino, dejando ambos saldos consistentes

#### Scenario: Cuenta origen o destino ajena
- **GIVEN** un socio autenticado y una cuenta origen o destino cuyo `socio_id` no le pertenece
- **WHEN** solicita la transferencia
- **THEN** el sistema responde 404 sin revelar si la cuenta existe, y no modifica ningún saldo

### Requirement: Validaciones de negocio de la transferencia

El sistema SHALL rechazar la transferencia cuando: la cuenta origen es igual a la destino; alguna cuenta no está `ACTIVA`; las cuentas tienen `moneda_id` distinto (sin conversión de moneda en v1); el monto no es mayor a cero; o el monto excede el `saldo_disponible` de la cuenta origen.

#### Scenario: Origen igual a destino
- **WHEN** el socio solicita una transferencia con la misma cuenta como origen y destino
- **THEN** el sistema rechaza la solicitud (400) y no modifica saldos

#### Scenario: Cuenta inactiva
- **WHEN** la cuenta origen o destino no está en estado `ACTIVA`
- **THEN** el sistema rechaza la solicitud (400) y no modifica saldos

#### Scenario: Monedas distintas
- **WHEN** la cuenta origen y la destino tienen `moneda_id` distinto
- **THEN** el sistema rechaza la solicitud (400) sin realizar conversión de moneda

#### Scenario: Saldo insuficiente
- **WHEN** el monto solicitado excede el `saldo_disponible` de la cuenta origen
- **THEN** el sistema rechaza la solicitud (400) y no modifica saldos

### Requirement: Locking determinístico y atomicidad

El sistema SHALL bloquear ambas cuentas (`SELECT ... FOR UPDATE`) dentro de una única transacción, en orden determinístico por `id` ascendente, para evitar deadlocks entre transferencias concurrentes. Toda la operación SHALL ser atómica: ante cualquier error, el sistema SHALL revertir ambos saldos sin dejar cambios parciales.

#### Scenario: Transferencias concurrentes sobre las mismas cuentas
- **GIVEN** dos transferencias concurrentes que involucran el mismo par de cuentas en cualquier orden
- **WHEN** ambas se ejecutan simultáneamente
- **THEN** el sistema las serializa sin producir deadlock ni saldo negativo

#### Scenario: Error a mitad de la operación
- **GIVEN** una transferencia en curso que falla antes de confirmar
- **WHEN** ocurre el error
- **THEN** el sistema revierte la transacción completa y ningún saldo queda modificado

### Requirement: Auditoría de ambas patas

Cada transferencia exitosa SHALL generar exactamente dos filas en `transaccion` (`TRANSFERENCIA_SALIDA` en la cuenta origen y `TRANSFERENCIA_ENTRADA` en la cuenta destino) vinculadas entre sí mediante una columna self-FK nullable, además de una entrada de bitácora (`registrar_accion()`, `modulo='AHORROS'`).

#### Scenario: Registro de transacción vinculada
- **WHEN** una transferencia se completa exitosamente
- **THEN** el sistema crea dos filas `transaccion` vinculadas entre sí, una por cada cuenta

#### Scenario: Registro de bitácora
- **WHEN** una transferencia se completa exitosamente
- **THEN** el sistema registra una entrada de bitácora en el módulo `AHORROS` describiendo el movimiento

### Requirement: Sin comisión ni límites en v1

El sistema SHALL NOT cobrar comisión ni aplicar límites de monto o frecuencia a las transferencias entre cuentas propias en esta versión.

#### Scenario: Transferencia sin cargo de comisión
- **WHEN** una transferencia se completa exitosamente
- **THEN** el monto acreditado en destino es igual al monto debitado en origen, sin descuentos adicionales

#### Scenario: Transferencia de monto elevado dentro del saldo disponible
- **GIVEN** una cuenta origen con saldo disponible suficiente
- **WHEN** el socio solicita una transferencia de ese monto, sin importar su magnitud
- **THEN** el sistema la procesa sin aplicar límite alguno de monto o frecuencia
