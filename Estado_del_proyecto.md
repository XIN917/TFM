# Estado del proyecto — TFM Automatización DEHú (MGS)

*Resumen de contexto y decisiones cerradas. Para tareas pendientes y preguntas abiertas, ver `TODO.md`.*

---

## 1. Contexto del proyecto

- **Proyecto**: automatización de la recepción y procesamiento de comunicaciones de la administración pública (DEHú/LEMA) para MGS Seguros, con IA (OCR + LLM) para interpretar, clasificar y generar tickets/notificaciones, integrado con el sistema interno de Ticketing.
- **Judit — tutora de empresa (mentora en MGS Seguros)**: revisa el diseño (casos de uso, requisitos, diagramas UML).
- **Marc — ponente de la universidad**: supervisa el TFM desde el lado académico.
- **JM (manager de empresa)**: directrices funcionales/negocio.
- **Dani**: gestiona el sistema de Ticketing internamente.

## 1.1 Aclaración: Mi Carpeta Ciudadana vs. LEMA

Actualmente los usuarios de MGS acceden **manualmente** a las notificaciones a través de la interfaz web de **Mi Carpeta Ciudadana** (la vía de acceso para persona física/jurídica dentro del Punto Único DEHú). El proyecto automatiza ese acceso manual sustituyéndolo por los **servicios web LEMA** (la vía para Grandes Destinatarios del mismo Punto Único). No son sistemas distintos: son dos formas de acceso al mismo DEHú. Todo el diseño ya realizado (especificación, diagramas, casos de uso, certificado solicitado a Silvia) sigue siendo correcto y aplica sobre LEMA.

## 2. Documentos generados hasta ahora

| Archivo | Contenido |
|---|---|
| `Propuesta_de_proyecto.md` | Propuesta original del proyecto |
| `Especificacion_Requisitos.md` | RF-01 a RF-11 + RNF |
| `casos_de_uso_final.png` | Diagrama de casos de uso, 3 actores (DEHú/LEMA, Usuario/Operador, Departamento) |
| `Casos_de_Uso.md` | Descripciones textuales de cada caso de uso (formato simplificado), más el diagrama en código PlantUML |
| `Diagramas_de_flujo.md` | Flujo principal (RF-01–RF-09) y flujo de reconciliación, separados |
| `TODO.md` | Traspaso entre sesiones — checklist, preguntas pendientes, código ER |
| `ER_Explicacion.md` | Explicación detallada de cada tabla y relación del ER |
| `DEHu_Campos_Respuesta_Servicios.md` | Campos de petición y respuesta de los 6 servicios DEHú/LEMA, con dependencias entre llamadas |
| `estructura_memoria_TFM.md` | Estructura orientativa de la memoria del TFM (aportada por la universidad) |
| `Gantt.md` | Planificación completa: código Mermaid del diagrama de Gantt + tabla de períodos, lista para la memoria |
| `Diagrama_Clases.md` | Diagrama de clases en capas (`modelo`/`aplicacion`/`infraestructura`/`api`), código PlantUML, y análisis de cumplimiento SOLID/GRASP |
| `Estado_Tecnico_Ticketing.md` | ⚠️ Análisis de arquitectura del proyecto interno de Ticketing (MGS) — información propietaria de la empresa, **no incluir si el repo se hace público** |

## 3. Hechos técnicos clave (guía LEMA)

- Protocolo: SOAP 1.1 + WS-Security con certificado X.509.
- Dos familias de servicios: LEMA (`localiza`, `peticionAcceso`, `consultaAnexos`, `consultaAcusePdf`) y ConsultaRealizadas (`localizaRealizadas`, `consultaRealizadas`).
- Restricción de 1 día: `consultaAnexos()` y `consultaAcusePdf()` no acceden a elementos con más de 1 día de antigüedad.
- Anexos por URL directa y por referencia — ambos se descargan siempre.
- Paginación soportada; sin filtro por fecha en la petición (filtrado del lado de la aplicación).
- Sin canal de envío/respuesta hacia la administración — fuera de alcance del MVP.

## 4. Decisiones de diseño cerradas

- Actores del caso de uso: solo entidades externas (DEHú/LEMA, Usuario/Operador, Departamento).
- Certificado electrónico: arquitectura/autenticación, no caso de uso.
- Ticket como canal principal; email como alternativa configurable (RF-08.2).
- Reclasificación: el Ticketing no soporta transferir de cola → finalizar ticket + crear uno nuevo (RF-08.5). Confirmado con el OpenAPI real: `PATCH /tickets/{id}` no permite modificar `cola`.
- RF-11: edición in-place si no cambia el departamento; finalizar+crear si cambia. Se usa el campo nativo `ticketRelacionado` de la API para enlazar tickets.
- "Asignar a departamento" no requiere la misma separación ticket/email que "Modificar" (es un mecanismo automático, no una decisión activa del actor).
- "Aceptar clasificación propuesta": no se renombra.
- Reclasificación restringida a rol de administrador: confirmado que aplica.
- RF-10 (Agente IA de reclasificación automática): diseñado y cerrado — evento de cancelación → Agente IA evalúa con umbral de confianza → autocorrección vía MCP o escalado a revisión humana. MCP acotado a esta única acción.
- Aprendizaje continuo del Agente IA a partir del feedback humano: dirección de evolución del proyecto, ya anotada en RF-09; mecanismo técnico por definir.
- Diagnóstico sistemático de errores de clasificación: idea a explorar; datos ya contemplados en el ER (`CLASIFICACION`, `INTERPRETACION`).
- RF-11.2/11.3: la distinción editable/no-editable ya no depende de `tipoEnvio` de DEHú (correspondencia 1/2 = notificación/comunicación no verificada contra la guía LEMA), sino de si la comunicación ya tiene una acción de derivación (`DERIVACION`) asociada.
- ER revisado a fondo esta sesión — renombrado general de tablas, tabla `USUARIO` añadida con clave compartida a `PERSONA`; no se incluye tabla de eventos, la trazabilidad e idempotencia quedan cubiertas por `CLASIFICACION` y `DERIVACION`. Enviado a Judit para primera revisión, pendiente de respuesta.

