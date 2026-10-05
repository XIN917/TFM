# PRD — HVOrganismosPublicos

Documento de construcción del aplicativo `HVOrganismosPublicos` (MGS Seguros): alcance, arquitectura, módulos, reglas de implementación, datos, integraciones, endpoints y criterios de aceptación. Contiene lo necesario para implementar sin consultar otros documentos.

| | |
|---|---|
| Producto | `HVOrganismosPublicos` |
| Área Eclipse | `HV` (Consulta) |
| Git | `HV/HVOrganismosPublicos` (workspace local: `Consultas/HVOrganismosPublicos`) |
| BBDD | `HV_OrganismosPublicos` en SQLPortal (DESA y PROD; tamaño extra por documentos) |
| Plataforma | Java 8, Java EE 7 (`javax.*`), WAS traditional 9, CDI, JAX-RS, JDBC. Sin JPA, Spring ni Jakarta. |
| Familia de referencia | AYTicketing / AYCalendarios (SQL propio, hexagonal, OpenAPI v1). No SISiso/Personas: no hay maestro CICS, así que no hay DAO ni EJB. |

---

## 1. Overview

**Problema:** cada departamento entra a DEHú con certificado local, revisa el buzón entero y decide qué es suyo. Hay carga manual, certificados en los puestos y comunicaciones que se pierden o llegan tarde.

**Solución:** sustituir la consulta en pantalla por LEMA (servicios web de Gran Destinatario). El sistema sondea DEHú, clasifica cada envío por departamento sin abrir el documento, descarga y persiste los documentos, y crea un ticket en la cola de Ticketing del departamento. Si la clasificación no alcanza el umbral, la revisa un Operador. Si el departamento cancela el ticket por estar en la cola equivocada, un Agente IA intenta corregirlo, como máximo dos veces.

El nombre `OrganismosPublicos` deja abierta la puerta a otras fuentes, pero el MVP es solo DEHú/LEMA.

### 1.1 Ciclo del MVP

1. Detectar el envío con `localiza()` y guardar los metadatos que `peticionAcceso()` no devuelve.
2. Clasificar por departamento antes de abrir. La señal base es organismo emisor y concepto. No se lee el documento.
3. Según `tipoEnvio`, solo si la confianza supera el umbral:
   - Comunicación (`1`): descargar en el mismo ciclo, resumir el documento principal y seguir.
   - Notificación (`2`): el lote de madrugada comparece, guarda documento, anexos y acuse cuando existan, y deriva enseguida. No se resume.
4. Publicar el evento de comunicación clasificada (RF-07).
5. Ejecutar la acción del canal: ticket obligatorio, email deseable (RF-08).
6. Revisión humana (RF-09) y reclasificación por Operador o Agente IA (RF-10).

Generar el evento (RF-07) y ejecutar la acción (RF-08) son pasos separados, para que Ticketing no sea la única salida posible.

### 1.2 Fuera de alcance

- Enviar cualquier cosa a la Administración. LEMA es solo lectura.
- Usar los correos de aviso (sede de la DGSFP o DEHú) como fuente. La fuente es DEHú.
- Automatismos que abran expedientes o actúen sobre otros sistemas de negocio.
- MCP genérico: el Agente IA solo puede finalizar un ticket y crear otro (RF-10).
- Multi-certificado / multi-razón social.
- Buzón interno (RF-08.3; se mantiene el hueco de numeración).
- Rol `consulta` para departamentos en el frontal HV, hasta que decidan IT y Seguridad.
- Aprendizaje continuo del modelo con el feedback del Operador. Persistir ese feedback sí es deseable (RF-09.3).
- RF-12 (`localizaRealizadas` / `consultaRealizadas`): reconciliación y evaluación de la clasificación con `REALIZADAS`, solo si sobra tiempo.

---

## 2. Actores y roles

| Actor externo | Qué hace |
|---|---|
| DEHú/LEMA | Fuente SOAP. El sistema llama; LEMA no empuja. |
| Operador / Administrador | Usa el frontal HV: cola de revisión y consulta local (RF-09, RF-11). |
| Departamento | Trabaja en el frontal de Ticketing, no en HV. Si cancela un ticket, dispara RF-10. |

Roles en `USUARIO` (autenticación contra el directorio de personal MGS):

| Rol | Permisos |
|---|---|
| `operador` | Ver la cola de revisión y aceptar la propuesta de la IA. |
| `administrador` | Lo mismo que `operador`, y además reclasificar (RF-09.6). |

Cada resolución de `REVISION` guarda el `USUARIO` que la hizo.

---

## 3. Arquitectura

- **Entre componentes:** orientada a eventos (bus corporativo / n8n). Ingesta, ejecución y reclasificación no se llaman en cadena.
- **Dentro de Java:** hexagonal: `modelo` / `aplicacion` / `infraestructura` / `api`.

n8n orquesta: dispara el sondeo y el OCR/LLM, y publica y consume eventos. Java es dueño del estado y de la ventana de 24 h: el conector LEMA y la persistencia de documentos viven en Java, no en un flujo n8n.

```
DEHú/LEMA (SOAP 1.1 + WS-Security X.509)
        │
        ▼
  LemaGateway ──► JDBC ──► HV_OrganismosPublicos
        │                    ▲
        ▼                    │
  OCR/LLM (n8n) ── persiste INTERPRETACION / TEXTOEXTRAIDO
        │
        ▼
  Publicador RF-07 ──► Bus / n8n ──► EjecutarDerivacionService
                                          │
                          TicketingGateway ┴ EmailNotificacionGateway
                                          │
Departamento cancela ticket ─(outbox de Ticketing)─► evento RF-10
                                          │
                              AgenteReclasificacionService
                                          │
                              AgenteIAGateway (MCP, proceso externo)

Operador ──► Vue Web ──► ApiFront (JAX-RS) ──► servicios de revisión/consulta
```

