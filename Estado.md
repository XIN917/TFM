# Estado del proyecto — TFM Automatización DEHú (MGS)

Resumen vivo. Pendientes y preguntas: `TODO.md`. Índice de documentos: `README.md`.

---

## 1. Contexto

Automatizar la recepción y tramitación de comunicaciones de la administración pública (DEHú/LEMA) en MGS Seguros: clasificar el departamento con los metadatos de `localiza()` antes de abrir el documento, y derivar al Ticketing interno.

Hoy el acceso es **manual** a **DEHú** (habitualmente por el enlace del correo de aviso). Mi Carpeta Ciudadana es otro portal, también puede abrir el mismo buzón. El proyecto sustituye esa consulta en pantalla por **LEMA** (Grandes Destinatarios): mismo buzón, servicios web. Seguridad Informática confirma: certificado de producción ya existe; el de pruebas lo genera Sistemas; contratación/custodia/renovación es de Seguridad.

| Rol | Función |
|---|---|
| Tutora de empresa | Revisa diseño (casos de uso, requisitos, UML) |
| Ponente | Supervisión académica |
| Manager de empresa | Directrices de negocio |
| Responsable de Ticketing | Sistema interno de tickets |

**Nombre:** `HVOrganismosPublicos`. `HV` es el área de aplicativos de Consulta. `OrganismosPublicos` es más amplio que DEHú a propósito (otras fuentes en el futuro), pero no amplía el MVP, que sigue siendo DEHú/LEMA. Justificarlo en la memoria.

**Infraestructura** (Soporte Datacenter / Altia, 10/09/2026): GIT `HV/HVOrganismosPublicos` (R/W Desarrollo); BBDD `HV_OrganismosPublicos` (DESA y PROD, SQLPortal, tamaño extra por documentos). Pendiente de grupos de acceso.

**Confidencialidad:** `Estado_Tecnico_Ticketing.md` y `Analisis_Frontend_AYTicketing.md` son internos de MGS. Fuera del repo; no republicar fragmentos de código real de Ticketing.

## 2. Dónde está el trabajo

Análisis y diseño del MVP cerrados en lo esencial (ER aprobado por la tutora de empresa). La clasificación por departamento es previa a la apertura: organismo emisor y concepto son la señal base. La comparecencia de las notificaciones es el lote de madrugada: si la clasificación superó el umbral, se abre, se archiva y se deriva enseguida. Siguen abiertos el catálogo de Siniestros y Sucursal Central, la validación de RRHH y la comprobación en LEMA de Distribución. El resumen de las notificaciones no entra todavía.

| Artefacto | Dónde |
|---|---|
| Requisitos RF-01–RF-12 | `Especificacion_Requisitos.md` |
| PRD para implementación (autocontenido) | `PRD.md` |
| Casos de uso | `Diagramas/Casos_de_Uso.md` |
| Flujos | `Diagramas/Diagramas_de_flujo.md` |
| ER | `Diagramas/ER_Explicacion.md` |
| Clases (RF-08, 09, 10, 11.4, 11.5) y patrones | `Diagramas/Diagrama_Clases.md` |
| Componentes | `Diagramas/Diagrama_Componentes.md` |
| Gantt | `Diagramas/Gantt.md` |
| Estructura de la memoria | `Estructura_Memoria.md` |

RF-11.5 (cambio de departamento) está diagramada como propuesta de alto nivel, no como diseño técnico cerrado: depende de la API «audiencia back» de Ticketing. El diagrama de secuencia de RF-10 se descartó: el flujo ya está en `Diagramas_de_flujo.md` y la invocación MCP en `Diagrama_Clases.md`.

## 3. Decisiones cerradas

