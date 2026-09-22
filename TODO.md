# TODO

Tareas abiertas y preguntas. Contexto y decisiones cerradas: `Estado_del_proyecto.md`.

## Próximos pasos

- [x] Enseñar al ponente la estructura de la memoria (`Estructura_Memoria.md`)
- [ ] Esperar respuestas de departamentos (correos ya enviados). Guion: `_local/Preguntas_departamentos.md`
- [ ] Decidir el periodo de retención de `TEXTOEXTRAIDO` (tras hablarlo con los compañeros)
- [ ] Diseño de interfaz RF-09.2 (documento como vista principal, texto extraído como panel auxiliar)
- [ ] Decidir alcance del diagnóstico sistemático de errores de clasificación
- [ ] Definir el rol `consulta` en `USUARIO` — bloqueado hasta IT/Seguridad
- [ ] Actualizar `Diagrama_Componentes.md`: el evento de cancelación (RF-10.1) sale de un outbox en BD, no de Ticketing publicando al bus; falta dibujar el proceso intermedio
- [ ] Recalcular el Gantt: duración real de «Infraestructura» (depende del acceso) y festivos que cruzan tareas
- [ ] Revisar con el equipo si `AgenteIAClient` (infra) debe invocar `TicketingGateway` — lectura literal de RF-10.4, no cerrado con nadie
- [ ] Valorar extraer el colaborador «finalizar ticket + crear uno nuevo» (hoy en RF-09.6, RF-10.4 y RF-11.5) — YAGNI de momento
- [ ] **23/09.** Ajustar requisitos con `tipoEnvio`. Investigar si la comunicación (`1`) no tiene fecha límite de respuesta: en ese caso se puede descargar al momento sin perjudicar al destinatario. La notificación (`2`) sí tiene plazo y comparecencia; no tratar los dos envíos igual.

## Preguntas pendientes

**Áreas de negocio** (Fiscal y DGS hechos; resto en espera de respuesta)

- RF-03: texto de `localiza()` primero; PDF solo residual. Anexo casi nunca para clasificar (entrevista Fiscal). Confirmar con el resto de áreas.
- Comparecencia eager (batch madrugada) vs lazy (ellas / botón HV): pendiente de que el resto de áreas confirmen si les perjudica arrancar el plazo de respuesta al comparecer. Antes, ver el paso del 23/09: puede que solo la notificación tenga ese plazo.

**Ponente**

- [x] «Informe de sostenibilidad» (gestión FIB) vs «Análisis de sostenibilidad e implicaciones éticas» — se omite el Informe de gestión por ahora; el capítulo ético se queda.
- (no bloqueante) Expansión de las siglas SE/PRO de los entornos LEMA

**Responsable de Ticketing**

- Capacidades reales de audiencia back (RF-11.4 / RF-11.5)
- Campo `motivoResolucion` en `POST /tickets/{id}/estado` — ¿contradice lo de «sin motivo en frontend»?
- Quién consume el outbox de cancelación y cómo llega al bus / n8n
- ¿La creación de tickets permite deduplicar por identificador externo (el `identificador` DEHú)? Relevante para RF-08.5
- Canal email (RF-08.2): no bloqueante; el director lo dejó como deseable

**IT / Seguridad**

- ¿El Departamento consulta sus tickets desde el frontal propio? Alcance del rol `consulta` (¿limitado al propio departamento? ¿cómo se modela en `USUARIO`?)
