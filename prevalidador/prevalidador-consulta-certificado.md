# API `prevalidadorConsultaCertificado`

Devuelve el **certificado de inspección** en JSON, con las mismas secciones que el reporte público [`/reporte/:inspeccionId`](../../src/app/public/reporte/).

La inspección se localiza por **`solicitud_id`**: documento en `inspecciones` donde `datosAsignacion.solicitudInspeccionId` coincide (relación 1:1 con la solicitud).

**Requisitos previos:** [`prevalidadorLogin`](./prevalidador-auth.md) y una solicitud creada por el prevalidador (p. ej. [`prevalidadorSolicitudInspeccion`](./prevalidador-solicitud-inspeccion.md)) que ya tenga inspección asignada.

---

## Endpoint

| | |
|---|---|
| **Métodos** | `GET`, `POST` |
| **URL (prod)** | `https://us-central1-vec-v2.cloudfunctions.net/prevalidadorConsultaCertificado` |
| **Auth** | `Authorization: Bearer <idToken>` |

---

## Parámetros

| Parámetro | Ubicación | Requerido | Descripción |
|---|---|---|---|
| `solicitud_id` | query (GET) o body JSON (POST) | Sí | ID del documento en `solicitudesInspeccion` |
| `timezone` | query o body JSON | No | Zona horaria **IANA** del cliente (p. ej. `America/Tijuana`). Por defecto: `America/Mexico_City`. La fecha del encabezado usa el mismo formato que el reporte web (`AAAA-MM-DD HH:mm (UTC±HH)`) en esa zona. |

Alias aceptados: `solicitudId`.

### Ejemplo GET

```bash
curl -s -G \
  "https://us-central1-vec-v2.cloudfunctions.net/prevalidadorConsultaCertificado" \
  --data-urlencode "solicitud_id=SOLICITUD_ID" \
  --data-urlencode "timezone=America/Mazatlan" \
  -H "Authorization: Bearer ${ID_TOKEN}"
```

### Ejemplo POST

```bash
curl -s -X POST \
  "https://us-central1-vec-v2.cloudfunctions.net/prevalidadorConsultaCertificado" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${ID_TOKEN}" \
  -d '{"solicitud_id":"SOLICITUD_ID","timezone":"America/Mazatlan"}'
```

---

## Respuesta exitosa (`200`)

```json
{
  "success": true,
  "solicitud_id": "...",
  "inspeccion_id": "...",
  "zona_horaria": "America/Mazatlan",
  "certificado": {
    "encabezado": { },
    "datosGeneralesYVehiculares": { },
    "resultadoVerificacion": null,
    "fotografias": { },
    "ubicacion": { },
    "leyenda": { }
  }
}
```

### Secciones (`certificado`)

| Sección | Contenido (equivalente al HTML) |
|---|---|
| `encabezado` | Folio, VEC LLC, patente, aduana, fecha, equipo BlueDriver, MAC, URLs y texto QR de datos |
| `datosGeneralesYVehiculares` | Propietario, país, marca, VIN, año, modelo, odómetro CarInfo |
| `resultadoVerificacion` | `null` si emisiones no están `finalizado`; si no, monitores en dos columnas + resultado final |
| `fotografias` | `fotos` y `fotosVin`: arrays de **URLs**; resumen IA, odómetro en tablero |
| `ubicacion` | Lat/long formateadas (4 decimales) y `mapaEstaticoUrl` (Google Static Maps, mismas dimensiones que el reporte) |
| `leyenda` | Textos SEMARNAT / EPA / sitio |

No se incluyen imágenes QR en base64 (solo `urlReportePublico` y `textoQrDatosInspeccion` para generarlas externamente).

La respuesta incluye `zona_horaria` con la zona IANA aplicada al campo `encabezado.fecha`.

---

## Errores

| HTTP | `error` | Cuándo |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Falta `solicitud_id` |
| 401 | `missing-token`, `invalid-token`, … | Token ausente o inválido |
| 403 | `forbidden` | La solicitud no es del prevalidador |
| 404 | `not-found`, `certificado-no-disponible` | Solicitud inexistente o sin inspección |
| 409 | `inspeccion-ambigua` | Más de una inspección para la misma solicitud |
| 405 | `METHOD_NOT_ALLOWED` | No es GET ni POST |
| 500 | `INTERNAL_ERROR` | Error no controlado |

---

## Índice Firestore

Consulta:

`inspecciones` WHERE `datosAsignacion.solicitudInspeccionId` == `solicitud_id`

Índice en `firestore.indexes.json` (campo único en colección `inspecciones`).

---

## Candado 1:1

Al crear asignación + inspección (`AsignacionesService.crearAsignacionEInspeccion`), se valida que no exista otra inspección con el mismo `solicitudInspeccionId`.
