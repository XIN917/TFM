# Explicación del ER — Automatización DEHú (MGS)

*Este documento explica qué representa cada tabla y cómo se relacionan entre sí.*

---

## Diagrama completo

```mermaid
erDiagram
  COMUNICACION ||--o{ DOCUMENTO : contiene
  COMUNICACION ||--o| INTERPRETACION : produce
  INTERPRETACION ||--o| TEXTOEXTRAIDO : almacena_texto_en
  COMUNICACION ||--o{ CLASIFICACION : acumula
  COMUNICACION ||--o{ DERIVACION : dispara
  COMUNICACION ||--o{ REVISION : puede_escalar_a
  USUARIO |o--o{ REVISION : resuelve
  USUARIO |o--o{ CLASIFICACION : clasifica

  COMUNICACION {
    uuid id PK
    string identificador
    string codigoOrigen
    string concepto
    string organismoEmisorCodigo
    string organismoEmisorNombre
    string organismoEmisorRaizCodigo
    string organismoEmisorRaizNombre
    datetime fechaPuestaDisposicion
    int tipoEnvio
    int vinculo
    string titularNombre
    string titularNif
    datetime fechaEvento
    string estado
    text metadatosPublicos
    datetime fechaIngesta
  }
  DOCUMENTO {
    uuid id PK
    uuid comunicacion_id FK
    string tipo
    string nombre
    string mimeType
    string hashSha256
    string algoritmoHash
    string csvResguardo
    string referenciaDocumento
    text metadatos
    string rutaAlmacenamiento
    datetime fechaDescarga
  }
  INTERPRETACION {
    uuid id PK
    uuid comunicacion_id FK
    json entidadesExtraidas
    float scoreConfianza
    datetime fechaProcesado
  }
  TEXTOEXTRAIDO {
    uuid id PK
    uuid interpretacion_id FK
    text textoExtraido
    datetime fechaCreacion
  }
  CLASIFICACION {
    uuid id PK
    uuid comunicacion_id FK
    string origen "ia | operador"
    string modelo "nullable"
    int nIntentos "nullable; 1, 2, 3… si origen = ia"
    string usuario_id FK "nullable; USUARIO.id si origen = operador"
    string departamentoAsignado FK
    string canalAsignado
    float scoreConfianza
    string resultado
    datetime fecha
  }
  DERIVACION {
    uuid id PK
    uuid comunicacion_id FK
    string canal
    string identificadorExterno
    string titulo
    string resumen
    string departamento FK
    string estado
    boolean esReclasificacion
    string ticketRelacionadoId
    datetime fechaEjecucion
  }
  REVISION {
    uuid id PK
    uuid comunicacion_id FK
    string usuario_id FK "nullable, = USUARIO.id cuando resuelto"
    string motivo
    boolean resuelto
    datetime fechaEntrada
    datetime fechaResolucion
  }
  DEPARTAMENTO {
    string id PK
    string nombre
    string colaDestino
    boolean permiteEmail
    text criteriosReparto
    boolean activo
  }
  USUARIO {
    string id PK, FK "= PERSONA.id (tabla externa, clave compartida)"
    string rol
    boolean activo
  }

  REALIZADAS {
    uuid id PK
    string identificador
    string codigoOrigen
    string concepto
    string organismoEmisorCodigo
    string organismoEmisorNombre
    string organismoEmisorRaizCodigo
    string organismoEmisorRaizNombre
    datetime fechaPuestaDisposicion
    int tipoEnvio
    int vinculo
    string titularNombre
    string titularNif
    string estadoDehu
    text referenciaPdfAcuse
    string csvResguardo
    string rutaDocumento "nullable"
    string hashSha256 "nullable"
    string algoritmoHash "nullable"
    string departamentoEsperado FK "nullable"
    string departamentoPropuesto FK "nullable"
    float scoreConfianza "nullable"
    string modelo "nullable"
    datetime fechaClasificacion "nullable"
    datetime fechaCarga
  }

  CLASIFICACION }o--|| DEPARTAMENTO : referencia
  DERIVACION }o--|| DEPARTAMENTO : referencia
  REALIZADAS }o--o| DEPARTAMENTO : referencia
```