| Bloque | RF | Dónde |
|---|---|---|
| Ingesta y clasificación | 01–07 | n8n dispara; `LemaGateway`, repositorios y persistencia IA en Java |
| Ejecución de canal | 08 | `EjecutarDerivacionService` + `NotificacionGateway` |
| Reclasificación automática | 10 | `ApiConsumer` + `AgenteReclasificacionService` + MCP fuera de WAS |
| Revisión y consulta | 09, 11 | `ApiFront` + `HVOrganismosPublicosWeb` |

### 3.1 Convenciones Java

- Paquete raíz: `es.mgs.hv.organismosPublicos`.
- Estereotipos como Ticketing: `@Servicio`, `@Repositorio`, `@Endpoint`, `@Transaccional`. Sin `AggregateRoot` ni eventos de dominio CDI.
- Cada `*Service` recibe sus dependencias por constructor con `@Inject`. Los tests lo instancian con `new Servicio(gatewayFalso, repoFalso)`, sin servidor.
- `beans.xml` con `bean-discovery-mode="annotated"` en Business y Comun (`src/META-INF`) y en Front y Web (`WebContent/WEB-INF`).
- `modelo` solo depende de interfaces; las implementaciones están en `infraestructura`.
- Identificadores: `java.util.UUID`, generado por el dominio al crear la entidad, no por la BBDD.
- Patrones: Strategy + Adapter en canales (`NotificacionGateway`), Gateway y Repository (Fowler), `NotificacionGatewayResolver` con `@Inject @Any Instance<NotificacionGateway>`. Canal nuevo = clase nueva, sin `switch` en el servicio.

### 3.2 Qué no crear

- DAO, EJB/EJBClient/Utils, WebService SOAP propio ni EAR batch aparte.
- El sondeo es un endpoint Back, lo dispare n8n o un timer del mismo WAR.
- El Agente IA no es un WAR en WAS. En Java solo existe `AgenteIAClient`.

---

## 4. Estructura de directorios

`# UML` = clase del modelo de diseño (contrato en §4.1).

```
HVOrganismosPublicos/
├── .gitignore
│
├── HVOrganismosPublicosBeans/
│   └── src/es/mgs/hv/organismosPublicos/bean/
│       ├── eventos/
│       │   ├── EventoComunicacionClasificada.java      # RF-07
│       │   └── EventoCancelacionTicket.java            # RF-10.1
│       ├── exception/
│       │   ├── OrganismosPublicosException.java
│       │   ├── EntidadNoEncontradaException.java
│       │   └── ReglaNegocioException.java                  # 409; el mensaje distingue el caso
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
│           ├── logging/
│           │   └── ConfiguradorLogs.java
│           ├── properties/                                   # carga de ficheros por entorno
│           ├── exceptions/
│           │   ├── PeticionNoValidaException.java          # entrada mal formada; la rechaza el adaptador REST
│           │   └── IntegrationException.java               # fallo de un adaptador de salida
│           └── util/
│               ├── Fechas.java
│               ├── ConversionUtils.java
│               └── ObjectMappers.java
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
│           │   ├── Pagina.java
│           │   ├── TipoEnvio.java
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
│               │   ├── jdbc/
│               │   │   └── JdbcTemplate.java
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
│   ├── build.xml                                                    # ant generate
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
│   │   │   ├── CacheControlFilter.java
│   │   │   ├── Cached.java
│   │   │   └── LoggingFilter.java
│   │   ├── exception/mappers/                              # uno por excepción de Beans y Comun; UnknownExceptionMapper, WebApplicationExceptionMapper y RespuestasError
│   │   └── providers/
│   │       ├── GestorBackendFeature.java
│   │       └── JsonProvider.java
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
├── HVOrganismosPublicosTest/                                        # el test vive en el mismo paquete que la clase; JUnit 5 y fakes de los puertos
│
└── HVOrganismosPublicosFT/                                          # fase 3
    └── tests/
        ├── operador-revision.spec.ts
        └── operador-consulta.spec.ts
```

| Módulo | Contenido |
|---|---|
| Beans | Excepciones de aplicación (`EntidadNoEncontradaException`, `ReglaNegocioException`) y DTOs de evento. El 409 es `ReglaNegocioException`. Sin SQL ni SOAP. |
| Comun | log4j2, carga de properties, `PeticionNoValidaException`, `IntegrationException` y `ObjectMappers`. |
| Business | Modelo, aplicación e infraestructura (`JdbcTemplate`, `*SQLServer`, `LemaClient`, `TicketingClient`, `AgenteIAClient`). |
| ApiFront + WasEAR | REST del Operador. EAR: WAR + Business + Beans + Comun. |
| Web + WebEAR | SPA Vue (UIBaseProject) y servlets de seguridad. EAR: WAR + Comun, sin Business. |
| ApiBack | Endpoints que llama n8n: sondeo, interpretación, derivación. |
| ApiConsumer | Evento de cancelación (RF-10). |
| Migraciones | SQL del esquema. |
| Properties | `{local\|desa\|usua\|prod}#*.properties`, en el disco del servidor (`PATH_PROPERTIES_SERVIDOR`). |
| Server | Scripts wsadmin: datasource SQLPortal y certificado LEMA. |
| Test | JUnit 5, sin WAS. Mismo paquete que la clase probada y fakes de los puertos. |
| FT | Playwright E2E contra desa/usua. |

Classpath (ya aplicado en RAD): Business → Beans + Comun; Test → Business; Front → Beans + Comun + Business; Web → Comun. `/lib` de WasEAR: Beans, Comun, Business. `/lib` de WebEAR: Comun.

### 4.1 Contrato de clases

| Clase | Métodos |
|---|---|
| `Comunicacion` | `tieneDerivacionAsociada`, `tieneDerivacionExitosa`, `tieneRevisionActiva`, `derivacionActiva`, `contarIntentosReclasificacion`, `registrarDerivacion`, `registrarClasificacion`, `escalarARevision`. Crea `Documento`, `Clasificacion`, `Derivacion` y `Revision`. |
| `Documento` | `verificarIntegridad` |
| `Derivacion` | `esModificableInPlace`, `actualizar` |
| `Revision` | `resolver(Usuario)` |
| `ComunicacionRepository` | `save`, `query`, `queryPorIdentificadorDehu`, `queryListado`, `queryRevisionesPendientes` |
| `LemaGateway` | `localiza`, `peticionAcceso`, `consultaAnexos`, `consultaAcusePdf` (implementado por `LemaClient`) |
| `TicketingGateway` | `crearTicket`, `finalizarTicket`, `modificarTicket` |
| `NotificacionGateway` | `soportaCanal`, `ejecutar` |
| `AgenteIAGateway` | `intentarAutocorreccion` |

