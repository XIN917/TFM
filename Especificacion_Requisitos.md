# Especificación de Requisitos
## Automatización de la recepción y procesamiento de comunicaciones de la administración pública (DEHú/LEMA)

---

## 0. Notas de anclaje técnico

Esta especificación se ha construido sobre la base real de la **Guía de integración para Grandes Destinatarios (LEMA)** de DEHú, no sobre supuestos genéricos de "una API pública". Esto tiene implicaciones directas en el diseño del MVP que conviene dejar explícitas antes de entrar en los requisitos:

- **Protocolo**: SOAP 1.1 sobre WSDL, no REST/JSON. Los mensajes van firmados mediante **WS-Security con certificado X.509** (no hay API key ni OAuth).
- **Dos familias de servicios, no una sola API de "consulta"**:
  - *Servicios LEMA (pendientes de acceso)*: `localiza()`, `peticionAcceso()`, `consultaAnexos()`, `consultaAcusePdf()`.
  - *Servicios ConsultaRealizadas (ya comparecidas/leídas)*: `localizaRealizadas()`, `consultaRealizadas()`.
- **Restricción temporal crítica**: `consultaAnexos()` y `consultaAcusePdf()` no permiten consultar anexos ni acuses de notificaciones con más de **1 día** de antigüedad. Esto obliga a que el almacenamiento (RF-02) sea **inmediato y no diferible** tras la comparecencia — no es una tarea que se pueda posponer o reintentar días después.
- **Límite de peticiones**: máximo 1000 peticiones por operación, sin límite de peticiones continuas. Condiciona el diseño del *scheduler* de sondeo (RF-01).
- **Onboarding previo obligatorio**: el alta como Gran Destinatario requiere autoregistro en el portal DEHú, firma de una Declaración Responsable, y validación en entorno de pruebas (SE) antes de producción (PRO). Esto es una dependencia de proyecto, no un requisito funcional del sistema, pero condiciona el calendario.
- Los anexos pueden ser **por URL directa** o **por referencia** (identificador que hay que resolver con `consultaAnexos()`); solo los segundos requieren descarga activa por parte del Gran Destinatario.

Estas particularidades se reflejan como notas técnicas dentro de cada requisito funcional relevante.

---

## 1. Alcance del MVP

El MVP cubre el ciclo **de recepción**: desde la detección de una comunicación nueva en DEHú hasta la **publicación de un evento de comunicación clasificada** y la **ejecución de la acción resultante** (crear ticket, depositar en un buzón, o enviar un email, según lo determine el contexto), con posibilidad de intervención humana cuando el sistema no tenga confianza suficiente para decidir.

Es importante distinguir dos cosas que en una primera versión de este documento quedaron colapsadas en una sola: **generar el evento** (el hecho de que una comunicación ha sido clasificada y está lista para actuar sobre ella) y **ejecutar la acción** que consume ese evento. Modelarlas por separado es lo que permite que el sistema de ticketing no sea la única salida posible — si resulta inviable, cualquier otro canal (email, buzón interno, un automatismo futuro) puede engancharse al mismo evento sin rediseñar el pipeline de detección/almacenamiento/IA.

**Explícitamente fuera de alcance del MVP** (ver sección 5):
- Envío de comunicaciones o respuestas hacia la administración.
- Automatismos corporativos que modifiquen sistemas de negocio más allá de notificar (apertura automática de expedientes, lanzamiento de procesos operativos) — el MVP contempla el enrutado del evento a un canal de notificación (ticket, buzón o email), no la ejecución de acciones de negocio.
- Integración general de agentes de IA vía MCP con capacidad de invocar acciones arbitrarias sobre sistemas corporativos (se contempla como evolución post-MVP de alcance amplio). El MVP acota este uso a un caso concreto y ya confirmado: el Agente IA de reclasificación (RF-10) invoca, vía MCP, la acción de finalizar el ticket original y crear uno nuevo en la cola correcta cuando su propia confianza de autocorrección supera el umbral. Cualquier otra invocación de acciones por parte de agentes IA fuera de este caso queda fuera de alcance del MVP.
- Gestión multi-certificado / multi-entidad si la compañía opera con más de una razón social frente a DEHú.

---

## 2. Actores

Siguiendo la definición estricta de actor UML (entidad externa a la frontera del sistema), se distingue entre los actores que interactúan con el sistema desde fuera y los componentes que forman parte de la implementación interna.

### 2.1 Actores (externos al sistema)

