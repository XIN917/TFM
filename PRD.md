# PRD — HVOrganismosPublicos

Documento de producto e implementación del aplicativo `HVOrganismosPublicos` (MGS Seguros). Consolida lo necesario para construir el sistema: alcance, arquitectura, módulos, requisitos, modelo de datos, integraciones y convenciones. El detalle académico de cada RF sigue en `Especificacion_Requisitos.md`; este PRD es la brújula de construcción.

| | |
|---|---|
| Producto | `HVOrganismosPublicos` |
| Área Eclipse | `HV` (Consulta) |
| Git | `HV/HVOrganismosPublicos` (workspace local: `Consultas/HVOrganismosPublicos`) |
| BBDD | `HV_OrganismosPublicos` en SQLPortal (DESA y PROD; extra de tamaño por documentos) |
| Plataforma | Java 8, Java EE 7 (`javax.*`), WAS traditional 9, CDI, JAX-RS, JDBC (sin JPA/Spring/Jakarta) |
| Familia de referencia | AYTicketing / AYCalendarios (SQL propio, hexagonal, OpenAPI v1). **No** SISiso/Personas (no hay maestro CICS → no hay módulo DAO ni EJB). |

Fuentes: `Especificacion_Requisitos.md`, `Estado_del_proyecto.md`, `Estructura_Repositorio.md`, `DEHu_Campos_Respuesta_Servicios.md`, `Diagramas/*`.

---

## 1. Overview

Hoy cada departamento entra a **DEHú** con certificado local (habitualmente por el enlace del correo de aviso), mira el listado entero y decide si algo es suyo. Mi Carpeta Ciudadana es otro portal; también puede abrir el mismo buzón. El producto sustituye la consulta en pantalla por **LEMA** (servicios web de Gran Destinatario): sondeo automático, persistencia inmediata de documentos, clasificación (metadatos de `localiza()` primero; OCR+LLM del documento solo si hace falta), evento corporativo y derivación al **Ticketing** interno (canal obligatorio). Si la IA no alcanza el umbral, un **Operador** revisa. Si el departamento cancela por mala cola, un **Agente IA** (fuera de WAS, vía MCP) intenta autocorregir como máximo dos veces.

El nombre `OrganismosPublicos` es más amplio que DEHú a propósito (otras fuentes en el futuro). El **MVP es solo DEHú/LEMA**. No dilata el alcance.

**Problema:** carga manual, certificados en puestos, comunicaciones oficiales que se pierden o llegan tarde.

**Propuesta de valor:** una sola ingesta centralizada, trazable, con IA y ticket en la cola correcta (o revisión humana), sin que el departamento recorra el buzón completo.

### 1.1 Objetivo del MVP

Ciclo de **recepción**: detectar en DEHú → almacenar documento/anexos/acuse → interpretar y clasificar → publicar evento → ejecutar notificación (ticket; email deseable) → permitir revisión humana y reclasificación (operador o agente).

Hay que separar siempre:

1. **Generar el evento** (comunicación clasificada, RF-07).
2. **Ejecutar la acción** (RF-08, canal por configuración).

Así Ticketing no es la única salida posible.

### 1.2 Fuera de alcance (MVP)

- Envío de escritos a la administración (LEMA es solo lectura; el envío sería REG/REC / sedes distintas → otro sistema).
- Automatismos que abran expedientes o toquen más sistemas de negocio que notificar.
- MCP genérico: solo la acción «finalizar ticket + crear uno nuevo» del Agente IA (RF-10.4).
- Multi-certificado / multi-razón social.
- Buzón interno (hueco RF-08.3; no implementar).
- `localizaRealizadas` / `consultaRealizadas` (RF-12): candidato si hay tiempo, no comprometido.
- Aprendizaje continuo del modelo a partir del feedback del Operador (evolución; persistir el feedback sí es deseable, RF-09.3).

---

## 2. Actores y roles

| Actor externo | Qué hace |
|---|---|
| DEHú/LEMA | Fuente SOAP. El sistema llama; LEMA no empuja. |
| Operador / Administrador | Frontal propio: cola de revisión y consulta local (RF-09, RF-11). |
| Departamento | Consume tickets en Ticketing (no el frontal HV). Cancela si la cola es incorrecta → evento RF-10. |

Roles en `USUARIO` (autenticación contra directorio de personal MGS):

| Rol | Permisos |
|---|---|
| `operador` | Ver cola de revisión, aceptar propuesta IA. |
| `administrador` | Lo de operador **más** reclasificar (RF-09.6). |

Rol `consulta` (departamento en frontal HV): **bloqueado** hasta IT/Seguridad. No implementar hasta que exista decisión.

Toda resolución de revisión queda asociada al `USUARIO` que la ejecutó.

---

## 3. Arquitectura del sistema

Estilos (justificarlos en la memoria, no como efecto secundario de SOLID):

- **Entre componentes:** orientada a eventos (bus corporativo / n8n). Pipeline, ejecución y reclasificación no se llaman en cadena rígida.
- **Dentro de Java:** hexagonal — `modelo` / `aplicacion` / `infraestructura` / `api`.

n8n **orquesta** (dispara sondeo, OCR/LLM, publica/consume eventos). Java **posee** el estado y la ventana legal: el conector LEMA y la persistencia de anexos/acuse viven en `ApiBack` + `Business`, no solo en un flujo n8n. Si n8n cae a las 25 h, el anexo ya no se puede recuperar por API.

```
DEHú/LEMA (SOAP 1.1 + WS-Security X.509)
        │
        ▼
  LemaGateway ──► JDBC ──► HV_OrganismosPublicos
        │                    ▲
        │                    │
        ▼                    │
  OCR/LLM (fuera; n8n) ── persistir INTERPRETACION / TEXTOEXTRAIDO
        │
        ▼
  Publicador RF-07 ──► Bus / n8n ──► EjecutarDerivacionService
                                          │
                          TicketingGateway ┴ EmailNotificacionGateway
                                          │
Departamento cancela ticket ─(outbox, no webhook)─► evento RF-10
                                          │
                              AgenteReclasificacionService
                                          │
                              AgenteIAGateway (MCP, proceso externo)
                                          │
Operador ──► Vue Web ──► ApiFront (JAX-RS) ──► servicios de revisión/consulta
```

