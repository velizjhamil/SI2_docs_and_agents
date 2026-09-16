# Design: Transferencias entre cuentas propias

## Technical Approach

Camino de autoservicio paralelo al de ventanilla, sin tocar los endpoints staff: dependencia de identidad `get_current_socio` en `deps.py`, endpoint `POST /api/v1/ahorros/transferencias` con helper propio de lock de dos filas, migración SQL aditiva `007`, y en móvil un service + pantalla calcados de `cuentas_service.dart` / `portfolio_screen.dart`.

## Architecture Decisions

### Decision: la dependencia resuelve identidad; la propiedad se valida en el `WHERE`

**Choice**: `get_current_socio(usuario=Depends(get_current_user), db)` → `select(Socio).where(Socio.usuario_id == usuario.id)`; sin socio o socio no `ACTIVO` → **403** (`"Operación disponible solo para socios"`). La propiedad de las cuentas se verifica dentro del helper de lock con `CuentaAhorro.socio_id == socio.id` **en el `WHERE`**: cuenta inexistente y cuenta ajena dan el mismo `None` → **404 `"Cuenta no encontrada"`**.
**Alternatives considered**: dependencia que lea el body y valide ambas cuentas; filtrar `socio_id` después de cargar la fila.
**Rationale**: sin rama que distinga los dos casos, la fuga de información es imposible por construcción. Validar en la dependencia obligaría a lockear filas fuera de la transacción del endpoint. El 403 habla del llamante, no de un recurso.

### Decision: helper nuevo `_lock_cuentas_propias()`, no extender `_get_cuenta_visible()`

**Choice**: función nueva en `ahorros.py`; `_get_cuenta_visible()` queda intacta.
**Alternatives considered**: parametrizar `_get_cuenta_visible()` con `socio_id`.
**Rationale**: esa función autoriza por `_socio_visible(admin, ...)` (regla de tenant staff) y tiene 3 llamadores que no deben cambiar; mezclar dos modelos de autorización en una función es la vía típica a un bypass.

### Decision: dos `SELECT ... FOR UPDATE` secuenciales por `id` ascendente

**Choice**: `ids = sorted([origen_id, destino_id])` y dos consultas de una fila en ese orden, con `.with_for_update(of=CuentaAhorro)` y **sin `joinedload`**; `moneda` se carga después del lock.
**Alternatives considered**: un solo `WHERE id IN (...) ORDER BY id ... FOR UPDATE`; conservar `joinedload(moneda, innerjoin=True)` como en `_get_cuenta_visible()`.
**Rationale**: con `IN ... ORDER BY`, el planner puede tomar los locks en orden de scan (antes del sort), así que el orden ascendente no queda garantizado y se pierde la protección anti-deadlock; dos sentencias lo garantizan a costa de un round-trip. `of=CuentaAhorro` evita lockear las filas de `moneda`/`socio`: sin él toda transferencia compite por la misma fila de moneda.

### Decision: `transaccion_contraparte_id` bidireccional, en la misma transacción

**Choice**: `ALTER TABLE transaccion ADD COLUMN IF NOT EXISTS transaccion_contraparte_id INT REFERENCES transaccion(id)` (nullable) + índice. Escritura: `INSERT` SALIDA `RETURNING id` → `INSERT` ENTRADA con contraparte `RETURNING id` → `UPDATE` de la salida; todo antes del único `db.commit()`.
**Alternatives considered**: vínculo unidireccional; columna `grupo_transferencia UUID`.
**Rationale**: nombre y forma calcan `transaccion_reversion_id` de `bd.sql`. Bidireccional permite auditar desde cualquiera de las dos patas; el `UPDATE` extra vive en la misma transacción, así que no hay estado visible a medias.

### Decision: la glosa va a bitácora, no a `transaccion`

**Choice**: `transaccion` no tiene columna `glosa`; se guarda en `registrar_accion(descripcion=...)` y se devuelve en la respuesta.
**Rationale**: la propuesta autoriza una sola columna nueva y la bitácora ya es el registro de texto libre del módulo. **Rechazado**: añadir `glosa VARCHAR(255)` a `transaccion`.

### Decision: el entry point prometido es el tile ya existente del dashboard, con `Navigator.push` directo

