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

**Convenciones del repo:** (1) Sin nombres de personas en los `.md` del TFM; reparto solo por departamento en `Criterios_Clasificacion.md` y `PRD.md` §6. (2) Notas crudas de interlocución, experimentación IA y código interno de Ticketing **fuera** de este repo. (3) Las reglas de reparto solo se describen en `Criterios_Clasificacion.md` (fuente única, con IDs); el resto de documentos cita los IDs.

## 2. Dónde está el trabajo

Análisis y diseño del MVP cerrados en lo esencial (ER aprobado por la tutora de empresa). La clasificación por departamento es previa a la apertura: organismo emisor y concepto son la señal base. El diseño temporal vigente contempla un lote de madrugada para comparecer las notificaciones ya clasificadas, archivarlas y derivarlas; la política temporal definitiva sigue abierta. Siguen abiertas reglas de reparto (ver `Criterios_Clasificacion.md`). La obtención de texto y la interpretación de información relevante del documento principal se contemplan para ambos tipos de envío, condicionadas a la autorización corporativa del tratamiento mediante IA externa.

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
| Criterios de reparto (vivo) | `Criterios_Clasificacion.md` |
| Campos LEMA / guías | `DEHu_Campos_Respuesta_Servicios.md`, `docs/` |

RF-11.5 (cambio de departamento) está diagramada como propuesta de alto nivel, no como diseño técnico cerrado: depende de la API «audiencia back» de Ticketing. El diagrama de secuencia de RF-10 se descartó: el flujo ya está en `Diagramas_de_flujo.md` y la invocación MCP en `Diagrama_Clases.md`.

## 3. Decisiones cerradas

