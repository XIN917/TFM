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