### 3.1 Bloques

| Bloque | RF | Quién |
|---|---|---|
| Pipeline ingesta + clasificación | 01–07 | n8n dispara; Java `LemaGateway` + repos + persistencia IA |
| Ejecución de canal | 08 | `EjecutarDerivacionService` + `NotificacionGateway` |
| Reclasificación automática | 10 | `ApiConsumer` + `AgenteReclasificacionService` + MCP fuera de WAS |
| Gestión / revisión | 09, 11 | `ApiFront` + `HVOrganismosPublicosWeb` |

Línea discontinua (no cerrada con el equipo):

- Modificar ticket in-place (audiencia **back** de Ticketing, RF-11.4/11.5).
- Cómo el outbox de cancelación de Ticketing llega al bus / n8n (Ticketing **no** publica webhook).

### 3.2 Capas Java y convenciones

Paquete raíz: `es.mgs.hv.organismosPublicos`.

Estereotipos como Ticketing: `@Servicio`, `@Repositorio`, `@Endpoint`, `@Transaccional`. Sin `AggregateRoot` ni eventos de dominio CDI (los cubre RF-07 a nivel corporativo).

**CDI:** `@Inject` en el **constructor** de cada `*Service`, no solo en campos. WAS ensambla implementaciones reales. En `HVOrganismosPublicosTest`: `new Servicio(gatewayFalso, repoFalso)` sin servidor. Sin constructor visible el unitario no es defendible en el TFM.

`beans.xml` con `bean-discovery-mode="annotated"` en Business, Comun (`src/META-INF`) y en Front/Web (`WebContent/WEB-INF`).

Interfaces de `modelo` (el código no depende de clases de `infraestructura`):

| Interfaz | Operaciones |
|---|---|
| `ComunicacionRepository` | `save`, `query`, `siguienteId` (+ consultas de listado que hagan falta) |
| `LemaGateway` | `localiza`, `peticionAcceso`, `consultaAnexos`, `consultaAcusePdf` |
| `TicketingGateway` | `crearTicket`, `finalizarTicket`, `modificarTicket` |
| `NotificacionGateway` | `soportaCanal`, `ejecutar` |
| `AgenteIAGateway` | `intentarAutocorreccion` |

Catálogo completo de carpetas y clases: sección 4. Las de `Diagrama_Clases.md` se marcan **UML**; el resto cierra RF-01–07, OpenAPI, WAS, Vue y tests.

Patrones a respetar: Strategy + Adapter en canales (`NotificacionGateway` / `TicketingNotificacionGateway`); Gateway y Repository (Fowler); `NotificacionGatewayResolver` con `@Inject @Any Instance<NotificacionGateway>`. No Observer/State/Factory Method en el modelo de clases.

### 3.3 Qué no crear

DAO, EJB/EJBClient/Utils, WebService SOAP legacy, BatchV1 EAR aparte. JDBC en `Business/infraestructura`. El sondeo es un endpoint Back (n8n o timer en el mismo WAR).

El Agente IA **no** es un WAR en WAS. Solo `AgenteIAClient` que habla con el proceso externo.

---

## 4. Estructura de directorios

