## Why

El socio ya consulta sus cuentas desde la app móvil (`GET /ahorros/mis-cuentas`), pero **no puede mover dinero sin pasar por ventanilla**: todo endpoint de movimiento (`depositar`, `retirar`, `abrir_cuenta`, `cambiar_estado_cuenta`) exige rol staff vía `require_operaciones`. No existe ningún camino de autoservicio para mover fondos.

Transferir entre cuentas propias es el caso de menor riesgo para abrir ese camino —el dinero nunca sale del patrimonio del socio— y establece el patrón de autorización de autoservicio que módulos posteriores reutilizarán.

*Fuera de alcance*: el Pago Móvil de Cuotas de Crédito es un change futuro e independiente (el dominio de créditos no existe: `creditos.py` vacío, sin modelos ni migración).

## What Changes

- **SI2_backend_api**
  - Nuevo endpoint de autoservicio `POST /api/v1/ahorros/transferencias` (origen, destino, monto, glosa opcional).
  - **Autorización**: nueva dependencia de autoservicio en `deps.py` (socio autenticado, distinta de `require_operaciones`) que resuelve el `Socio` desde `usuario_id` y exige que **ambas** cuentas tengan ese `socio_id`. Cuenta ajena → 404 sin revelar existencia. El aislamiento multi-tenant queda garantizado porque el socio pertenece a una sola cooperativa.
  - **Validaciones**: origen ≠ destino, ambas `ACTIVA`, misma `moneda_id` (sin conversión: distinta moneda se **rechaza** en v1), monto > 0 y ≤ `saldo_disponible`.
  - **Concurrencia**: una sola transacción con `SELECT ... FOR UPDATE` sobre ambas cuentas en **orden determinístico por `id` ascendente** para evitar deadlocks (hoy `_get_cuenta_visible()` lockea una sola fila).
  - **Auditoría de ambas patas**: dos filas en `transaccion` (`TRANSFERENCIA_SALIDA` / `TRANSFERENCIA_ENTRADA`) vinculadas por una nueva columna self-FK nullable (migración nueva, al estilo de `transaccion_reversion_id`), más un `registrar_accion()` en bitácora (`modulo='AHORROS'`).
  - Sin comisión y sin límites de monto/frecuencia en esta v1.
- **SI2_mobile_app** (`forest_microfinance`): nuevo `transferencias_service.dart` con el patrón de `cuentas_service.dart` (client inyectable, JWT desde `AuthService.tokenJWT`, resultado never-throw en español) y pantalla de transferencia (origen/destino desde las cuentas reales, monto, confirmación) accesible desde dashboard/portfolio.
- **SI2_frontend_web**: sin cambios.

## Capabilities

### New Capabilities
- `autoservicio-socio`: autorización de operaciones iniciadas por el socio autenticado sobre recursos propios, separada de la autorización staff (`require_operaciones`).
- `ahorros/transferencias`: transferencia interna de saldo entre cuentas de ahorro del mismo socio, con locking de ambas cuentas y registro de auditoría de las dos patas.
- `mobile/transferencias`: flujo móvil de transferencia entre cuentas propias.

### Modified Capabilities
- Ninguna. `openspec/specs/` aún no contiene specs consolidadas.

## Impact

| Área | Impacto | Detalle |
|---|---|---|
| `SI2_backend_api/app/api/v1/deps.py` | Nuevo | Dependencia de autoservicio del socio |
| `SI2_backend_api/app/api/v1/endpoints/ahorros.py` | Modificado | Endpoint de transferencia + helper de lock de dos cuentas |
| `SI2_backend_api/app/schemas/schemas.py` | Nuevo | Schemas de request/response de transferencia |
| `SI2_backend_api/migrations/` | Nuevo | Columna de vínculo entre patas en `transaccion` |
| `SI2_backend_api/tests/` | Nuevo | `test_transferencias.py` (autorización, saldo, moneda, concurrencia) |
| `SI2_mobile_app/lib/services/`, `lib/models/`, `lib/screens/` | Nuevo | Service, modelo y pantalla de transferencia |
| `SI2_mobile_app/test/` | Nuevo | Tests de service y pantalla |

## Riesgos

| Riesgo | Prob. | Mitigación |
|---|---|---|
| Socio mueve fondos de cuenta ajena | Media | Verificar `socio_id` de **ambas** cuentas contra el socio autenticado; tests negativos obligatorios |
| Deadlock entre transferencias concurrentes | Media | Lock siempre por `id` ascendente dentro de una única transacción |
| Descuadre (una pata escrita sin la otra) | Baja | Ambas patas en la misma transacción atómica; rollback total ante error |
| Cuentas en distinta moneda | Media | Rechazo explícito en v1; no se inventa conversión |

## Rollback

Revertir el commit del endpoint y del service móvil. La migración es aditiva (columna nullable) y puede quedar aplicada sin romper nada; su `downgrade` la elimina si se requiere. No hay backfill ni datos destruidos: las transferencias ya ejecutadas quedan como filas `transaccion` válidas y auditadas.

## Criterios de Éxito

- [ ] Un socio autenticado transfiere saldo entre dos cuentas propias activas y ambos saldos quedan consistentes.
- [ ] Todo intento sobre una cuenta que no le pertenece se rechaza (404) y queda registrado.
- [ ] Cada transferencia genera exactamente dos filas `transaccion` vinculadas entre sí y una entrada de bitácora.
- [ ] Transferencias concurrentes sobre las mismas cuentas no producen deadlock ni saldo negativo.
- [ ] La app móvil completa el flujo contra el backend real usando el patrón de `cuentas_service.dart`.
