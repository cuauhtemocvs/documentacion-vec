# Autenticación de prevalidadores (integración con VEC)

Guía para **servicios backend** de organizaciones prevalidadoras que consumen las **APIs HTTP de VEC**. No hay acceso directo a la base de datos de VEC: todo el intercambio de datos ocurre mediante estas APIs y un token de sesión.

Índice general: [README.md](./README.md).

---

## Alcance de datos

VEC solo devuelve información asociada al prevalidador autenticado (identificado por el token):

| Recurso | Qué puede ver el prevalidador |
|---|---|
| Su cuenta de prevalidador | Datos propios del usuario de integración |
| Solicitudes de inspección | Las que él registró o que están a su nombre |
| Inspecciones / certificados | Las vinculadas a sus solicitudes (o asignadas explícitamente a él) |

Cualquier otro dato del ecosistema VEC queda **fuera de alcance**.

---

## Resumen del flujo

```mermaid
sequenceDiagram
  participant Integrador as Servidor integrador
  participant Login as prevalidadorLogin
  participant API as API VEC
  participant VEC as VEC (admin)

  Integrador->>Login: POST email + password
  Login-->>Integrador: idToken + expiresIn
  Integrador->>API: Authorization Bearer idToken
  API->>VEC: validar prevalidador y recurso
  VEC-->>API: datos permitidos
  API-->>Integrador: respuesta JSON
  Note over Integrador,Login: Si el token expira, repetir login
```

---

## 1. Login — `prevalidadorLogin`

### Endpoint

| | |
|---|---|
| **Método** | `POST` |
| **URL (prod)** | `https://us-central1-vec-v2.cloudfunctions.net/prevalidadorLogin` |
| **Content-Type** | `application/json` |

### Request

```json
{
  "email": "prevalidador@ejemplo.com",
  "password": "su-contraseña"
}
```

Las credenciales las proporciona **VEC** al dar de alta la integración.

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
| `idToken` | Token de sesión. Enviar en APIs posteriores: `Authorization: Bearer …`. Suele expirar en ~1 h (`expiresIn` en segundos). |
| `expiresIn` | Segundos de validez (típicamente `"3600"`). |

> **No se devuelve refresh token.** Renueve sesión volviendo a llamar `prevalidadorLogin` con las mismas credenciales.

### Errores

| HTTP | `error` | Cuándo |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Email o password faltante/inválido |
| 401 | `invalid-credentials` | Email o contraseña incorrectos |
| 403 | `not-prevalidador` / `prevalidador-inactivo` | Cuenta no habilitada como prevalidador activo en VEC |
| 405 | `METHOD_NOT_ALLOWED` | No es POST |
| 500 | `INTERNAL_ERROR` | Fallo interno |

---

## 2. Listar clientes — `prevalidadorListaClientes`

Documentación: **[prevalidador-lista-clientes.md](./prevalidador-lista-clientes.md)**

`GET` con `Authorization: Bearer <idToken>`. Devuelve clientes con contrato vigente asociados al prevalidador del token.

---

## 3. Crear solicitud — `prevalidadorSolicitudInspeccion`

Documentación: **[prevalidador-solicitud-inspeccion.md](./prevalidador-solicitud-inspeccion.md)**

`POST` con datos del vehículo y `cliente_id`. El prevalidador y el estatus inicial `pendiente` los asigna VEC. **Guarde el `solicitud.id` de la respuesta.**

---

## 4. Actualizar solicitud — `prevalidadorActualizaSolicitudInspeccion`

Documentación: **[prevalidador-actualiza-solicitud-inspeccion.md](./prevalidador-actualiza-solicitud-inspeccion.md)**

`POST` con `solicitud_id` y los mismos datos del vehículo que en la creación. Permite corregir una solicitud en estatus `pendiente` o `enProceso`. Si la inspección vinculada ya está finalizada, la API responde `409` y no modifica ningún registro.

---

## 5. Listar solicitudes — `prevalidadorListaSolicitudes`

Documentación: **[prevalidador-lista-solicitudes.md](./prevalidador-lista-solicitudes.md)**

`GET` con `Authorization: Bearer <idToken>`. Devuelve las solicitudes del prevalidador autenticado, con filtros opcionales por fecha, VIN y orden.

---

## 6. Consultar certificado — `prevalidadorConsultaCertificado`

Documentación: **[prevalidador-consulta-certificado.md](./prevalidador-consulta-certificado.md)**

`GET` o `POST` con `solicitud_id` (el mismo ID del paso 3), después de que VEC haya asignado la inspección y esta se haya ejecutado. Devuelve el certificado en JSON.

---

## 7. Renovar sesión

Cuando una API responda `401` con `"Token inválido o expirado."`:

1. Llamar de nuevo `prevalidadorLogin`.
2. Guardar el nuevo `idToken`.
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

Opcional: cachear el token y renovarlo antes de que venza `expiresIn`.

---

## 7. Header en todas las APIs

```http
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
Content-Type: application/json
```

Ejemplo:

```bash
curl -s -X GET \
  "https://us-central1-vec-v2.cloudfunctions.net/prevalidadorListaClientes" \
  -H "Authorization: Bearer ${ID_TOKEN}"
```

---

## 8. Modelo de asociación (referencia)

Entender estos vínculos ayuda a interpretar respuestas y errores `403` / `404`.

### Solicitudes

Cada solicitud creada por API queda asociada al prevalidador del token:

```
solicitud de inspección
  └── prevalidador: { id, nombre }
```

### Inspecciones

Una inspección en VEC puede vincularse al prevalidador de dos formas (VEC las trata como equivalentes para autorización):

**Directa** (preferida cuando existe el dato):

```
inspección
  └── prevalidador: { id, nombre }
```

**Por solicitud** (habitual tras asignar crédito en panel/app VEC):

```
inspección
  └── solicitud de inspección vinculada
        └── prevalidador: { id, nombre }
```

La API de certificado localiza la inspección por `solicitud_id` (relación 1:1). Ver [prevalidador-consulta-certificado.md](./prevalidador-consulta-certificado.md).

---

## 9. Seguridad

- Credenciales de integración **solo en su servidor** (variables de entorno, gestor de secretos).
- HTTPS obligatorio en todas las llamadas.
- No compartir el `idToken` con aplicaciones cliente finales si no es estrictamente necesario; prefiera que su backend llame a VEC.
- Limite reintentos de login en su lado (protección ante fuerza bruta).
- Ante robo de credenciales, notificar a **VEC** de inmediato para rotación.

---

## Alta de cuenta

Para obtener usuario, contraseña y ambiente de pruebas (si aplica), solicitar al administrador **VEC** de su organización: debe existir un prevalidador **activo** vinculado a los clientes/contratos que usará la integración.
