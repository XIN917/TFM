# TODO (retomar en próxima sesión)

## Contexto rápido
ER del sistema propio revisado a fondo y enviado a Judit para su primera revisión (pendiente de respuesta). Código ER completo y explicación detallada de cada tabla en `ER_Explicacion.md`. Descripciones textuales de casos de uso completadas en `Casos_de_Uso.md` (formato simplificado, provisional hasta validar diseño).

## Próximos pasos
- [x] Diagramas de flujo (versión inicial) — ver `Diagramas_de_flujo.md`
- [x] Diagrama ER (versión inicial, revisado a fondo) — ver `ER_Explicacion.md`
- [ ] Diagrama de arquitectura de componentes (siguiente pieza — la dependencia estricta entre llamadas LEMA, documentada en `DEHu_Campos_Respuesta_Servicios.md`, condiciona el orden de los pasos)
- [x] Descripciones textuales de casos de uso — ver `Casos_de_Uso.md`
- [x] Planificación / Gantt — ver `Gantt.md`
- [x] Diagrama de clases (rebanada RF-09: aceptar/reclasificar) — ver `Diagrama_Clases.md`, con capas modelo/aplicacion/infraestructura/api y análisis SOLID/GRASP
- [ ] Extender el diagrama de clases a RF-08 (ejecución de derivación), RF-10 (Agente IA reclasificación) y RF-11.4 (modificación in-place) — mismo patrón de capas ya validado
- [ ] Diagrama de secuencia RF-10 (evaluar si hace falta)
- [ ] Diseño de interfaz RF-09.2 (documento como vista principal, texto extraído como panel auxiliar)
- [ ] Decidir alcance del diagnóstico sistemático de errores de clasificación
- [ ] Definir rol de "consulta" en `USUARIO` (departamento consultando sus propios tickets desde el frontal propio) — bloqueado hasta hablar con Silvia (ver preguntas pendientes)
- [ ] Confirmar duración real de la subtarea "Infraestructura" en el Gantt (7 días laborables estimados, pendiente de fecha real de acceso — dependencia externa no controlada, igual que el alta como Gran Destinatario)
- [ ] Revisar fechas del Gantt contra los festivos configurados (11 sep, 24 sep, 12 oct, 8 dic, 25 dic, 1 ene, 6 ene) — varias tareas los cruzan y pueden desplazarse ligeramente al recalcular en la herramienta
- [ ] **Antes de hacer público el repo del TFM**: revisar que `Estado_Tecnico_Ticketing.md` (y cualquier fragmento de código real de Ticketing) no esté incluido — es información propietaria de MGS, no publicable sin autorización

## Preguntas pendientes para compañeros

**Para Dani:**
- [ ] Capacidades reales de audiencia back (API de modificación de tickets, RF-11.4/11.5)
- [ ] Confirmar el campo `motivoResolucion` en `POST /tickets/{id}/estado` — ¿contradice lo de "sin motivo en frontend"?
- [ ] Mecanismo de detección de cancelación de ticket (RF-10.1): ¿Ticketing publica un evento accesible externamente (ecosistema de eventos de la empresa / n8n), ofrece webhook, o hay que hacer polling? Dani mencionó que se publica un evento — falta confirmar dónde y si es suscribible. **No es bloqueante ahora** (aún no empieza desarrollo); confirmar cuando se aborde la implementación de RF-10.
- [ ] ¿El endpoint de creación de tickets permite deduplicar por un identificador externo (el `identificador` DEHú)? Relevante para RF-08.5: si una llamada tiene éxito en Ticketing pero el sistema falla antes de registrar la `DERIVACION` localmente, hace falta poder detectar el duplicado del lado de Ticketing, no solo consultando la base de datos propia.
- [ ] Condiciones exactas del canal email (RF-08.2)
- [ ] Confirmar si el propio equipo de Ticketing tiene alguna preferencia/restricción sobre replicar su patrón de arquitectura (CDI, Repository/Gateway a medida) en un sistema nuevo, o si prefieren otra convención para el proyecto del TFM

