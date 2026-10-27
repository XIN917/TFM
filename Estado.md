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

Análisis y diseño del MVP cerrados en lo esencial (ER aprobado por la tutora de empresa). La clasificación por departamento es previa a la apertura: organismo emisor y concepto son la señal base. Una notificación ya clasificada por encima del umbral comparece cuando el Operador pulsa el botón de descarga en el frontal (RF-01.5); no hay proceso automático de comparecencia. Siguen abiertas reglas de reparto (ver `Criterios_Clasificacion.md`). La obtención de texto y la interpretación de información relevante del documento principal se contemplan para ambos tipos de envío, condicionadas a la autorización corporativa del tratamiento mediante IA externa.

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
- Flujos (05/10/2026): sondeo, autocorrección, comparecencia bajo demanda, operador, consulta local, reconciliación. Comunicación clasificada: `peticionAcceso()`, anexos si hay `anexosReferencia`, sin acuse, interpretación del principal si está autorizada y ticket. Notificación clasificada: queda `pendiente_descarga`, a la espera de que el Operador pulse el botón (27/10/2026: cierra la pendiente de continuidad del flujo separado).
- Comparecencia (27/10/2026): ya no hay lote de madrugada automático. El Operador pulsa un botón de descarga en el frontal para cada notificación clasificada por encima del umbral, o con departamento asignado por revisión humana (RF-09.5/09.6); se abre el documento, se guardan anexos y acuse, y queda lista para derivar — la deriva el proceso independiente de RF-08. Por debajo del umbral no se abre. La interpretación del documento principal de las notificaciones está contemplada si está autorizada; sigue pendiente decidir cómo derivar cuando la interpretación no esté autorizada o falle.
- Fuente y reparto: DEHú/LEMA, no los correos de aviso. Reglas en `Criterios_Clasificacion.md` (fuente única); `PRD.md` §6 solo guarda el catálogo de departamentos (`id`, nombre) que usa el prompt de n8n como `enum` — Java no necesita ni consulta las reglas. Los departamentos de clasificación son de primer nivel: subáreas y unidades internas (p. ej. el Servicio Jurídico de RRHH) no lo son.
- Atributos de LEMA y realizadas (05/10/2026). Se guardan como columnas los atributos que devuelve LEMA, incluidos los blobs sin documentar (`COMUNICACION.metadatosPublicos`, `DOCUMENTO.metadatos`); no hay columna con la respuesta en bruto. `REALIZADAS` guarda los envíos realizados anteriores al arranque para probar la clasificación, sin relación con el flujo ni acciones (RF-12.4–12.6, opcional). Un identificador vive en una sola tabla; un envío posterior al arranque que falte en `COMUNICACION` es anomalía de reconciliación.
- `PRD.md` autocontenido para implementación (diccionario, LEMA, endpoints, criterios de aceptación, semilla §6). Copia de trabajo en `HVOrganismosPublicos`; al copiar, no arrastrar notas locales ni nombres de personas.
- Acceso LEMA: sin token ni usuario; cada llamada va firmada con X.509 (WS-Security). Pruebas `se-gd-dehuws.redsara.es` (certificado autofirmado, alta propia; instrucciones en `docs/Instrucciones certificado LEMA pruebas.docx`); producción `gd-dehuws.redsara.es`.
- `TEXTOEXTRAIDO` tabla 1:1 opcional de `INTERPRETACION` (sugerencia de la tutora de empresa). Retención por parámetro global; el valor se fija tras contrastarlo con las áreas. Sin retención permanente para entrenamiento en el MVP.
- ER: tablas renombradas; `USUARIO` con clave compartida a `PERSONA`; sin tabla `EVENTO` (trazabilidad e idempotencia con `CLASIFICACION` + `DERIVACION`).
- `CLASIFICACION` (22/09/2026): una fila por clasificación. `origen` `ia` | `operador`; `modelo` y `versionPrompt` si hubo LLM (27/10/2026); `nIntentos` solo en filas `ia` (el tope de reclasificaciones cuenta `nIntentos > 1`); `usuario_id` solo en filas `operador`. `departamentoAsignado` obligatorio.
- Reparto (28/09/2026): el modelo clasifica solo por departamento; no hay catálogo de tipos. `tipoAsignado` y `tipoDetectado` eliminados del modelo. El administrador reclasifica eligiendo departamento.
- Orquestación (27/10/2026): n8n programa el sondeo y aloja los flujos de IA (clasificación e interpretación). Java dirige el flujo, es dueño del estado y llama a esos flujos por HTTP, fuera de transacción; el `estado` de `COMUNICACION` hace de cola y lo que falla se reintenta en el siguiente ciclo. Las reglas viven en el prompt versionado, no en BBDD: se elimina `DEPARTAMENTO.criteriosReparto`. El flujo devuelve `departamentoId`, `estado`, `score`, `modelo` y `versionPrompt`. Detalle en `PRD.md` §3 y RF-02.
- Comparecencia de notificaciones (fecha a confirmar): sustituye al lote de madrugada automático. Botón en el frontal, pulsado por un usuario con la funcionalidad Descargar, sin que acceda a DEHú; dispara `peticionAcceso()` y, desde las filas `DOCUMENTO` que deja pendientes esa misma llamada, `consultaAnexos()`/`consultaAcusePdf()` con reintento acotado. Se registra qué usuario y cuándo (`COMUNICACION.usuarioDescarga`/`fechaClicDescarga`). Aplica también a una notificación resuelta por revisión humana (RF-09.5/09.6). Vigilancia de descargas incompletas y aviso de vencimiento sin descargar: propuestos como proceso aparte (`/vigilanciaDescargas`), priorizado como Could — en el MVP se cubren con la vista de pendientes (RF-11.7/11.8) y el reintento manual. Detalle en `PRD.md` RF-01.5/RF-03.
- Decisiones tomadas al implementar y pasadas al PRD (27/10/2026): derivación por intentos (`en_curso` → `exito` / `fallo` / `desconocido`, sin reintentar `desconocido`); permisos por funcionalidades de SeguridadMGS en lugar de roles en `USUARIO`; transacciones cortas sin llamadas HTTP dentro.
- RF-07/RF-08 (27/10/2026; diseño aprobado, implementación pendiente). Prioridad: crear el ticket con la API back disponible de Ticketing. La derivación inicial es un proceso independiente de la ingesta (`POST /derivaciones`, programado por n8n) sobre las comunicaciones listas; ni los ciclos ni aceptar una revisión derivan, y las reclasificaciones siguen su flujo. Un `fallo` se reintenta como máximo 2 veces (`maxReintentos`, `0` los desactiva), contando las filas `DERIVACION` existentes, sin columnas nuevas; es independiente del límite de 2 de RF-10. `desconocido` y `en_curso` no cambian solos; lo que no se resuelve va a una conciliación manual excepcional. La publicación de RF-07 queda **pendiente de decisión** (Infra): el estado persistido desacopla la ingesta de la derivación, pero no cumple RF-07. Se descartan, por basarse en infraestructura no confirmada, la publicación en Kafka con reserva y republicación, el webhook provisional y las columnas `eventoIntentadoEn` y `reintentable`. Detalle en `PRD.md` RF-07/RF-08.
- Procesamiento documental (RF-04/RF-05): se contempla para comunicaciones y notificaciones, sobre el documento principal, mediante extracción nativa, OCR o capacidades multimodales de OpenAI o Mistral. Su finalidad principal es extraer e interpretar información útil para la tramitación, no producir únicamente un resumen. Los anexos y acuses se archivan, pero no se interpretan en este alcance. El procesamiento externo de documentos reales requiere validación previa de Seguridad y Protección de Datos.
- Frontend (RF-11.6): las comunicaciones y notificaciones se presentan en dos listados separados según `tipoEnvio`; la consulta sigue siendo local.
- Alcance de interpretación por departamento: la guía de Kit Digital de Canal Directo es material de dominio para evaluar fases y requerimientos, sin catálogo de salida cerrado para el MVP. La clasificación documental por subdepartamentos de Siniestros es evolución futura fuera del MVP por falta de criterios suficientes.
- Fuente de avisos: existen ejemplos reales remitidos por Notific@ y por DEHú, de ambos tipos de envío, con acceso indicado en DEHú. Los avisos por correo no son fuente de ingesta. La cobertura de LEMA para envíos dirigidos a personas físicas cuyos avisos recibe MGS sigue pendiente de comprobar.
- Aprendizaje continuo del Agente IA y diagnóstico sistemático de errores: evolución futura, no diseño cerrado.