| Actor | Rol |
|---|---|
| DEHú/LEMA | Fuente externa de comunicaciones oficiales (SOAP, WS-Security); actor de la detección y consulta (RF-01, RF-02). |
| Usuario/Operador | Persona que revisa y, si procede, reclasifica las comunicaciones que la IA no puede resolver con confianza suficiente (RF-09). |
| Departamento | Equipo/área de negocio destinataria de la notificación (ticket, buzón o email, según RF-08); consulta lo que le llega. |

### 2.2 Componentes internos (no son actores UML)

| Componente | Rol |
|---|---|
| Orquestador (n8n u equivalente) | Coordina el flujo completo (RF-01 a RF-08) y publica/consume eventos. |
| Motor de IA (OCR + LLM) | Extrae texto, interpreta y clasifica el contenido (RF-03 a RF-05); pieza interna, no visible como actor en el diagrama de casos de uso salvo que en el futuro se externalice como un módulo de agentes IA independiente y consumible por otras aplicaciones — decisión de arquitectura pendiente de valorar con la empresa, fuera del alcance de este documento. |
| Repositorio documental interno | Almacena documentos, anexos, acuses y metadatos (RF-02). |
| Canales de notificación (ticket / buzón / email) | Consumidores del evento de RF-07, seleccionados según configuración en RF-08. |
| Agente IA de reclasificación | Componente distinto del Motor de IA de RF-03/RF-05: se activa por el evento de reclasificación solicitada (RF-10), evalúa su propia confianza de autocorrección, e invoca vía MCP la acción de gestión de ticket cuando la supera. |

---

## 3. Requisitos funcionales

### RF-01 — Detección y consulta de nuevas comunicaciones

| | |
|---|---|
| **Descripción** | El sistema debe sondear periódicamente DEHú para detectar comunicaciones y notificaciones pendientes de acceso, y obtener su contenido. |
| **Sub-requisitos** | RF-01.1 Invocar `localiza()` de forma periódica (frecuencia configurable) para obtener el listado de envíos pendientes en cualquiera de los PUC integrados. Si la lista devuelta está vacía, el sistema no ejecuta ningún procesamiento adicional y simplemente espera hasta el siguiente ciclo de sondeo.<br>RF-01.2 Si la lista no está vacía, iterar sobre cada elemento devuelto e invocar, para cada uno, `peticionAcceso()` para comparecer la notificación / acceder al contenido de la comunicación, obteniendo documento(s) y metadatos, antes de pasar al siguiente elemento de la lista.<br>RF-01.3 Deduplicar mediante el identificador único devuelto por DEHú, para garantizar idempotencia ante reintentos o fallos parciales.<br>RF-01.4 Registrar y reintentar (con backoff) los fallos de conectividad, expiración de certificado o errores SOAP, sin perder el envío de la cola de pendientes. |
| **Nota técnica** | El límite de 1000 peticiones por operación condiciona el tamaño de lote del sondeo; al no haber límite de peticiones continuas, es preferible un polling frecuente con lotes pequeños a uno infrecuente con lotes grandes. |
| **Criterio de aceptación** | Ninguna comunicación pendiente permanece sin detectar más de N minutos (SLA a definir); ninguna comunicación se procesa dos veces. |

### RF-02 — Almacenamiento de la comunicación y sus documentos

| | |
|---|---|
| **Descripción** | El sistema debe persistir de forma inmediata y completa el documento principal, sus anexos y su acuse, junto con los metadatos asociados. |
| **Sub-requisitos** | RF-02.1 Persistir el documento principal y metadatos devueltos por `peticionAcceso()` (nombre, tipo MIME, fecha de evento, identificador, código de origen).<br>RF-02.2 Descargar y persistir **todos** los anexos de la comunicación, sin excepción, para no arriesgar la pérdida de información relevante que pueda venir en cualquiera de ellos. La vía de descarga depende del tipo: (a) **tipo URL directa** → petición HTTP estándar a esa dirección, sin llamar a DEHú; (b) **tipo referencia** (sin URL accesible) → invocar `consultaAnexos()`. Ambas ramas son obligatorias cuando el anexo correspondiente existe, y terminan almacenando el anexo de la misma forma (RF-02.4/02.5).<br>RF-02.3 Descargar y almacenar el acuse mediante `consultaAcusePdf()` **inmediatamente** tras la comparecencia — no es una tarea diferible.<br>RF-02.4 Verificar la integridad del documento comparando el hash (SHA-256) recibido en la respuesta con el hash calculado localmente.<br>RF-02.5 Mantener trazabilidad completa: identificador DEHú, código de origen, fecha de evento, y timestamps internos de ingesta. |
| **Nota técnica — CRÍTICA** | `consultaAnexos()` y `consultaAcusePdf()` **no permiten consultar elementos de notificaciones con más de 1 día de antigüedad**. Si RF-02.2/02.3 fallan y no se reintentan con éxito dentro de esa ventana, el anexo o el acuse se pierde de forma **irrecuperable** vía API. Esto debe traducirse en una alerta operativa de alta prioridad ante cualquier fallo de descarga, no en un reintento silencioso de baja prioridad. |
| **Criterio de aceptación** | El 100% de los documentos comparecidos en el día tienen su acuse y sus anexos-referencia almacenados antes de que expire la ventana de 24h. |

