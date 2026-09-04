# TODO (retomar en próxima sesión)

## Contexto rápido
ER del sistema propio revisado a fondo y aprobado por la tutora de empresa en su primera revisión. Código ER completo y explicación detallada de cada tabla en `ER_Explicacion.md`. Descripciones textuales de casos de uso completadas en `Casos_de_Uso.md` (formato simplificado, provisional hasta validar diseño). Diagrama de clases combinado en `Diagrama_Clases.md` ya cubre las cinco rebanadas de flujo (RF-09, RF-08, RF-10, RF-11.4 y, desde esta sesión, RF-11.5 — finalizar+crear, diagramada como propuesta de alto nivel, no como diseño técnico cerrado) más una sección nueva de patrones de diseño aplicados (GoF/PoEAA: Strategy, Adapter, Gateway, Repository) con los descartados justificados (Observer, State, Factory Method). Buzón interno (RF-08.3) retirado del alcance activo del MVP en todos los documentos (ver más abajo).

## Próximos pasos
- [x] Diagramas de flujo (versión inicial) — ver `Diagramas_de_flujo.md`
- [x] Diagrama ER (versión inicial, revisado a fondo) — ver `ER_Explicacion.md`
- [x] Diagrama de arquitectura de componentes — ver `Diagrama_Componentes.md`
- [x] Descripciones textuales de casos de uso — ver `Casos_de_Uso.md`
- [x] Planificación / Gantt — ver `Gantt.md`
- [x] Diagrama de clases (rebanada RF-09: aceptar/reclasificar) — ver `Diagrama_Clases.md`, con capas modelo/aplicacion/infraestructura/api y análisis SOLID/GRASP
- [x] Extender el diagrama de clases a RF-08 (ejecución de derivación), RF-10 (Agente IA reclasificación) y RF-11.4 (modificación in-place) — mismo patrón de capas ya validado. Análisis SOLID/GRASP actualizado: OCP y Polymorphism, antes marcados como pendientes, quedan demostrados en RF-08 (`NotificacionGateway` + `NotificacionGatewayResolver`). Ver `Diagrama_Clases.md`.
- [x] RF-11.5 (cambio de departamento vía RF-11) ya diagramada — reutiliza literalmente el mecanismo finalizar+crear de RF-09.6/RF-10.4 (`TicketingGateway.finalizarTicket()` + `crearTicket()`). Se diagrama como **propuesta de alto nivel, no como diseño técnico cerrado**: sigue condicionada a confirmar la API "audiencia back" de Ticketing (ver `Especificacion_Requisitos.md`, sección 6). Ver `Diagrama_Clases.md`.
- [x] ~~Diagrama de secuencia RF-10~~ — **Evaluado y descartado**, motivo detallado en `Diagrama_Clases.md` (sección "Pendiente").
- [ ] **Revisar con alguien del equipo (no confirmado)**: la decisión de que sea `AgenteIAClient` (infraestructura), y no `AgenteReclasificacionService` (aplicación), quien invoque `TicketingGateway` vía MCP — es una lectura literal de RF-10.4 ("el agente invoca"), razonamiento propio de esta sesión, no algo cerrado con nadie
- [ ] Valorar extraer a un colaborador compartido el mecanismo "finalizar ticket original + crear uno nuevo en la cola correcta", que ahora se reconstruye por separado en tres sitios (RF-09.6, RF-10.4, RF-11.5) — deliberadamente no forzado todavía (YAGNI): son solo tres usos y RF-11.5 sigue sin cerrar como diseño técnico
- [x] ~~Inconsistencia pendiente entre documentos (buzón)~~ — **Resuelto**: `Especificacion_Requisitos.md` y `Diagrama_Componentes.md` actualizados para quitar el buzón del alcance activo, coherente con `Diagrama_Clases.md`. RF-08.3 se retira dejando hueco en la numeración (RF-08.1, RF-08.2, RF-08.4, RF-08.5) en vez de renumerar, porque RF-08.4/RF-08.5 ya están referenciados por número desde RF-09.6, RF-10.4, RF-11.5 y varios documentos del proyecto — ver nota de numeración en RF-08 de `Especificacion_Requisitos.md`. Si se retoma el buzón, se reincorpora como RF-08.3 y como tercer destino de `Ejecutor / Selector de Canal` en `Diagrama_Componentes.md`, sin afectar al resto.
- [ ] Diseño de interfaz RF-09.2 (documento como vista principal, texto extraído como panel auxiliar)
- [ ] Decidir alcance del diagnóstico sistemático de errores de clasificación
- [ ] Definir rol de "consulta" en `USUARIO` (departamento consultando sus propios tickets desde el frontal propio) — bloqueado hasta hablar con el responsable de IT/Seguridad (ver preguntas pendientes)
- [ ] Confirmar duración real de la subtarea "Infraestructura" en el Gantt (7 días laborables estimados, pendiente de fecha real de acceso — dependencia externa no controlada, igual que el alta como Gran Destinatario)
- [ ] Revisar fechas del Gantt contra los festivos configurados (ver `Gantt.md`) — varias tareas los cruzan y pueden desplazarse ligeramente al recalcular en la herramienta
- [ ] Revisar el diagrama de arquitectura de componentes (`Diagrama_Componentes.md`) a la luz de la conclusión sobre RF-10.1 (ver más abajo): la flecha `Ticketing → Bus de Eventos Corporativo → Agente IA` asume que Ticketing publica directamente al bus, pero el mecanismo real confirmado es un outbox en BD — falta identificar (y decidir cómo dibujar) el proceso intermedio que lee esa tabla y la convierte en el evento que sí llega al bus/n8n
- [ ] **Antes de hacer público el repo del TFM**: revisar que `Estado_Tecnico_Ticketing.md` (y cualquier fragmento de código real de Ticketing) no esté incluido — es información propietaria de MGS, no publicable sin autorización

