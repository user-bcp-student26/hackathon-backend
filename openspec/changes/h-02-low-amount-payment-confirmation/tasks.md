# Tasks

## 1. Configuración y cliente W2

- [ ] 1.1 Añadir `bcp.wallet-mock.base-url` (por defecto `https://bcp-wallet-mock.yefri.workers.dev/groups/3`) y `bcp.wallet-mock.scenario` (por defecto `APPROVED`) en `application.properties`; verificar con `mvn test` que se leen.
- [ ] 1.2 Implementar `W2PaymentClient.createPayment(...)` con `HttpClient` y header `Idempotency-Key`; verificar con prueba contra servidor HTTP local simulado que se envían el header y el cuerpo.
- [ ] 1.3 Mapear respuestas 200, 202, 400, 404, 409, 503 y 504 a un tipo `W2Result`; verificar con una prueba por código.

## 2. Contador diario

- [ ] 2.1 Crear la tabla `daily_payment_totals` en `SqliteStore.initialize()`; verificar con `PRAGMA table_info(daily_payment_totals)`.
- [ ] 2.2 Crear `store/DailyTotalRepository` con `reserve(walletId, limaDate, cents, limitCents)` y `release` / `add`, dentro de `withConnection`; verificar que no reserva por encima de 50000 céntimos.
- [ ] 2.3 Añadir la tabla a `SqliteStore.clearBusinessData()`; verificar que queda vacía.
- [ ] 2.4 Calcular `limaDate` con `America/Lima`; verificar con pruebas que cruzan la medianoche de Lima.

## 3. Validaciones y servicio

- [ ] 3.1 Implementar validación de monto (0 < monto ≤ 50000 céntimos) y de origen propio en `service/PaymentValidation`; verificar con pruebas de 0, 0.01, 500 y 500.01.
- [ ] 3.2 Implementar validación de destino (existe, distinto del origen) según el mapeo abierto; verificar con destino inexistente e igual al origen.
- [ ] 3.3 Implementar `PaymentService.createPayment` con el orden origen → monto → destino → tope, liberando la reserva en fallos sin operación; verificar que un error de validación no llama a W2.
- [ ] 3.4 Resolver la decisión abierta sobre 202 antes de implementar el resultado terminal (ver `design.md`).

## 4. Endpoint y contrato

- [ ] 4.1 Añadir `POST /v1/payments` a `openapi.yaml` con códigos 200/400/404/409/503/504 y el campo `reason`; verificar que `/q/openapi` responde 200.
- [ ] 4.2 Implementar `PaymentsResource` con autenticación Basic existente; verificar con curl que sin credenciales responde 401.

## 5. Pruebas y documentación

- [ ] 5.1 Pruebas de contrato para los escenarios de `specs/payments/low-amount-transfer/spec.md` en `H01ContractTest` o clase nueva; verificar con `mvn test`.
- [ ] 5.2 Carpeta de pagos en la colección Postman con éxito, monto inválido, destino inválido y tope diario; verificar con Newman sin fallos.
- [ ] 5.3 Documentar en `README.md` la dependencia de W2, el riesgo de reintento sin idempotencia y cómo probar pagos; verificar que los comandos se ejecutan tal cual.

## 6. Integración

- [ ] 6.1 Reiniciar el servicio en el puerto 8090 con el build nuevo; verificar con curl un pago aprobado y otro rechazado por tope.

## Workflow follow-up

- Archivar el change tras la revisión del equipo con `openspec archive h-02-low-amount-payment-confirmation`.
