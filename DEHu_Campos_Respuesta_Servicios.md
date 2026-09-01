# DEHú/LEMA — Campos de respuesta por servicio

*Extraído y verificado contra el Anexo I (ejemplos de petición/respuesta) de la Guía de Integración para Gran Destinatario. Solo se listan campos confirmados en los ejemplos reales del PDF — no se han inferido ni completado con supuestos.*

---

## Servicios LEMA (pendientes de acceso)

### 1. `localiza()` — listado de comunicaciones pendientes

Por cada `item` dentro de `envios`:

| Campo | Tipo / ejemplo | Nota |
|---|---|---|
| `identificador` | string | Identificador único DEHú del envío |
| `codigoOrigen` | string/int | |
| `concepto` | string | Asunto/título de la comunicación |
| `organismoEmisor.codigoOrganismo` | string | |
| `organismoEmisor.nombreOrganismo` | string | Organismo emisor directo |
| `organismoEmisorRaiz.codigoOrganismo` | string | |
| `organismoEmisorRaiz.nombreOrganismo` | string | Organismo "padre" (ej. Ministerio) |
| `fechaPuestaDisposicion` | datetime ISO | |
| `tipoEnvio` | int | |
| `vinculo` | int | Tipo de relación (representación/apoderamiento) |
| `titular.nombreTitular` | string | |
| `titular.nifTitular` | string | |
| `metadatosPublicos` | base64 (blob opaco) | Contenido no documentado en la guía |

Paginación: `hayMasResultados`, `opcionesRespuestaLocaliza` (`totalResultados`, `totalPag`, `paginaActual`).

**⚠️ Estos campos (`concepto`, `organismoEmisor*`, `vinculo`, `titular`) NO vuelven a aparecer en `peticionAcceso()`. Si no se capturan aquí, se pierden.**

---

### 2. `peticionAcceso()` — acceso al contenido de un envío

| Campo | Tipo / ejemplo | Nota |
|---|---|---|
| `identificador` | string | Mismo valor que en `localiza()` |
| `codigoOrigen` | string/int | Mismo valor que en `localiza()` |
| `fechaEvento` | datetime ISO | |
| `documento.nombre` | string | |
| `documento.contenido` | referencia MTOM (`href="cid:..."`) | Binario adjunto al SOAP |
| `documento.hashDocumento.hash` | base64 | |
| `documento.hashDocumento.algoritmoHash` | string | Siempre `sha256` en los ejemplos |
| `documento.mimeType` | string | |
| `documento.metadatos` | base64 (blob opaco) | |
| `documento.csvResguardo` | string | Código de justificante del documento principal |
| `anexos.anexosReferencia[].nombre` | string | Solo anexos tipo *referencia* (los de URL directa no pasan por aquí) |
| `anexos.anexosReferencia[].mimeType` | string | |
| `anexos.anexosReferencia[].referenciaDocumento` | base64 | Identificador para pasar a `consultaAnexos()` |

**No incluye**: `concepto`, `organismoEmisor`, `organismoEmisorRaiz`, `vinculo`, `titular`.

---

### 3. `consultaAnexos()` — descarga de un anexo tipo referencia

Se invoca una vez por cada `referenciaDocumento` obtenida en `peticionAcceso()`.

| Campo | Tipo / ejemplo | Nota |
|---|---|---|
| `documento.nombre` | string | |
| `documento.contenido` | referencia MTOM | Binario del anexo |
| `documento.mimeType` | string | |

Respuesta mínima: no trae hash ni csvResguardo (a diferencia del documento principal).

---

### 4. `consultaAcusePdf()` — descarga del acuse/resguardo

Dos variantes de petición (`csvResguardo` o `referencia`), misma forma de respuesta:

| Campo | Tipo / ejemplo | Nota |
|---|---|---|
| `acusePdf.nombreAcuse` | string | |
| `acusePdf.contenido` | referencia MTOM | Binario del PDF del acuse |
| `acusePdf.mimeType` | string | |
| `acusePdf.metadatos` | string | En los ejemplos contiene el propio `csvResguardo`/código DEHU — confirmar formato exacto en pruebas, la guía no lo especifica más allá del ejemplo |

---

## Servicios ConsultaRealizadas (ya comparecidas/leídas)

### 5. `localizaRealizadas()` — listado de envíos ya realizados

Mismos campos que `localiza()` (`identificador`, `codigoOrigen`, `concepto`, `organismoEmisor`, `organismoEmisorRaiz`, `fechaPuestaDisposicion`, `tipoEnvio`, `vinculo`, `titular`), **más**:

| Campo | Tipo / ejemplo | Nota |
|---|---|---|
| `estado` | string, ej. `EXPIRADA` | Estado del envío en DEHú — no confundir con `COMUNICACION.estado` interno del sistema propio |
| `referenciaPdfAcuse` | base64 | Referencia directa al PDF del acuse |
| `csvResguardo` | string | |

Paginación: `totalPaginas`, `paginaActual` (sin `hayMasResultados` en el ejemplo).

**Sin filtro por fecha en la petición** (ya documentado en la especificación de requisitos).

---

### 6. `consultaRealizadas()` — contenido de un envío ya realizado

| Campo | Tipo / ejemplo | Nota |
|---|---|---|
| `identificador` | string | |
| `codigoOrigen` | string/int | |
| `fechaEvento` | datetime ISO | |
| `documento.nombre` | string | |
| `documento.contenido.tipoMIME` | string | Estructura ligeramente distinta a `peticionAcceso()` (anidada bajo `contenido`) |
| `documento.hashDocumento.hash` | base64 | |
| `documento.hashDocumento.algoritmoHash` | string | `sha256` |
| `anexos.anexosReferencia[].nombre` | string | |
| `anexos.anexosReferencia[].mimeType` | string | |
| `anexos.anexosReferencia[].referenciaDocumento` | base64 | |

**No incluye** `csvResguardo` a nivel de documento (a diferencia de `peticionAcceso()`).

---

## Resumen — qué campo viene de dónde

| Campo | localiza() | peticionAcceso() | localizaRealizadas() | consultaRealizadas() |
|---|:---:|:---:|:---:|:---:|
| identificador / codigoOrigen | ✓ | ✓ | ✓ | ✓ |
| concepto | ✓ | | ✓ | |
| organismoEmisor / organismoEmisorRaiz | ✓ | | ✓ | |
| vinculo / titular | ✓ | | ✓ | |
| fechaPuestaDisposicion / fechaEvento | ✓ | ✓ | ✓ | ✓ |
| estado (DEHú) | | | ✓ | |
| documento (nombre, mimeType, hash) | | ✓ | | ✓ |
| csvResguardo (documento) | | ✓ | | |
| anexos (referencia) | | ✓ | | ✓ |
| referenciaPdfAcuse / csvResguardo (envío) | | | ✓ | |

---

## Datos de petición por servicio

*Lo que hay que enviar en cada llamada, y de dónde sale cada valor.*

| Servicio | Campos de la petición | Origen del valor |
|---|---|---|
| `localiza()` | `nifTitular`; opcional `opcionesLocaliza.pagina` | NIF propio (certificado); paginación gestionada por el sistema |
| `peticionAcceso()` | `identificador`, `codigoOrigen` | Del `item` devuelto por `localiza()` |
| `consultaAnexos()` | `nifReceptor`, `identificador`, `codigoOrigen`, `referencia` | `identificador`/`codigoOrigen` de `localiza()`; `referencia` = `referenciaDocumento` de cada anexo devuelto por `peticionAcceso()` |
| `consultaAcusePdf()` | `nifReceptor`, `identificador`, `codigoOrigen`, y **una de dos**: `identificadorAcusePdf.csvResguardo` o `identificadorAcusePdf.referencia` | `csvResguardo` = el que devuelve `peticionAcceso()` en `documento.csvResguardo` |
| `localizaRealizadas()` | `nifTitular`, `tipoEnvio` | NIF propio; **sin filtro por fecha** (ya documentado en la especificación) |
| `consultaRealizadas()` | `identificador`, `codigoOrigen`, `nifPeticion`, `nombrePeticion`, `concepto` | Del `item` devuelto por `localizaRealizadas()` |

### Dependencia estricta entre llamadas

```
localiza()
  └─ identificador, codigoOrigen ──▶ peticionAcceso()
                                        ├─ documento.csvResguardo ──▶ consultaAcusePdf() (vía csvResguardo)
                                        └─ anexos[].referenciaDocumento ──▶ consultaAnexos() (vía referencia)

localizaRealizadas()
  └─ identificador, codigoOrigen, concepto ──▶ consultaRealizadas()
```

No son llamadas independientes: cada servicio de "detalle" depende de un identificador o referencia obtenido en el servicio de "listado" que lo precede. Esto condiciona el orden de los pasos en el diagrama de arquitectura de componentes.

---

## Nota sobre la extracción del PDF

Este documento se ha construido revisando directamente el texto extraído del PDF original (`DEHuGuia_Integracion_para_Gran_Destinatario.pdf`), páginas 22-49 (Anexo I). Se han evitado los tramos con artefactos de extracción (saltos de línea con guion mal codificados, presentes sobre todo en URLs de namespaces XML y en la sección de endpoints WSDL) — ninguno de los campos aquí listados proviene de esas zonas afectadas.