### RF-03 — Procesamiento mediante IA (OCR + interpretación de contenido)

| | |
|---|---|
| **Descripción** | El sistema debe extraer texto estructurado de los documentos recibidos (PDF, escaneados o nativos) para permitir su interpretación. |
| **Sub-requisitos** | RF-03.1 Aplicar OCR a documentos sin capa de texto (escaneados).<br>RF-03.2 Extraer texto nativo cuando el documento ya lo contenga.<br>RF-03.3 Normalizar el resultado en un formato común independientemente del origen (documento principal, anexo referenciado, anexo por URL). |
| **Criterio de aceptación** | Todo documento almacenado en RF-02 tiene una representación textual asociada, con indicador de calidad/confianza del OCR cuando aplique. |

### RF-04 — Interpretación automática del contenido

| | |
|---|---|
| **Descripción** | El sistema debe interpretar semánticamente el texto extraído mediante un modelo LLM para identificar entidades y naturaleza de la comunicación. |
| **Sub-requisitos** | RF-04.1 Identificar tipo de comunicación (p. ej. demanda judicial, requerimiento, notificación administrativa, comunicación informativa).<br>RF-04.2 Extraer entidades relevantes: organismo emisor, referencia/expediente si existe, plazos, importes, partes involucradas.<br>RF-04.3 Determinar si la comunicación se relaciona con un expediente ya existente en los sistemas internos (búsqueda por referencia/entidad) o si requiere apertura de uno nuevo — como *dato de salida*, no como acción automática ejecutada en el MVP (ver sección 5).<br>RF-04.4 Producir un score de confianza asociado a la interpretación. |
| **Criterio de aceptación** | Cada comunicación procesada produce una estructura de datos con tipo, entidades y score de confianza, verificable por un humano. |

### RF-05 — Clasificación y determinación del departamento/equipo

| | |
|---|---|
| **Descripción** | El sistema debe determinar, en base a la interpretación anterior, qué área de negocio debe recibir la comunicación (siniestros, laboral, comercial/concursos públicos, fiscal, etc.). |
| **Sub-requisitos** | RF-05.1 Aplicar reglas de negocio y/o modelo de clasificación sobre el resultado de RF-04.<br>RF-05.2 Asociar cada clasificación a un departamento/equipo y, de forma independiente, al canal de notificación que le corresponda (ticket, buzón o email — ver RF-08).<br>RF-05.3 Marcar como "sin clasificar" cualquier comunicación cuyo score de confianza (RF-04.4 y/o de la propia clasificación) esté por debajo de un umbral configurable, derivándola a RF-09.<br>RF-05.4 Registrar toda comunicación clasificada en un historial trazable, sea cual sea el resultado: si la confianza es baja, en la cola de revisión pendiente (RF-09.1); si es alta, en un historial de comunicaciones procesadas — ninguna comunicación queda sin dejar rastro tras su clasificación. |
| **Criterio de aceptación** | Toda comunicación queda clasificada con un departamento y una confianza, o marcada explícitamente para revisión humana — nunca queda sin resolución de estado. |

### RF-06 — Generación automática del contenido estructurado del ticket

| | |
|---|---|
| **Descripción** | El sistema debe construir el contenido del ticket a partir de los datos interpretados y clasificados, en un formato apto para el sistema de ticketing destino. |
| **Sub-requisitos** | RF-06.1 Generar título, descripción/resumen, campos estructurados (tipo, organismo, referencia, plazo, departamento) y adjuntos vinculados (documento + anexos + acuse almacenados en RF-02).<br>RF-06.2 Incluir en el ticket la trazabilidad hacia el identificador DEHú original, para permitir auditoría y evitar duplicados. |
| **Criterio de aceptación** | El contenido generado es válido según el esquema/API del sistema de ticketing destino, sin intervención manual, para el 100% de los casos clasificados con confianza suficiente. |

### RF-07 — Generación y publicación del evento de comunicación clasificada