## 5. Planificación (Gantt)

Planificación completa cerrada — ver `Gantt.md` (código Mermaid + períodos).

**5 fases:** Análisis y Diseño → Desarrollo → Testing → Memoria → Revisión Final (esta última al final, ya que incluye la revisión de la propia memoria una vez escrita).

**Desarrollo:** Infraestructura → Pipeline → Motor IA (OCR + clasificación) → Integración con Ticketing → Gestión de derivaciones y consultas (Operador) [Backend + Frontend en paralelo] → Despliegue continuo a producción (en paralelo, desde que el Pipeline está listo).

**Testing:** Pruebas en entorno SE + Validación en entorno PRO (LEMA) — sin subtarea intermedia de "integración e IA" (redundante).

**Memoria (6 bloques, mapeados contra `estructura_memoria_TFM.md`):** Introducción → Gestión del proyecto y planificación → Análisis → Desarrollo → Evaluación → Conclusiones.

**Deadlines de negocio:** Backend/Frontend antes del 25 dic; Desarrollo y Testing cierran ambos el 31 dic; Revisión Final en enero 2027 (4–15 ene).

**Calendario:** semana laboral lunes-viernes, con 7 festivos configurados (11 sep, 24 sep, 12 oct, 8 dic, 25 dic, 1 ene, 6 ene) — pendiente de recalcular fechas de tareas que los cruzan.

## 6. Arquitectura de software — capas y patrones

- **Stack técnico confirmado** (vía análisis del repositorio real de Ticketing, `Estado_Tecnico_Ticketing.md`): Java 8, **Java EE 7/8** (namespace `javax.*`, no Jakarta EE), IBM WebSphere Application Server traditional 9.0. Sin JPA/Hibernate — persistencia por JDBC puro con framework propio (`JdbcTemplate`/`RowMapper`). Sin Spring — inyección de dependencias con **CDI** (`@Inject`, `@Produces`). Endpoints con **JAX-RS**.
- **Decisión de diseño**: adoptar el mismo patrón arquitectónico ya en producción en Ticketing, para consistencia y mantenibilidad — Repository (interfaz de dominio + implementación de infraestructura), Service de aplicación, Controller JAX-RS, con los mismos estereotipos CDI que usa Ticketing (`@Repositorio`, `@Servicio`, `@Endpoint`, `@Transaccional`).
- **Versión simplificada respecto al DDD táctico completo de Ticketing**: sin `<<AggregateRoot>>` formal ni eventos de dominio CDI — el rol de estos últimos ya lo cubre RF-07 (evento de comunicación clasificada) a nivel de arquitectura de eventos corporativa, así que añadirlos aquí sería redundante. Decisión tomada explícitamente para evitar sobre-ingeniería dado el alcance y calendario del TFM.
- **Diagrama de clases en capas** (`modelo`/`aplicacion`/`infraestructura`/`api`) diseñado como rebanada vertical del flujo RF-09 (aceptar/reclasificar) — ver `Diagrama_Clases.md`. Pendiente extender el mismo patrón a RF-08 (ejecución de derivación), RF-10 (Agente IA) y RF-11.4 (modificación in-place).
- **Corrección DIP aplicada**: `TicketingGateway` y `LemaGateway` añadidas como interfaces de dominio (junto a `ComunicacionRepository`), implementadas por `TicketingClient`/`LemaClient` en infraestructura — antes los Services dependían directamente de las clases concretas, inconsistente con el tratamiento ya dado al Repository.
- **Análisis de cumplimiento SOLID/GRASP documentado** en `Diagrama_Clases.md`, con ejemplos concretos del propio dominio (`Revision.resolver()` como Information Expert, `TicketingGateway` como DIP, etc.). OCP y Polymorphism marcados explícitamente como pendientes — no hay caso real que los demuestre en la rebanada RF-09; aplicarán al diseñar RF-08.4 (selección de canal de notificación).
- **Nota de confidencialidad**: `Estado_Tecnico_Ticketing.md` contiene fragmentos de código real y detalles de arquitectura del sistema de Ticketing de MGS — información propietaria de la empresa. No debe publicarse si el repositorio del TFM se hace público (ver `TODO.md`).
