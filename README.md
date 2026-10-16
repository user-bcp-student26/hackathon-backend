# hackathon-backend
Software backend con agentes y supervisión humana.

## Probar la API (H-01)

Contrato: `src/main/resources/META-INF/resources/openapi.yaml` (H-01 onboarding, wallet y KYC).

Las wallets se guardan en la tabla SQLite `wallets` (`wallet_id`, `user_id`, `status`, `created_at`).
Al iniciar, los registros de wallet existentes en `records` se migran a esa tabla; `payments_enabled`
sigue derivándose del estado KYC.
Los contactos se guardan en la tabla SQLite `contacts` (`id`, `owner_id`, `name`, `phone`, `wallet_id`,
`created_at`). Los contactos existentes en `records` se migran a esa tabla al iniciar.
Al crear una wallet, el backend solicita su creación al mock BCP del grupo 3 (`POST /groups/3/kyc/verifications`)
con `documentType: TEST`, `document` con el formato `TEST-<user_id>` y `scenario: APPROVED`. El `wallet_id` y estado
recibidos se guardan junto al ID del usuario. La URL base se puede configurar con `BCP_WALLET_BASE_URL`.

### URL pública de pruebas

- Base URL (ngrok, temporal): `https://opulently-crowd-watch.ngrok-free.dev`
- Solo funciona mientras el servicio local y el túnel de ngrok estén activos. Si se reinicia ngrok, la URL cambia.
- Es una base de datos de pruebas: no uses datos reales.

### Levantar el servicio local

```powershell
mvn -q package -DskipTests
java '-Dquarkus.http.port=8090' -jar target\quarkus-app\quarkus-run.jar
```

Para exponerlo con ngrok: `ngrok http 8090`.

### Probar con Postman

1. Importa `postman/H01-onboarding-wallet-kyc.postman_collection.json` en Postman.
2. Cambia la variable de colección `baseUrl`:
   - Local: `http://localhost:8090`
   - Pública: `https://opulently-crowd-watch.ngrok-free.dev`
3. Ejecuta las peticiones en orden (carpetas 1 a 5). Cada prueba depende del estado de las anteriores.
   - `POST /users` genera un correo único y guarda `email`.
   - Los endpoints protegidos usan Basic Auth (`email` / `password`) desde las variables de la colección.
4. Para ejecutar toda la colección desde terminal: `npx newman run postman\H01-onboarding-wallet-kyc.postman_collection.json --env-var "baseUrl=https://opulently-crowd-watch.ngrok-free.dev"`.

### Probar con curl

```powershell
curl.exe -s -H "Content-Type: application/json" -d "@body.json" https://opulently-crowd-watch.ngrok-free.dev/users
curl.exe -s -u "correo@example.com:password123" -X POST https://opulently-crowd-watch.ngrok-free.dev/auth/login
```

### Endpoint `/wallet`

Requiere autenticación Basic Auth (`email` / `password`) del usuario.

| Método | Descripción | Respuestas |
|--------|-------------|------------|
| `GET /wallet` | Obtiene la wallet del usuario autenticado. | `200` wallet · `401` no autenticado · `404` el usuario aún no tiene wallet |
| `POST /wallet` | Genera la wallet del usuario autenticado (relación 1 a 1). No requiere KYC aprobado. | `201` wallet creada en estado `active` · `401` no autenticado · `409` el usuario ya tiene wallet |

Esquema `Wallet`:

```json
{
  "wallet_id": "3f2b8c1e-9a4d-4e6f-b7c2-1d5e8f9a0b12",
  "status": "active",
  "payments_enabled": false,
  "createdAt": "2026-10-16T14:51:04.886-05:00"
}
```

- `wallet_id`: UUID v4 generado por el sistema.
- `payments_enabled`: `true` solo si la última verificación KYC está `approved`.

Con curl:

```powershell
curl.exe -s -u "correo@example.com:password123" https://opulently-crowd-watch.ngrok-free.dev/wallet
curl.exe -s -u "correo@example.com:password123" -X POST https://opulently-crowd-watch.ngrok-free.dev/wallet
```