| | |
|---|---|
| **Descripción** | El sistema debe publicar un evento estructurado, dentro de la arquitectura orientada a eventos de la compañía, en cuanto una comunicación queda clasificada y con su contenido de ticket generado (RF-06). Este evento es el punto de desacople entre "la IA ha terminado su trabajo" y "algo actúa en consecuencia". |
| **Sub-requisitos** | RF-07.1 El evento debe incluir el contenido estructurado (RF-06), el departamento/cola destino (RF-05) y el identificador DEHú original, de forma que cualquier consumidor tenga lo necesario sin volver a consultar el pipeline.<br>RF-07.2 El evento se publica exista o no, en ese momento, un consumidor concreto activo — el acoplamiento con el canal de salida (RF-08) no debe ser directo. |
| **Nota de diseño** | Este requisito responde directamente a la arquitectura orientada a eventos ya existente en la compañía (n8n + ecosistema de eventos corporativo) descrita en la propuesta original del proyecto, y es lo que permite que RF-08 pueda cambiar de canal sin tocar RF-01 a RF-06. |
| **Criterio de aceptación** | Toda comunicación que complete RF-06 genera exactamente un evento publicado, verificable de forma independiente al resultado de RF-08. |

### RF-08 — Ejecución de la acción resultante según el contexto

| | |
|---|---|
| **Descripción** | El sistema debe consumir el evento de RF-07 y ejecutar la acción de notificación que corresponda según el contexto — no asumir que el destino es siempre el sistema de ticketing. |
| **Sub-requisitos** | RF-08.1 Crear el ticket en la cola correspondiente cuando el sistema de ticketing es el canal configurado para ese departamento/tipo de comunicación (comportamiento por defecto).<br>RF-08.2 Enviar un email al departamento/equipo destino como canal alternativo, configurable por departamento o activable como contingencia si el sistema de ticketing no está disponible o resulta inviable.<br>RF-08.3 Depositar la comunicación en un buzón interno cuando así lo determine la regla de negocio asociada al tipo de comunicación o departamento (mecanismo ya existente, mencionado en la propuesta original).<br>RF-08.4 La selección de canal debe resolverse por configuración/reglas, no por código específico de cada caso, para que añadir un canal nuevo no implique modificar RF-01 a RF-07.<br>RF-08.5 Confirmar la ejecución (idempotencia: no duplicar la acción si ya se ejecutó para el mismo identificador DEHú) y registrar el resultado (éxito/fallo), con reintento en caso de fallo transitorio del canal de destino. **Excepción confirmada**: en una reclasificación posterior al envío inicial, el ticketing no soporta transferir de cola (ver sección 5.2), por lo que el flujo correcto es finalizar el ticket original y crear uno nuevo en la cola correcta — este segundo ticket no debe tratarse como duplicado, sino como resultado esperado de RF-09/reclasificación, ya sea manual (Operador) o automática (RF-10, Agente IA vía MCP). |
| **Criterio de aceptación** | Toda comunicación clasificada con éxito produce exactamente una acción de notificación, en el canal correcto según configuración, en un tiempo máximo definido por SLA. |

### RF-09 — Revisión humana cuando la IA no sabe qué hacer

| | |
|---|---|
| **Descripción** | El sistema debe ofrecer un mecanismo de intervención humana para los casos en que la interpretación (RF-04) o la clasificación (RF-05) no alcancen el umbral de confianza requerido, o cuando se detecte una excepción/error en el flujo. |
| **Sub-requisitos** | RF-09.1 Generar un ticket o tarea de "revisión pendiente" (en cola específica) cuando el score de confianza esté por debajo del umbral.<br>RF-09.2 Presentar al revisor humano el documento, el texto extraído y la interpretación/clasificación propuesta por la IA (aunque de baja confianza), para agilizar la decisión.<br>RF-09.3 Permitir que la decisión humana (departamento correcto, tipo correcto) se registre como *feedback* reutilizable para mejorar el modelo/reglas en iteraciones futuras (deseable en MVP, no bloqueante) — modelado en el diagrama de flujo del Operador como un consumidor asíncrono más del mismo evento que dispara la acción normal (RF-08), sin bloquear ni retrasar el cierre del caso.<br>RF-09.4 Garantizar que ninguna comunicación queda "silenciosamente" sin acción ni revisión — toda comunicación detectada en RF-01 termina en una acción normal (RF-08) o en una tarea de revisión (RF-09).<br>RF-09.5 Si el Operador considera correcta la propuesta de la IA, debe poder aceptarla explícitamente, lo que dispara la ejecución de la acción normal (RF-08) con la clasificación propuesta sin modificar.<br>RF-09.6 Si el Operador considera incorrecta la propuesta, debe poder reclasificar manualmente (departamento y/o tipo) y asignar la comunicación al departamento correcto; esta acción se acoge a la misma excepción de idempotencia de RF-08.5 cuando ya existiera un ticket previo para el mismo identificador DEHú. |
| **Criterio de aceptación** | El 100% de las comunicaciones detectadas tienen un estado final trazable: acción ejecutada automáticamente, o en revisión humana. Ninguna se pierde entre pasos. |
| **Nota de diseño — aprendizaje continuo** | Que el Agente IA aprenda a partir del *feedback* humano registrado en RF-09.3 (es decir, que la corrección de un Operador mejore la precisión de futuras clasificaciones, no solo quede almacenada como dato) es una dirección importante de evolución del proyecto. El mecanismo concreto para lograrlo (ajuste de reglas, fine-tuning del modelo, optimización de prompts, u otro) no está definido todavía — depende de una exploración técnica que se abordará durante el desarrollo, según lo que resulte viable. No se trata de una pregunta abierta que quede fuera del alcance del proyecto, sino de una línea de trabajo propia que se incorporará si la exploración técnica lo permite. |