`ConstantesDominio`:

- Estado de `COMUNICACION`: `pendiente`, `en_proceso`, `en_revision`, `procesada` (`pendiente → en_proceso → en_revision / procesada`). Es el ciclo interno del sistema, no un estado de DEHú.
- Origen de clasificación: `ia`, `operador`.
- Canal: `ticket`, `email`.
- Tipo de documento: `principal`, `anexo`, `acuse`.
- Rol: `operador`, `administrador`.

Dependencias de cada servicio:

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

---

## 5. Requisitos funcionales

Solo las reglas que el código no puede violar.

### RF-01 Detección

- n8n o un timer llama al endpoint Back, que ejecuta `localiza()`. Si la lista está vacía, no se hace nada.
- Guardar, al procesar `localiza()` y antes de `peticionAcceso()`, los campos que esa segunda respuesta no devuelve: `concepto`, organismo emisor, organismo emisor raíz, `fechaPuestaDisposicion`, `tipoEnvio`, `vinculo`, titular y `metadatosPublicos`. `metadatosPublicos` se guarda tal como llega; no se interpreta ni entra en la clasificación.
- Idempotencia por `identificador` DEHú: un envío ya registrado no se reprocesa.
- Reintentos con backoff ante fallos SOAP, de certificado o de red, sin perder el envío pendiente.
- Lotes de menos de 1000 peticiones por operación LEMA.
- Comunicación (`tipoEnvio` `1`): `peticionAcceso(identificador, codigoOrigen)` en el mismo ciclo. Acceder no tiene efectos jurídicos y no hay acuse.
- Notificación (`tipoEnvio` `2`): el sondeo no comparece. `peticionAcceso()` es la comparecencia y puede abrir el plazo de respuesta. La ejecuta el lote de madrugada, solo si la clasificación ya superó el umbral: abre, guarda anexos y acuse cuando existan, y deriva enseguida. Por debajo del umbral no se abre.

### RF-02 Almacenamiento (crítico)

- En el mismo ciclo que `peticionAcceso()`, persistir el documento principal y todos los anexos. En notificaciones, también el acuse.
- Anexo con URL directa: descarga HTTP, sin LEMA. Anexo por referencia: `consultaAnexos()`, uno a uno.
- Acuse: `consultaAcusePdf()` con el `csvResguardo` de `peticionAcceso()`. Solo en notificaciones.
- `consultaAnexos()` y `consultaAcusePdf()` solo funcionan durante 24 h desde `peticionAcceso()`. Pasado ese plazo, el documento no se puede recuperar por API. Si la descarga falla, alerta de alta prioridad; no reintentos silenciosos.
- `Documento.verificarIntegridad()`: el SHA-256 del principal debe coincidir con el de DEHú.
- Binarios en almacenamiento de ficheros (`rutaAlmacenamiento`); en BBDD solo metadatos y hash.
- Guardar en `DOCUMENTO.metadatos` el `documento.metadatos` del principal y el `acusePdf.metadatos` del acuse, tal como llegan y sin interpretar. Si no se guardan al recibirlos, no se pueden recuperar: `consultaRealizadas()` no trae el del documento, y el acuse solo se puede volver a pedir durante 24 h.
- Guardar también `algoritmoHash` y, en cada anexo por referencia, `referenciaDocumento`. El binario va a fichero. No se persiste la paginación de `localiza()`.

### RF-05 Clasificación

- Se clasifica antes de abrir nada. La señal base es organismo emisor y concepto de `localiza()`. No se leen principal, anexos ni acuse. Otro metadato de `localiza()` solo entra si una muestra real demuestra que discrimina. El concepto se reconoce por la materia, sin igualdad exacta. Los DIR3 observados no son la clave universal del organismo.
- Resultado: el departamento. No hay catálogo de tipos. El prompt se construye solo con `DEPARTAMENTO.criteriosReparto` (reglas confirmadas y ya realizables con `localiza()`).
- Umbral de confianza configurable. Por debajo: `REVISION` (RF-09); no se abre el envío ni se publica acción automática.
- DGSFP no va a un solo departamento: la materia separa Coordinación DGS, SAC y Red de Mediación. Exclusivos y Mediación entran por concepto. Distribución → Red de Mediación está confirmada en negocio y fuera del prompt: el área remitente se ve en un aviso de la sede y no es un campo documentado de `localiza()`.
- TGSS salvo embargo → RRHH está fuera del prompt hasta validarlo con RRHH. No consultar hoy el buzón no excluye a un departamento como destino.

### RF-03/04 Extracción y resumen

- OCR/LLM solo del documento principal de las comunicaciones, para el resumen. Anexos y acuse se archivan sin procesar.
- Las notificaciones no se resumen en este alcance. El lote de madrugada las archiva. El resumen se retomará cuando la clasificación por departamento esté cerrada.
- Salida: entidades (organismo, expediente, plazos, importes, partes).
- n8n puede orquestar el OCR/LLM; Java persiste `INTERPRETACION`, `TEXTOEXTRAIDO` y `CLASIFICACION`.

### RF-06 Contenido del ticket

Título, resumen (si lo hay), campos, adjuntos e `identificador` DEHú.

### RF-07 Evento

- Exactamente un evento por comunicación con contenido RF-06 completo.
- Payload: contenido RF-06, departamento/cola e `identificador` DEHú.
- Se publica aunque no haya consumidor. Infra: AYEventos / n8n.
- No hay tabla `EVENTO`: la trazabilidad es `CLASIFICACION` + `DERIVACION`.

