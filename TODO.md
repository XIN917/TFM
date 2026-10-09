# TODO

Tareas abiertas y preguntas. Contexto y decisiones cerradas: `Estado.md`.

## Próximos pasos

- [ ] Cerrar reglas de Siniestros y validar Sucursal Central. Fiscal, Coordinación DGS, Red de Mediación y Canal Directo ya están en el catálogo del PRD. Interlocutores: `_local/interlocutores_reparto.md`. Resúmenes: `_local/reuniones/`
- [ ] Cuando haya acceso a LEMA, comprobar qué devuelve `localiza()` para una notificación DGSFP que hoy se ve como área remitente Distribución (`concepto`, `metadatosPublicos` u otro dato previo a comparecer). Hasta entonces no entra en el prompt. No abrir el documento para buscarlo. Exclusivos y Mediación sí entran, por concepto. Detalle en `_local/reuniones/`
- [ ] Validar con RRHH la regla TGSS salvo embargo. Está fuera del prompt. RRHH no consulta hoy el buzón DEHú. Interlocutor en `_local/interlocutores_reparto.md`
- [ ] Con acceso a LEMA, antes de cargar `REALIZADAS` (RF-12.4): cuántos envíos devuelve `localizaRealizadas()` y qué periodo cubre (el portal solo muestra 30 días); si `tipoEnvio` `1` devuelve comunicaciones; si un envío comparecido aparece al momento (en el portal sí); si trae `metadatosPublicos`. Comprobar también si se pueden recuperar anexos y acuse de envíos antiguos (el límite de 24 h aplica a `consultaAnexos()` y `consultaAcusePdf()`)
- [ ] En pruebas LEMA: nombre del elemento de los anexos con URL directa (no vienen en `anexosReferencia`; hasta confirmarlo no tienen columna) y formato real de `acusePdf.metadatos` (en los ejemplos contiene el `csvResguardo`)
- [ ] Decidir si RF-04.1 (identificar el tipo de comunicación) sigue en la especificación. El PRD y `INTERPRETACION` solo guardan entidades y score; sin catálogo de tipos (decisión del 28/09)
- [ ] Copiar `PRD.md` a `HVOrganismosPublicos`
- [ ] Si se hace RF-12.6, acordar el tamaño de la muestra, quién accede a los documentos y cuándo se borran (contienen datos personales de terceros)
- [ ] Decidir el periodo de retención de `TEXTOEXTRAIDO` (tras hablarlo con los compañeros)
- [ ] Diseño de interfaz RF-09.2: metadatos de `localiza()` y departamento propuesto. El documento no se abre para clasificar
- [ ] Decidir alcance del diagnóstico sistemático de errores de clasificación
- [ ] Definir el rol `consulta` en `USUARIO` — bloqueado hasta IT/Seguridad
- [ ] Actualizar `Diagrama_Componentes.md`: el evento de cancelación (RF-10.1) sale de un outbox en BD, no de Ticketing publicando al bus; falta dibujar el proceso intermedio
- [ ] Recalcular el Gantt: duración real de «Infraestructura» (depende del acceso) y festivos que cruzan tareas
- [ ] Revisar con el equipo si `AgenteIAClient` (infra) debe invocar `TicketingGateway` — lectura literal de RF-10.4, no cerrado con nadie
- [ ] Valorar extraer el colaborador «finalizar ticket + crear uno nuevo» (hoy en RF-09.6, RF-10.4 y RF-11.5) — YAGNI de momento
- [ ] Mejora pendiente del flujo 2 (no cambia el diagrama actual). Si alguien abre una notificación mal clasificada, el plazo de respuesta ya corre desde la comparecencia y hay que redirigir enseguida. Idea: esa persona deja un comentario («esto no es de siniestros, es de canal directo»); la IA lo interpreta y reasigna el departamento al momento, y se avisa por correo a la persona correcta. El ticket no se borra; cómo modificarlo está sin decidir (Ticketing no transfiere de cola)

## Preguntas pendientes

**Áreas de negocio**

