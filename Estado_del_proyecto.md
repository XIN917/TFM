# Estado del proyecto — TFM Automatización DEHú (MGS)

*Resumen de contexto y decisiones cerradas. Para tareas pendientes y preguntas abiertas, ver `TODO.md`.*

---

## 1. Contexto del proyecto

- **Proyecto**: automatización de la recepción y procesamiento de comunicaciones de la administración pública (DEHú/LEMA) para MGS Seguros, con IA (OCR + LLM) para interpretar, clasificar y generar tickets/notificaciones, integrado con el sistema interno de Ticketing.
- **Tutora de empresa (mentora en MGS Seguros)**: revisa el diseño (casos de uso, requisitos, diagramas UML).
- **Ponente de la universidad**: supervisa el TFM desde el lado académico.
- **Manager de empresa**: directrices funcionales/negocio.
- **Responsable de Ticketing**: gestiona el sistema de Ticketing internamente.

## 1.1 Aclaración: Mi Carpeta Ciudadana vs. LEMA

Actualmente los usuarios de MGS acceden **manualmente** a las notificaciones a través de la interfaz web de **Mi Carpeta Ciudadana** (la vía de acceso para persona física/jurídica dentro del Punto Único DEHú). El proyecto automatiza ese acceso manual sustituyéndolo por los **servicios web LEMA** (la vía para Grandes Destinatarios del mismo Punto Único). No son sistemas distintos: son dos formas de acceso al mismo DEHú. Todo el diseño ya realizado (especificación, diagramas, casos de uso, certificado solicitado al responsable de IT/Seguridad) sigue siendo correcto y aplica sobre LEMA.

## 2. Documentos del proyecto

Índice completo y descripción de cada documento: ver [`README.md`](README.md).

⚠️ `Estado_Tecnico_Ticketing.md` (análisis de arquitectura del sistema interno de Ticketing de MGS) y `Analisis_Frontend_AYTicketing.md` (análisis del frontend de ese mismo sistema) son información propietaria de la empresa y **no deben incluirse** ahora que el repositorio del TFM es público — no aparecen en el índice del README por ese motivo; se guardan fuera del repo, junto al resto de notas internas.

## 3. Hechos técnicos clave (guía LEMA)

Ver `Especificacion_Requisitos.md`, sección 0 (Notas de anclaje técnico) — protocolo SOAP/WS-Security, familias de servicios, restricción de 1 día, límite de peticiones, onboarding.

## 4. Decisiones de diseño cerradas

- Actores del caso de uso: solo entidades externas (DEHú/LEMA, Usuario/Operador, Departamento).
- Certificado electrónico: arquitectura/autenticación, no caso de uso.
- Ticket como único canal obligatorio del MVP; email (RF-08.2) confirmado por el director de empresa como deseable, no bloqueante para el cierre del MVP — independientemente de si el ticket es alcanzable o no en un caso concreto. Ver `Especificacion_Requisitos.md`, nota de diseño en RF-08.
- Reclasificación: el Ticketing no soporta transferir de cola → finalizar ticket + crear uno nuevo (RF-08.5). Confirmado con el OpenAPI real: `PATCH /tickets/{id}` no permite modificar `cola`.
- RF-11: edición in-place si no cambia el departamento; finalizar+crear si cambia. Se usa el campo nativo `ticketRelacionado` de la API para enlazar tickets.
- "Asignar a departamento" no requiere la misma separación ticket/email que "Modificar" (es un mecanismo automático, no una decisión activa del actor).
- "Aceptar clasificación propuesta": no se renombra.
- Reclasificación restringida a rol de administrador: confirmado que aplica.
- RF-10 (Agente IA de reclasificación automática): diseñado y cerrado — evento de cancelación → Agente IA evalúa con umbral de confianza → autocorrección vía MCP o escalado a revisión humana. MCP acotado a esta única acción.
- **RF-10.1 (mecanismo de detección de cancelación de ticket) resuelto mediante análisis interno del sistema de Ticketing** (documento no publicable, información propietaria de MGS, ver `Estado_Tecnico_Ticketing.md`): Ticketing no expone webhook saliente ni cola de mensajes; el cambio de estado de un ticket se propaga mediante un mecanismo de persistencia interno (patrón outbox), no una notificación de red directa. Sigue pendiente confirmar con el responsable de Ticketing quién consume esa información y cómo llega al ecosistema de eventos corporativo/n8n — no bloqueante para el diseño de alto nivel (ver `TODO.md`).
- Aprendizaje continuo del Agente IA a partir del feedback humano: dirección de evolución del proyecto, ya anotada en RF-09; mecanismo técnico por definir.
- Diagnóstico sistemático de errores de clasificación: idea a explorar; datos ya contemplados en el ER (`CLASIFICACION`, `INTERPRETACION`).
- RF-11.2/11.3: la distinción editable/no-editable ya no depende de `tipoEnvio` de DEHú (correspondencia 1/2 = notificación/comunicación no verificada contra la guía LEMA), sino de si la comunicación ya tiene una acción de derivación (`DERIVACION`) asociada.
- ER revisado a fondo esta sesión — renombrado general de tablas, tabla `USUARIO` añadida con clave compartida a `PERSONA`; no se incluye tabla de eventos, la trazabilidad e idempotencia quedan cubiertas por `CLASIFICACION` y `DERIVACION`. Enviado a la tutora de empresa para primera revisión — aprobado.