- Actores de caso de uso: solo entidades externas (DEHú/LEMA, Operador, Departamento). El certificado es arquitectura, no caso de uso.
- Canal obligatorio del MVP: ticket. Email (RF-08.2) deseable, no bloqueante (director de empresa). Buzón interno (RF-08.3) fuera del alcance activo; se deja el hueco de numeración.
- Ticketing no transfiere de cola → reclasificar es finalizar + crear. Confirmado en OpenAPI (`PATCH /tickets/{id}` no toca `cola`). Enlace con `ticketRelacionado`.
- RF-11: edición in-place si no cambia el departamento; finalizar + crear si cambia. Es editable si ya hay `DERIVACION`, no según `tipoEnvio`.
- Reclasificación restringida a administrador. No se renombra «Aceptar clasificación propuesta». «Asignar a departamento» no replica la separación ticket/email de «Modificar»: es automático, no una decisión del actor.
- RF-10 cerrado a nivel de diseño: cancelación → Agente IA (umbral) → autocorrección MCP o escalado humano. MCP acotado a esa acción. Ticketing no expone webhook: el cambio de estado sale por outbox interno. Falta quién lo consume (ver `TODO.md`).
- `tipoEnvio`: `1` comunicación, `2` notificación. `vinculo`: `1` titular, `2` destinatario (plan de pruebas GD v2.0, §2.1.2–2.1.3).
- Descarga según `tipoEnvio` (FAQ DEHú). Comunicación: acceder no tiene efectos jurídicos ni genera acuse; se descarga en el mismo ciclo en que queda clasificada. Notificación: `peticionAcceso()` es la comparecencia y puede abrir el plazo de respuesta, que fija el organismo emisor. Sin comparecer, rechazo tácito a los 10 días naturales (Ley 39/2015, art. 43.2).
- Clasificación sin leer el documento (RF-05 y PRD): organismo emisor y concepto de `localiza()` son la señal base, antes de abrir. Otro metadato de `localiza()` solo si una muestra real demuestra que discrimina. Sin confianza suficiente, revisión y no se abre. El anexo no entra. El resumen de este alcance es el de la comunicación (`tipoEnvio` `1`).
- Flujos (05/10/2026): sondeo, autocorrección, lote de madrugada, operador, consulta local, reconciliación. Comunicación clasificada: `peticionAcceso()`, anexos si hay `anexosReferencia`, sin acuse, resumen del principal y ticket. Notificación clasificada: espera al lote de madrugada.
- Comparecencia: lote de madrugada. Si la clasificación superó el umbral, se abre el documento, se guardan anexos y acuse cuando existan, y se deriva enseguida. No hay confirmación manual. Por debajo del umbral no se abre. El ticket de la notificación sale sin resumen.
- Fuente y reparto (05/10/2026; detalle en `_local/reuniones/`, interlocutores en `_local/interlocutores_reparto.md`). La fuente es DEHú, no los correos de aviso. El prompt solo lleva reglas confirmadas y realizables con `localiza()`; los DIR3 observados no son la clave. Semilla en `PRD.md`. Distribución → Red de Mediación está confirmada en negocio y fuera del prompt hasta ver el dato en LEMA. TGSS salvo embargo → RRHH está fuera del prompt hasta validarlo con RRHH. Siniestros y Sucursal Central siguen sin regla activa.
- Atributos de LEMA y realizadas (05/10/2026). Se guardan como columnas los atributos que devuelve LEMA, incluidos los blobs sin documentar (`COMUNICACION.metadatosPublicos`, `DOCUMENTO.metadatos`); no hay columna con la respuesta en bruto. `REALIZADAS` guarda los envíos realizados anteriores al arranque para probar la clasificación, sin relación con el flujo ni acciones (RF-12.4–12.6, opcional). Un identificador vive en una sola tabla; un envío posterior al arranque que falte en `COMUNICACION` es anomalía de reconciliación.
- `PRD.md` autocontenido (05/10/2026): incluye lo necesario para implementar (diccionario de datos, correspondencia con LEMA, endpoints, criterios de aceptación) sin citar otros documentos del repo ni `_local`. Pensado para copiarlo a `HVOrganismosPublicos`.
- Acceso LEMA: sin token ni usuario; cada llamada va firmada con X.509 (WS-Security). Pruebas `se-gd-dehuws.redsara.es` (certificado autofirmado, alta propia; instrucciones en `docs/Instrucciones certificado LEMA pruebas.docx`); producción `gd-dehuws.redsara.es`.
- `TEXTOEXTRAIDO` tabla 1:1 opcional de `INTERPRETACION` (sugerencia de la tutora de empresa). Retención por parámetro global; el valor se fija tras contrastarlo con las áreas. Sin retención permanente para entrenamiento en el MVP.
- ER: tablas renombradas; `USUARIO` con clave compartida a `PERSONA`; sin tabla `EVENTO` (trazabilidad e idempotencia con `CLASIFICACION` + `DERIVACION`).
- `CLASIFICACION` (22/09/2026): una fila por clasificación. `origen` `ia` | `operador`; `modelo` si hubo LLM; `nIntentos` solo en filas `ia` (el tope de reclasificaciones cuenta `nIntentos > 1`); `usuario_id` solo en filas `operador`. `departamentoAsignado` obligatorio.
- Reparto (28/09/2026): el modelo clasifica solo por departamento; no hay catálogo de tipos. `tipoAsignado` y `tipoDetectado` eliminados del modelo. Qué organismos y materias van a cada departamento se guarda en `DEPARTAMENTO.criteriosReparto` y alimenta el prompt; el administrador reclasifica eligiendo departamento.
- Aprendizaje continuo del Agente IA y diagnóstico sistemático de errores: evolución futura, no diseño cerrado.

