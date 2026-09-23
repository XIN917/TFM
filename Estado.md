# Estado del proyecto — TFM Automatización DEHú (MGS)

Resumen vivo. Pendientes y preguntas: `TODO.md`. Índice de documentos: `README.md`.

---

## 1. Contexto

Automatizar la recepción y tramitación de comunicaciones de la administración pública (DEHú/LEMA) en MGS Seguros: clasificar (cascada de RF-03 en hipótesis, ver §3) y derivar al Ticketing interno.

Hoy el acceso es **manual** a **DEHú** (habitualmente por el enlace del correo de aviso). Mi Carpeta Ciudadana es otro portal, también puede abrir el mismo buzón. El proyecto sustituye esa consulta en pantalla por **LEMA** (Grandes Destinatarios): mismo buzón, servicios web. Seguridad Informática confirma: certificado de producción ya existe; el de pruebas lo genera Sistemas; contratación/custodia/renovación es de Seguridad.

| Rol | Función |
|---|---|
| Tutora de empresa | Revisa diseño (casos de uso, requisitos, UML) |
| Ponente | Supervisión académica |
| Manager de empresa | Directrices de negocio |
| Responsable de Ticketing | Sistema interno de tickets |

**Nombre:** `HVOrganismosPublicos`. `HV` es el prefijo del área de aplicativos de Consulta. `OrganismosPublicos` se eligió a propósito más amplio que Carpeta Ciudadana/DEHú: el sistema podría, más adelante, cubrir consultas a otros organismos públicos y avisar al departamento que deba gestionarlas. Es una dirección de naming, no un cambio de alcance: el MVP sigue siendo DEHú/LEMA (ver `Especificacion_Requisitos.md`, sección 5). Para la memoria: justificar el nombre en gestión del proyecto o en decisiones técnicas, dejando claro que no dilata el MVP.

**Infraestructura** (Soporte Datacenter / Altia, 10/09/2026): GIT `HV/HVOrganismosPublicos` (R/W Desarrollo); BBDD `HV_OrganismosPublicos` (DESA y PROD, SQLPortal, tamaño extra por documentos). Pendiente de grupos de acceso.

**Confidencialidad:** `Estado_Tecnico_Ticketing.md` y `Analisis_Frontend_AYTicketing.md` son internos de MGS. Fuera del repo; no republicar fragmentos de código real de Ticketing.

## 2. Dónde está el trabajo

Análisis y diseño del MVP cerrados en lo esencial (ER aprobado por la tutora de empresa), salvo la confirmación de la clasificación sin leer el documento y la política de comparecencia de las notificaciones, pendientes del resto de áreas. Requisitos y PRD ya clasifican sin leer el documento. Casos de uso, y algunas frases de RF-06.1 y RF-09.4, siguen el diseño anterior.