---

## Entidad central: `COMUNICACION`

Cada fila representa **un envío recibido de DEHú** (notificación o comunicación, según el sentido oficial de DEHú, distinguido por `tipoEnvio`). Todo lo demás del modelo cuelga de aquí.

| Campo | Origen | Nota |
|---|---|---|
| `identificador`, `codigoOrigen` | `localiza()` | Identificadores propios de DEHú; se usan para deduplicar (RF-01.3) y para encadenar las siguientes llamadas a LEMA |
| `concepto`, `organismoEmisorCodigo`, `organismoEmisorNombre`, `organismoEmisorRaizCodigo`, `organismoEmisorRaizNombre`, `fechaPuestaDisposicion`, `vinculo`, `titularNombre`, `titularNif` | `localiza()` | No vuelven en `peticionAcceso()`. Se capturan al detectar. `fechaPuestaDisposicion` sirve para el plazo de comparecencia |
| `tipoEnvio` | `localiza()` | `1` comunicación, `2` notificación (plan de pruebas GD v2.0, §2.1.2–2.1.3). La comunicación se descarga en el ciclo en que queda clasificada y no tiene acuse. La notificación se comparece en el lote de madrugada, solo si la clasificación superó el umbral, y comparecerla puede arrancar plazo (FAQ DEHú) |
| `estado` | Interno | Ciclo de vida propio del sistema: `pendiente → en_proceso → en_revision / procesada` — no es un estado de DEHú |
| `fechaEvento`, `fechaIngesta` | Mixto | `fechaEvento` viene de `peticionAcceso()` o `consultaRealizadas()`; `fechaIngesta` es el timestamp interno de cuándo se procesó |
| `metadatosPublicos` | `localiza()` | Base64 tal como llega (`nvarchar(max)`); la guía no documenta su contenido. No vuelve en `peticionAcceso()`. No se interpreta ni entra en la clasificación |

---

## Resumen completo de relaciones

```
COMUNICACION 1───N DOCUMENTO
COMUNICACION 1───1 INTERPRETACION    (opcional)
INTERPRETACION 1───1 TEXTOEXTRAIDO   (opcional)
COMUNICACION 1───N CLASIFICACION
COMUNICACION 1───N DERIVACION
COMUNICACION 1───N REVISION          (opcional)
USUARIO      0..1───N REVISION
USUARIO      0..1───N CLASIFICACION
DEPARTAMENTO 1───N CLASIFICACION
DEPARTAMENTO 1───N DERIVACION
DEPARTAMENTO 1───N REALIZADAS        (opcional; sin relación con COMUNICACION)
```

`(opcional)` marca las relaciones donde no toda `COMUNICACION` tiene necesariamente una fila asociada — `INTERPRETACION` solo existe tras el procesamiento IA; `TEXTOEXTRAIDO` solo existe mientras no haya expirado su periodo de retención (ver más abajo); `REVISION` solo existe si la comunicación escaló a revisión humana, y puede tener más de una fila si la comunicación escala a revisión en más de una ocasión (p. ej. una reclasificación posterior vuelve a caer por debajo del umbral).

`0..1` en `USUARIO` significa que la fila puede no tener usuario. En `CLASIFICACION`, `usuario_id` va vacío si `origen = ia` y es obligatorio si `origen = operador`. En `REVISION`, va vacío mientras `resuelto = false` y se rellena al resolverla.

## Relaciones desde `COMUNICACION`

### `DOCUMENTO` (1:N)

Una comunicación puede tener varios documentos: el principal, cada anexo y, si es notificación (`tipoEnvio` `2`), el acuse — todos en la misma tabla, distinguidos por `tipo`. La comunicación (`1`) no genera acuse (FAQ DEHú). Solo el documento principal pasa por el pipeline de IA (RF-03); anexos y acuse se archivan pero no se procesan (ver `INTERPRETACION`).