**Para Silvia:**
- [ ] Roles de acceso al frontal propio: ¿el Departamento necesita consultar sus propios tickets desde el frontal del sistema (no desde Ticketing directamente)? Judit mencionó que sí — pendiente de que Silvia lo confirme antes de diseñar el rol `consulta` y su alcance (¿limitado al propio departamento? ¿cómo se modela esa pertenencia en `USUARIO`?)

**Interno (implementación, no bloqueante para el diseño):**
- [ ] Al comprobar idempotencia (RF-08.5), la condición debe ser `DERIVACION.estado = 'exito'`, no solo "existe una fila" — una fila con `estado='fallo'` debe permitir reintento, no bloquearlo.

## Decisiones ya cerradas (no reabrir sin motivo)

**De esta sesión (arquitectura / diagrama de clases):**
- Stack técnico confirmado vía análisis real del repo de Ticketing: Java 8, Java EE 7/8 (`javax.*`, no Jakarta EE), sin JPA/Spring, CDI + JAX-RS, WebSphere traditional 9.0. Ver `Estado_Tecnico_Ticketing.md` (⚠️ no publicar, es propietario de MGS).
- Capas del diagrama de clases: `modelo` (dominio + interfaces Repository/Gateway) / `aplicacion` (Services) / `infraestructura` (implementaciones + clientes SOAP/REST) / `api` (Controllers JAX-RS) — mismo patrón que ya usa Ticketing internamente (Repository a medida, no JPA; Service envolviendo Repository; estereotipos CDI propios).
- Sin `<<AggregateRoot>>` formal ni eventos de dominio CDI en el diseño propio — evitar duplicar lo que RF-07 ya cubre a nivel de arquitectura de eventos corporativa. Decisión consciente de simplicidad (KISS/YAGNI) frente al DDD táctico completo de Ticketing.
- `TicketingGateway`/`LemaGateway` añadidas como interfaces de dominio (principio DIP) — los Services de `aplicacion` nunca dependen directamente de `TicketingClient`/`LemaClient` (clases concretas de `infraestructura`).
- Rebanada RF-09 (aceptar/reclasificar) es la única diagramada por ahora, a propósito — patrón por validar antes de replicar a RF-08/RF-10/RF-11.4.

**De sesiones anteriores:**
- RF-11.4/11.5 confirmado con OpenAPI real: PATCH /tickets no permite cambiar cola → edición in-place solo si no cambia departamento
- `ticketRelacionado` (campo nativo) para enlazar tickets en reclasificación
- Estados de `COMUNICACION.estado`: pendiente → en_proceso → en_revision / procesada
- Aprendizaje continuo del Agente IA: dirección futura, ya en RF-09 de la especificación

**De esta sesión:**
- `COMUNICACION`–`REVISION` cambiada de 1:1 a 1:N (feedback de Judit tras su revisión del ER): una comunicación puede escalar a revisión más de una vez (p. ej. si tras una reclasificación la nueva propuesta vuelve a caer bajo el umbral). Cada escalada genera una fila nueva; las anteriores quedan cerradas (`resuelto = true`). Para localizar la revisión activa hay que filtrar por `resuelto = false`, no asumir fila única. `ER_Explicacion.md` actualizado (diagrama, resumen de relaciones, sección `REVISION`).

**De sesión anterior (revisión completa del ER):**
- Campos `concepto`, `organismoEmisorCodigo`, `organismoEmisorNombre` añadidos a `COMUNICACION` — vienen de `localiza()`, no de `peticionAcceso()` (verificado contra la guía DEHú)
- `csvResguardo` añadido a `DOCUMENTO` (antes `DOCUMENTO_ADJUNTO`)
- `tipoAsignado` añadido a `CLASIFICACION` (antes `REGISTRO_CLASIFICACION`) — cubre que RF-09.6 permite reclasificar departamento y/o tipo
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