**Choice**: `dashboard_screen.dart` — `_buildQuickActionTile()` (línea 481) gana un parámetro `VoidCallback? onTap` y envuelve su `Container` en `InkWell`; solo el tile `Icons.swap_horiz` ('Envíos' / 'Transferencias', línea 324) lo recibe, los otros cinco pasan `null` y siguen inertes. El callback hace `await Navigator.of(context).push(MaterialPageRoute(builder: (_) => TransferenciaScreen()))` y, si vuelve `true`, invoca `onRetry` — el `VoidCallback` que `_CuentasBody` **ya recibe** (`onRetry: _recargar`, línea 148) — para refrescar saldos sin plomería nueva.
**Alternatives considered**: ruta nombrada `/transferencias` en la tabla `routes` de `main.dart`; envolver la grilla entera en un gesto.
**Rationale**: el spec promete la pantalla "accesible desde dashboard/portfolio" y el tile ya existe, pero hoy es un `Container` sin gesto (`dashboard_screen.dart` no tiene un solo `Navigator`, `onTap` ni `InkWell`). `MaterialPageRoute` es el único push real del código (`login_screen.dart:54`); la tabla `routes` de `main.dart` solo registra transiciones de sesión (`/login`, `/dashboard`) y `/dashboard` ni siquiera se usa. El constructor en el `builder` además deja inyectar services en el widget test, cosa que una ruta nombrada `const` no permite. Por eso `main.dart` **no** se modifica.

### Decision: `portfolio_screen.dart` no se modifica en v1; `services_screen.dart` queda como entry point adicional

**Choice**: el dashboard es el entry point que exige el spec. `services_screen.dart` se wirea **además** (su `_buildServiceTile` ya expone `onTap: () {}` vacío, línea 160), no como reemplazo. `portfolio_screen.dart` no se toca.
**Rationale**: portfolio hoy no tiene ningún punto de entrada de acciones — su única interacción es el `IconButton` de refresco del AppBar (línea 73); abrir uno inventaría un patrón de UI que el change no pide. Queda como decisión explícita, no como silencio. El catálogo de Servicios es donde el usuario ya busca la operación por nombre, así que wirear su tile cuesta una línea y evita un callejón sin salida.

## Data Flow

    TransferenciaScreen ──POST /ahorros/transferencias──→ endpoint
                        get_current_socio (403 si no es socio)│
                                                              ▼
                                     _lock_cuentas_propias(socio, ids)
                                FOR UPDATE id menor ─→ FOR UPDATE id mayor
                                            │ (404 si ajena o inexistente)
                                validaciones: ACTIVA, misma moneda, saldo
                                                              ▼
        origen −monto ; destino +monto ; INSERT SALIDA ↔ ENTRADA ; bitácora
                                                    un solo COMMIT
                                                              ▼
                          TransferenciaOut (ambas cuentas con saldo nuevo)

## File Changes

| File | Action | Description |
|------|--------|-------------|
| `SI2_backend_api/app/api/v1/deps.py` | Modify | `get_current_socio` |
| `SI2_backend_api/app/api/v1/endpoints/ahorros.py` | Modify | `_lock_cuentas_propias()`, `_registrar_transferencia()`, `POST /transferencias` |
| `SI2_backend_api/app/schemas/schemas.py` | Modify | `TransferenciaCreate`, `TransferenciaOut` |
| `SI2_backend_api/migrations/007_sprint3_transferencias.sql` | Create | Self-FK `transaccion_contraparte_id` + índice |
| `SI2_backend_api/tests/test_transferencias.py` | Create | Autorización, validaciones, doble pata, concurrencia |
| `SI2_mobile_app/lib/models/transferencia_model.dart` | Create | Modelo de respuesta (tolerante a int/double/string) |
| `SI2_mobile_app/lib/services/transferencias_service.dart` | Create | `TransferenciasService` + `TransferenciaResult` |
| `SI2_mobile_app/lib/screens/transferencia_screen.dart` | Create | Origen/destino/monto/confirmación |
| `SI2_mobile_app/lib/screens/dashboard_screen.dart` | Modify | `onTap` opcional en `_buildQuickActionTile()` + `InkWell`; solo el tile `swap_horiz` ('Envíos'/'Transferencias') navega y refresca con `onRetry` al volver |
| `SI2_mobile_app/lib/screens/services_screen.dart` | Modify | `onTap` del tile "Transferencias entre Cuentas" (entry point adicional del catálogo) |
| `SI2_mobile_app/lib/screens/portfolio_screen.dart` | None | Sin cambios en v1 — no tiene puntos de entrada de acciones hoy (decisión explícita arriba) |
| `SI2_mobile_app/lib/main.dart` | None | Sin cambios — se usa `MaterialPageRoute`, no ruta nombrada |
| `SI2_mobile_app/test/transferencias_service_test.dart` | Create | `MockClient`, patrón de `cuentas_service_test.dart` |
| `SI2_mobile_app/test/transferencia_screen_test.dart` | Create | Services inyectados + entry point del dashboard |