### RF-08 Derivación

- El canal sale de `Clasificacion.canalAsignado` y del catálogo `DEPARTAMENTO`, no de un `if` por tipo.
- Ticket obligatorio. Email (RF-08.2) no obligatorio, pero planificado justo después del ticket (`EmailNotificacionGateway`). Si no da tiempo, no bloquea el cierre del MVP.
- Idempotencia: si ya hay una `DERIVACION` con `estado=exito` (`tieneDerivacionExitosa()`), no se repite. Un `fallo` sí se reintenta.
- Reclasificación (RF-09.6, RF-10.4, RF-11.5): finalizar el ticket original y crear uno nuevo con `esReclasificacion=true` y `ticketRelacionadoId` = ticket original. No cuenta como duplicado. Ticketing no transfiere de cola (`PATCH` no toca `cola`).

### RF-09 Revisión humana

- Confianza baja → `REVISION` con `resuelto=false`. Ninguna comunicación se pierde entre pasos.
- UI (09.2): metadatos de `localiza()` y clasificación propuesta. No se abre el documento para decidir el departamento.
- Aceptar (09.5) fija el departamento. Una comunicación sigue el ciclo de descarga del sondeo. Una notificación no se abre aquí: queda para el lote de madrugada, que archiva y deriva.
- Reclasificar (09.6) → solo `administrador`. Elige el departamento; finalizar + crear.
- Feedback (09.3): deseable, asíncrono, no bloquea el cierre.

### RF-10 Agente IA

- Entrada: evento de cancelación del ticket (outbox de Ticketing).
- Intentos = filas `CLASIFICACION` con `origen=ia` y `nIntentos>1`. Límite: 2.
- En el límite: escalar a RF-09 sin llamar al agente.
- Con margen: `AgenteIAGateway.intentarAutocorreccion`. El agente propone; si supera el umbral, él mismo finaliza + crea (`AgenteIAClient` usa `TicketingGateway`). Si no, escala a revisión.
- No hay bucle interno: el siguiente intento llega con la siguiente cancelación.

### RF-11 Consulta local

- Nunca llama a DEHú.
- Editable si y solo si existe al menos una `DERIVACION` (`tieneDerivacionAsociada()`), no según `tipoEnvio`. En revisión pendiente, solo lectura.
- 11.4, mismo departamento: `modificarTicket` in-place (depende de la audiencia back de Ticketing).
- 11.5, cambio de departamento: finalizar + crear, como en 09.6.

### RF-12 Reconciliación y evaluación (opcional)

Batch con `localizaRealizadas`, que no filtra por fecha y exige `tipoEnvio`. No es consulta interactiva. «Realizada» quiere decir que DEHú ya no la tiene pendiente (aceptada, rechazada o expirada), no que un departamento la haya tramitado. Una notificación comparecida pasa a realizadas.

- Un identificador vive en un solo sitio, y manda `COMUNICACION`. Lo que ya está ahí se descarta.
- `fechaPuestaDisposicion` posterior al arranque y ausente en `COMUNICACION`: anomalía de reconciliación (RF-12.3). También aparece si un área lo abrió a mano en el portal.
- Anterior al arranque y ausente en `COMUNICACION`: va a `REALIZADAS` (RF-12.4), en producción, porque el buzón de pruebas no tiene los envíos reales de MGS. No tiene efectos jurídicos.
- `REALIZADAS` se clasifica con el mismo prompt y umbral (RF-12.5) y nunca crea ticket, revisión ni derivación.
- Descarga opcional del principal de una muestra con `consultaRealizadas()` (RF-12.6), para analizar o etiquetar. No entra en la clasificación. Anexos y acuse no se piden. Acceso restringido y borrado al terminar la evaluación.

---

## 6. Base de datos

SQL Server (`HV_OrganismosPublicos`), JDBC propio (`JdbcTemplate` + `*RepositorySQLServer`). Scripts en `HVOrganismosPublicosMigraciones`.

| Tabla | Relación con `COMUNICACION` | Rol |
|---|---|---|
| `COMUNICACION` | raíz | Un envío DEHú. `estado` interno (no el de DEHú). |
| `DOCUMENTO` | 1:N | `principal` / `anexo` / `acuse` |
| `INTERPRETACION` | 1:1 opcional | Resultado de la IA |
| `TEXTOEXTRAIDO` | 1:1 opcional desde `INTERPRETACION` | Texto completo, con retención configurable |
| `CLASIFICACION` | 1:N, solo inserciones | Historial de clasificaciones |
| `DERIVACION` | 1:N, solo inserciones | Resultado de RF-08 |
| `REVISION` | 1:N opcional | Una fila por escalado |
| `DEPARTAMENTO` | catálogo | Colas, canales y criterios de reparto |
| `USUARIO` | catálogo | Roles de la app |
| `REALIZADAS` | ninguna | Envíos ya realizados para evaluar la clasificación (RF-12.4–12.6). Fuera del flujo |

Campos:

