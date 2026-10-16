# Design

## Context

El pago se ejecuta en el backend externo W2 · Pago a contacto (mock), documentado en `openAPI.txt.txt` (servidor `https://bcp-wallet-mock.yefri.workers.dev/groups/3`). El backend local no registra pagos. Solo llama a W2 y mantiene un contador diario por wallet para el tope de PEN 500. Ver `proposal.md` y `specs/payments/low-amount-transfer/spec.md`.

## Goals / Non-Goals

**Goals:**
- Un endpoint local `POST /v1/payments` que valida, llama a W2 y devuelve un resultado de intento.
- Validar monto, destino y tope diario antes de llamar a W2.
- Contador diario por wallet en SQLite con corte en Lima.

**Non-Goals:**
- Historial de pagos local y tabla de pagos.
- Idempotencia de reintentos del cliente (diferida en H-02). Cada intento envía una `Idempotency-Key` nueva.
- Consumo de callbacks, `/resolve` y reversos (fuera del alcance de `POST /payments`).

## Decisions

- **Cliente HTTP hacia W2**: `W2PaymentClient` con `java.net.http.HttpClient`. La URL base va en `bcp.wallet-mock.base-url` en `application.properties`, con el valor por defecto de arriba.
- **`Idempotency-Key`**: UUID generado por intento. Evita el error 400 de W2 por header faltante. No da idempotencia real ante reintentos del cliente.
- **Mapeo de origen**: `walletId` de W2 = wallet propia del pagador autenticado.
- **Contador diario**: tabla `daily_payment_totals(wallet_id, lima_date, total_cents, PRIMARY KEY(wallet_id, lima_date))`. Se usa `ZoneId.of("America/Lima")` para `lima_date`. Se actualiza dentro de `SqliteStore.withConnection` (sincronizado).
- **Reserva antes de llamar a W2**: se comprueba `total + monto <= 50000` y se reserva el monto dentro del mismo lock. Si W2 responde fallo sin operación (`PROVIDER_ERROR`, `INSUFFICIENT_FUNDS`, `WALLET_NOT_ACTIVE`), la reserva se libera. Si W2 aprueba o responde 504, la reserva se queda.
- **Orden de validación fijo**: origen → monto → destino → tope diario. Errores estables.
- **Montos en céntimos (entero)**: `amountCents` de W2 es entero (1 a 100000000). PEN 500 = 50000.

## Open decisions

- **Resultado 202**: W2 responde 202 con `PENDING` para la creación. Según el contrato, la confirmación llega por callback o `/resolve`, que quedan fuera de este cambio. Opciones: (a) devolver `pending` al cliente y relajar la regla de resultado terminal; (b) consumir `/control/events` y `/callbacks/payments` en la misma petición; (c) llamar a `/payments/{id}/resolve` una vez y tratar PENDING como `failed`. **Pendiente de decisión.**
- **Mapeo de destino**: W2 solo acepta `recipient` ∈ {`contact-demo`, `merchant-demo`}, no un `idWallet` arbitrario. Hay que decidir cómo se mapean las wallets de este backend a esos valores. **Pendiente de decisión.**
- **Escenario**: el cliente no elige el escenario en producción. Se configura por propiedad y por defecto es `APPROVED`.

## Risks / Trade-offs

- **[Reintento tras corte de red]** → Sin idempotencia real, un reintento crea otro cobro. Riesgo aceptado por H-02 hasta definir idempotencia.
- **[Contador en una sola instancia]** → El lock de `SqliteStore` cubre el MVP. Varias instancias requieren otra estrategia.
- **[Dependencia de disponibilidad de W2]** → Si W2 no responde, el intento falla por timeout del cliente.
