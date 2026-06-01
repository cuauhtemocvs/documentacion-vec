# Autenticación de Prevalidadores (integraciones server-to-server)

Integración pensada para **servicios backend** que consumen Cloud Functions HTTP (`onRequest`). No hay acceso directo a Firestore desde el cliente del prevalidador: todo pasa por APIs con token.

## Alcance de datos

| Colección | Acceso |
|---|---|
| `prevalidadores/{uid}` | Solo el propio registro (`uid` del token) |
| `solicitudesInspeccion` | Solo donde `prevalidador.id === uid` |
| `inspecciones` | Donde `prevalidador.id === uid` **o** la solicitud ligada pertenece al prevalidador |

Cualquier otra colección queda **fuera de alcance**.

## Resumen del flujo

```mermaid
sequenceDiagram
  participant Servidor as Servidor integrador
  participant Login as prevalidadorLogin
  participant API as API onRequest
  participant FS as Firestore Admin

  Servidor->>Login: POST email + password
  Login-->>Servidor: idToken + expiresIn
  Servidor->>API: Authorization Bearer idToken
  API->>API: verifyIdToken
  API->>FS: prevalidador activo + recurso asociado
  API-->>Servidor: datos filtrados
  Note over Servidor,Login: Si idToken expira, repetir prevalidadorLogin
```

---

## 1. Login — `prevalidadorLogin`

### Endpoint

| | |
|---|---|
| **Método** | `POST` |
| **URL (prod)** | `https://us-central1-vec-v2.cloudfunctions.net/prevalidadorLogin` |
| **Content-Type** | `application/json` |
| **CORS** | Cualquier origen (`Access-Control-Allow-Origin: *`) |

### Request

```json
{
  "email": "prevalidador@ejemplo.com",
  "password": "su-contraseña"
}
```

### Response exitosa (200)

```json
{
  "success": true,
  "tokenType": "Bearer",
  "idToken": "eyJhbGciOiJSUzI1NiIs...",
  "expiresIn": "3600",
  "prevalidador": {
    "id": "mBbzLMgHM8hruaQv7rjSA8bz2",
    "nombre": "CAAAREM",
    "email": "prevalidador@ejemplo.com"
  }
}
```

| Campo | Uso |
|---|---|
| `idToken` | JWT de Firebase Auth. Enviar en APIs posteriores: `Authorization: Bearer …`. Expira en ~1 h (`expiresIn` segundos). |
| `expiresIn` | Segundos de validez (típicamente `"3600"`). |

> **No se devuelve `refreshToken`.** La Web API Key de Firebase queda solo en el servidor (Cloud Functions). El integrador renueva sesión volviendo a llamar `prevalidadorLogin` con las mismas credenciales.

### Errores

| HTTP | `error` | Cuándo |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Email o password faltante/inválido |
| 401 | `invalid-credentials` | Email o contraseña incorrectos |
| 403 | `not-prevalidador` / `prevalidador-inactivo` | No es prevalidador activo |
| 405 | `METHOD_NOT_ALLOWED` | No es POST |
| 500 | `INTERNAL_ERROR` | Fallo interno |

---

## 2. Listar clientes — `prevalidadorListaClientes`

Documentación completa: **[prevalidador-lista-clientes.md](./prevalidador-lista-clientes.md)**

Resumen: `GET` con `Authorization: Bearer <idToken>`. Devuelve clientes con contrato vigente asociados al prevalidador del token (mismo criterio que el select **Cliente** del modal de solicitud). Índice de docs: [README.md](./README.md).

---

## 3. Crear solicitud — `prevalidadorSolicitudInspeccion`

Documentación completa: **[prevalidador-solicitud-inspeccion.md](./prevalidador-solicitud-inspeccion.md)**

Resumen: `POST` con `cliente_id`, `vin`, `fabricante`, `modelo`, `pais`, `anio_modelo`, `nombre_propietario`. Prevalidador y estatus `pendiente` se asignan en servidor.

---

## 4. Renovar sesión (sin endpoint extra)

Cuando una API responda `401` con `"Token inválido o expirado."`:

1. Volver a llamar `prevalidadorLogin` con email y password (credenciales en el servidor integrador, nunca en el navegador del usuario final si aplica).
2. Guardar el nuevo `idToken` y `expiresIn`.
3. Reintentar la petición fallida.

Ejemplo (pseudo-código):

```typescript
async function obtenerIdToken(): Promise<string> {
  const res = await fetch(PREVALIDADOR_LOGIN_URL, {
    method: "POST",
    headers: {"Content-Type": "application/json"},
    body: JSON.stringify({
      email: process.env.PREVALIDADOR_EMAIL,
      password: process.env.PREVALIDADOR_PASSWORD,
    }),
  });
  const data = await res.json();
  if (!data.success) throw new Error(data.message);
  return data.idToken;
}
```

Opcional: cachear el token en memoria y renovarlo unos minutos antes de `expiresIn`.

---

## 5. Usar el token en APIs `onRequest`

Header obligatorio en cada request:

```http
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
Content-Type: application/json
```

Ejemplo curl:

```bash
curl -X POST "https://us-central1-vec-v2.cloudfunctions.net/miApiPrevalidador" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${ID_TOKEN}" \
  -d '{"solicitudInspeccionId": "abc123"}'
```

---

## 6. Implementar una API con restricción por asociación