## Interfaces / Contracts

```python
class TransferenciaCreate(BaseModel):
    cuenta_origen_id: int = Field(..., ge=1)
    cuenta_destino_id: int = Field(..., ge=1)
    monto: Decimal = Field(..., gt=0, max_digits=12, decimal_places=2)
    glosa: str | None = Field(None, max_length=255)

class TransferenciaOut(BaseModel):          # 201
    transaccion_salida_id: int
    transaccion_entrada_id: int
    cuenta_origen: CuentaAhorroOut          # saldo ya actualizado
    cuenta_destino: CuentaAhorroOut
    monto: Decimal
    glosa: str | None
    fecha_hora: datetime
```

Errores con `detail` en español: **403** no es socio; **404** cuenta inexistente o ajena; **400** origen == destino, cuenta no `ACTIVA`, `moneda_id` distinta, saldo insuficiente. Las validaciones cruzadas van en el endpoint (400), no en un `model_validator` (422), porque el cuerpo 422 de FastAPI es una lista y el móvil muestra `detail` como string.

```dart
Future<TransferenciaResult> transferir({
  required int cuentaOrigenId, required int cuentaDestinoId,
  required double monto, String? glosa, http.Client? client,
}) // JWT de AuthService.tokenJWT; never-throw; timeout 10s
```

`TransferenciaResult.success(...)` / `.failure(String)`, igual en forma a `CuentasResult`. Token nulo → `'No hay sesión activa...'`; 400 → `detail` del backend; 401 → mensaje de sesión; 404 → `'Una de las cuentas seleccionadas no está disponible.'` (no revela propiedad); resto → mensaje de conexión. Timeout 10 s, no 5 s como la lectura, por ser escritura de dinero.

La pantalla toma las cuentas de `CuentasService.listarMisCuentas()`; el dropdown de destino excluye el origen y filtra por `status == 'ACTIVA'` y misma `currency.currencyCode` — defensa en profundidad de UX, nunca sustituto del backend.

## Testing Strategy

| Layer | What to Test | Approach |
|-------|-------------|----------|
| Unit (mobile) | Mapeo de 201/400/401/404/timeout a `TransferenciaResult` | `MockClient`, patrón de `cuentas_service_test.dart` |
| Widget (mobile) | Dropdowns filtrados, validación de monto, confirmación, error visible; tap en el tile 'Transferencias' del dashboard abre `TransferenciaScreen` y el retorno `true` dispara `onRetry` | `pumpWidget` con ambos services inyectados; `_CuentasBody` con `onRetry` espía |
| Integration | Feliz; cuenta ajena → 404; origen == destino, no `ACTIVA`, moneda distinta, saldo insuficiente → 400; staff sin socio → 403; exactamente 2 filas `transaccion` mutuamente vinculadas + 1 bitácora | `TestClient` + Postgres real con el `skipif` de `test_savings.py` |
| Concurrency | A→B y B→A simultáneas: sin deadlock, sin saldo negativo, suma de saldos constante | Dos sesiones sobre la misma pareja de cuentas |

## Threat Matrix

N/A — no hay routing de comandos, shell, subprocesos, automatización VCS/PR, clasificación de ejecutables ni integración de procesos. El límite de seguridad aquí es autorización HTTP + aislamiento de datos, cubierto por las decisiones 1–3 y los tests negativos obligatorios.

## Migration / Rollout

`007_sprint3_transferencias.sql` con el estilo de `006` (`BEGIN; ALTER TABLE ... IF NOT EXISTS; CREATE INDEX IF NOT EXISTS; COMMIT;`): idempotente, sin backfill, columna nullable, filas existentes válidas. Sin feature flag; el endpoint no existe hasta el despliegue.

## Open Questions

- [ ] Sin clave de idempotencia en v1: si la app agota el timeout con la transacción ya confirmada, un reintento del usuario duplica la transferencia. Mitigación actual: timeout de 10 s y refresco de saldos ante resultado ambiguo. `Idempotency-Key` queda como change posterior.
- [ ] `canal` se registra como `'MOVIL'` (ventanilla usa `'WEB'`). La columna no tiene CHECK; confirmar si algún reporte asume `'WEB'`.
