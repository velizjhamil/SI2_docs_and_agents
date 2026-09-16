# Tasks: Transferencias entre cuentas propias

## Review Workload Forecast

| Field | Value |
|-------|-------|
| Estimated changed lines | ~450 backend + ~600 mobile (~1050 total, tests incl.) |
| 400-line budget risk | High |
| Chained PRs recommended | Yes |
| Suggested split | PR1 backend-core → PR2 backend-tests → PR3 mobile-service → PR4 mobile-ui |
| Delivery strategy | ask-on-risk |
| Chain strategy | pending — orchestrator must ask user |

Decision needed before apply: Yes
Chained PRs recommended: Yes
Chain strategy: pending
400-line budget risk: High

### Suggested Work Units

| Unit | Goal | Likely PR | Focused test command | Runtime harness | Rollback boundary |
|------|------|-----------|----------------------|-----------------|-------------------|
| 1 | Migración 007 + `get_current_socio` + `_lock_cuentas_propias` + endpoint | PR 1 | `pytest tests/test_transferencias.py -k auth_or_lock` | `uvicorn` local + Postgres dev | Revert endpoint/deps commit; migración es aditiva, `downgrade` la retira |
| 2 | `test_transferencias.py` completo (negocio, auditoría, concurrencia) | PR 2 | `pytest tests/test_transferencias.py` | Postgres real, `skipif` de `test_savings.py` | Revert commit de tests, sin afectar PR 1 |
| 3 | Modelo + `transferencias_service.dart` + tests de service | PR 3 | `flutter test test/transferencias_service_test.dart` | `MockClient` inyectado | Revert commit; no toca pantallas |
| 4 | `transferencia_screen.dart` + wiring dashboard/services + tests | PR 4 | `flutter test test/transferencia_screen_test.dart` | Services inyectados vía `pumpWidget` | Revert `onTap`/`InkWell` en dashboard y services_screen; tile vuelve a inerte |

## Phase 1: Backend — Migración (Foundation)

- [x] 1.1 Crear `SI2_backend_api/migrations/007_sprint3_transferencias.sql`: `transaccion_contraparte_id` self-FK nullable + índice, estilo `006` — *mecánico, bajo riesgo*
- [x] 1.2 Aplicar en dev y verificar `IF NOT EXISTS` idempotente sin backfill — *mecánico*

## Phase 2: Backend — Autorización y schemas

- [x] 2.1 RED: test en `test_transferencias.py` — usuario sin `Socio` asociado → 403 en endpoint protegido
- [x] 2.2 GREEN: implementar `get_current_socio` en `deps.py` (resuelve `Socio` por `usuario_id`, 403 si no `ACTIVO`) — **toca autorización, requiere cuidado**
- [x] 2.3 Añadir `TransferenciaCreate`/`TransferenciaOut` en `schemas.py` — *mecánico*

## Phase 3: Backend — Locking y endpoint (Core)

- [x] 3.1 RED: test — cuenta origen/destino ajena → 404 sin revelar existencia
- [x] 3.2 RED: test — origen==destino, cuenta inactiva, moneda distinta, saldo insuficiente → 400
- [x] 3.3 RED: test — dos transferencias concurrentes mismo par de cuentas, sin deadlock ni saldo negativo
- [x] 3.4 GREEN: `_lock_cuentas_propias()` en `ahorros.py` — dos `SELECT...FOR UPDATE` por id ascendente, `of=CuentaAhorro`, sin `joinedload` — **toca locking, requiere cuidado**
- [x] 3.5 GREEN: `_registrar_transferencia()` — dos `INSERT` en `transaccion` vinculados por `transaccion_contraparte_id` + `registrar_accion()` bitácora `AHORROS` — **toca migración/auditoría**
- [x] 3.6 GREEN: `POST /api/v1/ahorros/transferencias` — valida negocio, invoca lock+registro, un solo `commit` — **toca autorización/locking**
- [x] 3.7 Confirmar que todos los RED de 2.1 y 3.1–3.3 pasan en verde

## Phase 4: Backend — Auditoría y límites (Verification)

- [x] 4.1 Test: transferencia exitosa genera exactamente 2 filas `transaccion` vinculadas + 1 bitácora
- [x] 4.2 Test: sin comisión ni límite de monto — acreditado == debitado

## Phase 5: Mobile — Modelo y servicio (Foundation)

- [x] 5.1 Crear `lib/models/transferencia_model.dart` (tolerante int/double/string) — *mecánico*
- [x] 5.2 RED: `test/transferencias_service_test.dart` — mapeo 201/400/401/404/timeout con `MockClient`
- [x] 5.3 GREEN: `lib/services/transferencias_service.dart` — `TransferenciasService`/`TransferenciaResult`, patrón `cuentas_service.dart`, JWT `AuthService.tokenJWT`, timeout 10s — *mecánico, sigue patrón existente*

## Phase 6: Mobile — Pantalla (Core UI)

- [x] 6.1 RED: `test/transferencia_screen_test.dart` — dropdowns filtrados (excluye origen, misma moneda, `ACTIVA`), validación de monto, confirmación, mensajes de error
- [x] 6.2 GREEN: `lib/screens/transferencia_screen.dart` — origen/destino desde `CuentasService.listarMisCuentas()`, confirmación, mensajes de rechazo en español

## Phase 7: Mobile — Wiring de navegación (Integration)

- [x] 7.1 Modificar `dashboard_screen.dart`: `onTap` opcional + `InkWell` en `_buildQuickActionTile()`; solo tile `swap_horiz` navega con `MaterialPageRoute` y llama `onRetry` si retorna `true` — *mecánico, patrón acotado por design*
- [x] 7.2 Modificar `services_screen.dart`: `onTap` del tile "Transferencias entre Cuentas" abre `TransferenciaScreen` — *mecánico*
- [x] 7.3 RED/GREEN: extender test de dashboard — tap en tile 'Transferencias' abre pantalla y `onRetry` espiado se dispara al volver `true`

## Phase 8: Verificación end-to-end

- [ ] 8.1 Backend: `pytest tests/test_transferencias.py` contra Postgres real
- [ ] 8.2 Mobile: `flutter test` completo
- [ ] 8.3 Manual: app real contra backend real — feliz, cuenta ajena, moneda distinta, saldo insuficiente
- [ ] 8.4 Verificar cada Criterio de Éxito de `proposal.md`
