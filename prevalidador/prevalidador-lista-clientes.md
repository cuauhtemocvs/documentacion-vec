# API `prevalidadorListaClientes`

Lista los **clientes** que un prevalidador puede usar al registrar solicitudes de inspección. Misma lógica y mismos campos que el select **Cliente** del modal de solicitud en el panel VEC.

**Requisito previo:** token obtenido con [`prevalidadorLogin`](./prevalidador-auth.md#1-login--prevalidadorlogin).

---

## Endpoint

| | |
|---|---|
| **Function** | `prevalidadorListaClientes` |
| **Método** | `GET` |
| **URL (prod)** | `https://us-central1-vec-v2.cloudfunctions.net/prevalidadorListaClientes` |
| **Región** | `us-central1` |
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

Un cliente entra en la lista si, en Firestore (`clientes/{id}`), existe **al menos un contrato** que cumpla **ambas** condiciones:

| Condición | Campo |
|---|---|
| Prevalidador asignado | `contratos[].prevalidador.id === uid` del token |
| Vigencia actual | Fecha de hoy ∈ `[vigencia.inicio, vigencia.fin]` (día calendario, `YYYY-MM-DD`) |

Equivale a `ModalSolicitudInspeccionComponent.cargarClientesPorPrevalidador` en el front Angular.

**No se incluyen:**

- Clientes sin contratos
- Contratos con otro `prevalidador.id`
- Contratos vencidos o aún no iniciados
- Contratos sin `vigencia.inicio` / `vigencia.fin` válidos

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
      "etiqueta": "1234 - Taller Norte (Razón Social SA de CV)"
    }
  ]
}
```

### Campos de cada cliente

| Campo | Tipo | Descripción |
|---|---|---|
| `id` | string | ID del documento en `clientes/{id}` |
| `nombre` | string | Razón social / nombre del cliente |
| `alias` | string | Nombre corto en el panel |
| `numeroPatente` | string | Número de patente del cliente |
| `rfc` | string | RFC |
| `etiqueta` | string | Texto del select en el panel: `{patente} - {alias} ({nombre})` |

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

No es error HTTP; el integrador debe manejar el caso (equivalente al mensaje *"No hay clientes con contrato vigente para este prevalidador"* en el panel).

---

## Errores

| HTTP | `error` | Cuándo |
|---|---|---|
| 401 | `missing-token` | Falta header `Authorization` o no es `Bearer …` |
| 401 | `invalid-token` | Token inválido o expirado → renovar con `prevalidadorLogin` |
| 403 | `not-prevalidador` / `prevalidador-inactivo` | Usuario Auth válido pero no es prevalidador activo en Firestore |
| 405 | `METHOD_NOT_ALLOWED` | No es `GET` |
| 500 | `INTERNAL_ERROR` | Fallo interno al consultar Firestore |

Formato de error:

```json
{
  "success": false,
  "error": "invalid-token",
  "message": "Token inválido o expirado."
}
```