```
HVOrganismosPublicos/
├── .gitignore
│
├── HVOrganismosPublicosBeans/
│   └── src/es/mgs/hv/organismosPublicos/bean/
│       ├── id/
│       │   ├── ComunicacionId.java
│       │   ├── DocumentoId.java
│       │   ├── InterpretacionId.java
│       │   ├── ClasificacionId.java
│       │   ├── DerivacionId.java
│       │   └── RevisionId.java
│       ├── eventos/
│       │   ├── EventoComunicacionClasificada.java      # RF-07
│       │   └── EventoCancelacionTicket.java            # RF-10.1
│       ├── exception/
│       │   ├── OrganismosPublicosException.java
│       │   ├── EntidadNoEncontradaException.java
│       │   ├── OperacionNoPermitidaException.java
│       │   ├── BusinessRuleViolationException.java
│       │   ├── DerivacionDuplicadaException.java
│       │   ├── VentanaAnexoExpiradaException.java
│       │   ├── IntegridadDocumentoException.java
│       │   └── LimiteReclasificacionException.java
│       └── seguridad/
│           └── UsuarioAutenticado.java
│
├── HVOrganismosPublicosComun/
│   └── src/
│       ├── META-INF/beans.xml
│       └── es/mgs/hv/organismosPublicos/comun/
│           ├── Constantes.java
│           ├── annotation/
│           │   ├── Servicio.java
│           │   ├── Repositorio.java
│           │   ├── Endpoint.java
│           │   ├── Transaccional.java
│           │   └── Client.java
│           ├── properties/
│           │   ├── PropiedadesLoader.java
│           │   └── GestorPropiedadesProvider.java
│           ├── exceptions/
│           │   ├── BadInputException.java
│           │   └── IntegrationException.java
│           └── util/
│               ├── Fechas.java
│               └── ConversionUtils.java
│
├── HVOrganismosPublicosBusiness/
│   └── src/
│       ├── META-INF/beans.xml
│       └── es/mgs/hv/organismosPublicos/business/
│           ├── modelo/
│           │   ├── Comunicacion.java                      # UML
│           │   ├── Documento.java                         # UML
│           │   ├── Interpretacion.java                    # UML
│           │   ├── TextoExtraido.java                     # UML
│           │   ├── Clasificacion.java                     # UML
│           │   ├── Derivacion.java                        # UML
│           │   ├── Revision.java                          # UML
│           │   ├── Departamento.java                      # UML
│           │   ├── Usuario.java                           # UML
│           │   ├── CambiosDerivacion.java                 # UML
│           │   ├── ResultadoAutocorreccion.java           # UML
│           │   ├── EnvioPendiente.java
│           │   ├── ContenidoTicket.java
│           │   ├── FiltroConsultaComunicaciones.java
│           │   ├── ConstantesDominio.java
│           │   ├── ComunicacionRepository.java            # UML
│           │   ├── DepartamentoRepository.java
│           │   ├── UsuarioRepository.java
│           │   ├── LemaGateway.java                       # UML
│           │   ├── TicketingGateway.java                  # UML
│           │   ├── NotificacionGateway.java               # UML
│           │   ├── AgenteIAGateway.java                   # UML
│           │   ├── InterpretacionIAGateway.java
│           │   ├── EventosGateway.java
│           │   ├── AlmacenDocumentosGateway.java
│           │   └── MailGateway.java
│           ├── aplicacion/
│           │   ├── ingesta/
│           │   │   └── IngestarComunicacionesService.java           # RF-01/02
│           │   ├── interpretacion/
│           │   │   ├── InterpretarComunicacionService.java          # RF-03/04
│           │   │   └── RegistrarInterpretacionService.java
│           │   ├── clasificacion/
│           │   │   ├── ClasificarComunicacionService.java           # RF-05
│           │   │   └── GenerarContenidoTicketService.java           # RF-06
│           │   ├── eventos/
│           │   │   └── PublicarComunicacionClasificadaService.java  # RF-07
│           │   ├── derivacion/
│           │   │   ├── EjecutarDerivacionService.java               # UML RF-08
│           │   │   └── NotificacionGatewayResolver.java             # UML
│           │   ├── revision/
│           │   │   ├── AceptarClasificacionService.java             # UML RF-09.5
│           │   │   └── ReclasificarComunicacionService.java         # UML RF-09.6
│           │   ├── consulta/
│           │   │   ├── ConsultarComunicacionesService.java          # RF-11
│           │   │   └── ConsultarRevisionService.java                # RF-09.1/09.2
│           │   ├── modificacion/
│           │   │   └── ModificarDerivacionService.java              # UML RF-11.4/11.5
│           │   └── reclasificacion/
│           │       └── AgenteReclasificacionService.java            # UML RF-10
│           └── infraestructura/
│               ├── persistencia/
│               │   ├── Database.java
│               │   ├── ComunicacionRepositorySQLServer.java         # UML
│               │   ├── DepartamentoRepositorySQLServer.java
│               │   └── UsuarioRepositorySQLServer.java
│               ├── almacenamiento/
│               │   └── AlmacenDocumentosFileSystem.java
│               ├── lema/
│               │   └── LemaClient.java                              # UML
│               ├── ticketing/
│               │   └── TicketingClient.java                         # UML
│               ├── notificacion/
│               │   ├── TicketingNotificacionGateway.java            # UML
│               │   └── EmailNotificacionGateway.java                # UML
│               ├── mail/
│               │   └── SmtpMailClient.java
│               ├── eventos/
│               │   └── AyEventosClient.java
│               ├── ia/
│               │   ├── InterpretacionIAClient.java
│               │   └── AgenteIAClient.java                          # UML MCP
│               └── http/
│                   └── RestClientSupport.java
│
├── HVOrganismosPublicosApiFrontV1WebService/                       # fase 1
│   ├── openapi/
│   │   ├── openapi.yaml
│   │   └── paths/
│   │       ├── revisiones.yaml
│   │       └── comunicaciones.yaml
│   ├── gen/                                                        # ant generate; no editar
│   ├── src/es/mgs/hv/organismosPublicos/api/front/v1/
│   │   ├── impl/
│   │   │   ├── RevisionesApiImpl.java                              # UML RevisionController
│   │   │   └── ComunicacionesApiImpl.java                          # UML RegistroController
│   │   ├── mappers/
│   │   │   ├── ComunicacionMapper.java
│   │   │   ├── RevisionMapper.java
│   │   │   └── DerivacionMapper.java
│   │   ├── listeners/
│   │   │   └── ApiServletContextListener.java
│   │   ├── filters/
│   │   │   └── CacheControlFilter.java
│   │   ├── exception/mappers/
│   │   │   ├── EntidadNoEncontradaExceptionMapper.java
│   │   │   ├── BadInputExceptionMapper.java
│   │   │   └── OrganismosPublicosExceptionMapper.java
│   │   └── providers/
│   │       └── GestorBackendFeature.java
│   └── WebContent/WEB-INF/
│       ├── web.xml
│       ├── beans.xml
│       ├── ibm-web-ext.xml
│       └── ibm-web-bnd.xml
├── HVOrganismosPublicosApiFrontV1WasEAR/                            # WAR + Beans + Comun + Business
│
├── HVOrganismosPublicosWeb/                                         # fase 1
│   ├── src/es/mgs/hv/organismosPublicos/web/
│   │   ├── OrganismosPublicosApplication.java
│   │   ├── servlet/AbstractHttpServlet.java
│   │   ├── controller/UsuarioGetController.java
│   │   ├── api/JsonProvider.java
│   │   ├── exception/SeguridadAPIException.java
│   │   ├── shared/jndi/Referencia.java
│   │   └── seguridad/
│   │       ├── filtro/SeguridadFilter.java
│   │       ├── autenticacion/SeguridadProvider.java
│   │       └── csrf/HttpCSRFTokenProvider.java
│   ├── frontend/src/
│   │   ├── main.ts
│   │   ├── App.vue
│   │   ├── router.ts
│   │   ├── client/                                                 # tsclient Front
│   │   ├── store/
│   │   │   ├── revisiones.store.ts
│   │   │   └── comunicaciones.store.ts
│   │   ├── views/
│   │   │   ├── ListaRevisiones/ListaRevisionesView.vue             # RF-09.1
│   │   │   ├── DetalleRevision/DetalleRevisionView.vue             # RF-09.2/09.5/09.6
│   │   │   ├── ListaComunicaciones/ListaComunicacionesView.vue     # RF-11.1
│   │   │   ├── DetalleComunicacion/DetalleComunicacionView.vue     # RF-11
│   │   │   └── Error/ErrorView.vue
│   │   └── components/
│   │       ├── DocumentoViewer.vue
│   │       └── TextoExtraidoPanel.vue
│   └── WebContent/WEB-INF/
│       ├── web.xml
│       ├── beans.xml
│       ├── ibm-web-ext.xml
│       └── ibm-web-bnd.xml
├── HVOrganismosPublicosWebEAR/                                      # WAR + Comun
│
├── HVOrganismosPublicosApiBackV1WebService/                         # fase 2
│   ├── openapi/openapi.yaml
│   ├── gen/
│   ├── src/es/mgs/hv/organismosPublicos/api/back/v1/
│   │   ├── impl/
│   │   │   ├── IngestaApiImpl.java                                 # POST sondeo RF-01/02
│   │   │   ├── InterpretacionesApiImpl.java                        # POST OCR/LLM
│   │   │   └── DerivacionesApiImpl.java                            # POST RF-08
│   │   ├── mappers/
│   │   ├── exception/mappers/
│   │   └── listeners/ApiServletContextListener.java
│   └── WebContent/WEB-INF/
├── HVOrganismosPublicosApiBackV1WasEAR/
│
├── HVOrganismosPublicosApiConsumerV1WebService/                     # fase 3
│   ├── openapi/openapi.yaml
│   ├── gen/
│   ├── src/es/mgs/hv/organismosPublicos/api/consumer/v1/
│   │   └── impl/CancelacionesApiImpl.java                          # RF-10
│   └── WebContent/WEB-INF/
├── HVOrganismosPublicosApiConsumerV1WasEAR/
│
├── HVOrganismosPublicosMigraciones/
│   ├── V1__esquema.sql
│   ├── V2__indices.sql
│   └── V3__seed_departamento.sql
│
├── HVOrganismosPublicosProperties/                                  # disco servidor, no EAR
│   ├── local#organismosPublicos.properties
│   ├── desa#organismosPublicos.properties
│   ├── usua#organismosPublicos.properties
│   ├── prod#organismosPublicos.properties
│   ├── local#log4j2.properties
│   ├── desa#log4j2.properties
│   ├── usua#log4j2.properties
│   └── prod#log4j2.properties
│
├── HVOrganismosPublicosServer/
│   ├── datasource.py
│   └── certificado_lema.py
│
├── HVOrganismosPublicosTest/
│   └── src/es/mgs/hv/organismosPublicos/test/
│       ├── aplicacion/
│       │   ├── IngestarComunicacionesServiceTest.java
│       │   ├── InterpretarComunicacionServiceTest.java
│       │   ├── ClasificarComunicacionServiceTest.java
│       │   ├── GenerarContenidoTicketServiceTest.java
│       │   ├── PublicarComunicacionClasificadaServiceTest.java
│       │   ├── EjecutarDerivacionServiceTest.java
│       │   ├── AceptarClasificacionServiceTest.java
│       │   ├── ReclasificarComunicacionServiceTest.java
│       │   ├── AgenteReclasificacionServiceTest.java
│       │   ├── ModificarDerivacionServiceTest.java
│       │   └── ConsultarComunicacionesServiceTest.java
│       └── modelo/
│           ├── ComunicacionTest.java
│           └── DocumentoTest.java
│
└── HVOrganismosPublicosFT/                                          # fase 3
    └── tests/
        ├── operador-revision.spec.ts
        └── operador-consulta.spec.ts
```

