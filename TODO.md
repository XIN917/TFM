# TODO

Tareas abiertas y preguntas. Contexto y decisiones cerradas: `Estado.md`.

## Próximos pasos

- [ ] Esperar respuestas del resto de departamentos (correos ya enviados). Hechos: Fiscal, Coordinación DGS y Red de Mediación (25/09). Guion: `_local/Preguntas_departamentos.md`. Resúmenes: `_local/reuniones/`
- [ ] Ver de dónde sale el área remitente de Red de Mediación y reproducir ese reparto dentro del DIR3 `E00119006` (25/09). No está en el aviso DEHú ni en `localiza()` documentado. El aviso de la sede queda fuera. No preguntar al área. No abrir documentos para buscarlo, de momento. Detalle en `_local/reuniones/`
- [ ] Alinear casos de uso, RF-09.4, la aceptación de RF-01 y RF-06.1 con el diagrama del 23/09 (descarga y acuse según `tipoEnvio`, comparecencia en flujo propio). La clasificación sin leer el documento ya está en requisitos y PRD. No tocar el flujo 2.
- [ ] Decidir el periodo de retención de `TEXTOEXTRAIDO` (tras hablarlo con los compañeros)
- [ ] Diseño de interfaz RF-09.2 (documento como vista principal, texto extraído como panel auxiliar)
- [ ] Decidir alcance del diagnóstico sistemático de errores de clasificación
- [ ] Definir el rol `consulta` en `USUARIO` — bloqueado hasta IT/Seguridad
- [ ] Actualizar `Diagrama_Componentes.md`: el evento de cancelación (RF-10.1) sale de un outbox en BD, no de Ticketing publicando al bus; falta dibujar el proceso intermedio
- [ ] Recalcular el Gantt: duración real de «Infraestructura» (depende del acceso) y festivos que cruzan tareas
- [ ] Revisar con el equipo si `AgenteIAClient` (infra) debe invocar `TicketingGateway` — lectura literal de RF-10.4, no cerrado con nadie
- [ ] Valorar extraer el colaborador «finalizar ticket + crear uno nuevo» (hoy en RF-09.6, RF-10.4 y RF-11.5) — YAGNI de momento
- [ ] Mejora pendiente del flujo 2 (no cambia el diagrama actual). Si alguien abre una notificación mal clasificada, el plazo de respuesta ya corre desde la comparecencia y hay que redirigir enseguida. Idea: esa persona deja un comentario («esto no es de siniestros, es de canal directo»); la IA lo interpreta y reasigna el departamento al momento, y se avisa por correo a la persona correcta. El ticket no se borra; cómo modificarlo está sin decidir (Ticketing no transfiere de cola)

## Preguntas pendientes

**Áreas de negocio** (Fiscal, Coordinación DGS y Red de Mediación hechos; SAC y el resto en espera)

- Clasificación sin leer el documento: ya escrita en RF-05 y en el PRD (organismo y concepto de `localiza()`, antes de abrir). El caso DGSFP está en el paso de arriba. Pendiente de confirmar con el resto de departamentos.
- Resumen de la notificación: no se hace mientras la descarga no sea inmediata. La comunicación sí, porque se descarga en el mismo ciclo. Pendiente de confirmar con el resto de departamentos.
- Comparecencia eager (batch madrugada) vs lazy (ellas / botón HV), **solo notificaciones** (`tipoEnvio` `2`): el diagrama ya dibuja el botón en el frontal HV (responsable de área, sin volver a clasificar). Pendiente de que el resto de áreas confirmen si les perjudica arrancar el plazo de respuesta al comparecer. Las comunicaciones (`1`) no tienen ese plazo (FAQ DEHú, 23/09) y se descargan en el mismo ciclo.

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
- Certificado de LEMA para pruebas: la guía permite uno autofirmado, distinto del de producción, dado de alta solo en el entorno de pruebas (`se-gd-dehuws.redsara.es`). No llamarlo contra producción. Quién lo genera: Sistemas.
- ¿El Departamento consulta sus tickets desde el frontal propio? Alcance del rol `consulta` (¿limitado al propio departamento? ¿cómo se modela en `USUARIO`?)