- `hashSha256` y `algoritmoHash` — el hash y el algoritmo que devuelve DEHú (`hashDocumento`). La integridad (RF-02.4) compara el hash
- `csvResguardo` — código de justificante del documento principal en `peticionAcceso()`. Sirve para volver a pedir el acuse
- `referenciaDocumento` — referencia del anexo en `peticionAcceso()` o `consultaRealizadas()`, la que se pasa a `consultaAnexos()`. Vacía en principal y acuse
- `metadatos` — `documento.metadatos` del principal (`peticionAcceso()`) o `acusePdf.metadatos` del acuse (`consultaAcusePdf()`), tal como llegan (`nvarchar(max)`). Vacío en anexos: `consultaAnexos()` no lo devuelve. No se interpreta
- `rutaAlmacenamiento` — el binario (`documento.contenido` o `acusePdf.contenido`). No se guarda el `href` MTOM

Las columnas son los atributos que devuelve LEMA según la guía, sin el binario. Los blobs `metadatosPublicos` y `metadatos` se guardan aunque no se interpreten, porque después no se pueden recuperar: `consultaRealizadas()` no los trae y el acuse solo se puede volver a pedir durante 24 h. La paginación de la llamada no se guarda: no es un atributo del envío. La URL de un anexo directo no tiene columna hasta que las pruebas confirmen el nombre de su elemento.

### `INTERPRETACION` (1:1 opcional)

Resultado de OCR + LLM sobre el **documento principal** (RF-03/RF-04): entidades y score de confianza. No decide el departamento: eso se hace al clasificar, antes de abrir el documento. Es opcional porque hasta que no se procesa, la comunicación no tiene interpretación todavía. El texto extraído en sí no vive aquí — ver `TEXTOEXTRAIDO`.

### `TEXTOEXTRAIDO` (1:1 opcional, desde `INTERPRETACION`)

Texto extraído del documento principal, separado de `INTERPRETACION` a propósito: es el campo más pesado y el menos consultado en el día a día (listados, dashboard), así que aislarlo evita penalizar esas consultas frecuentes. Se enlaza por `interpretacion_id` (no por `comunicacion_id` directamente) para mantener la jerarquía `COMUNICACION → INTERPRETACION → TEXTOEXTRAIDO` y no duplicar la relación con `COMUNICACION` en dos sitios.

Sujeto a una política de retención: se purga pasado un periodo definido por **parámetro global configurable** (no hardcodeado). El valor concreto se fijará tras contrastarlo con los compañeros.

### `CLASIFICACION` (1:N)

Historial completo de decisiones de clasificación — no se sobrescribe, se acumula. Cada fila es un intento.

- `origen` — `ia` u `operador`
- `modelo` — nombre del modelo si hubo LLM; vacío si la clasificación salió solo de metadatos o si `origen = operador`
- `nIntentos` — `1`, `2`, `3`… solo en filas `ia`. Vacío si `origen = operador`. La inicial es `1`; el tope de reclasificaciones (RF-10.2) cuenta filas `origen = ia` con `nIntentos > 1`, sin campo contador en `COMUNICACION`
- `usuario_id` (FK, nullable) — el operador de esa fila. Obligatorio si `origen = operador`; vacío si `origen = ia`. Varias clasificaciones de operadores distintos quedan en filas distintas, cada una con su id. No sustituye a `REVISION.usuario_id`, que es quién resolvió esa escalada
- `departamentoAsignado` (FK, obligatorio) — a qué departamento va la comunicación. Es la única categorización: el modelo clasifica por departamento, con `DEPARTAMENTO.criteriosReparto`. RF-09.6 corrige el departamento. Toda fila tiene departamento.
- Esta tabla es el historial de auditoría: no se sobrescribe, se acumula

### `DERIVACION` (1:N)

El resultado de RF-08: cada vez que se ejecuta una acción real hacia un departamento (crear ticket o, si se implementa RF-08.2, enviar email), queda una fila aquí.