## 5. Planificación (Gantt)

Planificación completa cerrada — ver `Gantt.md` (código Mermaid + períodos).

**5 fases:** Análisis y Diseño → Desarrollo → Testing → Memoria → Revisión Final (esta última al final, ya que incluye la revisión de la propia memoria una vez escrita).

**Desarrollo:** Infraestructura → Pipeline → Motor IA (OCR + clasificación) → Integración con Ticketing → Gestión de derivaciones y consultas (Operador) [Backend + Frontend en paralelo] → Despliegue continuo a producción (en paralelo, desde que el Pipeline está listo).

**Testing:** Pruebas en entorno SE + Validación en entorno PRO (LEMA) — sin subtarea intermedia de "integración e IA" (redundante).

**Memoria (6 bloques, mapeados contra `estructura_memoria_TFM.md`):** Introducción → Gestión del proyecto y planificación → Análisis → Desarrollo → Evaluación → Conclusiones.

**Deadlines de negocio:** Backend/Frontend antes del 25 dic; Desarrollo y Testing cierran ambos el 31 dic; Revisión Final en enero 2027 (4–15 ene).

**Calendario:** semana laboral lunes-viernes, festivos configurados en `Gantt.md` — pendiente de recalcular fechas de tareas que los cruzan.

## 6. Arquitectura de software — capas, patrones y componentes