## Preguntas pendientes para compañeros

**Para el responsable de Ticketing:**
- [ ] Capacidades reales de audiencia back (API de modificación de tickets, RF-11.4/11.5)
- [ ] Confirmar el campo `motivoResolucion` en `POST /tickets/{id}/estado` — ¿contradice lo de "sin motivo en frontend"?
- [x] ~~Mecanismo de detección de cancelación de ticket (RF-10.1)~~ — **Resuelto mediante análisis interno del sistema de Ticketing** (documento no publicable, información propietaria de MGS): no expone webhook saliente ni cola de mensajes; el cambio de estado de un ticket se propaga mediante un mecanismo de persistencia interno (patrón outbox), no una notificación de red directa. **Sigue pendiente confirmar con el responsable de Ticketing** quién consume esa información y cómo llega al ecosistema de eventos corporativo/n8n. No bloqueante para el diseño de alto nivel; si acaban implementando RF-10 antes de tener esta respuesta, la alternativa de fallback sería un proceso propio que haga poll directo a la fuente de persistencia interna.
- [ ] ¿El endpoint de creación de tickets permite deduplicar por un identificador externo (el `identificador` DEHú)? Relevante para RF-08.5: si una llamada tiene éxito en Ticketing pero el sistema falla antes de registrar la `DERIVACION` localmente, hace falta poder detectar el duplicado del lado de Ticketing, no solo consultando la base de datos propia.
- [ ] Condiciones exactas del canal email (RF-08.2) — **no bloqueante**: el director de empresa confirmó que el canal ticket es el prioritario y el email queda como deseable, no obligatorio para el cierre del MVP (ver `Especificacion_Requisitos.md`, nota de diseño en RF-08)
- [ ] Confirmar si el propio equipo de Ticketing tiene alguna preferencia/restricción sobre replicar su patrón de arquitectura (CDI, Repository/Gateway a medida) en un sistema nuevo, o si prefieren otra convención para el proyecto del TFM

