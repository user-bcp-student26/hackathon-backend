# Spec Delta

## Purpose

Permite al pagador enviar un pago de bajo monto desde su wallet propia hacia otra wallet, ejecutado por el backend externo W2 · Pago a contacto (mock). El backend local valida monto, destino y tope diario antes de llamar al mock, y traduce la respuesta del mock al resultado del intento.

El backend local NO registra pagos. Solo mantiene un contador diario de pagos exitosos por wallet para aplicar el tope.

## ADDED Requirements

### Requirement: Ejecución delegada en W2
El sistema SHALL ejecutar cada pago llamando a `POST /payments` del backend W2 (`https://bcp-wallet-mock.yefri.workers.dev/groups/3`, configurable). El sistema SHALL NOT debitar ni acreditar saldo internamente.

#### Scenario: Pago enviado a W2
- **WHEN** un pago pasa las validaciones locales
- **THEN** el sistema envía `POST /payments` con el header `Idempotency-Key` y el cuerpo `walletId`, `recipient`, `amountCents`, `currency` (`PEN`) y `scenario`

#### Scenario: Escenario configurable para pruebas
- **WHEN** no se configura un escenario
- **THEN** el sistema usa `APPROVED`

### Requirement: Origen propio del pagador
El sistema SHALL usar como origen la wallet propia del pagador autenticado. El cliente SHALL NOT poder indicar otra wallet origen.

#### Scenario: Pago con origen propio
- **WHEN** un pagador autenticado envía un pago válido
- **THEN** el sistema envía a W2 el `walletId` asociado a su usuario

#### Scenario: Intento de indicar otro origen
- **WHEN** la petición incluye una wallet origen distinta de la del pagador
- **THEN** el sistema responde 400 y no llama a W2

### Requirement: Validación de monto
El sistema SHALL rechazar montos menores o iguales a 0 o mayores a PEN 500 (50000 céntimos) con 400, antes de llamar a W2.

#### Scenario: Monto cero o negativo
- **WHEN** un pagador envía un monto menor o igual a 0
- **THEN** el sistema responde 400 y no llama a W2

#### Scenario: Monto sobre el tope por operación
- **WHEN** un pagador envía un monto mayor a PEN 500
- **THEN** el sistema responde 400 y no llama a W2

#### Scenario: Monto en rango
- **WHEN** un pagador envía un monto mayor a 0 y no mayor a PEN 500
- **THEN** el sistema continúa con la validación de destino

### Requirement: Validación de destino
El sistema SHALL rechazar destinos inexistentes o iguales a la wallet origen con 404 o 400 respectivamente, antes de llamar a W2.

#### Scenario: Destino inexistente
- **WHEN** un pagador envía un destino que no existe
- **THEN** el sistema responde 404 y no llama a W2

#### Scenario: Destino igual al origen
- **WHEN** el destino es la wallet propia del pagador
- **THEN** el sistema responde 400 y no llama a W2

### Requirement: Tope diario de pagos exitosos
El sistema SHALL limitar el acumulado de pagos exitosos por wallet a PEN 500 por día calendario de Lima (`America/Lima`). El acumulado SHALL guardarse como contador diario por wallet en SQLite, sin registrar pagos individuales. Los intentos fallidos SHALL NOT sumar al acumulado.

#### Scenario: Pago que excede el tope diario
- **WHEN** el contador del día más el monto solicitado supera PEN 500
- **THEN** el sistema responde 409 `DAILY_LIMIT_EXCEEDED` y no llama a W2

#### Scenario: Pago dentro del tope
- **WHEN** el contador del día más el monto solicitado no supera PEN 500
- **THEN** el sistema llama a W2

#### Scenario: Contador tras pago exitoso
- **WHEN** W2 confirma el pago como aprobado
- **THEN** el sistema suma el monto al contador del día de la wallet

#### Scenario: Reinicio a medianoche de Lima
- **WHEN** cambia la fecha calendario en Lima
- **THEN** el contador de la wallet empieza en cero para el nuevo día

### Requirement: Mapeo de respuestas exitosas y pendientes de W2
El sistema SHALL traducir la respuesta 200 de W2 con `status` `SUCCEEDED` a `succeeded` con `reference` igual al `id` del pago. La respuesta 202 de W2 SHALL NOT convertirse en un resultado terminal sin resolver la decisión abierta de `design.md`.

#### Scenario: Pago aprobado
- **WHEN** W2 responde 200 con `status` `SUCCEEDED`
- **THEN** la respuesta del intento es `succeeded` con la referencia del pago

### Requirement: Mapeo de errores de W2
El sistema SHALL traducir los errores de W2 así: 400 y 404 se devuelven al cliente con el mismo código; 409 `IDEMPOTENCY_CONFLICT` se devuelve como 409; 409 `WALLET_NOT_ACTIVE` o `INSUFFICIENT_FUNDS` y 503 `PROVIDER_ERROR` son `failed` con su motivo. El 504 `TIMEOUT_AFTER_PROCESSING` es `failed` con el `paymentId`, y su monto SHALL sumarse al contador diario porque el proveedor lo aprobó.

#### Scenario: Saldo insuficiente
- **WHEN** W2 responde 409 `INSUFFICIENT_FUNDS`
- **THEN** la respuesta del intento es `failed` con motivo `INSUFFICIENT_FUNDS`

#### Scenario: Timeout después de procesar
- **WHEN** W2 responde 504 con `paymentId`
- **THEN** la respuesta del intento es `failed` con motivo `TIMEOUT_AFTER_PROCESSING` y el `paymentId`, y el monto se suma al contador diario

### Requirement: Resultado terminal único por intento
El sistema SHALL devolver exactamente un resultado terminal por intento (`succeeded` o `failed`). Un intento SHALL NOT tener estados intermedios visibles para el pagador.

#### Scenario: Respuesta de pendiente
- **WHEN** W2 responde 202
- **THEN** el sistema NO devuelve un resultado terminal hasta resolver la decisión abierta de `design.md`

### Requirement: Tratamiento de falta de respuesta
Si el cliente no recibe respuesta dentro de su tiempo de espera, SHALL tratar el intento como fallido y no mostrarlo como pendiente.

#### Scenario: Cliente sin respuesta
- **WHEN** el cliente no recibe respuesta a la petición de pago
- **THEN** el cliente muestra el intento como fallido

### Requirement: Respuesta de fallo con motivo
Cuando un intento termina en `failed`, la respuesta SHALL incluir un código de motivo estable para que el cliente ofrezca reintentar o volver.

#### Scenario: Fallo con motivo
- **WHEN** un intento termina en `failed`
- **THEN** la respuesta incluye `reason` con un código legible por el cliente
