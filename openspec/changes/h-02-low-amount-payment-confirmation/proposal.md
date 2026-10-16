# Proposal

## Why

El pagador necesita enviar un pago de bajo monto desde su wallet hacia otra wallet y saber el resultado. Origen: issue H-02 de `user-bcp-student26/hackathon` (https://github.com/user-bcp-student26/hackathon/issues/2). El backend no registra pagos: la ejecución la hace el backend externo W2 · Pago a contacto (mock).

## What Changes

- Nuevo endpoint `POST /v1/payments` que valida y delega la ejecución a `POST /payments` de W2.
- Validación local de monto (0 < monto ≤ PEN 500), origen propio y destino (existe y es distinto del origen), antes de llamar a W2.
- Tope diario de PEN 500 por wallet y día calendario de Lima, guardado como contador (sin registrar pagos).
- Traducción de la respuesta de W2 a un resultado de intento (`succeeded` / `failed` con `reason`).
- **Fuera de alcance**: historial local de pagos, idempotencia de reintentos, callbacks y `/resolve`, reversos, tope por proveedor y métricas.

## Capabilities

### New Capabilities
- `payments/low-amount-transfer`: pago de bajo monto delegado a W2, con validaciones locales y tope diario.

### Modified Capabilities
- Ninguna.

## Impact

- **API**: `POST /v1/payments` en `src/main/resources/META-INF/resources/openapi.yaml`. El contrato H-01 no cambia.
- **Código**: recurso `PaymentsResource`, `PaymentService`, `W2PaymentClient` y `DailyTotalRepository` sobre SQLite.
- **Datos**: tabla `daily_payment_totals`. Sin pagos persistidos.
- **Dependencias**: `java.net.http` (JDK). Sin librerías nuevas.
- **Externo**: dependencia de disponibilidad de `https://bcp-wallet-mock.yefri.workers.dev/groups/3`.
- **Pruebas**: contrato, Postman/Newman y curl.