- **COMUNICACION:** `id` (UUID, PK), `identificador` (índice único), `codigoOrigen`, `concepto`, `organismoEmisorCodigo`, `organismoEmisorNombre`, `organismoEmisorRaizCodigo`, `organismoEmisorRaizNombre`, `fechaPuestaDisposicion`, `tipoEnvio`, `vinculo`, `titularNombre`, `titularNif`, `fechaEvento`, `estado` (interno), `metadatosPublicos` (base64 de `localiza()`, `nvarchar(max)`; no se interpreta ni entra en el prompt), `fechaIngesta`.
- **DOCUMENTO:** `id`, `comunicacion_id`, `tipo`, `nombre`, `mimeType`, `hashSha256`, `algoritmoHash`, `csvResguardo` (el del documento en `peticionAcceso()`), `referenciaDocumento` (anexo por referencia), `metadatos` (`documento.metadatos` o `acusePdf.metadatos` en base64, `nvarchar(max)`; vacío en anexos; no se interpreta), `rutaAlmacenamiento`, `fechaDescarga`.
- **INTERPRETACION:** `id`, `comunicacion_id`, `entidadesExtraidas` (JSON en `nvarchar`), `scoreConfianza`, `fechaProcesado`.
- **TEXTOEXTRAIDO:** `id`, `interpretacion_id`, `textoExtraido`, `fechaCreacion`.
- **CLASIFICACION:** `id`, `comunicacion_id`, `origen` (`ia` | `operador`), `modelo` (solo si hubo LLM), `nIntentos` (`1`, `2`, `3`…; solo si `origen=ia`), `usuario_id` (solo si `origen=operador`), `departamentoAsignado` (obligatorio), `canalAsignado`, `scoreConfianza`, `resultado`, `fecha`.
- **DERIVACION:** `id`, `comunicacion_id`, `canal`, `identificadorExterno`, `titulo`, `resumen`, `departamento` (FK), `estado` (`exito` | `fallo`), `esReclasificacion`, `ticketRelacionadoId`, `fechaEjecucion`.
- **REVISION:** `id`, `comunicacion_id`, `usuario_id` (nulo hasta resolver), `motivo`, `resuelto`, `fechaEntrada`, `fechaResolucion`.
- **DEPARTAMENTO:** `id`, `nombre`, `colaDestino`, `permiteEmail`, `criteriosReparto` (texto), `activo`.
- **USUARIO:** `id` (= `PERSONA.id` de Personas, sin FK física), `rol`, `activo`.
- **REALIZADAS:** `id`, `identificador` (único; nunca presente en `COMUNICACION`), `codigoOrigen`, `concepto`, `organismoEmisorCodigo`, `organismoEmisorNombre`, `organismoEmisorRaizCodigo`, `organismoEmisorRaizNombre`, `fechaPuestaDisposicion`, `tipoEnvio`, `vinculo`, `titularNombre`, `titularNif`, `estadoDehu`, `referenciaPdfAcuse`, `csvResguardo` (del envío), `rutaDocumento`, `hashSha256`, `algoritmoHash` (solo si se descargó el principal), `departamentoEsperado` (FK, etiquetado), `departamentoPropuesto` (FK), `scoreConfianza`, `modelo`, `fechaClasificacion`, `fechaCarga`.

Semilla de `DEPARTAMENTO`. `criteriosReparto` es el texto del prompt: solo reglas confirmadas cuyo dato ya está en `localiza()`. Los DIR3 son ejemplos observados (`E00119006` DGSFP, `EA0028512` AEAT, `E00127105` Catastro, `EA0042298` TGSS); no son la clave de la regla. Faltan las colas reales de Ticketing.

| Departamento | `criteriosReparto` |
|---|---|
| Coordinación DGS | DGSFP: Inspección, análisis de balances, Solvencia, DEC |
| SAC | DGSFP: Modelo 2B, Reclamaciones. El concepto puede llevar variación de formato |
| Red de Mediación | DGSFP: Exclusivos, Mediación |
| Asesoría Fiscal | AEAT; Dirección General del Catastro; TGSS cuando el concepto es embargo o levantamiento de embargo, aunque venga abreviado |
| Canal Directo | Entidad Pública Empresarial red.es, con concepto o referencia asociados a Kit Digital (por ejemplo KD). «Kit Digital» no es un literal obligatorio |
| Servicio Jurídico | AEPD |

Fuera del prompt, y por tanto sin fila activa de criterio:

- DGSFP + Distribución → Red de Mediación. Regla de negocio confirmada. El área remitente se ve en un aviso de la sede, no en `localiza()` documentado. Entra si una muestra real lo muestra en `concepto`, `metadatosPublicos` u otro metadato previo a comparecer.
- DGSFP + Corredores → Red de Mediación. Observado, sin consolidar.
- TGSS salvo embargo → RRHH. Identificado por Asesoría Fiscal; pendiente de validación directa con RRHH. RRHH no consulta hoy el buzón DEHú.
- Siniestros y Sucursal Central. Sin regla hasta cerrar su catálogo. Un ayuntamiento no implica un departamento.

Agrupar varias notificaciones por referencia de acuerdo es una necesidad identificada en Canal Directo y no es requisito del MVP. La clasificación de ese departamento es la fila de arriba.

---

## 7. Integraciones externas

### 7.1 LEMA

- SOAP 1.1 firmado con certificado X.509 (WS-Security). No hay REST, token ni usuario.
- El certificado es la identidad de la compañía. Producción: ya existe. Pruebas: autofirmado, lo genera Sistemas. Custodia y renovación: Seguridad. Si caduca, se corta la ingesta.
- Entornos: SE (pruebas, `se-gd-dehuws.redsara.es`) y PRO (producción, `gd-dehuws.redsara.es`), con properties distintas.
- El alta de Gran Destinatario (Declaración Responsable + validación en SE) es una dependencia de calendario, no de código.

Orden de llamadas:

```
localiza(nifTitular, pagina)
  → identificador, codigoOrigen → peticionAcceso
       → csvResguardo → consultaAcusePdf                       (solo notificaciones)
       → anexosReferencia[].referenciaDocumento → consultaAnexos (uno a uno)
```

```
localizaRealizadas(nifTitular, tipoEnvio)
  → identificador, codigoOrigen, concepto → consultaRealizadas
```

Binarios por MTOM. Paginación de `localiza()`: `hayMasResultados`, `opcionesRespuestaLocaliza` (`totalResultados`, `totalPag`, `paginaActual`). Paginación de `localizaRealizadas()`: `totalPaginas`, `paginaActual`, sin `hayMasResultados` en el ejemplo.

Campos confirmados en los ejemplos del Anexo I de la guía de integración:

| Servicio | Elemento | Destino |
|---|---|---|
| `localiza()` (por `item` de `envios`) | `identificador`, `codigoOrigen` | `COMUNICACION` |
| | `concepto` | `COMUNICACION.concepto` |
| | `organismoEmisor.codigoOrganismo` / `.nombreOrganismo` | `organismoEmisorCodigo` / `organismoEmisorNombre` |
| | `organismoEmisorRaiz.codigoOrganismo` / `.nombreOrganismo` | `organismoEmisorRaizCodigo` / `organismoEmisorRaizNombre` |
| | `fechaPuestaDisposicion` (datetime ISO) | `fechaPuestaDisposicion` |
| | `tipoEnvio` (int: `1` comunicación, `2` notificación; otro valor es error) | `tipoEnvio` |
| | `vinculo` (int: `1` titular, `2` destinatario; si aparece como ambos, titular) | `vinculo` |
| | `titular.nombreTitular` / `titular.nifTitular` | `titularNombre` / `titularNif` |
| | `metadatosPublicos` (base64, contenido no documentado) | `COMUNICACION.metadatosPublicos` |
| `peticionAcceso()` | `fechaEvento` | `COMUNICACION.fechaEvento` |
| | `documento.nombre`, `documento.mimeType` | `DOCUMENTO` (principal) |
| | `documento.contenido` (MTOM, `href="cid:..."`) | Fichero (`rutaAlmacenamiento`) |
| | `documento.hashDocumento.hash` / `.algoritmoHash` (`sha256` en los ejemplos) | `hashSha256` / `algoritmoHash` |
| | `documento.metadatos` (base64) | `DOCUMENTO.metadatos` (principal) |
| | `documento.csvResguardo` | `DOCUMENTO.csvResguardo`; entrada de `consultaAcusePdf()` |
| | `anexos.anexosReferencia[].nombre` / `.mimeType` / `.referenciaDocumento` | `DOCUMENTO` (anexo); `referenciaDocumento` es la entrada de `consultaAnexos()` |
| `consultaAnexos()` | `documento.nombre`, `documento.contenido`, `documento.mimeType` | `DOCUMENTO` (anexo). No trae hash ni `csvResguardo` |
| `consultaAcusePdf()` | `acusePdf.nombreAcuse`, `acusePdf.contenido`, `acusePdf.mimeType` | `DOCUMENTO` (acuse) |
| | `acusePdf.metadatos` (en los ejemplos contiene el propio `csvResguardo`; formato por confirmar en pruebas) | `DOCUMENTO.metadatos` (acuse) |
| `localizaRealizadas()` | Los de `localiza()` salvo `metadatosPublicos`, que no aparece en el ejemplo | `REALIZADAS` |
| | `estado` (ej. `EXPIRADA`) | `REALIZADAS.estadoDehu` |
| | `referenciaPdfAcuse` (base64), `csvResguardo` | `REALIZADAS.referenciaPdfAcuse` / `.csvResguardo` |
| `consultaRealizadas()` | `identificador`, `codigoOrigen`, `fechaEvento`, `documento.nombre`, `documento.hashDocumento.hash` / `.algoritmoHash` | `REALIZADAS` (RF-12.6) o `COMUNICACION` (RF-12.3) |
| | `documento.contenido.tipoMIME` (anidado bajo `contenido`, a diferencia de `peticionAcceso()`) | Tipo MIME del principal |
| | `anexos.anexosReferencia[].nombre` / `.mimeType` / `.referenciaDocumento` | Detalle de RF-12.3. En RF-12.6 los anexos no se piden |

`peticionAcceso()` no devuelve `concepto`, `organismoEmisor`, `organismoEmisorRaiz`, `vinculo` ni `titular`. `consultaRealizadas()` no trae `csvResguardo` del documento. Los anexos con URL directa no vienen en `anexosReferencia`, y el nombre de su elemento no está confirmado en los ejemplos; no tienen columna hasta confirmarlo en pruebas.

Datos de cada petición:

| Servicio | Campos | Origen |
|---|---|---|
| `localiza()` | `nifTitular`; opcional `opcionesLocaliza.pagina` | NIF propio; paginación del sistema |
| `peticionAcceso()` | `identificador`, `codigoOrigen` | `item` de `localiza()` |
| `consultaAnexos()` | `nifReceptor`, `identificador`, `codigoOrigen`, `referencia` | `referencia` = `referenciaDocumento` de cada anexo de `peticionAcceso()` |
| `consultaAcusePdf()` | `nifReceptor`, `identificador`, `codigoOrigen` y una de dos: `identificadorAcusePdf.csvResguardo` o `identificadorAcusePdf.referencia` | `csvResguardo` = `documento.csvResguardo` de `peticionAcceso()` |
| `localizaRealizadas()` | `nifTitular`, `tipoEnvio` | NIF propio. Sin filtro por fecha |
| `consultaRealizadas()` | `identificador`, `codigoOrigen`, `nifPeticion`, `nombrePeticion`, `concepto` | `item` de `localizaRealizadas()` |

### 7.2 Ticketing

- HTTP / OpenAPI. Se crea el ticket en la cola `DEPARTAMENTO.colaDestino`.
- `modificarTicket` necesita la audiencia back, no confirmada.
- Las cancelaciones no salen por webhook sino por un outbox interno.

### 7.3 Bus de eventos / n8n

n8n orquesta RF-01 a RF-08. Java publica en el bus corporativo con el mismo patrón que Ticketing/AYEventos. n8n y el bus se tratan como piezas distintas hasta que Infra confirme el cableado.

### 7.4 Agente IA / MCP

Proceso fuera de WAS, accedido por `AgenteIAGateway`. Su única herramienta MCP es finalizar + crear ticket. El OCR/LLM de la ingesta es un componente distinto del agente de reclasificación.

### 7.5 Seguridad

Login del Operador contra el directorio de personal con `GestorBackendFilter`, como Ticketing Front (`codApp` pendiente de Infra). `USUARIO` solo guarda los roles de esta app.

### 7.6 Correo

SMTP corporativo vía `EmailNotificacionGateway` (RF-08.2).

---

## 8. Endpoints

OpenAPI 3, recursos en camelCase plural, `x-area: hv`, `x-subarea: def`, `x-version: v1`. Prefijo en el gateway: `/api/{front|back|consumer}/hv/def/v1`. Context root en WAS: Front `HVOrganismosPublicos/api/front/v1`, Web `appt/HVOrganismosPublicosWeb`. El navegador nunca ve SOAP ni el certificado.

