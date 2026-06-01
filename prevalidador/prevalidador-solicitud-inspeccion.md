# API `prevalidadorSolicitudInspeccion`

Crea un registro en **`solicitudesInspeccion`** con el mismo payload y validaciones que el modal **Nueva solicitud de inspección** (modo crear) del panel VEC.

- El **prevalidador** se toma del `idToken` (no se envía en el body).
- El **estatus** inicial es siempre `pendiente` (no editable en creación).
- El **cliente** se resuelve por `cliente_id` y debe tener contrato vigente con ese prevalidador (mismo criterio que `prevalidadorListaClientes`).

**Requisitos previos:** [`prevalidadorLogin`](./prevalidador-auth.md) y, recomendado, [`prevalidadorListaClientes`](./prevalidador-lista-clientes.md) para obtener un `cliente_id` válido.

---

## Endpoint

| | |
|---|---|
| **Método** | `POST` |
| **URL (prod)** | `https://us-central1-vec-v2.cloudfunctions.net/prevalidadorSolicitudInspeccion` |
| **Content-Type** | `application/json` |
| **Auth** | `Authorization: Bearer <idToken>` |

---

## Request body

| Campo | Tipo | Requerido | Validación |
|---|---|---|---|
| `cliente_id` | string | Sí | Debe existir y ser elegible para el prevalidador del token |
| `vin` | string | Sí | No vacío |
| `fabricante` | string | Sí | No vacío |
| `modelo` | string | Sí | No vacío |
| `pais` | string | Sí | No vacío |
| `anio_modelo` | number o string | Sí | Entero entre `1900` y año actual + 1 |
| `nombre_propietario` | string | Sí | No vacío |

También se aceptan alias en camelCase (`clienteId`, `anioModelo`, `nombrePropietario`) por compatibilidad.

### Ejemplo

```json
{
  "cliente_id": "abc123cliente",
  "vin": "1HGBH41JXMN109186",
  "fabricante": "Honda",
  "modelo": "Civic",
  "pais": "México",
  "anio_modelo": 2022,
  "nombre_propietario": "Juan Pérez"
}
```

```bash
curl -s -X POST \
  "https://us-central1-vec-v2.cloudfunctions.net/prevalidadorSolicitudInspeccion" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${ID_TOKEN}" \
  -d '{
    "cliente_id": "abc123cliente",
    "vin": "1HGBH41JXMN109186",
    "fabricante": "Honda",
    "modelo": "Civic",
    "pais": "México",
    "anio_modelo": 2022,
    "nombre_propietario": "Juan Pérez"
  }'
```

---

## Response exitosa (201)

```json
{
  "success": true,
  "solicitud": {
    "id": "nuevoDocId",
    "vin": "1HGBH41JXMN109186",
    "fabricante": "Honda",
    "modelo": "Civic",
    "pais": "México",
    "anioModelo": "2022",
    "nombrePropietario": "Juan Pérez",
    "estatus": "pendiente",
    "cliente": {
      "id": "abc123cliente",
      "nombre": "Razón Social SA de CV",
      "alias": "Taller Norte",
      "numeroPatente": "1234",
      "rfc": "XAXX010101000"
    },
    "prevalidador": {
      "id": "mBbzLMgHM8hruaQv7rjSA8bz2",
      "nombre": "CAAAREM"
    },
    "createdBy": {
      "tipo": "prevalidador",
      "id": "mBbzLMgHM8hruaQv7rjSA8bz2",
      "nombre": "CAAAREM",
      "email": "prevalidador@ejemplo.com"
    },
    "modifiedAt": null,
    "modifiedBy": null
  }
}
```

### Auditoría al crear por API

| Campo | Valor |
|---|---|
| `createdAt` | Fecha/hora de creación en VEC (servidor) |
| `createdBy` | Prevalidador del token (`tipo: "prevalidador"`) |
| `modifiedAt` | `null` hasta la primera edición en el panel VEC |
| `modifiedBy` | `null` hasta que un usuario del panel VEC edite la solicitud |

`fechaRegistro` refleja la misma fecha de alta (visible en el panel VEC).

**Siguiente paso (cuando la inspección esté hecha en VEC):** [prevalidadorConsultaCertificado](./prevalidador-consulta-certificado.md) con el mismo `solicitud.id`.

---

## Errores

| HTTP | `error` | Cuándo |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Campos faltantes o `anio_modelo` fuera de rango |
| 401 | `missing-token` / `invalid-token` | Token ausente o inválido |
| 403 | `not-prevalidador` / `prevalidador-inactivo` | Token no es prevalidador activo |
| 403 | `cliente-no-elegible` | `cliente_id` sin contrato vigente con este prevalidador |
| 404 | `cliente-not-found` | `cliente_id` no existe |
| 405 | `METHOD_NOT_ALLOWED` | No es POST |
| 500 | `INTERNAL_ERROR` | Fallo interno |

### Ejemplo validación (400)

```json
{
  "success": false,
  "error": "VALIDATION_ERROR",
  "message": "Datos de entrada inválidos",
  "details": [
    { "field": "vin", "message": "Requerido" },
    { "field": "anio_modelo", "message": "El año máximo es 2027" }
  ]
}
```

---

## Flujo integrador

```mermaid
sequenceDiagram
  participant API as Integrador
  participant Login as prevalidadorLogin
  participant Clientes as prevalidadorListaClientes
  participant Crear as prevalidadorSolicitudInspeccion

  API->>Login: POST credenciales
  Login-->>API: idToken
  API->>Clientes: GET Bearer
  Clientes-->>API: clientes[].id
  API->>Crear: POST body + Bearer
  Crear-->>API: solicitud.id (201)
  Note over API: Después: operación VEC + consulta certificado
```

Ver flujo completo: [README.md](./README.md) y [prevalidador-consulta-certificado.md](./prevalidador-consulta-certificado.md).
