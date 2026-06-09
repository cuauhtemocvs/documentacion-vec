# API `prevalidadorListaSolicitudes`

Lista las **solicitudes de inspección** registradas por el prevalidador autenticado. Solo se incluyen solicitudes que pertenecen a su cuenta (mismo criterio de acceso que [`prevalidadorConsultaCertificado`](./prevalidador-consulta-certificado.md)).

**Requisito previo:** token obtenido con [`prevalidadorLogin`](./prevalidador-auth.md).

---

## Endpoint

| | |
|---|---|
| **API** | `prevalidadorListaSolicitudes` |
| **Método** | `GET` |
| **URL (prod)** | `https://us-central1-vec-v2.cloudfunctions.net/prevalidadorListaSolicitudes` |
| **Auth** | `Authorization: Bearer <idToken>` |
| **Body** | Ninguno |
| **CORS** | Cualquier origen (pensado para server-to-server) |

---

## Query params (opcionales)

| Parámetro | Tipo | Default | Descripción |
|---|---|---|---|
| `orden` | `ASC` \| `DESC` | `DESC` | Orden por `fechaRegistro` |
| `fechaIni` | `yyyy-mm-dd` | Hoy − 1 mes (UTC) | Incluye solicitudes con `fechaRegistro` ≥ inicio del día |
| `fechaFin` | `yyyy-mm-dd` | Hoy (UTC) | Incluye solicitudes con `fechaRegistro` ≤ fin del día |
| `vin` | string | — | Filtro exacto por VIN (insensible a mayúsculas/minúsculas) |

También se aceptan alias `fecha_ini` y `fecha_fin`.

---

## Request

```http
GET /prevalidadorListaSolicitudes?orden=DESC&fechaIni=2025-05-08&fechaFin=2025-06-08 HTTP/1.1
Host: us-central1-vec-v2.cloudfunctions.net
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
```

### Ejemplo curl

```bash
curl -s -G \
  "https://us-central1-vec-v2.cloudfunctions.net/prevalidadorListaSolicitudes" \
  -H "Authorization: Bearer ${ID_TOKEN}" \
  --data-urlencode "orden=DESC" \
  --data-urlencode "fechaIni=2025-05-08" \
  --data-urlencode "fechaFin=2025-06-08" \
  --data-urlencode "vin=1HGBH41JXMN109186"
```

---

## Response exitosa (200)

```json
{
  "success": true,
  "total": 1,
  "prevalidador": {
    "id": "mBbzLMgHM8hruaQv7rjSA8bz2",
    "nombre": "CAAAREM"
  },
  "filtros": {
    "orden": "DESC",
    "fechaIni": "2025-05-08",
    "fechaFin": "2025-06-08",
    "vin": "1HGBH41JXMN109186"
  },
  "solicitudes": [
    {
      "id": "nuevoDocId",
      "cliente_id": "abc123cliente",
      "vin": "1HGBH41JXMN109186",
      "fabricante": "Honda",
      "modelo": "Civic",
      "pais": "México",
      "anio_modelo": "2022",
      "nombre_propietario": "Juan Pérez",
      "fechaRegistro": "2025-06-08T18:42:11.123Z"
    }
  ]
}
```

### Campos de cada solicitud

| Campo | Tipo | Descripción |
|---|---|---|
| `id` | string | Identificador de la solicitud en VEC (`solicitud_id` para otras APIs) |
| `cliente_id` | string | Cliente enviado al crear la solicitud |
| `vin` | string | VIN |
| `fabricante` | string | Fabricante |
| `modelo` | string | Modelo |
| `pais` | string | País |
| `anio_modelo` | string | Año modelo |
| `nombre_propietario` | string | Nombre del propietario |
| `fechaRegistro` | string | Fecha de alta en ISO 8601 con zona horaria (UTC, sufijo `Z`) |

Los campos de vehículo y cliente coinciden con el body de [`prevalidadorSolicitudInspeccion`](./prevalidador-solicitud-inspeccion.md). Para actualizar una solicitud listada, use [`prevalidadorActualizaSolicitudInspeccion`](./prevalidador-actualiza-solicitud-inspeccion.md) con su `id` como `solicitud_id`.

### Lista vacía

```json
{
  "success": true,
  "total": 0,
  "prevalidador": { "id": "...", "nombre": "..." },
  "filtros": { "orden": "DESC", "fechaIni": "...", "fechaFin": "...", "vin": null },
  "solicitudes": []
}
```

No es error HTTP.

---

## Errores

| HTTP | `error` | Cuándo |
|---|---|---|
| 400 | `INVALID_ORDER` | `orden` distinto de `ASC` o `DESC` |
| 400 | `INVALID_DATE` | `fechaIni` / `fechaFin` con formato inválido |
| 400 | `INVALID_DATE_RANGE` | `fechaIni` posterior a `fechaFin` |
| 401 | `missing-token` / `invalid-token` | Token ausente o inválido |
| 403 | `not-prevalidador` / `prevalidador-inactivo` | Cuenta no habilitada |
| 405 | `METHOD_NOT_ALLOWED` | No es `GET` |
| 500 | `INTERNAL_ERROR` | Fallo interno en VEC |