| Módulo | Contenido |
|---|---|
| Beans | Ids, excepciones, DTOs de evento. Sin SQL ni SOAP. |
| Comun | log4j2, carga de properties. |
| Business | Hexagonal: modelo, aplicación, infraestructura (`*Jdbc`, `LemaClient`, `TicketingClient`, `AgenteIAClient`). |
| ApiFront + WasEAR | REST Operador. EAR: WAR + Business + Beans + Comun. |
| Web + WebEAR | SPA Vue (UIBaseProject) + servlets de entrada/seguridad. EAR: WAR + Comun. **Sin Business.** |
| ApiBack | Hooks n8n: sondeo LEMA, pipeline, derivación. |
| ApiConsumer | Evento de cancelación (RF-10). |
| Migraciones | SQL del esquema. |
| Properties | `{desa\|local\|prod\|usua}#*.properties` y log4j2. En disco del servidor (`PATH_PROPERTIES_SERVIDOR`), no en el EAR. |
| Server | wsadmin: datasource SQLPortal, certificado LEMA. |
| Test | JUnit 5, Java 8, Mockito. Sin WAS. |
| FT | Playwright E2E contra desa/usua. No sustituye a Test. |

OpenAPI: `x-area: hv`, `x-subarea: def`, `x-version: v1`. Gateway: `/api/{front\|back}/hv/def/v1`. `def` = default; Infra puede asignar otro código de 3 letras si HV crece.

Context root WAS (después, `ibm-web-ext.xml`): Front `HVOrganismosPublicos/api/front/v1`; Web `appt/HVOrganismosPublicosWeb`.

Fases de construcción (plazo TFM; backend/frontend antes del 25 dic 2026):

| Fase | Qué |
|---|---|
| 1 | Beans, Comun, Business, Front+Web, Migraciones, Properties, Server, Test |
| 2 | ApiBack (n8n dispara sondeo y derivación) |
| 3 | ApiConsumer y FT cuando exista el evento de cancelación y haya desa |

Classpath (ya aplicado en RAD): Business → Beans+Comun; Test → Business; Front → Beans+Comun+Business; Web → Comun. WasEAR `/lib`: Beans, Comun, Business. WebEAR `/lib`: Comun.

### 4.1 Contrato de clases

El árbol de arriba es la única estructura de directorios. Aquí no se repite. Paquete raíz: `es.mgs.hv.organismosPublicos`. **UML** = `Diagramas/Diagrama_Clases.md`. Constructor `@Inject` en cada `*Service`.

Métodos UML:

| Clase | Métodos |
|---|---|
| `Comunicacion` | `tieneDerivacionAsociada`, `tieneDerivacionExitosa`, `tieneRevisionActiva`, `derivacionActiva`, `contarIntentosReclasificacion`, `registrarDerivacion`, `registrarClasificacion`, `escalarARevision` |
| `Documento` | `verificarIntegridad` |
| `Derivacion` | `esModificableInPlace`, `actualizar` |
| `Revision` | `resolver(Usuario)` |
| `ComunicacionRepository` | UML: `save`, `query`, `siguienteId`. Añadir en la misma interfaz: `queryPorIdentificadorDehu`, `queryListado`, `queryRevisionesPendientes` |
| `LemaGateway` | `localiza`, `peticionAcceso`, `consultaAnexos`, `consultaAcusePdf` |
| `TicketingGateway` | `crearTicket`, `finalizarTicket`, `modificarTicket` |
| `NotificacionGateway` | `soportaCanal`, `ejecutar` |
| `AgenteIAGateway` | `intentarAutocorreccion` |

