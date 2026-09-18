# Estado del proyecto — TFM Automatización DEHú (MGS)

Resumen vivo. Pendientes y preguntas: `TODO.md`. Índice de documentos: `README.md`.

---

## 1. Contexto

Automatizar la recepción y tramitación de comunicaciones de la administración pública (DEHú/LEMA) en MGS Seguros: clasificar (metadatos LEMA primero; OCR + LLM solo si hace falta) y derivar al Ticketing interno.

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

Análisis y diseño del MVP cerrados en lo esencial (ER aprobado por la tutora de empresa), salvo RF-03 (cascada de clasificación) y la política de comparecencia, pendientes del resto de áreas. Introducción de la memoria cerrada (18/09/2026): `memoria/01_Introduccion.md`.

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
- RF-11.2/11.3: editable o no según si ya hay `DERIVACION`, no según `tipoEnvio` de DEHú.
- RF-03 (hipótesis de trabajo; requisitos y diagramas aún no reescritos): clasificar primero con metadatos de `localiza()` (organismo emisor + concepto). OCR/LLM del documento principal **solo si eso no basta**; anexo solo si el principal sigue ambiguo. El acuse se archiva (RF-02) y no entra al pipeline. Fiscal y DGS: con emisor + concepto cubren ~99 %; el anexo casi nunca sirve para clasificar. Confirmar con el resto de áreas (`TODO.md`). El contrato escrito sigue siendo «OCR siempre del principal».
- `TEXTOEXTRAIDO` tabla 1:1 opcional de `INTERPRETACION` (sugerencia de la tutora de empresa). Retención por parámetro global; el valor se fija tras contrastarlo con las áreas. Sin retención permanente para entrenamiento en el MVP.
- ER: tablas renombradas; `USUARIO` con clave compartida a `PERSONA`; sin tabla `EVENTO` (trazabilidad e idempotencia con `CLASIFICACION` + `DERIVACION`).
- Aprendizaje continuo del Agente IA y diagnóstico sistemático de errores: evolución futura, no diseño cerrado.
- **Plataforma:** Java EE 7 + WAS 9 + `javax.*` (tutora de empresa, sept. 2026). Desarrollo querría Jakarta EE + servidor más moderno; depende de Sistemas, sin fecha. HVOrganismosPublicos no nace en Jakarta. Detalle: `PRD.md`.

## 4. Arquitectura (resumen)

Detalle en `Diagrama_Clases.md` y `Diagrama_Componentes.md`.

- **Módulos Eclipse:** ver `PRD.md` §4 (familia Ticketing; sin DAO/EJB). Incluye `Test` (JUnit para el TFM) y `FT` (Playwright).
- **Stack alineado con Ticketing (cerrado):** Java 8, **Java EE 7** (`javax.*`), **WAS 9.0**, JDBC propio (sin JPA/Spring), CDI, JAX-RS. Jakarta EE descartado para este aplicativo hasta que Sistemas mueva de WAS 9. Módulos y CDI por constructor: `PRD.md`. Misma convención de estereotipos (`@Repositorio`, `@Servicio`, `@Endpoint`, `@Transaccional`), sin `AggregateRoot` ni eventos de dominio CDI (los cubre RF-07 a nivel corporativo).
- **Estilos:** orientada a eventos entre componentes (bus corporativo desacopla pipeline / ejecución / reclasificación); hexagonal por dentro de cada componente. Justificarlo explícitamente en la memoria, no dejarlo como efecto secundario de SOLID.
- **Frontal:** subconjunto CRUD + listado + detalle + un flujo de estado, sobre librería de terceros. No replicar el acabado visual de Ticketing (vive en librerías internas, fuera de plazo).
- **Componentes:** línea discontinua solo para lo no confirmado — modificar ticket (audiencia back) y evento de cancelación (proceso que lee el outbox).

## 5. Planificación

Ver `Gantt.md`. Cinco fases: Análisis y diseño → Desarrollo → Testing → Memoria → Revisión final.

Deadlines de negocio: backend/frontend antes del 25 dic; desarrollo y testing cierran el 31 dic; revisión final 4–15 ene 2027. Festivos pendientes de recalcular (ver `TODO.md`).

## 6. Memoria

Estructura definida en `Estructura_Memoria.md` (sin «Análisis de antecedentes»; «Estado de la cuestión» cubre soluciones existentes o análogas). Introducción cerrada. Siguiente capítulo de calendario: gestión del proyecto (desde el 21/09).