### RF-10 — Reclasificación automática mediante Agente IA

| | |
|---|---|
| **Descripción** | Cuando el Departamento reporta que una clasificación ya enviada es incorrecta, el sistema debe publicar un evento que active un Agente IA capaz de intentar autocorregir la clasificación antes de requerir intervención humana, reduciendo la necesidad de que el Operador esté monitorizando constantemente. |
| **Sub-requisitos** | RF-10.1 Consumir el evento de reclasificación solicitada que se genera cuando el Departamento cancela la notificación asignada (equivalente a reportarla como mal clasificada). El reporte en sí ocurre **fuera de la frontera del sistema**, dentro del propio sistema de Ticketing (probablemente como un cambio de estado del ticket — ver limitación confirmada en sección 6, sin campo de motivo en frontend); el MVP no diseña esa interfaz, solo reacciona al evento resultante, reutilizando el mismo patrón de evento de RF-07.<br>RF-10.2 Al generarse el evento de cancelación, incrementar y persistir (asociado al identificador DEHú de la comunicación) un contador de intentos de autocorrección — de forma análoga a la deduplicación de RF-01.3, el contador no puede ser una variable local a esta ejecución, porque el Departamento puede volver a cancelar en un ciclo posterior y el sistema debe recordar los intentos previos.<br>RF-10.3 Antes de intentar la reclasificación, comprobar el contador persistido: si ya alcanzó el límite (2), escalar directamente a la cola de revisión humana (RF-09) sin invocar al Agente IA. Si aún hay margen, el Agente IA genera una nueva propuesta de clasificación junto con su propio score de confianza, sobre el mismo criterio de umbral usado en RF-04.4/RF-05.3.<br>RF-10.4 Si la confianza de la autocorrección supera el umbral, el agente invoca — vía MCP — la acción de finalizar el ticket original y crear uno nuevo en la cola correcta, acogiéndose a la excepción de idempotencia ya prevista en RF-08.5/sección 5.2.<br>RF-10.5 Si la confianza no supera el umbral en este intento, se escala a la cola de revisión humana (RF-09) — el siguiente reintento, si lo hay, vendrá de una futura cancelación del Departamento, ya contabilizada por el contador de RF-10.2, no de un bucle interno de este mismo proceso. |
| **Nota técnica** | El Agente IA expone la acción de gestión de tickets (finalizar/crear) como una herramienta MCP invocable de forma autónoma; ningún otro caso de invocación de acciones por agentes IA está en alcance del MVP (ver sección 1). |
| **Nota de diseño** | Este requisito responde a una propuesta explícita de negocio (JM) de automatizar la corrección de errores de clasificación sin depender de que el Operador esté disponible, apoyándose en el mismo mecanismo de eventos ya establecido en RF-07/RF-10.1. |
| **Criterio de aceptación** | Toda comunicación reportada como mal clasificada obtiene, sin excepción, una resolución automática (ticket reclasificado) o una escalada a revisión humana — nunca queda sin resultado ni requiere aviso adicional fuera de la propia creación del ticket. |

### RF-11 — Consulta local de registros por el Operador