- **Stack técnico confirmado** (vía análisis del repositorio real de Ticketing, `Estado_Tecnico_Ticketing.md`): Java 8, **Java EE 7/8** (namespace `javax.*`, no Jakarta EE), IBM WebSphere Application Server traditional 9.0. Sin JPA/Hibernate — persistencia por JDBC puro con framework propio (`JdbcTemplate`/`RowMapper`). Sin Spring — inyección de dependencias con **CDI** (`@Inject`, `@Produces`). Endpoints con **JAX-RS**.
- **Decisión de diseño**: adoptar el mismo patrón arquitectónico ya en producción en Ticketing, para consistencia y mantenibilidad — Repository (interfaz de dominio + implementación de infraestructura), Service de aplicación, Controller JAX-RS, con los mismos estereotipos CDI que usa Ticketing (`@Repositorio`, `@Servicio`, `@Endpoint`, `@Transaccional`).
- **Versión simplificada respecto al DDD táctico completo de Ticketing**: sin `<<AggregateRoot>>` formal ni eventos de dominio CDI — el rol de estos últimos ya lo cubre RF-07 (evento de comunicación clasificada) a nivel de arquitectura de eventos corporativa, así que añadirlos aquí sería redundante. Decisión tomada explícitamente para evitar sobre-ingeniería dado el alcance y calendario del TFM.
- **Diagrama de clases en capas** (`modelo`/`aplicacion`/`infraestructura`/`api`) — ya extendido a cinco rebanadas verticales: RF-09 (aceptar/reclasificar, cerrada con revisión previa), RF-08 (ejecución de la acción resultante), RF-10 (reclasificación automática vía Agente IA), RF-11.4 (modificación in-place) y RF-11.5 (cambio de departamento, finalizar+crear) — ver `Diagrama_Clases.md`. RF-11.5 se diagrama como propuesta de alto nivel, no como diseño técnico cerrado, condicionada a la API "audiencia back" de Ticketing. El diagrama incluye también una sección de patrones de diseño aplicados (GoF/PoEAA: Strategy, Adapter, Gateway, Repository).
- **Corrección DIP aplicada**: `TicketingGateway` y `LemaGateway` añadidas como interfaces de dominio (junto a `ComunicacionRepository`), implementadas por `TicketingClient`/`LemaClient` en infraestructura — antes los Services dependían directamente de las clases concretas, inconsistente con el tratamiento ya dado al Repository.
- **Análisis de cumplimiento SOLID/GRASP actualizado y consolidado** en `Diagrama_Clases.md`, con ejemplos concretos del propio dominio en las cuatro rebanadas. OCP y Polymorphism, marcados como pendientes tras la rebanada RF-09 (sin caso real que los demostrara), quedan **demostrados en la rebanada RF-08**: `NotificacionGateway` (interfaz común para los canales soportados — ticket y email) con selección de implementación en tiempo de ejecución vía `NotificacionGatewayResolver` (GRASP Pure Fabrication) — añadir un canal de notificación nuevo no requiere tocar el Service ni el Resolver.
- **Diagrama de arquitectura de componentes cerrado** (`Diagrama_Componentes.md`, código PlantUML) — vista de sistema completo en cuatro bloques: Pipeline de Ingesta y Clasificación (RF-01–RF-07, orquestado por n8n), Ejecución de Acciones (RF-08), Reclasificación Automática (RF-10, Agente IA), y Gestión y Revisión (RF-09/RF-11, frontal + API REST + BD). Decisión clave reflejada: el desacople Pipeline/Ejecución vía el Bus de Eventos Corporativo es un principio de diseño explícito (RNF escalabilidad), no un detalle incidental. El diagrama usa línea discontinua exclusivamente para las dos dependencias aún no confirmadas del lado externo: la modificación de tickets desde la API (RF-11.4/11.5, pendiente de la API "audiencia back" de Ticketing) y el evento de cancelación de Ticketing hacia el Bus de Eventos Corporativo (RF-10.1, pendiente de identificar el proceso intermedio que lee el outbox de Ticketing — ver punto anterior).
- **Nota de confidencialidad**: `Estado_Tecnico_Ticketing.md` contiene fragmentos de código real y detalles de arquitectura del sistema de Ticketing de MGS — información propietaria de la empresa. No debe publicarse si el repositorio del TFM se hace público (ver `TODO.md`). El repositorio del TFM ya es público; se revisó que `TODO.md` no contuviera identificadores propios del código de Ticketing (nombres de clases/tablas reales) y se sanitizó la entrada correspondiente a RF-10.1 dejando solo la conclusión general.
- **Análisis del frontend real de Ticketing** (documento no publicable, información propietaria de MGS, ver `Analisis_Frontend_AYTicketing.md`, fuera del repo): SPA moderna con TypeScript, servida como estáticos dentro del mismo WAR/EAR de WebSphere; el backend Java solo autentica y sirve ficheros. La mayor parte del acabado visual y funcional avanzado (listados con múltiples vistas, filtros con plantillas guardadas, panel contextual, sistema de diseño y accesibilidad) no vive en esa app, sino en librerías de componentes internas de la compañía — no es replicable en el plazo del TFM y no debe usarse como referencia de esfuerzo. **Decisión**: el frontal del TFM apunta a un subconjunto funcional (CRUD + listado + detalle + un flujo de estado tipo workflow) sobre una librería de componentes de terceros ya existente, no a igualar el acabado visual completo de Ticketing.