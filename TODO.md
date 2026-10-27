# TODO

Tareas abiertas y preguntas. Contexto y decisiones cerradas: `Estado.md`.

## Próximos pasos

- [ ] Cerrar con las áreas las reglas en estado Confirmada/Pendiente de `Criterios_Clasificacion.md` (R10–R12): TGSS salvo embargo, Corredores y Sucursal Central
- [ ] Nueva versión del prompt con R13–R16 y «Área de Siniestros» en el `enum` de `schema.json` (v1.0 no la incluye)
- [ ] Few-shot: separar los ejemplos que van al prompt de los que se usan para evaluar (no evaluar con los mismos). Faltan ejemplos de R15, R16 y de otras reglas
- [ ] Siniestros reparte hoy por descarte (lo que no es de otra área), pero el prompt prohíbe clasificar por descarte, y hay envíos de Siniestros cuyo concepto es solo una referencia de expediente, sin materia, que no encajan en R13–R16 (una liquidación autonómica también podría ser tributaria). Sin ejemplos del resto de áreas no se puede medir cuántos envíos ajenos caerían en Siniestros. Decidir entre reglas positivas por organismo y materia (lo dudoso a revisión) o Siniestros como destino por defecto, y reunir ejemplos de las demás áreas para evaluarlo
- [ ] Conciliar con Asesoría Fiscal y RRHH el destino inicial de los embargos TGSS (R03)
- [ ] Confirmar la cola de Ticketing de AEPD → RRHH (R09)
- [ ] Con acceso a LEMA: dónde llega el área remitente Distribución de DGSFP (R07: `concepto`, `metadatosPublicos` u otro metadato previo a comparecer). No abrir el documento para buscarlo
- [ ] Con acceso a LEMA, antes de cargar `REALIZADAS` (RF-12.4): cuántos envíos devuelve `localizaRealizadas()` y qué periodo cubre (el portal solo muestra 30 días); si `tipoEnvio` `1` devuelve comunicaciones; si un envío comparecido aparece al momento (en el portal sí); si trae `metadatosPublicos`. Comprobar también si se pueden recuperar anexos y acuse de envíos antiguos (el límite de 24 h aplica a `consultaAnexos()` y `consultaAcusePdf()`)
- [ ] En pruebas LEMA: nombre del elemento de los anexos con URL directa (no vienen en `anexosReferencia`; hasta confirmarlo no tienen columna) y formato real de `acusePdf.metadatos` (en los ejemplos contiene el `csvResguardo`)
- [ ] Comentar con la tutora de empresa el cambio del ER: fuera `DEPARTAMENTO.criteriosReparto`, nueva `CLASIFICACION.versionPrompt` (la clasificación la llama n8n)
- [ ] Decidir la confianza de la clasificación: el prompt v1.0 devuelve `score` siempre `null`, pero RF-02 decide con un umbral sobre el score. Elegir umbral o el `estado` del prompt (`clasificado` / `revision_humana` / `sin_clasificar`)
- [ ] Regenerar las imágenes del ER y de clases en `Diagramas/img/` tras el cambio
- [ ] Copiar `PRD.md` a `HVOrganismosPublicos`
- [ ] Si se hace RF-12.6, acordar el tamaño de la muestra, quién accede a los documentos y cuándo se borran (contienen datos personales de terceros)
- [ ] Decidir el periodo de retención de `TEXTOEXTRAIDO` (tras hablarlo con los compañeros)
- [ ] Validar con Seguridad y Protección de Datos las condiciones para remitir documentos reales a OpenAI o Mistral (datos personales, contratos, retención y transferencias). Definir el comportamiento alternativo de derivación si el tratamiento no se autoriza o falla; no asumir que el ticket se genera sin interpretación hasta que se acuerde.
- [ ] Seleccionar y evaluar modelo/API de OpenAI o Mistral con documentos de prueba digitales, escaneados y mixtos; comprobar calidad de extracción, interpretación orientada a la tramitación, coste y latencia. Definir y validar el indicador de confianza de interpretación sin asumir que sea una probabilidad calibrada.
- [ ] Confirmar con Canal Directo qué información operativa necesita identificar de los documentos de Kit Digital (fases, requerimientos y actuaciones). La guía disponible es material de dominio, no un catálogo obligatorio de salida ya validado.
- [ ] Verificar durante la integración con LEMA si la autorización de Gran Destinatario de MGS permite consultar los envíos dirigidos a personas físicas cuyos avisos llegan al buzón corporativo. No asumir una integración adicional con Notific@.
- [ ] Contrastar las fechas calculadas de comparecencia con fechas de vencimiento de avisos reales y validar las reglas jurídicas aplicables. No usar una fecha orientativa como vencimiento confirmado ni como único criterio de acceso.
- [ ] Diseño de interfaz RF-09.2: metadatos de `localiza()` y departamento propuesto. El documento no se abre para clasificar
- [ ] Decidir alcance del diagnóstico sistemático de errores de clasificación
- [ ] Definir el rol `consulta` en `USUARIO` — bloqueado hasta IT/Seguridad
- [ ] Actualizar `Diagrama_Componentes.md`: el evento de cancelación (RF-10.1) sale de un outbox en BD, no de Ticketing publicando al bus; falta dibujar el proceso intermedio
- [ ] Recalcular el Gantt: duración real de «Infraestructura» (depende del acceso) y festivos que cruzan tareas
- [ ] Revisar con el equipo si `AgenteIAClient` (infra) debe invocar `TicketingGateway` — lectura literal de RF-10.4, no cerrado con nadie
- [ ] Valorar extraer el colaborador «finalizar ticket + crear uno nuevo» (hoy en RF-09.6, RF-10.4 y RF-11.5) — YAGNI de momento
- [ ] Mejora pendiente del flujo 2 (no cambia el diagrama actual). Si alguien abre una notificación mal clasificada, el plazo de respuesta ya corre desde la comparecencia y hay que redirigir enseguida. Idea: esa persona deja un comentario («esto no es de siniestros, es de canal directo»); la IA lo interpreta y reasigna el departamento al momento, y se avisa por correo a la persona correcta. El ticket no se borra; cómo modificarlo está sin decidir (Ticketing no transfiere de cola)
- [ ] Cerrar la política temporal de sondeo y comparecencia de las notificaciones. La necesidad es conservar el mayor margen posible antes de la comparecencia sin retrasar innecesariamente la derivación. Valorar si el sondeo principal se realiza de madrugada y, tras clasificar el envío con los metadatos de `localiza()`, la comparecencia, descarga y derivación se ejecutan inmediatamente dentro del mismo flujo. Si se adopta esta solución, integrar el actual flujo independiente de «lote de madrugada» en el flujo principal de sondeo. Hasta cerrar esta decisión, RF-01 mantiene el diseño vigente.

## Preguntas pendientes

**Áreas de negocio**

- Notificaciones judiciales que afectan a empleados: sin destino inicial validado (no hay regla).
- Interpretación documental: se contempla para los documentos principales de notificaciones y comunicaciones, condicionada a la autorización de IA externa. Sigue pendiente acordar la salida operativa cuando no se pueda procesar el documento.

**Ponente**

- Cómputo de la fecha límite de comparecencia: revisar con el ponente, el área jurídica y la integración LEMA la relación entre el artículo 43.2 y las reglas de cómputo del artículo 30 de la Ley 39/2015. Existe un caso observado de AEAT en que el vencimiento del aviso no coincidió con el desplazamiento esperado por festivo autonómico; no generalizar a partir de ese caso. La fecha calculada por la aplicación es orientativa, y todavía no se ha verificado cómo obtener un vencimiento confirmado mediante LEMA. Distinguirlo de los plazos de actuación que figuran en los documentos.

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