`ConstantesDominio`: estados `pendiente\|en_proceso\|en_revision\|procesada`; origen `ia_inicial\|ia_reclasificacion\|operador`; canal `ticket\|email`; tipo documento `principal\|anexo\|acuse`; rol `operador\|administrador`.

Colaboradores de aplicación:

| Servicio | Interfaces |
|---|---|
| `IngestarComunicacionesService` | `LemaGateway`, `ComunicacionRepository`, `AlmacenDocumentosGateway` |
| `InterpretarComunicacionService` | `InterpretacionIAGateway`, `ComunicacionRepository` |
| `RegistrarInterpretacionService` | `ComunicacionRepository` |
| `ClasificarComunicacionService` | `ComunicacionRepository`, `DepartamentoRepository` |
| `GenerarContenidoTicketService` | `ComunicacionRepository` |
| `PublicarComunicacionClasificadaService` | `EventosGateway`, `ComunicacionRepository` |
| `EjecutarDerivacionService` | `ComunicacionRepository`, `NotificacionGatewayResolver` |
| `NotificacionGatewayResolver` | `@Any Instance<NotificacionGateway>` |
| `AceptarClasificacionService` | `ComunicacionRepository`, `TicketingGateway` |
| `ReclasificarComunicacionService` | `ComunicacionRepository`, `TicketingGateway` |
| `ConsultarComunicacionesService` | `ComunicacionRepository` |
| `ConsultarRevisionService` | `ComunicacionRepository` |
| `ModificarDerivacionService` | `ComunicacionRepository`, `TicketingGateway` |
| `AgenteReclasificacionService` | `ComunicacionRepository`, `AgenteIAGateway` (`LIMITE_INTENTOS = 2`) |

`LemaClient`: un client, cuatro métodos SOAP. Endpoints HTTP: sección 8. RF-12 (`ReconciliarRealizadasService`): no crear hasta que el calendario lo permita.

---

## 5. Requisitos funcionales (contrato de implementación)

Texto completo: `Especificacion_Requisitos.md`. Aquí: regla que el código no puede violar + dónde cae.

### RF-01 Detección

- n8n (o timer) llama a Back; Java `localiza()`. Lista vacía → no hacer nada.
- Por cada item: `peticionAcceso(identificador, codigoOrigen)`.
- Idempotencia por `identificador` DEHú (no reprocesar).
- Reintentos con backoff ante SOAP/certificado/red; no perder el pendiente.
- Lotes pequeños y frecuentes (máx. 1000 peticiones/operación LEMA).
- **Capturar en `localiza()`** `concepto`, organismo emisor (y raíz si se guarda), `tipoEnvio`: **no vuelven** en `peticionAcceso()`.

### RF-02 Almacenamiento — crítico

- Persistir documento principal + **todos** los anexos + acuse **en el mismo ciclo** que la comparecencia.
- Anexo URL directa: HTTP, sin LEMA. Anexo referencia: `consultaAnexos()`.
- Acuse: `consultaAcusePdf()` con `csvResguardo` de `peticionAcceso()`.
- Hash SHA-256 del principal vs el de DEHú (`Documento.verificarIntegridad()`).
- **Ventana 24 h:** `consultaAnexos` / `consultaAcusePdf` no sirven para notificaciones de más de 1 día. Fallo de descarga = alerta de alta prioridad, no reintento silencioso a 48 h.
- Tipos de `DOCUMENTO`: `principal` | `anexo` | `acuse`. Ficheros en almacenamiento (campo `rutaAlmacenamiento`); BBDD con metadatos y hash.

### RF-03 a RF-06 IA y contenido de ticket

- OCR/LLM **solo del documento principal**. Anexos y acuse se archivan y no entran al pipeline (KISS; pendiente confirmar con compañeros si el anexo aporta a clasificar).
- Salida: tipo, entidades (organismo, expediente, plazos, importes, partes), score, departamento, canal (`ticket` / `email`).
- Umbral configurable: por debajo → RF-09 (no publicar acción automática).
- RF-06: título, resumen, campos, adjuntos, trazabilidad `identificador` DEHú.
- n8n puede orquestar OCR/LLM; Java persiste `INTERPRETACION` + `TEXTOEXTRAIDO` y el historial `CLASIFICACION`.

### RF-07 Evento

Exactamente un evento por comunicación que complete RF-06. Payload: contenido RF-06 + departamento/cola + identificador DEHú. Publicar aunque aún no haya consumidor. Infra: AYEventos / n8n. Trazabilidad en BD via `CLASIFICACION` + `DERIVACION` (no hay tabla `EVENTO`).

### RF-08 Derivación

- Canal por `Clasificacion.canalAsignado` / catálogo `DEPARTAMENTO`, **no** `if` por tipo de comunicación.
- MVP obligatorio: ticket. Email (RF-08.2) deseable, no bloqueante. Implementar `EmailNotificacionGateway` si hay tiempo; el Resolver ya debe admitirlo.
- Idempotencia: no repetir acción si ya hay `DERIVACION` con `estado=exito` para ese identificador (`tieneDerivacionExitosa()`). Un `fallo` **sí** se reintenta.
- **Excepción:** reclasificación (09.6, 10.4, 11.5) = finalizar ticket original + crear uno nuevo. Segundo ticket **no** es duplicado. `esReclasificacion=true`, `ticketRelacionadoId` = ticket original. Ticketing **no** transfiere de cola (`PATCH` no toca `cola`).

### RF-09 Revisión humana

- Confianza baja → `REVISION` (`resuelto=false`) y cola específica. Ninguna comunicación se pierde entre pasos.
- UI: documento como vista principal; texto extraído como panel auxiliar (diseño de interfaz aún abierto).
- Aceptar (09.5) → RF-08 con la clasificación propuesta.
- Reclasificar (09.6) → solo administrador; finalizar+crear.
- Feedback (09.3) deseable, asíncrono, no bloquea el cierre.

### RF-10 Agente IA