- Actores de caso de uso: solo entidades externas (DEHú/LEMA, Operador, Departamento). El certificado es arquitectura, no caso de uso.
- Canal obligatorio del MVP: ticket. Email (RF-08.2) deseable, no bloqueante (director de empresa). Buzón interno (RF-08.3) fuera del alcance activo; se deja el hueco de numeración.
- Ticketing no transfiere de cola → reclasificar es finalizar + crear. Confirmado en OpenAPI (`PATCH /tickets/{id}` no toca `cola`). Enlace con `ticketRelacionado`.
- RF-11: edición in-place si no cambia el departamento; finalizar + crear si cambia. Es editable si ya hay `DERIVACION`, no según `tipoEnvio`.
- Reclasificación restringida a administrador. No se renombra «Aceptar clasificación propuesta». «Asignar a departamento» no replica la separación ticket/email de «Modificar»: es automático, no una decisión del actor.
- RF-10 cerrado a nivel de diseño: cancelación → Agente IA (umbral) → autocorrección MCP o escalado humano. MCP acotado a esa acción. Ticketing no expone webhook: el cambio de estado sale por outbox interno. Falta quién lo consume (ver `TODO.md`).
- `tipoEnvio`: `1` comunicación, `2` notificación. `vinculo`: `1` titular, `2` destinatario (plan de pruebas GD v2.0, §2.1.2–2.1.3).
- Descarga según `tipoEnvio` (FAQ DEHú). Comunicación: acceder no tiene efectos jurídicos ni genera acuse; se descarga en el mismo ciclo en que queda clasificada. Notificación: `peticionAcceso()` es la comparecencia y puede abrir el plazo de respuesta, que fija el organismo emisor. La regla general de rechazo por falta de acceso se recoge en el artículo 43.2 de la Ley 39/2015; la fecha de vencimiento calculada por la aplicación es orientativa hasta contrastarla con vencimientos reales y validar las reglas jurídicas aplicables.
- Clasificación sin leer el documento (RF-02 y PRD): organismo emisor y concepto de `localiza()` son la señal base, antes de abrir. Otro metadato de `localiza()` solo si una muestra real demuestra que discrimina. Sin confianza suficiente, revisión y no se abre. El anexo no entra. La interpretación documental posterior se orienta a identificar asunto, requerimientos, actuaciones expresas, plazos documentados y otros datos útiles para la tramitación, tanto en comunicaciones como en notificaciones, cuando el tratamiento esté autorizado.
- Flujos (05/10/2026): sondeo, autocorrección, lote de madrugada, operador, consulta local, reconciliación. Comunicación clasificada: `peticionAcceso()`, anexos si hay `anexosReferencia`, sin acuse, interpretación del principal si está autorizada y ticket. Notificación clasificada: en el diseño temporal vigente espera al lote de madrugada, cuya continuidad como flujo separado está pendiente de decisión.
- Comparecencia: lote de madrugada. Si la clasificación superó el umbral, se abre el documento, se guardan anexos y acuse cuando existan, y se deriva enseguida. No hay confirmación manual. Por debajo del umbral no se abre. La interpretación del documento principal de las notificaciones está contemplada si está autorizada; sigue pendiente decidir cómo derivar cuando la interpretación no esté autorizada o falle.
- Fuente y reparto: DEHú/LEMA, no los correos de aviso. Reglas en `Criterios_Clasificacion.md`; copia para implementación en `PRD.md` §6. El prompt solo lleva reglas evaluables con metadatos de `localiza()`; los DIR3 observados no son la clave. Los departamentos de clasificación son de primer nivel: subáreas y unidades internas (p. ej. el Servicio Jurídico de RRHH) no lo son.
- Atributos de LEMA y realizadas (05/10/2026). Se guardan como columnas los atributos que devuelve LEMA, incluidos los blobs sin documentar (`COMUNICACION.metadatosPublicos`, `DOCUMENTO.metadatos`); no hay columna con la respuesta en bruto. `REALIZADAS` guarda los envíos realizados anteriores al arranque para probar la clasificación, sin relación con el flujo ni acciones (RF-12.4–12.6, opcional). Un identificador vive en una sola tabla; un envío posterior al arranque que falte en `COMUNICACION` es anomalía de reconciliación.
- `PRD.md` autocontenido para implementación (diccionario, LEMA, endpoints, criterios de aceptación, semilla §6). Copia de trabajo en `HVOrganismosPublicos`; al copiar, no arrastrar notas locales ni nombres de personas.
- Acceso LEMA: sin token ni usuario; cada llamada va firmada con X.509 (WS-Security). Pruebas `se-gd-dehuws.redsara.es` (certificado autofirmado, alta propia; instrucciones en `docs/Instrucciones certificado LEMA pruebas.docx`); producción `gd-dehuws.redsara.es`.
- `TEXTOEXTRAIDO` tabla 1:1 opcional de `INTERPRETACION` (sugerencia de la tutora de empresa). Retención por parámetro global; el valor se fija tras contrastarlo con las áreas. Sin retención permanente para entrenamiento en el MVP.
- ER: tablas renombradas; `USUARIO` con clave compartida a `PERSONA`; sin tabla `EVENTO` (trazabilidad e idempotencia con `CLASIFICACION` + `DERIVACION`).
- `CLASIFICACION` (22/09/2026): una fila por clasificación. `origen` `ia` | `operador`; `modelo` y `versionPrompt` si hubo LLM (27/10/2026); `nIntentos` solo en filas `ia` (el tope de reclasificaciones cuenta `nIntentos > 1`); `usuario_id` solo en filas `operador`. `departamentoAsignado` obligatorio.
- Reparto (28/09/2026): el modelo clasifica solo por departamento; no hay catálogo de tipos. `tipoAsignado` y `tipoDetectado` eliminados del modelo. El administrador reclasifica eligiendo departamento.
- Prompt de clasificación (27/10/2026): todas las llamadas automáticas a IA las hace n8n, también la clasificación. Las reglas viven en el prompt versionado, no en BBDD: se elimina `DEPARTAMENTO.criteriosReparto` (era una copia sin uso). n8n devuelve departamento, score, `modelo` y `versionPrompt`; Java valida que el departamento existe y está activo, y guarda la clasificación (`POST /clasificaciones`). Cambia el ER aprobado: comentarlo con la tutora de empresa.
- Procesamiento documental (RF-04/RF-05): se contempla para comunicaciones y notificaciones, sobre el documento principal, mediante extracción nativa, OCR o capacidades multimodales de OpenAI o Mistral. Su finalidad principal es extraer e interpretar información útil para la tramitación, no producir únicamente un resumen. Los anexos y acuses se archivan, pero no se interpretan en este alcance. El procesamiento externo de documentos reales requiere validación previa de Seguridad y Protección de Datos.
- Frontend (RF-11.6): las comunicaciones y notificaciones se presentan en dos listados separados según `tipoEnvio`; la consulta sigue siendo local.
- Alcance de interpretación por departamento: la guía de Kit Digital de Canal Directo es material de dominio para evaluar fases y requerimientos, sin catálogo de salida cerrado para el MVP. La clasificación documental por subdepartamentos de Siniestros es evolución futura fuera del MVP por falta de criterios suficientes.
- Fuente de avisos: existen ejemplos reales remitidos por Notific@ y por DEHú, de ambos tipos de envío, con acceso indicado en DEHú. Los avisos por correo no son fuente de ingesta. La cobertura de LEMA para envíos dirigidos a personas físicas cuyos avisos recibe MGS sigue pendiente de comprobar.
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