| Artefacto | Dónde |
|---|---|
| Requisitos RF-01–RF-11 | `Especificacion_Requisitos.md` |
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
- RF-11: edición in-place si no cambia el departamento; finalizar + crear si cambia.
- Reclasificación restringida a administrador. No se renombra «Aceptar clasificación propuesta». «Asignar a departamento» no replica la separación ticket/email de «Modificar»: es automático, no una decisión del actor.
- RF-10 cerrado a nivel de diseño: cancelación → Agente IA (umbral) → autocorrección MCP o escalado humano. MCP acotado a esa acción. Ticketing no expone webhook: el cambio de estado sale por outbox interno. Falta quién lo consume (ver `TODO.md`).
- `tipoEnvio` (22/09/2026): `1` comunicación, `2` notificación. Plan de pruebas funcionales para Gran Destinatario v2.0 (SGAD, 18/03/2024), §2.1.2 y §2.1.3. `vinculo`: `1` titular, `2` destinatario. RF-11.2/11.3 sigue siendo editable o no según si ya hay `DERIVACION`, no según `tipoEnvio`.
- Descarga según `tipoEnvio` (23/09/2026, FAQ DEHú; la leyenda 1/2 no está en la guía de integración, sí en el plan de pruebas GD v2.0): la comunicación (`1`) no tiene plazo de lectura ni efectos jurídicos por acceder a ella, y DEHú no genera acuse. Se descarga en el mismo ciclo que `localiza()`. La notificación (`2`) sí: `peticionAcceso()` es la comparecencia y puede abrir el plazo de alegaciones o recursos. Ese plazo lo fija el organismo emisor (puesta a disposición → caducidad) y lo gestiona el PUC, no DEHú. Los diez días naturales son el rechazo tácito de la Ley 39/2015, art. 43.2, no una cifra de la FAQ. Cuándo comparecer la notificación sigue pendiente de las áreas (`TODO.md`).
- Diagrama de flujo (23/09/2026). Seis flujos: 1 sondeo, 2 autocorrección, 3 comparecencia, 4 operador, 5 consulta local, 6 reconciliación. En el sondeo los dos tipos se clasifican antes de abrir, con organismo emisor y concepto de `localiza()`. Si la confianza no basta, cola de revisión y no se abre; no hay segunda clasificación. Comunicación ya clasificada: `peticionAcceso()` en el mismo ciclo, anexos solo si la respuesta trae `anexosReferencia`, sin `consultaAcusePdf()`, extraer y resumir el principal, ticket del departamento ya asignado. Notificación: pendiente de comparecencia, sin descarga ni ticket. El flujo 2 (cancelar, contador ≤ 2, finalizar + crear) no se tocó; la idea de reasignar por comentario está solo en `TODO.md`.
- Comparecencia en ese diagrama (hipótesis; las áreas no lo han cerrado): no se comparece en el sondeo. Flujo propio en el frontal HV. Actor: responsable de área, no el operador ni el ticket de Ticketing. La lista muestra datos de `localiza()` y el día 10 calculado desde `fechaPuestaDisposicion`. El botón de confirmar es la comparecencia. No hay resumen del documento: el resumen solo acompaña a la descarga inmediata, y esta no lo es hasta que lo confirmen las áreas. El ticket del departamento ya asignado se genera sin ese resumen. No se muestra el vencimiento del correo de cortesía: no está en los ejemplos de `localiza()` ni de `peticionAcceso()`. Eager (batch) frente a este botón sigue en `TODO.md`.
- Clasificación sin leer el documento (23/09/2026, escrita en RF-05 y en el PRD; pendiente de confirmar con el resto de departamentos): organismo emisor y concepto de `localiza()`, antes de abrir. Si la confianza no basta, revisión y no se abre. El resumen solo se hace en la descarga inmediata: la comunicación, en el mismo ciclo. La notificación no se resume mientras la descarga no sea inmediata (pendiente de confirmar con el resto de departamentos). El anexo no entra a clasificar. Fiscal y DGS: emisor + concepto cubren ~99 %.
- Acceso LEMA (23/09/2026, guía de integración v3.0 §2.2 y §5; especificación del servicio web, copia pública de la v1.7): no hay token, OAuth ni usuario y contraseña. La llamada va firmada con certificado X.509 (WS-Security; el `BinarySecurityToken` es ese certificado). Cl@ve es solo del portal. En preproducción el certificado puede ser autofirmado y el alta es distinta de la de producción; los envíos de prueba se piden al Centro de Servicios («Creación de envíos para LEMA en pruebas»). Endpoints: pruebas `se-gd-dehuws.redsara.es`, producción `gd-dehuws.redsara.es`. El de producción ya existe; el de pruebas lo genera Sistemas.
- `TEXTOEXTRAIDO` tabla 1:1 opcional de `INTERPRETACION` (sugerencia de la tutora de empresa). Retención por parámetro global; el valor se fija tras contrastarlo con las áreas. Sin retención permanente para entrenamiento en el MVP.
- ER: tablas renombradas; `USUARIO` con clave compartida a `PERSONA`; sin tabla `EVENTO` (trazabilidad e idempotencia con `CLASIFICACION` + `DERIVACION`).
- `CLASIFICACION` (22/09/2026): una fila por clasificación. `origen` `ia` | `operador`; `modelo` si hubo LLM; `nIntentos` solo en filas `ia` (el tope de reclasificaciones cuenta `nIntentos > 1`); `usuario_id` solo en filas `operador`. `departamentoAsignado` obligatorio. No hay catálogo de tipos: `tipoAsignado` y `tipoDetectado` no se rellenan en el MVP. La categorización es el departamento.
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