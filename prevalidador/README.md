# Sincronizacion de documentacion (prevalidador)

Este folder incluye el script `sync-docs.ps1` para copiar la documentacion de APIs de `prevalidador` al repo publico de documentacion.

## Estructura esperada

Para que funcione sin parametros, ambos repos deben estar como carpetas hermanas:

- `...\vec-v2`
- `...\documentacion-vec`

Ejemplo:

- `C:\dev\VEC-V2-Integrado\vec-v2`
- `C:\dev\VEC-V2-Integrado\documentacion-vec`

## Que hace el script

- Toma todos los archivos `*.md` de `functions/docs/prevalidador`.
- Los copia a `documentacion-vec/prevalidador`.
- Si la carpeta destino no existe, la crea.
- Sobrescribe archivos existentes para mantenerlos sincronizados.

## Como ejecutarlo

Desde la raiz de `vec-v2`, correr:

```powershell
powershell -ExecutionPolicy Bypass -File ".\functions\docs\prevalidador\sync-docs.ps1"
```

## Validacion rapida

Al terminar, deberias ver mensajes como:

- `Sincronizado: <archivo>.md`
- `Listo. Documentacion sincronizada en: <ruta-destino>`

## Uso opcional con ruta manual

Si alguien no tiene la estructura de carpetas esperada, puede indicar la ruta destino explicitamente:

```powershell
powershell -ExecutionPolicy Bypass -File ".\functions\docs\prevalidador\sync-docs.ps1" -DestinationRepo "D:\repos\documentacion-vec"
```
