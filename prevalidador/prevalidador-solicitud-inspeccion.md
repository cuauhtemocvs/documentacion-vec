# API `prevalidadorSolicitudInspeccion`

Registra una **nueva solicitud de inspección** en VEC.

- El **prevalidador** se identifica con el token de sesión (no se envía en el body).
- El **estatus** inicial es siempre `pendiente` (no editable en creación).
- El **cliente** se indica con `cliente_id` **o** con `numero_patente` (al menos uno; si vienen ambos, manda `cliente_id`).

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
| `cliente_id` | string | Condicional | Requerido si no se envía `numero_patente`. Debe existir y ser elegible para el prevalidador del token. `null`, `"null"`, `"undefined"` y `""` se tratan como no enviados |
| `numero_patente` | string o number | Condicional | Requerido si no se envía `cliente_id`. 4 dígitos numéricos (`^\d{4}$`) tras normalizar: se quitan espacios y se completan ceros a la izquierda (`123` → `"0123"`) |
| `vin` | string | Sí | No vacío |
| `fabricante` | string | Sí | No vacío |
| `modelo` | string | Sí | No vacío |
| `pais` | string | Sí | No vacío |
| `anio_modelo` | number o string | Sí | Entero entre `1900` y año actual + 1 |
| `nombre_propietario` | string | Sí | No vacío |

También se aceptan alias en camelCase (`clienteId`, `numeroPatente`, `anioModelo`, `nombrePropietario`) por compatibilidad.

Si se envían **ambos** `cliente_id` y `numero_patente`, **tiene prioridad `cliente_id`** (es único en el sistema) y **no se realiza la búsqueda de patente compartida**; `numero_patente` se ignora. La resolución por patente solo aplica cuando se envía `numero_patente` sin `cliente_id`.

### Resolución por `numero_patente`

1. Se buscan clientes con ese número de patente (exacto tras `trim`).
2. Si no hay ninguno → `404 cliente-not-found`.
3. Si ninguno es elegible (contrato vigente con el prevalidador del token) → `403 cliente-no-elegible`.
4. Si algún cliente tiene `patenteCompartida === true` → **caso patente compartida**: la solicitud se crea con embed parcial de cliente.
5. En caso contrario → **caso normal**: se toma el **primer cliente elegible** (mismo embed completo que con `cliente_id`). Si hay varios con `patenteCompartida === false` (dato inconsistente), también se usa el primero elegible.

### Ejemplo con `cliente_id`

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

### Ejemplo con `numero_patente` (cliente único)

```json
{
  "numero_patente": "1234",
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
    "numero_patente": "1234",
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

### Cliente resuelto (por `cliente_id` o patente no compartida)

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

### Patente compartida

Cuando `numero_patente` corresponde a clientes con `patenteCompartida === true`, el embed de cliente queda así (el resto se completa al asignar en VEC):

```json
{
  "success": true,
  "solicitud": {
    "id": "nuevoDocId",
    "vin": "1HGBH41JXMN109186",
    "estatus": "pendiente",
    "cliente": {
      "id": null,
      "nombre": null,
      "alias": null,
      "numeroPatente": "5678",
      "rfc": null,
      "patenteCompartida": true
    },
    "prevalidador": {
      "id": "mBbzLMgHM8hruaQv7rjSA8bz2",
      "nombre": "CAAAREM"
    }
  }
}
```

### Metadatos de la solicitud creada

| Campo | Valor |
|---|---|
| `createdBy` | Prevalidador de la sesión (`tipo: "prevalidador"`) |
| `modifiedAt` | `null` hasta que VEC modifique la solicitud |
| `modifiedBy` | `null` hasta que VEC modifique la solicitud |

La fecha de alta de la solicitud queda registrada en VEC como `fechaRegistro` (consultable en [`prevalidadorListaSolicitudes`](./prevalidador-lista-solicitudes.md)).

**Consultar solicitudes creadas:** [prevalidadorListaSolicitudes](./prevalidador-lista-solicitudes.md).

**Corregir datos de una solicitud existente:** [prevalidadorActualizaSolicitudInspeccion](./prevalidador-actualiza-solicitud-inspeccion.md) con el mismo `solicitud.id`.

**Siguiente paso (cuando la inspección esté hecha en VEC):** [prevalidadorConsultaCertificado](./prevalidador-consulta-certificado.md) con el mismo `solicitud.id`.

---

## Errores

| HTTP | `error` | Cuándo |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Campos faltantes, `anio_modelo` fuera de rango, o `numero_patente` no es de 4 dígitos |
| 401 | `missing-token` / `invalid-token` | Token ausente o inválido |
| 403 | `not-prevalidador` / `prevalidador-inactivo` | Token no es prevalidador activo |
| 403 | `cliente-no-elegible` | Cliente(s) sin contrato vigente con este prevalidador |
| 404 | `cliente-not-found` | `cliente_id` no existe, o ningún cliente con ese `numero_patente` |
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
  alt Por cliente_id
    API->>Clientes: GET Bearer
    Clientes-->>API: clientes[].id
    API->>Crear: POST cliente_id + Bearer
  else Por numero_patente
    API->>Crear: POST numero_patente + Bearer
  end
  Crear-->>API: solicitud.id (201)
  Note over API: Después: operación VEC + consulta certificado
```

Ver flujo completo: [README.md](./README.md) y [prevalidador-consulta-certificado.md](./prevalidador-consulta-certificado.md).
