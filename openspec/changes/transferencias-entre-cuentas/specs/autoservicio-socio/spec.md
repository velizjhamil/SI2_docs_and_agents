## Purpose

Provee una vía de autorización para operaciones que el propio socio autenticado inicia sobre sus recursos, distinta y separada de la autorización de staff (`require_operaciones`). Es el patrón reutilizable que los módulos de autoservicio (transferencias y futuros) consumen para resolver el `Socio` desde el usuario autenticado y restringir el acceso a recursos propios.

## ADDED Requirements

### Requirement: Dependencia de autorización de autoservicio

El sistema SHALL exponer en `deps.py` una dependencia de autoservicio, distinta de `require_operaciones`, que a partir del `Usuario` autenticado (`get_current_user`) resuelve el `Socio` asociado por `Socio.usuario_id` y lo entrega al endpoint. Esta dependencia SHALL NOT otorgar el acceso que otorgan los roles de staff (`CAJERO`, `OFICIAL_CREDITO`, `ADMINISTRADOR`, `SUPERADMIN`).

#### Scenario: Usuario autenticado con socio asociado
- **GIVEN** un usuario autenticado cuyo `usuario_id` está vinculado a un registro `Socio`
- **WHEN** accede a un endpoint protegido por la dependencia de autoservicio
- **THEN** el sistema resuelve el `Socio` y permite continuar la operación

#### Scenario: Usuario autenticado sin socio asociado
- **GIVEN** un usuario autenticado (p. ej. staff) sin registro `Socio` vinculado a su `usuario_id`
- **WHEN** intenta acceder a un endpoint protegido por la dependencia de autoservicio
- **THEN** el sistema rechaza la solicitud con un error de autorización (403) sin revelar detalles internos

### Requirement: Aislamiento de recursos ajenos

El sistema SHALL restringir cada operación de autoservicio a recursos cuyo `socio_id` coincida con el `Socio` resuelto por la dependencia de autoservicio. Toda referencia a un recurso de otro socio SHALL responder 404 sin revelar su existencia. El aislamiento multi-tenant queda garantizado porque el socio pertenece a una sola cooperativa.

#### Scenario: Recurso propio
- **GIVEN** un socio autenticado y un recurso cuyo `socio_id` coincide con el suyo
- **WHEN** el socio opera sobre ese recurso vía autoservicio
- **THEN** el sistema permite la operación

#### Scenario: Recurso de otro socio
- **GIVEN** un socio autenticado y un recurso cuyo `socio_id` pertenece a otro socio
- **WHEN** el socio intenta operar sobre ese recurso vía autoservicio
- **THEN** el sistema responde 404 sin indicar si el recurso existe