| | |
|---|---|
| **Descripción** | El Operador debe poder consultar, bajo demanda, los registros ya almacenados en el repositorio local (independientemente del sondeo periódico de RF-01), diferenciando entre registros con y sin acción de derivación generada porque su capacidad de modificación es distinta. |
| **Sub-requisitos** | RF-11.1 Buscar y mostrar registros del repositorio local, sin invocar a DEHú en ningún momento de esta consulta.<br>RF-11.2 Si el registro no tiene ninguna acción de derivación asociada (es decir, es una comunicación tal como la recibió DEHú, sin haber generado aún ninguna acción hacia un departamento), el resultado es de solo lectura — no existe acción de modificación.<br>RF-11.3 Si el registro sí tiene al menos una acción de derivación asociada (ya se generó un ticket u otro canal de salida), el Operador puede modificarlo, propagando el cambio al sistema de Ticketing (no una edición puramente local — ver Nota de diseño).<br>RF-11.4 Si la modificación **no cambia el departamento/cola destino** (p. ej. editar la descripción/resumen generado por la IA), el sistema actualiza el ticket existente in-place mediante la API de modificación de Ticketing, sin finalizar ni crear un ticket nuevo.<br>RF-11.5 Si la modificación **sí cambia el departamento/cola destino**, aplica el mismo mecanismo ya descrito en RF-08.5/sección 5.2: finalizar el ticket original y crear uno nuevo en la cola correcta — no se trata como el mismo caso que RF-11.4. |
| **Nota técnica** | La distinción editable/no editable no depende del campo `tipoEnvio` de DEHú (su correspondencia con "notificación"/"comunicación" no está verificada contra la guía de integración — ver también sección 6), sino de si la comunicación ya generó una acción de derivación (ticket u otro canal) en el propio sistema. Lo que DEHú entrega es siempre de solo lectura; lo editable es únicamente lo que el sistema propio ha producido a partir de ello. Una comunicación en cola de revisión pendiente (RF-09), al no tener todavía ninguna derivación generada, es de solo lectura bajo esta misma regla. |
| **Nota de diseño** | El criterio que separa RF-11.4 de RF-11.5 no es "modificar vs. reclasificar" en abstracto, sino si la modificación cambia o no la cola/departamento destino del ticket. Esta distinción es diseño propuesto y está **condicionada a la disponibilidad de la API de "audiencia back" del sistema de Ticketing** (aún no implementada — ver sección 6), que es la que permitiría a la aplicación invocar la modificación del ticket. Hasta que esa API exista y se confirmen sus capacidades reales (qué campos admite modificar, si un cambio de contenido dispara algún evento de estado no deseado) con el equipo de Ticketing, esta sub-especificación se mantiene como propuesta de alto nivel, no como diseño técnico cerrado. |
| **Criterio de aceptación** | El Operador puede consultar cualquier registro local sin depender de la disponibilidad de DEHú; solo se le ofrece la opción de modificar el ticket asociado cuando el registro ya tiene una acción de derivación generada — la comunicación original, tal como la entregó DEHú, nunca se modifica; esa modificación actualiza el ticket in-place o genera uno nuevo según si cambia o no la cola destino. |

### RF-12 — Reconciliación periódica con comunicaciones ya realizadas *(sujeto a disponibilidad de tiempo del proyecto)*

| | |
|---|---|
| **Descripción** | Proceso batch, independiente del ciclo reactivo de RF-01, que detecta comunicaciones que DEHú marca como ya completadas (`localizaRealizadas()`) pero que no existen en el repositorio local — posible síntoma de un fallo silencioso del flujo principal. |
| **Sub-requisitos** | RF-12.1 Invocar `localizaRealizadas()` de forma periódica (frecuencia distinta e independiente de RF-01.1) y filtrar el resultado por `fechaPuestaDisposicion` en el lado de la aplicación, ya que la API no admite filtro por fecha en la petición.<br>RF-12.2 Para cada elemento, consultar el repositorio local; si ya existe, no se realiza ninguna acción.<br>RF-12.3 Si no existe, invocar `consultaRealizadas()` para obtener el detalle y almacenarlo. |
| **Nota de alcance** | Este proceso no forma parte del alcance comprometido del MVP — se diseña como candidato a implementar si el calendario del proyecto lo permite (ver sección 1 y conclusiones/trabajo futuro de la memoria). |
| **Criterio de aceptación** | (aplicable solo si se implementa) Ninguna comunicación marcada como realizada en DEHú queda sin al menos un registro local, dentro del margen de la frecuencia configurada del proceso. |

---

## 4. Requisitos no funcionales