Errores: `401` no autenticado, `403` sin rol, `404` no existe, `409` regla de negocio (derivación duplicada, revisión ya resuelta, límite de RF-10, comunicación no editable).

### 8.1 Front (Operador) — fase 1

| Método | Path | Rol | RF | Servicio |
|---|---|---|---|---|
| `GET` | `/revisiones` | operador, administrador | 09.1 | `ConsultarRevisionService` |
| `GET` | `/revisiones/{id}` | operador, administrador | 09.2 | `ConsultarRevisionService` (metadatos de `localiza()` y propuesta de departamento; el documento solo si ya consta abierto) |
| `GET` | `/revisiones/{id}/documentos/{documentoId}` | operador, administrador | 09.2 | binario local |
| `POST` | `/revisiones/{id}/aceptacion` | operador, administrador | 09.5 | `AceptarClasificacionService` |
| `POST` | `/revisiones/{id}/reclasificacion` | administrador | 09.6 | `ReclasificarComunicacionService` (body: `departamento`) |
| `GET` | `/comunicaciones` | operador, administrador | 11.1 | `ConsultarComunicacionesService` (query: `estado`, `texto`; cada ítem trae `editable`) |
| `GET` | `/comunicaciones/{id}` | operador, administrador | 11.1–11.3 | detalle con `editable` |
| `GET` | `/comunicaciones/{id}/documentos/{documentoId}` | operador, administrador | 11 | binario local |
| `PATCH` | `/comunicaciones/{id}` | operador, administrador | 11.4 / 11.5 | `ModificarDerivacionService` (body: `titulo`, `resumen`, `departamentoDestino`) |
| `GET` | `/departamentos` | operador, administrador | 09.6, 11.5 | `DEPARTAMENTO` activos |

### 8.2 Back (n8n) — fase 2

Credencial de servicio, no de Operador.

| Método | Path | RF | Servicio |
|---|---|---|---|
| `POST` | `/ciclosIngesta` | 01, 05 | `IngestarComunicacionesService`: `localiza()`, persiste metadatos y clasifica antes de abrir. Sin envíos nuevos → `204`. Una comunicación sobre el umbral sigue a `peticionAcceso()` en el mismo ciclo (RF-02). Una notificación sobre el umbral queda para el lote de madrugada. Por debajo del umbral, revisión y no se abre |
| `POST` | `/lotesMadrugada` | 01, 02, 06, 08 | Comparece las notificaciones ya clasificadas por encima del umbral, archiva documento, anexos y acuse cuando existan, y deriva. No vuelve a clasificar ni resume |
| `POST` | `/interpretaciones` | 03–06 | `RegistrarInterpretacionService` + `GenerarContenidoTicketService`. No clasifica. Solo hay documento que interpretar si ya se abrió |
| `POST` | `/eventos/comunicacionClasificada` | 07 | `PublicarComunicacionClasificadaService` (solo si n8n no publica al bus) |
| `POST` | `/derivaciones` | 08 | `EjecutarDerivacionService` |

El OCR/LLM se hace de una de dos formas, nunca las dos: n8n lo calcula y lo envía a `POST /interpretaciones`, o Java lo pide con `InterpretarComunicacionService`.

### 8.3 Consumer — fase 3

| Método | Path | RF | Servicio |
|---|---|---|---|
| `POST` | `/eventos/cancelaciones` | 10 | `AgenteReclasificacionService` (body: `identificador` DEHú o `comunicacionId`, y `ticketOriginalId`) |

Si el bus entrega el evento por otro medio, su adapter invoca este mismo contrato.

### 8.4 Rutas del frontal Vue

Vue 3 + TypeScript + Vite + Pinia sobre UIBaseProject, sin replicar el acabado visual de Ticketing.

| Ruta | Vista | Endpoints |
|---|---|---|
| `/revisiones` | `ListaRevisionesView` | `GET /revisiones` |
| `/revisiones/:id` | `DetalleRevisionView` | `GET /revisiones/{id}`, `POST …/aceptacion`, `POST …/reclasificacion` |
| `/comunicaciones` | `ListaComunicacionesView` | `GET /comunicaciones` |
| `/comunicaciones/:id` | `DetalleComunicacionView` | `GET /comunicaciones/{id}`, `PATCH` si `editable` |

---

## 9. Requisitos no funcionales

| Área | Implicación |
|---|---|
| Seguridad | Certificado solo en el servidor, con alerta de caducidad. Los logs no vuelcan el certificado. |
| Conservación legal | Documentos oficiales y trazabilidad de recepción a ticket. |
| Resiliencia | Una caída de SE/PRO no pierde envíos ya detectados. |
| Rendimiento | `TEXTOEXTRAIDO` fuera de los listados. |
| Auditabilidad | Log del resultado de cada llamada SOAP y de cada clasificación con su score. |
| Extensibilidad | Otra fuente distinta de DEHú = otro adapter de entrada al mismo bus. |
| Entornos | local / desa / usua / prod, y SE / PRO en LEMA. |

---

## 10. Configuración y operación

Properties (`{entorno}#organismosPublicos.properties`, patrón Ticketing):

- LEMA: URL/WSDL de SE y PRO, NIF titular, timeouts, tamaño de lote, frecuencia de sondeo (esta puede vivir en n8n).
- Umbral de confianza.
- Días de retención de `TEXTOEXTRAIDO`.
- JNDI del datasource SQLPortal.
- URLs de Ticketing Front/Back, AYEventos y Agente IA.
- `PATH_PROPERTIES_SERVIDOR` (resource-env-ref en WAS).

`HVOrganismosPublicosServer` configura el datasource e instala el certificado para WS-Security.

Binarios: ruta en el servidor o volumen. FILESTREAM no, salvo que se decida con Infra.

`.gitignore` del repo HV: `.metadata/`, compilados, `tsclient/`, `node_modules`, Playwright, `.env.*`. Se versionan `.project`, `.classpath` y `.settings`.