- Siniestros y Sucursal Central siguen sin regla activa. No convertir en regla lo observado (ayuntamientos, juzgados, Guardia Civil, contratación) hasta validarlo. Nombres en `_local/interlocutores_reparto.md`.
- DGSFP + Corredores → Red de Mediación: observado, sin consolidar. Fuera del prompt.
- Resumen del documento principal de las notificaciones: no forma parte del alcance actual. Se verá cuando la clasificación por departamento esté cerrada. La comunicación (`tipoEnvio` `1`) sí se resume.

**Ponente**

- Festivos en la fecha límite de comparecencia (`fechaPuestaDisposicion` + 10 días naturales, Ley 39/2015 arts. 43.2 y 30.3; `localiza()` no da la caducidad). Si el último día es sábado o domingo, pasa al primer día hábil siguiente (art. 30.5). Los festivos no se cuentan, igual que hace la AEAT. Comentarlo:
  - Caso AEAT: el último día cayó en un festivo autonómico del domicilio de la empresa. Por el art. 30.6 era inhábil y debía pasar al día hábil siguiente, pero el vencimiento del aviso DEHú no se movió.
  - El art. 30.6 suma los festivos del domicilio de la empresa y los de la sede de cada órgano emisor: nacionales, autonómicos y locales, que cambian cada año.
  - Sin contar festivos, la fecha puede salir antes que la real, pero nunca después.

- Término genérico: DEHú llama *envío* a ambos tipos (notificación y comunicación; `envios`/`tipoEnvio` en la API [1]). Propuesta: adoptar «envío» como término genérico y reservar «notificación»/«comunicación» para cada tipo. Consultar antes de tocar nada:
  - **Título**: ¿se puede cambiar sin trámite formal? Opción preferida: «Automatización de la recepción de notificaciones y comunicaciones electrónicas de la Administración Pública» (pareja que usa el propio portal); si no, se mantiene el actual.
  - **Modelo**: renombrar `COMUNICACION` → `ENVIO` (arrastra `Comunicacion`, `ComunicacionRepository`, ER, diagrama de clases, requisitos y flujos).
  - **Introducción**: si se adopta «envío», adaptar el párrafo de terminología del Contexto (hoy dice que se usa «comunicación» en sentido amplio). No tocar hasta hablarlo.
- ¿El código fuente es un entregable obligatorio del TFM? ¿Existe en la FIB la opción de memoria confidencial o de publicación aplazada (UPCommons)?

**MGS (responsable del TFM en la empresa)**

- Qué se puede incluir en la memoria: código propio (¿fragmentos?, ¿cuánto?), nombres de módulos y de sistemas corporativos, capturas. Mientras tanto se sigue el criterio de `memoria/Notas_pendientes.md` (módulos por función, sistemas corporativos en genérico).

**Responsable de Ticketing**

- Capacidades reales de audiencia back (RF-11.4 / RF-11.5)
- Campo `motivoResolucion` en `POST /tickets/{id}/estado` — ¿contradice lo de «sin motivo en frontend»?
- Quién consume el outbox de cancelación y cómo llega al bus / n8n
- ¿La creación de tickets permite deduplicar por identificador externo (el `identificador` DEHú)? Relevante para RF-08.5
**Infra**

- Canal email (RF-08.2): qué servidor SMTP corporativo y qué remitente usa la aplicación. No es obligatorio en el MVP, pero está planificado tras el ticket.

**IT / Seguridad**

- Certificados del acceso manual actual a DEHú: ¿qué certificado(s) usan hoy las áreas (representante de persona jurídica, o certificado personal + apoderamiento)? ¿A quién están emitidos y en cuántos puestos están instalados? La Propuesta afirma «certificados en equipos de usuarios concretos, asociados a personas apoderadas», sin confirmar. Mientras tanto, la Introducción usa redacción neutra («mediante certificado digital»).
- Certificado de LEMA para pruebas: la guía permite uno autofirmado, distinto del de producción, dado de alta solo en el entorno de pruebas (`se-gd-dehuws.redsara.es`). No llamarlo contra producción. Quién lo genera: Sistemas.
- ¿El Departamento consulta sus tickets desde el frontal propio? Alcance del rol `consulta` (¿limitado al propio departamento? ¿cómo se modela en `USUARIO`?)