# API `prevalidadorListaClientes`

Lista los **clientes** que su prevalidador puede usar al registrar solicitudes de inspección en **VEC**. Aplica los mismos criterios de elegibilidad que al crear una solicitud.

**Requisito previo:** token obtenido con [`prevalidadorLogin`](./prevalidador-auth.md#1-login--prevalidadorlogin).

---

## Endpoint

| | |
|---|---|
| **API** | `prevalidadorListaClientes` |
| **Método** | `GET` |
| **URL (prod)** | `https://us-central1-vec-v2.cloudfunctions.net/prevalidadorListaClientes` |
| **Auth** | `Authorization: Bearer <idToken>` |
| **Body** | Ninguno |
| **CORS** | Cualquier origen (pensado para server-to-server) |

---

## Flujo recomendado

```mermaid
sequenceDiagram
  participant Integrador
  participant Login as prevalidadorLogin
  participant Lista as prevalidadorListaClientes

  Integrador->>Login: POST email + password
  Login-->>Integrador: idToken
  Integrador->>Lista: GET + Bearer idToken
  Lista-->>Integrador: clientes[] (contrato vigente)
```

1. Llamar `prevalidadorLogin` y guardar `idToken`.
2. Llamar `prevalidadorListaClientes` con el header `Authorization`.
3. Usar el `id` del cliente elegido al crear solicitudes en APIs posteriores (objeto `cliente` embebido).

---

## Criterio de inclusión

Un cliente aparece si en **VEC** tiene **al menos un contrato** que cumpla **ambas** condiciones:

| Condición | Descripción |
|---|---|
| Prevalidador asignado | El contrato está asociado al mismo prevalidador que el token de sesión |
| Vigencia actual | La fecha de hoy está dentro del periodo de vigencia del contrato (`YYYY-MM-DD`) |

**No se incluyen:**

- Clientes sin contratos
- Contratos asociados a otro prevalidador
- Contratos vencidos o aún no iniciados
- Contratos sin periodo de vigencia válido

**Orden:** ascendente por `alias`.

---

## Request

```http
GET /prevalidadorListaClientes HTTP/1.1
Host: us-central1-vec-v2.cloudfunctions.net
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
```

### Ejemplo curl

```bash
curl -s -X GET \
  "https://us-central1-vec-v2.cloudfunctions.net/prevalidadorListaClientes" \
  -H "Authorization: Bearer ${ID_TOKEN}"
```

### Ejemplo (Node / fetch)

```typescript
const res = await fetch(
  "https://us-central1-vec-v2.cloudfunctions.net/prevalidadorListaClientes",
  {
    method: "GET",
    headers: {Authorization: `Bearer ${idToken}`},
  }
);
const data = await res.json();
if (!data.success) {
  throw new Error(`${data.error}: ${data.message}`);
}
console.log(data.clientes);
```

---

## Response exitosa (200)

```json
{
  "success": true,
  "total": 2,
  "prevalidador": {
    "id": "mBbzLMgHM8hruaQv7rjSA8bz2",
    "nombre": "CAAAREM"
  },
  "clientes": [
    {
      "id": "abc123cliente",
      "nombre": "Razón Social SA de CV",
      "alias": "Taller Norte",
      "numeroPatente": "1234",
      "rfc": "XAXX010101000",
      "usuarioPrevalidador": "usuario-taller-norte",
      "etiqueta": "1234 - Taller Norte (Razón Social SA de CV)"
    }
  ]
}
```

### Campos de cada cliente

| Campo | Tipo | Descripción |
|---|---|---|
| `id` | string | Identificador del cliente en VEC (`cliente_id` para otras APIs) |
| `nombre` | string | Razón social / nombre del cliente |
| `alias` | string | Nombre corto en el panel |
| `numeroPatente` | string | Número de patente del cliente |
| `rfc` | string | RFC |
| `usuarioPrevalidador` | string | Usuario del contrato vigente asociado al prevalidador de la sesión. Si no está definido, se devuelve `""` |
| `etiqueta` | string | Texto descriptivo: `{patente} - {alias} ({nombre})` |

### Uso al crear una solicitud

Al persistir una solicitud de inspección, el objeto `cliente` debe usar la misma forma (sin `etiqueta`):

```json
{
  "cliente": {
    "id": "abc123cliente",
    "nombre": "Razón Social SA de CV",
    "alias": "Taller Norte",
    "numeroPatente": "1234",
    "rfc": "XAXX010101000"
  }
}
```

### Lista vacía

Si el prevalidador no tiene clientes con contrato vigente:

```json
{
  "success": true,
  "total": 0,
  "prevalidador": { "id": "...", "nombre": "..." },
  "clientes": []
}
```

No es error HTTP; el integrador debe manejar el caso en su aplicación.

---

## Errores

| HTTP | `error` | Cuándo |
|---|---|---|
| 401 | `missing-token` | Falta header `Authorization` o no es `Bearer …` |
| 401 | `invalid-token` | Token inválido o expirado → renovar con `prevalidadorLogin` |
| 403 | `not-prevalidador` / `prevalidador-inactivo` | Cuenta no habilitada como prevalidador activo en VEC |
| 405 | `METHOD_NOT_ALLOWED` | No es `GET` |
| 500 | `INTERNAL_ERROR` | Fallo interno en VEC |

Formato de error:

```json
{
  "success": false,
  "error": "invalid-token",
  "message": "Token inválido o expirado."
}
```