- Entrada: evento de cancelación (fuera de HV; Ticketing outbox).
- Contador = filas `CLASIFICACION` con `origen=ia_reclasificacion` (no campo suelto). Límite **2**. Si ya está en el límite, escalar a RF-09 **sin** llamar al agente.
- Si hay margen: `AgenteIAGateway.intentarAutocorreccion`. El agente (MCP) propone y, si supera umbral, él invoca finalizar+crear (`AgenteIAClient` depende de `TicketingGateway`). Si no supera, escalar a revisión. No hay bucle interno: el siguiente intento es otra cancelación.
- **Diseño no cerrado con el equipo:** si el client de infra debe llamar a Ticketing o el Service compara el umbral. Hasta confirmación, implementar lo diagramado (MCP actúa).

### RF-11 Consulta local

- Nunca llama a DEHú.
- Editable **solo** si existe al menos una `DERIVACION` (`tieneDerivacionAsociada()`). **No** usar `tipoEnvio`.
- Comunicación en revisión pendiente (sin derivación) = solo lectura.
- 11.4: mismo departamento → `modificarTicket` in-place (condicionado a API back de Ticketing).
- 11.5: cambia departamento → mismo finalizar+crear que 09.6.

### RF-12 Reconciliación (opcional)

`localizaRealizadas` no filtra por fecha. Batch, no consulta interactiva. Fuera del MVP comprometido.

---

## 6. Base de datos

Motor: SQL Server (`HV_OrganismosPublicos`). Persistencia: `JdbcTemplate` en `ComunicacionRepositorySQLServer`. Scripts en `HVOrganismosPublicosMigraciones`.

`USUARIO.id` = `PERSONA.id` de Personas (clave compartida, no FK física obligatoria a otro catálogo). `PERSONA` no se crea aquí.

### 6.1 Tablas

| Tabla | Cardinalidad desde COMUNICACION | Rol |
|---|---|---|
| `COMUNICACION` | raíz | Un envío DEHú. `estado` interno: `pendiente` → `en_proceso` → `en_revision` \| `procesada`. No es el `estado` de DEHú. |
| `DOCUMENTO` | 1:N | principal / anexo / acuse |
| `INTERPRETACION` | 1:1 opcional | Tras IA |
| `TEXTOEXTRAIDO` | 1:1 opcional desde interpretación | Texto pesado; retención por **parámetro global** (valor TBD con compañeros). Sin retención permanente para entrenamiento en el MVP. |
| `CLASIFICACION` | 1:N append-only | `origen`: `ia_inicial` \| `ia_reclasificacion` \| `operador` |
| `DERIVACION` | 1:N | Resultado RF-08. `estado` éxito/fallo. |
| `REVISION` | 1:N opcional | Una fila por escalada; pendiente = `resuelto=false` |
| `DEPARTAMENTO` | catálogo | `colaDestino`, `permiteEmail`, `activo` |
| `USUARIO` | catálogo | `rol`, `activo` |

Índice único recomendado: `COMUNICACION(identificador)` (idempotencia RF-01.3).

Campos mínimos (alineados al ER):

**COMUNICACION:** `id` UUID PK, `identificador`, `codigoOrigen`, `concepto`, `organismoEmisorCodigo`, `organismoEmisorNombre`, `tipoEnvio` int, `fechaEvento`, `estado`, `fechaIngesta`.

**DOCUMENTO:** `id`, `comunicacion_id`, `tipo`, `nombre`, `mimeType`, `hashSha256`, `csvResguardo`, `rutaAlmacenamiento`, `fechaDescarga`.

**INTERPRETACION:** `id`, `comunicacion_id`, `tipoDetectado`, `entidadesExtraidas` (JSON/nvarchar), `scoreConfianza`, `fechaProcesado`.

**TEXTOEXTRAIDO:** `id`, `interpretacion_id`, `textoExtraido`, `fechaCreacion`.

**CLASIFICACION:** `id`, `comunicacion_id`, `origen`, `departamentoAsignado`, `tipoAsignado`, `canalAsignado`, `scoreConfianza`, `resultado`, `fecha`.

**DERIVACION:** `id`, `comunicacion_id`, `canal`, `identificadorExterno`, `titulo`, `resumen`, `departamento` FK, `estado`, `esReclasificacion`, `ticketRelacionadoId`, `fechaEjecucion`.

**REVISION:** `id`, `comunicacion_id`, `usuario_id` nullable, `motivo`, `resuelto`, `fechaEntrada`, `fechaResolucion`.

**DEPARTAMENTO:** `id`, `nombre`, `colaDestino`, `permiteEmail`, `activo`.

**USUARIO:** `id`, `rol`, `activo`.

Semilla `DEPARTAMENTO`: siniestros, laboral, comercial/concursos, fiscal, … (valores reales con el negocio). Umbral de confianza y retención de texto: properties, no constantes compiladas.

Dominio (Information Expert / Creator): `Comunicacion` crea `Documento`, `Clasificacion`, `Derivacion`, `Revision`. Métodos: `tieneDerivacionAsociada`, `tieneDerivacionExitosa`, `tieneRevisionActiva`, `derivacionActiva`, `contarIntentosReclasificacion`, `registrarDerivacion`, `registrarClasificacion`, `escalarARevision`. `Derivacion.esModificableInPlace` / `actualizar`. `Revision.resolver(usuario)`.

---

## 7. Integraciones externas

### 7.1 LEMA (SOAP 1.1 + WS-Security X.509)

No REST. Certificado = identidad de la compañía. Prod: Seguridad confirma que existe. Pruebas: lo genera Sistemas. Custodia/renovación: Seguridad. Caducidad = corte total de ingesta.

Entornos LEMA: **SE** (Servicios Estables, pruebas) y **PRO**. Properties distintas.

Orden de llamadas:

```
localiza(nifTitular, pagina)
  → identificador, codigoOrigen → peticionAcceso
       → csvResguardo → consultaAcusePdf
       → anexosReferencia[].referenciaDocumento → consultaAnexos (uno a uno)
```

Campos de respuesta verificados: `DEHu_Campos_Respuesta_Servicios.md`. Binarios por MTOM. Paginación `localiza`: `hayMasResultados`, `totalPag`, `paginaActual`.