Helpers en `functions/src/lib/prevalidadorAuth.ts`:

| Función | Uso |
|---|---|
| `verifyPrevalidadorBearerToken(authHeader)` | Valida JWT + prevalidador activo |
| `assertSolicitudInspeccionBelongsToPrevalidador(id, uid)` | Solicitud del prevalidador |
| `assertInspeccionBelongsToPrevalidador(id, uid)` | Inspección del prevalidador (directo o vía solicitud) |
| `resolveInspeccionPrevalidadorId(data)` | Resuelve el uid dueño desde el documento de inspección |
| `sendPrevalidadorAuthError(res, error)` | Respuesta JSON de error estándar |

### Ejemplo: consultar una solicitud

```typescript
import {onRequest} from "firebase-functions/v2/https";
import cors from "cors";
import {
  assertSolicitudInspeccionBelongsToPrevalidador,
  sendPrevalidadorAuthError,
  verifyPrevalidadorBearerToken,
} from "../lib/prevalidadorAuth";

const corsHandler = cors({origin: true});

export const prevalidadorGetSolicitud = onRequest(
  {region: "us-central1"},
  async (req, res) => {
    await corsHandler(req, res, async () => {
      try {
        const {uid} = await verifyPrevalidadorBearerToken(
          req.headers.authorization
        );
        const solicitudId = String(req.query.id ?? req.body?.id ?? "").trim();
        if (!solicitudId) {
          res.status(400).json({
            success: false,
            error: "VALIDATION_ERROR",
            message: "id es requerido.",
          });
          return;
        }

        const data = await assertSolicitudInspeccionBelongsToPrevalidador(
          solicitudId,
          uid
        );

        res.json({success: true, solicitud: {id: solicitudId, ...data}});
      } catch (error) {
        if (sendPrevalidadorAuthError(res, error)) return;
        res.status(500).json({success: false, error: "INTERNAL_ERROR"});
      }
    });
  }
);
```

### Ejemplo: consultar una inspección

```typescript
const {uid} = await verifyPrevalidadorBearerToken(req.headers.authorization);
const data = await assertInspeccionBelongsToPrevalidador(inspeccionId, uid);
```

### Listar solicitudes del prevalidador

Consulta Firestore (Admin SDK) filtrando por asociación:

```typescript
const snap = await db
  .collection("solicitudesInspeccion")
  .where("prevalidador.id", "==", uid)
  .get();
```

Registros legacy con `prevalidador` como string deben usar el UID como valor del string, o migrarse al formato `{ id, nombre }`.

### Listar inspecciones del prevalidador

1. Obtener IDs de solicitudes del prevalidador (consulta anterior).
2. Consultar `inspecciones` donde `datosAsignacion.solicitudInspeccionId` esté en ese listado (máx. 30 IDs por query `in` en Firestore; paginar o hacer varias queries si hace falta).

---

## 7. Modelo de asociación

### Solicitudes

```
solicitudesInspeccion/{id}
  └── prevalidador: { id: "<uid>", nombre: "..." }
```

### Inspecciones (dos formas compatibles)

**Preferido / futuro** — prevalidador embebido en la inspección:

```
inspecciones/{id}
  └── prevalidador: { id: "<uid>", nombre: "..." }
```

**Actual (fallback)** — enlace vía solicitud:

```
inspecciones/{id}
  └── datosAsignacion.solicitudInspeccionId → solicitudesInspeccion/{id}
        └── prevalidador: { id: "<uid>", nombre: "..." }
```

`assertInspeccionBelongsToPrevalidador` y `resolveInspeccionPrevalidadorId` evalúan **primero** el campo directo `inspeccion.prevalidador`; si no existe, siguen la cadena por solicitud. Así las APIs actuales siguen funcionando y, al migrar, basta con escribir `prevalidador` al crear o actualizar la inspección.

### Consultas para listar inspecciones

Cuando las inspecciones ya tengan `prevalidador` embebido:

```typescript
const snap = await db
  .collection("inspecciones")
  .where("prevalidador.id", "==", uid)
  .get();
```

Mientras coexistan registros sin ese campo, combinar con solicitudes (como en la sección 4) o migrar datos gradualmente.

---

## 8. Seguridad

- Credenciales del prevalidador solo en el **servidor integrador** (variables de entorno, secret manager).
- HTTPS obligatorio.
- No exponer la Web API Key de Firebase al integrador; solo la usan las Cloud Functions internamente.
- Rate limiting en el integrador al llamar login (evitar brute force).
- Validar siempre token + estatus activo + pertenencia del recurso en cada API.

---

## 9. Despliegue

```bash
firebase deploy --only functions:prevalidadorLogin,functions:prevalidadorListaClientes,functions:prevalidadorSolicitudInspeccion
```

### Multi-entorno (dev/qa/prod)

Define la Web API Key por entorno en archivos `.env` de Functions.
Por ahora usa `functions/.env.prod`:

- `functions/.env.prod`

Contenido:

```bash
FIREBASE_WEB_API_KEY=AIzaSy...
```

El código lee `process.env.FIREBASE_WEB_API_KEY` y, si no existe, usa un
fallback compatible con el proyecto actual.

Requisitos:

1. Prevalidador creado en admin (Auth + `prevalidadores/{uid}` con `estatus: "activo"`).
2. Solicitudes con `prevalidador.id` igual al UID del prevalidador.