---

## 11. Pruebas

| Capa | Qué demuestra |
|---|---|
| `HVOrganismosPublicosTest` (JUnit 5) | Umbral, límite de 2 en RF-10, idempotencia de derivación, finalizar + crear frente a in-place, integridad del hash, `tieneDerivacionExitosa` frente a `tieneDerivacionAsociada`. Gateways simulados. |
| `npm run test` (Vue) | `vue-tsc` + eslint. |
| FT Playwright | Flujos del Operador contra desa/usua. |
| SE LEMA | Llamadas SOAP reales, con certificado de pruebas y alta de Gran Destinatario. |

La evidencia para el TFM son los tests de `HVOrganismosPublicosTest`, no los E2E.

### 11.1 Criterios de aceptación

| RF | Criterio |
|---|---|
| 01 | Ninguna comunicación pendiente permanece sin detectar más de N minutos (SLA a definir); ninguna se procesa dos veces. |
| 02 | El 100% de las notificaciones comparecidas en el día tienen su acuse y sus anexos por referencia almacenados antes de que expire la ventana de 24 h. Las comunicaciones no tienen acuse; sí tienen documento principal y anexos por referencia. |
| 03 | El documento principal de toda comunicación (`tipoEnvio` `1`) descargada en el mismo ciclo tiene una representación textual asociada, con indicador de calidad o confianza del OCR cuando aplique. Las notificaciones, los anexos y el acuse no entran. |
| 04 | Cada comunicación procesada produce una estructura con entidades y score de confianza, verificable por un humano. |
| 05 | Toda comunicación queda clasificada con un departamento y una confianza a partir de las señales base, o marcada para revisión humana sin haber abierto el documento. |
| 06 | El contenido generado es válido según el esquema de la API de Ticketing, sin intervención manual, en el 100% de los casos clasificados con confianza suficiente. |
| 07 | Toda comunicación que complete RF-06 genera exactamente un evento publicado, verificable con independencia del resultado de RF-08. |
| 08 | Toda comunicación clasificada con éxito produce exactamente una acción por el canal ticket, en un tiempo máximo definido por SLA. El email (RF-08.2) no es requisito de cierre. |
| 09 | El 100% de las comunicaciones detectadas tienen un estado final trazable: acción automática o revisión humana. Ninguna se pierde entre pasos. |
| 10 | Toda comunicación reportada como mal clasificada obtiene una resolución automática (ticket reclasificado) o una escalada a revisión humana. Nunca queda sin resultado ni requiere un aviso aparte de la creación del ticket. |
| 11 | El Operador consulta cualquier registro local sin depender de DEHú. Solo puede modificar el ticket si ya hay una derivación. La comunicación tal como la entregó DEHú nunca se modifica; la modificación actualiza el ticket in-place o crea uno nuevo según cambie o no la cola. |
| 12 | (si se implementa) Ningún envío realizado en DEHú queda sin registro local, en `COMUNICACION` o en `REALIZADAS`, dentro del margen de la frecuencia configurada. Ninguno está en las dos tablas. Ningún envío de `REALIZADAS` genera una acción. |

---

## 12. Plan de construcción

Fases por módulo:

| Fase | Módulos |
|---|---|
| 1 | Beans, Comun, Business, Front + Web, Migraciones, Properties, Server, Test |
| 2 | ApiBack |
| 3 | ApiConsumer y FT, cuando existan el evento de cancelación y el entorno desa |

Orden de trabajo, pensado para avanzar sin n8n ni LEMA al principio:

1. Esquema y dominio: migraciones, entidades, repositorios y tests del modelo.
2. RF-08/09 con `TicketingGateway` falso: aceptar, reclasificar, idempotencia.
3. ApiFront + Web contra BBDD local.
4. `LemaGateway`, RF-01/02, alerta de 24 h y ApiBack.
5. RF-03 a RF-07 y n8n.
6. Ticketing real.
7. Email (RF-08.2).
8. RF-11.4/11.5, cuando exista la audiencia back.
9. RF-10 y Consumer, cuando exista el evento de cancelación.
10. RF-12, opcional.

Fechas: backend y frontend antes del 25 dic 2026; desarrollo y testing hasta el 31 dic; revisión del 4 al 15 ene 2027.

---

## 13. Decisiones abiertas

Mientras no se cierren, se implementa lo descrito aquí sin fijar valores en código.

| Tema | Impacto |
|---|---|
| Cómo aparece Distribución en `localiza()` | La regla DGSFP + Distribución → Red de Mediación está confirmada y fuera del prompt. Hay que ver si el dato llega en `concepto`, `metadatosPublicos` u otro metadato previo a comparecer (RF-05). |
| Catálogo de Siniestros y Sucursal Central | Sin reglas activas hasta cerrarlo. Tampoco Corredores, ni TGSS → RRHH hasta validarlo con RRHH (RF-05). |
| Resumen de notificaciones | No forma parte de este alcance. Se retomará cuando la clasificación por departamento esté cerrada (RF-03/04). |
| Retención de `TEXTOEXTRAIDO` | Valor del parámetro. |
| Audiencia back de Ticketing | Bloquea RF-11.4/11.5. |
| Quién publica el outbox de cancelación en el bus | Bloquea RF-10. |
| Deduplicado por `identificador` DEHú al crear tickets | RF-08.5; preguntar al equipo de Ticketing. |
| Si el umbral de RF-10 lo compara el agente o el servicio | Se implementa lo diagramado: el agente compara y actúa. |
| `codApp` de GestorBackend y grupos de Git | Infra. |
| Servidor SMTP y remitente del email | Infra. Bloquea probar RF-08.2 contra correo real; el código va por properties. |
| Colas reales de Ticketing por departamento | Semilla de `DEPARTAMENTO`. |
| Alcance real de `localizaRealizadas()` | Cuántos devuelve, qué periodo cubre (el portal solo enseña 30 días), si el tipo `1` trae comunicaciones, si aparece al momento de comparecer y si trae `metadatosPublicos`. Condiciona RF-12.4. |