Alta Gran Destinatario (Declaración Responsable + validación SE) es dependencia de calendario, no código.

### 7.2 Ticketing (HTTP / OpenAPI)

Crear ticket en cola `DEPARTAMENTO.colaDestino`. Finalizar + crear en reclasificación, con enlace `ticketRelacionado`. Deduplicar por identificador externo DEHú: **preguntar al equipo** (RF-08.5).

Audiencia back para `modificarTicket`: no confirmada. Hasta entonces RF-11.4/11.5 es diseño, no contrato cerrado.

Cancelación: cambio de estado en Ticketing; **sin webhook**. Outbox interno; falta quién lo publica al bus (TODO).

Departamento trabaja en el frontal de Ticketing, no en HV.

### 7.3 Bus de eventos / n8n / AYEventos

n8n orquesta RF-01–RF-08. Publicador Java hacia el bus corporativo (mismo patrón que Ticketing/AYEventos). n8n y el bus se tratan como piezas distintas hasta que Infra confirme el cableado del evento de cancelación.

### 7.4 Agente IA / MCP

Proceso fuera de WAS. Java: `AgenteIAGateway`. Herramienta MCP del MVP: finalizar + crear ticket. OCR/LLM del pipeline inicial puede ser el mismo u otro proceso; no mezclar con el agente de reclasificación en el diseño de componentes.

### 7.5 Directorio / seguridad MGS

Login Operador: directorio de personal + `GestorBackendFilter` (como Ticketing Front, `codApp` propio cuando Infra lo asigne). `USUARIO` solo roles de esta app.

### 7.6 Correo

SMTP corporativo vía `EmailNotificacionGateway` si se implementa RF-08.2. No bloquea el cierre del MVP.

---

## 8. Listado de endpoints

OpenAPI 3, kebab-case plural, `x-area: hv`, `x-subarea: def`, `x-version: v1`. Prefijo gateway: `/api/{front|back|consumer}/hv/def/v1`. En WAS local el context-root Front es `HVOrganismosPublicos/api/front/v1` (los paths de la tabla van **después** de ese prefijo). No exponer SOAP ni el certificado al navegador.

Códigos: `401` no autenticado; `403` sin rol; `404` no existe; `409` regla de negocio (derivación duplicada, revisión ya resuelta, límite RF-10).

### 8.1 Front (`x-audiencia: front`) — Operador

Implementación: `RevisionesApiImpl`, `ComunicacionesApiImpl`. Autenticación: directorio + `GestorBackendFilter`.

| Método | Path | Rol | RF | Servicio |
|---|---|---|---|---|
| `GET` | `/revisiones` | operador, administrador | 09.1 | `ConsultarRevisionService` |
| `GET` | `/revisiones/{id}` | operador, administrador | 09.2 | `ConsultarRevisionService` (documento, texto extraído, propuesta IA) |
| `GET` | `/revisiones/{id}/documentos/{documentoId}` | operador, administrador | 09.2 | binario del documento (no llama a LEMA) |
| `POST` | `/revisiones/{id}/aceptacion` | operador, administrador | 09.5 | `AceptarClasificacionService` |
| `POST` | `/revisiones/{id}/reclasificacion` | **administrador** | 09.6 | `ReclasificarComunicacionService` (body: `departamento`, `tipo`) |
| `GET` | `/comunicaciones` | operador, administrador | 11.1 | `ConsultarComunicacionesService` (query: estado, texto; cada ítem trae `editable`) |
| `GET` | `/comunicaciones/{id}` | operador, administrador | 11.1–11.3 | detalle; `editable` ⇔ existe `DERIVACION` |
| `GET` | `/comunicaciones/{id}/documentos/{documentoId}` | operador, administrador | 11 | binario local |
| `PATCH` | `/comunicaciones/{id}` | operador, administrador | 11.4 / 11.5 | `ModificarDerivacionService` (body: `titulo`, `resumen`, `departamentoDestino`). **409** si no es editable. Condicionado a audiencia back de Ticketing |
| `GET` | `/departamentos` | operador, administrador | 09.6, 11.5 | catálogo `DEPARTAMENTO` activo (desplegable de reclasificación) |

### 8.2 Back (`x-audiencia: back`) — n8n — fase 2

Implementación: `IngestaApiImpl`, `InterpretacionesApiImpl`, `DerivacionesApiImpl`. Credencial de servicio, no el Operador.

| Método | Path | RF | Servicio |
|---|---|---|---|
| `POST` | `/ciclos-ingesta` | 01, 02 | `IngestarComunicacionesService` (`localiza` + persistencia inmediata; lista vacía → `204`) |
| `POST` | `/interpretaciones` | 03–06 | `RegistrarInterpretacionService` + `ClasificarComunicacionService` + `GenerarContenidoTicketService` (n8n envía OCR/LLM ya calculado) |
| `POST` | `/eventos/comunicacion-clasificada` | 07 | `PublicarComunicacionClasificadaService` (si n8n no publica al bus directamente) |
| `POST` | `/derivaciones` | 08 | `EjecutarDerivacionService` (consumo del evento RF-07) |

`POST /interpretaciones` y el motor OCR en Java (`InterpretarComunicacionService`) son alternativas: n8n usa una de las dos, no las dos en el mismo ciclo.

### 8.3 Consumer (`x-audiencia: consumer`) — fase 3

| Método | Path | RF | Servicio |
|---|---|---|---|
| `POST` | `/eventos/cancelaciones` | 10 | `AgenteReclasificacionService` (body: identificador DEHú o `comunicacionId`, `ticketOriginalId`) |

Si el bus entrega el evento por otro adapter (no HTTP), este path es el contrato interno que ese adapter invoca.

### 8.4 Rutas del frontal Vue

No son la API. El SPA llama a Front.

| Ruta Vue | Vista | Endpoints |
|---|---|---|
| `/revisiones` | `ListaRevisionesView` | `GET /revisiones` |
| `/revisiones/:id` | `DetalleRevisionView` | `GET /revisiones/{id}`, `POST …/aceptacion` o `…/reclasificacion` |
| `/comunicaciones` | `ListaComunicacionesView` | `GET /comunicaciones` |
| `/comunicaciones/:id` | `DetalleComunicacionView` | `GET /comunicaciones/{id}`, `PATCH` si `editable` |