## 4. Arquitectura (resumen)

Detalle en `Diagrama_Clases.md` y `Diagrama_Componentes.md`.

- **Módulos Eclipse:** ver `PRD.md` §4 (familia Ticketing; sin DAO/EJB). Incluye `Test` (JUnit para el TFM) y `FT` (Playwright).
- **Stack alineado con Ticketing (cerrado; tutora de empresa, sept. 2026):** Java 8, **Java EE 7** (`javax.*`), **WAS 9.0**, JDBC propio (sin JPA/Spring), CDI, JAX-RS. Jakarta EE descartado para este aplicativo hasta que Sistemas mueva de WAS 9. Módulos y CDI por constructor: `PRD.md`. Misma convención de estereotipos (`@Repositorio`, `@Servicio`, `@Endpoint`, `@Transaccional`), sin `AggregateRoot` ni eventos de dominio CDI (los cubre RF-07 a nivel corporativo).
- **Estilos:** orientada a eventos entre componentes (bus corporativo para la reclasificación y, cuando se decida, para RF-07; hasta entonces, el estado persistido desacopla pipeline y ejecución); hexagonal por dentro de cada componente. Justificarlo explícitamente en la memoria, no dejarlo como efecto secundario de SOLID.
- **Frontal:** subconjunto CRUD + listado + detalle + un flujo de estado, sobre librería de terceros. No replicar el acabado visual de Ticketing (vive en librerías internas, fuera de plazo).
- **Componentes:** línea discontinua solo para lo no confirmado — modificar ticket (audiencia back), evento de cancelación (proceso que lee el outbox) y publicación de RF-07.

## 5. Planificación

Ver `Gantt.md`. Cinco fases: Análisis y diseño → Desarrollo → Testing → Memoria → Revisión final.

Deadlines de negocio: backend/frontend antes del 25 dic; desarrollo y testing cierran el 31 dic; revisión final 4–15 ene 2027. Festivos pendientes de recalcular (ver `TODO.md`).

## 6. Memoria

Estructura: `Estructura_Memoria.md`. Introducción cerrada (revisada el 23/09; `memoria/01_Introduccion.md` sincronizado con Overleaf). Gestión del proyecto: iniciada el 01/09 (Gantt en `Diagramas/Gantt.md`).