- `departamento` (FK) — a qué departamento se derivó
- `esReclasificacion` + `ticketRelacionadoId` — cubren el caso confirmado con Dani: Ticketing no permite cambiar de cola, así que una reclasificación finaliza el ticket original y crea uno nuevo, enlazados por este campo nativo de la API

### `REVISION` (1:N opcional)

Solo existe si la comunicación quedó por debajo del umbral de confianza y escaló a cola humana (RF-09.1). Es 1:N y no 1:1 porque una misma comunicación puede escalar a revisión más de una vez a lo largo de su ciclo de vida — por ejemplo, si tras una reclasificación (RF-10) la nueva propuesta vuelve a caer por debajo del umbral. Cada escalada genera una fila nueva; las anteriores quedan cerradas (`resuelto = true`) como historial.

- `usuario_id` (FK, nullable) — quién resolvió esa revisión concreta; `null` mientras `resuelto = false`, se rellena junto con `fechaResolucion` cuando se resuelve
- Cubre tanto el caso de aceptar la propuesta de la IA (RF-09.5) como el de reclasificar manualmente (RF-09.6) — en ambos casos, quien tocó esta `REVISION` queda registrado aquí
- Para saber si una comunicación tiene una revisión pendiente *activa* en un momento dado, la consulta debe filtrar por `resuelto = false` en vez de asumir una única fila por comunicación

---

## Entidades de apoyo

### `DEPARTAMENTO`

Catálogo/diccionario, no cuelga directamente de `COMUNICACION` — se referencia desde `CLASIFICACION`, `DERIVACION` y `REALIZADAS` (ver resumen de relaciones al principio del documento).

| Campo | Para qué |
|---|---|
| `id`, `nombre` | Identidad del departamento |
| `colaDestino` | Cola de Ticketing asociada, cuando el canal es ticket |
| `permiteEmail` | Si el email está habilitado como canal (normal o de contingencia) para este departamento — RF-08.2 |
| `criteriosReparto` | Organismos y materias confirmados cuyo dato ya está en `localiza()` (p. ej. «DGSFP: Modelo 2B, Reclamaciones»). Con este texto se construye el prompt. Una regla confirmada en negocio pero aún no localizable en LEMA no se escribe aquí. Cambiar el reparto es cambiar datos, no código. El concepto no se compara por igualdad exacta |
| `activo` | Si el departamento sigue operativo |

### `REALIZADAS`

Envíos que DEHú ya tiene como realizados (aceptados, rechazados o expirados), cargados con `localizaRealizadas()` para probar la clasificación (RF-12.4–12.6). No cuelga de `COMUNICACION` ni tiene documentos, clasificaciones, revisiones ni derivaciones: no entra en el flujo y no genera ninguna acción.

- Un identificador vive en un solo sitio. Si ya está en `COMUNICACION`, no se carga aquí
- `estadoDehu` — el estado DEHú del envío. No es el `estado` interno de `COMUNICACION`
- `referenciaPdfAcuse` (referencia al PDF del acuse, base64) y `csvResguardo` del envío — de `localizaRealizadas()`, tal como llegan
- `rutaDocumento`, `hashSha256`, `algoritmoHash` — solo si se descargó el principal con `consultaRealizadas()` para analizarlo o etiquetarlo (RF-12.6). Anexos y acuse no se piden
- `departamentoEsperado` — el departamento correcto, etiquetado a mano o con las áreas
- `departamentoPropuesto`, `scoreConfianza`, `modelo`, `fechaClasificacion` — la última clasificación con el mismo prompt y umbral que el flujo normal. Para comparar varias versiones de `criteriosReparto` haría falta una tabla hija

### `USUARIO`

Gestiona el control de acceso al frontal propio (operador/administrador — RF-09, restricción de reclasificación a admin).

- `id` — clave compartida con `PERSONA` (tabla externa de la empresa, con todos los empleados): `USUARIO.id` es directamente el `id` de `PERSONA`, no un UUID propio. Así se evita duplicar nombre/email, que ya viven en `PERSONA`.
- `PERSONA` no se dibuja en el ER porque es una tabla externa, gestionada por otro sistema — solo se anota la referencia.