**Para el responsable de IT/Seguridad:**
- [ ] Roles de acceso al frontal propio: ¿el Departamento necesita consultar sus propios tickets desde el frontal del sistema (no desde Ticketing directamente)? La tutora de empresa mencionó que sí — pendiente de que se confirme antes de diseñar el rol `consulta` y su alcance (¿limitado al propio departamento? ¿cómo se modela esa pertenencia en `USUARIO`?)

**Interno (implementación, no bloqueante para el diseño):**
- [ ] Al comprobar idempotencia (RF-08.5), la condición debe ser `DERIVACION.estado = 'exito'`, no solo "existe una fila" — una fila con `estado='fallo'` debe permitir reintento, no bloquearlo. **Reflejado ya en el diseño de clases de RF-08** (`Comunicacion.tieneDerivacionExitosa()`, distinto de `tieneDerivacionAsociada()`).

## Decisiones ya cerradas (no reabrir sin motivo)

*Resumen consolidado de todas las decisiones cerradas: ver `Estado_del_proyecto.md` §4 y §6 (esa es la fuente única). El detalle de diseño de clases (RF-08/RF-10/RF-11.4) vive en `Diagrama_Clases.md`; el de componentes en `Diagrama_Componentes.md`.*

Lo que sigue aquí es únicamente el historial de cambios del ER, que no está consolidado en ningún otro documento (el ER final sí lo está, en `ER_Explicacion.md`):

**Historial de revisión completa del ER (sesión anterior):**
- Campos `concepto`, `organismoEmisorCodigo`, `organismoEmisorNombre` añadidos a `COMUNICACION` — vienen de `localiza()`, no de `peticionAcceso()` (verificado contra la guía DEHú)
- `csvResguardo` añadido a `DOCUMENTO`
- `tipoAsignado` añadido a `CLASIFICACION` — cubre que RF-09.6 permite reclasificar departamento y/o tipo
- Tabla `EVENTO` eliminada del diseño: idempotencia cubierta por `DERIVACION`, trazabilidad por `CLASIFICACION`+`DERIVACION`, contador de reintentos de RF-10.2 derivado de `CLASIFICACION` (`COUNT(*) WHERE origen='ia_reclasificacion'`) — sin campo contador aparte
- Renombrado general de tablas (simplificación): `DOCUMENTO_ADJUNTO`→`DOCUMENTO`, `INTERPRETACION_IA`→`INTERPRETACION`, `REGISTRO_CLASIFICACION`→`CLASIFICACION`, `REVISION_PENDIENTE`→`REVISION`, `DEPARTAMENTO_DESTINO`→`DEPARTAMENTO`
- `ACCION_NOTIFICACION` renombrada a `DERIVACION` (no a `NOTIFICACION`) — evita colisión con "notificación" como tipo de envío oficial de DEHú (`tipoEnvio=1`)
- Valores de `CLASIFICACION.origen`: `ia_inicial`, `ia_reclasificacion`, `operador`
- `DEPARTAMENTO`: se descartó separar los canales en tabla aparte (`CANAL_DESTINO`) — el conjunto de canales es cerrado (ticket, email) y cada departamento tiene como máximo un destino por canal; estructura plana es suficiente. `emailDestino` no se modela en el ER (detalle de implementación, fase de desarrollo)
- Tabla `USUARIO` añadida — controla acceso al frontal propio (roles `operador`/`administrador`), con clave compartida con `PERSONA` (tabla externa de la empresa) en vez de duplicar datos de empleados
- `REVISION.usuario_id` (FK, nullable) — registra quién resolvió cada revisión pendiente; cubre tanto aceptar como reclasificar (RF-09.5/09.6).
- RF-11.2/11.3 modificados: el criterio editable/no-editable ya no depende de `tipoEnvio` de DEHú (correspondencia 1/2 = notificación/comunicación no verificada contra la guía — ningún ejemplo del Anexo I muestra el valor `1`), sino de si la comunicación ya tiene una `DERIVACION` asociada. Documento `Especificacion_Requisitos.md` actualizado en RF-11.2/11.3, nota técnica, descripción, criterio de aceptación, y sección 6 (riesgos)


---

Código ER completo y actualizado: ver `ER_Explicacion.md`.