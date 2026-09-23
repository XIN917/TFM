# TODO

Tareas abiertas y preguntas. Contexto y decisiones cerradas: `Estado.md`.

## Próximos pasos

- [ ] Esperar respuestas de departamentos (correos ya enviados). Guion: `_local/Preguntas_departamentos.md`
- [ ] Decidir el periodo de retención de `TEXTOEXTRAIDO` (tras hablarlo con los compañeros)
- [ ] Diseño de interfaz RF-09.2 (documento como vista principal, texto extraído como panel auxiliar)
- [ ] Decidir alcance del diagnóstico sistemático de errores de clasificación
- [ ] Definir el rol `consulta` en `USUARIO` — bloqueado hasta IT/Seguridad
- [ ] Actualizar `Diagrama_Componentes.md`: el evento de cancelación (RF-10.1) sale de un outbox en BD, no de Ticketing publicando al bus; falta dibujar el proceso intermedio
- [ ] Recalcular el Gantt: duración real de «Infraestructura» (depende del acceso) y festivos que cruzan tareas
- [ ] Revisar con el equipo si `AgenteIAClient` (infra) debe invocar `TicketingGateway` — lectura literal de RF-10.4, no cerrado con nadie
- [ ] Valorar extraer el colaborador «finalizar ticket + crear uno nuevo» (hoy en RF-09.6, RF-10.4 y RF-11.5) — YAGNI de momento

## Preguntas pendientes

**Áreas de negocio** (Fiscal y DGS hechos; resto en espera de respuesta)

- RF-03: texto de `localiza()` primero; PDF solo residual. Anexo casi nunca para clasificar (entrevista Fiscal). Confirmar con el resto de áreas.
- Comparecencia eager (batch madrugada) vs lazy (ellas / botón HV), **solo notificaciones** (`tipoEnvio` `2`): pendiente de que el resto de áreas confirmen si les perjudica arrancar el plazo de respuesta al comparecer. Las comunicaciones (`1`) no tienen ese plazo (FAQ DEHú, 23/09) y se descargan en el mismo ciclo.

**Ponente**

- Término genérico: DEHú llama *envío* a ambos tipos (notificación y comunicación; `envios`/`tipoEnvio` en la API [1]). Propuesta: adoptar «envío» como término genérico y reservar «notificación»/«comunicación» para cada tipo. Consultar antes de tocar nada:
  - **Título**: ¿se puede cambiar sin trámite formal? Opción preferida: «Automatización de la recepción de notificaciones y comunicaciones electrónicas de la Administración Pública» (pareja que usa el propio portal); si no, se mantiene el actual.
  - **Modelo**: renombrar `COMUNICACION` → `ENVIO` (arrastra `Comunicacion`, `ComunicacionRepository`, ER, diagrama de clases, requisitos y flujos).
  - **Introducción**: si se adopta «envío», adaptar el párrafo de terminología del Contexto (hoy dice que se usa «comunicación» en sentido amplio). No tocar hasta hablarlo.

**Responsable de Ticketing**

- Capacidades reales de audiencia back (RF-11.4 / RF-11.5)
- Campo `motivoResolucion` en `POST /tickets/{id}/estado` — ¿contradice lo de «sin motivo en frontend»?
- Quién consume el outbox de cancelación y cómo llega al bus / n8n
- ¿La creación de tickets permite deduplicar por identificador externo (el `identificador` DEHú)? Relevante para RF-08.5
- Canal email (RF-08.2): no bloqueante; el director lo dejó como deseable

**IT / Seguridad**

- Certificados del acceso manual actual a DEHú: ¿qué certificado(s) usan hoy las áreas (representante de persona jurídica, o certificado personal + apoderamiento)? ¿A quién están emitidos y en cuántos puestos están instalados? La Propuesta afirma «certificados en equipos de usuarios concretos, asociados a personas apoderadas», sin confirmar. Mientras tanto, la Introducción usa redacción neutra («mediante certificado digital»).
- ¿El Departamento consulta sus tickets desde el frontal propio? Alcance del rol `consulta` (¿limitado al propio departamento? ¿cómo se modela en `USUARIO`?)