Vue 3 + TypeScript + Vite + Pinia sobre **UIBaseProject**. Listado, detalle y un flujo de estado (revisión). Sin acabado visual interno de Ticketing.

---

## 9. Requisitos no funcionales

| Área | Implicación de código |
|---|---|
| Seguridad | Certificado X.509 solo en servidor; alertas de caducidad; logs sin volcar el binario del cert. |
| Acceso | Roles operador/administrador; auditoría de quién resolvió cada `REVISION`. |
| Conservación legal | Documentos oficiales + trazabilidad recepción → ticket. Texto extraído con retención configurable. |
| Resiliencia | Caídas SE/PRO sin perder ya detectadas; alerta si RF-02 falla dentro de 24 h. |
| Rendimiento | Lotes LEMA &lt; 1000; `TEXTOEXTRAIDO` fuera de listados. |
| Auditabilidad | Log de cada SOAP (resultado) y de cada clasificación + score. |
| Extensibilidad | Canal nuevo = nueva clase `NotificacionGateway`. Fuente nueva ≠ DEHú = otro adapter de entrada al mismo bus. |
| Entornos | local / desa / usua / prod + SE vs PRO LEMA. |

---

## 10. Configuración y operaciones

Properties (patrón Ticketing `{entorno}#….properties`):

- URL/WSDL LEMA SE y PRO, NIF titular, timeouts, tamaño de lote, frecuencia de sondeo (la frecuencia puede vivir en n8n).
- Umbral de confianza.
- Días de retención de `TEXTOEXTRAIDO`.
- JNDI datasource SQLPortal.
- URLs Ticketing Front/Back, AYEventos, endpoint del Agente IA.
- `PATH_PROPERTIES_SERVIDOR` (resource-env-ref WAS, como Ticketing).

`HVOrganismosPublicosServer`: wsadmin datasource + instalación del certificado para WS-Security.

Almacenamiento de binarios: ruta en servidor o volumen; no asumir FILESTREAM hasta decidirlo con Infra. La BBDD ya está pedida con tamaño extra.

`.gitignore` en raíz del repo HV: `.metadata/`, compilados, `tsclient/`, `node_modules`, Playwright, `.env.*`. Versionar `.project`, `.classpath`, `.settings`.

---

## 11. Pruebas

| Capa | Qué demuestra |
|---|---|
| `HVOrganismosPublicosTest` (JUnit 5) | Umbral, límite 2 de RF-10, idempotencia de derivación, finalizar+crear vs in-place, integridad hash, `tieneDerivacionExitosa` vs asociada. Gateways mockeados. |
| `npm run test` del Vue | `vue-tsc` + eslint, no dominio. |
| FT Playwright | Operador contra desa/usua, cuando exista entorno. |
| SE LEMA | Pruebas reales SOAP cuando haya certificado de pruebas y alta GD. |

Sin WAS en unitarios. Evidencia de evaluación del TFM = Test, no solo E2E.

---

## 12. Plan de construcción (slices)

Orden que permite ir cerrando RF sin tener n8n ni LEMA el primer día:

1. **Esquema + dominio** — migraciones, entidades, repositorio JDBC con tests de persistencia local si hay SQL; si no, tests del modelo en memoria.
2. **RF-08/09 en seco** — servicios + Test con `TicketingGateway` falso (aceptar, reclasificar, idempotencia).
3. **ApiFront + Web** — listado/detalle/revisión contra BD local.
4. **LemaGateway** — cliente SOAP; persistencia RF-01/02; alerta 24 h. ApiBack.
5. **RF-03–07** — persistir interpretación; publicar evento; n8n.
6. **Ticketing real** — crear ticket; deduplicado si la API lo permite.
7. **RF-11.4/11.5** — cuando exista audiencia back.
8. **RF-10 + Consumer** — cuando exista el evento de cancelación.
9. **Email / RF-12** — opcional.

Deadlines de negocio: desarrollo+testing hasta 31 dic 2026; revisión 4–15 ene 2027. Gantt en `Diagramas/Gantt.md`.

---

## 13. Decisiones abiertas (no inventar en código)

Hasta que se cierre, implementar el diseño actual y dejar puntos de extensión:

| Tema | Impacto |
|---|---|
| Retención de `TEXTOEXTRAIDO` | Valor del parámetro; no hardcodear años. |
| ¿El anexo aporta a clasificar? | Si sí, ampliar RF-03; hoy solo principal. |
| Audiencia back Ticketing | Bloquea cierre de RF-11.4/11.5. |
| Quién lee el outbox de cancelación | Bloquea RF-10 de verdad. |
| Deduplicado por identificador DEHú en POST tickets | RF-08.5. |
| `AgenteIAClient` → `TicketingGateway` | Confirmación de equipo. |
| Rol `consulta` | No modelar UI de departamento en HV. |
| `codApp` GestorBackend y grupos GIT | Infra. |
| Colas reales de Ticketing por departamento | Semilla `DEPARTAMENTO`. |

No republicar código interno de Ticketing (`Estado_Tecnico_Ticketing.md`, `Analisis_Frontend_AYTicketing.md`).

---

## 14. Invariantes (checklist al implementar)

1. Java EE 7 / `javax.*` / WAS 9 / Java 8. Nada de `jakarta.*` ni Spring ni JPA.
2. Constructor `@Inject`; tests con `new Servicio(...)`.
3. Persistencia LEMA en Java dentro de 24 h.
4. Metadatos de `localiza()` guardados antes de `peticionAcceso()`.
5. OCR solo del principal.
6. `CLASIFICACION` y `DERIVACION` se acumulan, no se pisan.
7. Editable RF-11 ⇔ existe `DERIVACION`, no `tipoEnvio`.
8. Reclasificación = finalizar + crear, nunca transferir cola.
9. Máximo 2 autocorrecciones IA; la 3.ª cancelación escala a humano.
10. Canal nuevo = clase nueva, no `switch` en el Service.
11. Agente IA fuera de WAS.
12. Sin DAO, sin EJB, sin EAR por cada Utility.
13. Email y RF-12 no bloquean el MVP; ticket sí.
14. No enviar nada a la Administración.