| Categoría | Requisito |
|---|---|
| **Seguridad** | Gestión segura del ciclo de vida del certificado X.509 usado para WS-Security (custodia, renovación, alertas de caducidad). El certificado es la identidad de la compañía frente a DEHú: su compromiso o expiración detiene toda la ingesta. |
| **Control de acceso** | El sistema debe autenticar a los usuarios internos (Operadores) contra el directorio de personal ya existente en la compañía, y aplicar control de acceso basado en roles sobre el frontal propio. Como mínimo se distinguen dos roles: `operador` (acceso a RF-09: aceptar o reclasificar comunicaciones en revisión) y `administrador` (además, ejecutar reclasificaciones — restricción ya confirmada con el equipo funcional). Toda resolución de una revisión pendiente (RF-09.5/RF-09.6) debe quedar asociada al usuario que la ejecutó, para trazabilidad. |
| **Cumplimiento normativo** | Conservación de las comunicaciones oficiales conforme a plazos legales aplicables (procedimiento administrativo, normativa sectorial de seguros). Trazabilidad íntegra desde la recepción hasta el ticket. |
| **Resiliencia** | El sistema debe tolerar caídas temporales de DEHú (entorno SE/PRO) sin pérdida de comunicaciones ya detectadas, especialmente considerando la ventana de 1 día para anexos/acuses (RF-02). |
| **Rendimiento** | Respetar el límite de 1000 peticiones por operación de LEMA; diseño de sondeo que evite saturar dicho límite en picos de volumen. |
| **Auditabilidad** | Registro (log) de cada llamada SOAP realizada, su resultado, y de cada decisión de clasificación tomada por la IA, con su score de confianza, para poder auditar decisiones automáticas. |
| **Escalabilidad y extensibilidad (principio de diseño, no solo objetivo deseable)** | El desacople entre generación de evento (RF-07) y ejecución de la acción (RF-08) no es incidental: es la decisión de arquitectura que permite (a) añadir un canal de salida nuevo sin tocar el pipeline de detección/IA, y (b) incorporar en el futuro nuevas fuentes de comunicaciones además de DEHú, publicando al mismo bus de eventos desde un adaptador de entrada distinto. El MVP debe demostrar este principio con al menos dos canales de salida reales (ticket + email), no solo documentarlo como intención. |
| **Entornos** | Soporte de los dos entornos de LEMA (SE - pruebas, PRO - producción) con configuración diferenciada, alineado con el proceso de alta como Gran Destinatario. |

---

## 5. Fuera de alcance del MVP (explícito)

- **Envío de respuestas o comunicaciones hacia la administración** — ver justificación en la sección 5.1.
- Automatismos corporativos que modifiquen sistemas de negocio (apertura automática de expedientes, lanzamiento de procesos operativos más allá de notificar) — quedan como evolución post-MVP, una vez validada la fiabilidad de la clasificación. El MVP sí contempla, como parte del alcance (RF-08), elegir entre distintos *canales de notificación* (ticket, buzón, email); lo que queda fuera es que el sistema *actúe* sobre otros sistemas de negocio más allá de notificar.
- Integración de agentes de IA vía MCP con capacidad de invocar acciones — la propuesta original lo contempla como capa de integración avanzada; el MVP se limita a generar el ticket para que la acción la decida un humano o un flujo posterior.
- Soporte multi-certificado o multi-razón social frente a DEHú.
- Uso de los servicios `localizaRealizadas()` / `consultaRealizadas()` (comunicaciones ya comparecidas/leídas) — el MVP se centra en el flujo de comunicaciones *nuevas*; estos servicios son candidatos naturales para una fase de reconciliación/auditoría posterior. **Restricción a tener en cuenta cuando se aborde esa fase**: `localizaRealizadas()` no admite filtro por fecha en la petición (solo `nifTitular` y `tipoEnvio`, confirmado contra la guía LEMA), y el volumen real que puede devolver está sin verificar — antes de diseñar esa integración hay que comprobar cuántos registros llega a devolver en la práctica. Si el volumen es alto, una consulta bajo demanda no es viable; el patrón correcto sería un **proceso batch que sincronice periódicamente** (p. ej. diario/semanal) en lugar de consultar en tiempo real, filtrando por fecha del lado de la aplicación una vez descargado el listado completo.

### 5.1 Sobre la posibilidad de un canal bidireccional (recepción + envío)

Se ha valorado explícitamente si el sistema podría ampliarse para enviar respuestas o escritos a la administración, y se descarta para el MVP por una **razón técnica de fondo, no por simple limitación de alcance**:

- **DEHú/LEMA es, por diseño, un sistema de recepción.** Los seis servicios web disponibles (`localiza`, `peticionAcceso`, `consultaAnexos`, `consultaAcusePdf`, `localizaRealizadas`, `consultaRealizadas`) son todos de lectura. No existe, dentro de LEMA, ninguna operación de envío o presentación de escritos.
- **El canal de envío equivalente es un sistema distinto**: el Registro Electrónico General/Común (REG/REC, conforme a la Ley 39/2015). Este registro está pensado principalmente para presentación interactiva (persona física/jurídica con certificado digital o Cl@ve rellenando un formulario en la sede electrónica), y no ofrece una interfaz de "Gran Destinatario" para envío masivo automatizado equiparable a la que LEMA ofrece para la recepción.
- **El Sistema de Interconexión de Registros (SIR)** tampoco es una opción válida aquí: conecta registros *entre administraciones*, no sirve como canal de presentación desde entidades privadas.
- En la práctica, un canal de salida real obligaría a integrarse con la **sede electrónica específica de cada organismo destinatario**, cada una con su propio trámite, formulario y, en muchos casos, sin un API estandarizado equivalente al de LEMA para terceros automatizados.

**Conclusión para el diseño**: un canal verdaderamente bidireccional no es "una extensión" de la integración con DEHú, sino **dos integraciones arquitectónicamente independientes**, con protocolos, modelos de acceso y niveles de automatización distintos. Se recomienda mantener esto fuera del MVP y documentarlo como una limitación estructural del ecosistema de administración electrónica española, no como una decisión de producto revisable en la siguiente iteración del mismo sistema.

### 5.2 Dependencia confirmada: reasignación de departamento en el sistema de Ticketing

Cuando un Departamento detecta que una comunicación ya enviada (clasificada con alta confianza, sin pasar por revisión) tiene una categoría incorrecta y la reclasifica, hace falta reenviarla al departamento correcto. **Confirmado con el equipo responsable del Ticketing**: el sistema **no** soporta transferir/reasignar un ticket existente a otra cola. El mecanismo real es: **finalizar el ticket original y crear uno nuevo en la cola correcta**.

Esto tiene una consecuencia directa sobre RF-08.5 (ver más abajo): la regla de "no duplicar la acción para el mismo identificador DEHú" necesita una excepción explícita para el caso de reclasificación, donde generar un segundo ticket es el comportamiento correcto y esperado, no un duplicado accidental.

---

## 6. Riesgos y dependencias técnicas

| Riesgo/Dependencia | Impacto |
|---|---|
| La correspondencia entre `tipoEnvio` (1/2) y los conceptos "notificación"/"comunicación" de DEHú, mencionada en versiones anteriores de RF-11, no está confirmada contra la guía de integración LEMA (Anexo I) — los ejemplos disponibles solo muestran el valor `2` | Se ha optado por no depender de `tipoEnvio` para decidir qué registros son editables en RF-11; el criterio real usado es si la comunicación ya generó una acción de derivación (ticket u otro canal) en el propio sistema. Si en el futuro se necesita el significado real de `tipoEnvio` para otro fin, debe verificarse por separado (pruebas contra entorno SE, o consulta directa a la documentación completa de LEMA). |
| Alta como Gran Destinatario (Declaración Responsable + validación SE) no completada a tiempo | Bloquea todo el proyecto; es una dependencia externa con tiempos no controlados por el equipo. |
| Pérdida de anexo/acuse por fallo no detectado dentro de la ventana de 1 día | Pérdida irrecuperable de documentación legal — requiere monitorización activa, no solo reintentos. |
| Certificado X.509 caducado o revocado | Corte total de ingesta hasta renovación y posible re-registro. |
| Baja confianza sistemática de la IA en ciertos tipos de comunicación (p. ej. escritos judiciales con formato no estándar) | Sobrecarga de la cola de revisión humana (RF-08); mitigable con reglas complementarias al modelo. |
| El sistema de Ticketing solo permite cambiar el estado del ticket a nivel backend (sin campo de motivo en frontend) — confirmado con el equipo de Ticketing (Dani) | No es posible registrar explícitamente el motivo de un cambio de estado (p. ej. una reclasificación) dentro del propio ticket; el origen del cambio debe inferirse a partir del evento que lo generó (RF-07/RF-10), no de un campo de texto libre. Afecta a la trazabilidad de RF-09/RF-10 si en el futuro se requiere justificación explícita legible por un humano. |
| La API de "audiencia back" del sistema de Ticketing, necesaria para que la aplicación pueda invocar la modificación de un ticket existente (RF-11.4/RF-11.5), **aún no existe** — está siendo solicitada por otro compañero, sin fecha ni detalles confirmados | Bloquea el cierre técnico de RF-11 (modificación de notificaciones); el diseño de alto nivel (edición in-place vs. finalizar+crear) puede avanzar, pero su implementación y validación quedan condicionadas a que esta API se publique. |