## 4. Arquitectura (resumen)

Detalle en `Diagrama_Clases.md` y `Diagrama_Componentes.md`.

- **Módulos Eclipse:** ver `PRD.md` §4 (familia Ticketing; sin DAO/EJB). Incluye `Test` (JUnit para el TFM) y `FT` (Playwright).
- **Stack alineado con Ticketing (cerrado; tutora de empresa, sept. 2026):** Java 8, **Java EE 7** (`javax.*`), **WAS 9.0**, JDBC propio (sin JPA/Spring), CDI, JAX-RS. Jakarta EE descartado para este aplicativo hasta que Sistemas mueva de WAS 9. Módulos y CDI por constructor: `PRD.md`. Misma convención de estereotipos (`@Repositorio`, `@Servicio`, `@Endpoint`, `@Transaccional`), sin `AggregateRoot` ni eventos de dominio CDI (los cubre RF-07 a nivel corporativo).
- **Estilos:** orientada a eventos entre componentes (bus corporativo desacopla pipeline / ejecución / reclasificación); hexagonal por dentro de cada componente. Justificarlo explícitamente en la memoria, no dejarlo como efecto secundario de SOLID.
- **Frontal:** subconjunto CRUD + listado + detalle + un flujo de estado, sobre librería de terceros. No replicar el acabado visual de Ticketing (vive en librerías internas, fuera de plazo).
- **Componentes:** línea discontinua solo para lo no confirmado — modificar ticket (audiencia back) y evento de cancelación (proceso que lee el outbox).

## 5. Planificación

Ver `Gantt.md`. Cinco fases: Análisis y diseño → Desarrollo → Testing → Memoria → Revisión final.

Deadlines de negocio: backend/frontend antes del 25 dic; desarrollo y testing cierran el 31 dic; revisión final 4–15 ene 2027. Festivos pendientes de recalcular (ver `TODO.md`).

## 6. Memoria

Estructura: `Estructura_Memoria.md`. Introducción cerrada (revisada el 23/09; `memoria/01_Introduccion.md` sincronizado con Overleaf). Gestión del proyecto: iniciada el 01/09 (Gantt en `Diagramas/Gantt.md`).