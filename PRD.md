# PRD — HVOrganismosPublicos

Documento de construcción del aplicativo `HVOrganismosPublicos` (MGS Seguros): alcance, arquitectura, módulos, reglas de implementación, datos, integraciones y endpoints. El texto completo de cada RF está en `Especificacion_Requisitos.md`; campos de LEMA en `DEHu_Campos_Respuesta_Servicios.md`; diagramas en `Diagramas/`.

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

1. Detectar el envío con `localiza()`.
2. Clasificar con organismo emisor y concepto, sin leer el documento.
3. Según `tipoEnvio`:
   - Comunicación (`1`): descargar, resumir el documento principal y seguir.
   - Notificación (`2`): queda pendiente de comparecencia (sección 13). Al comparecer se descargan documentos y acuse; no se resume.
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
- RF-12 (`localizaRealizadas` / `consultaRealizadas`): solo si sobra tiempo.

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
- Patrones: Strategy + Adapter en canales (`NotificacionGateway`), Gateway y Repository (Fowler), `NotificacionGatewayResolver` con `@Inject @Any Instance<NotificacionGateway>`. Canal nuevo = clase nueva, sin `switch` en el servicio.

### 3.2 Qué no crear

- DAO, EJB/EJBClient/Utils, WebService SOAP propio ni EAR batch aparte.
- El sondeo es un endpoint Back, lo dispare n8n o un timer del mismo WAR.
- El Agente IA no es un WAR en WAS. En Java solo existe `AgenteIAClient`.

---

## 4. Estructura de directorios

`# UML` = clase de `Diagramas/Diagrama_Clases.md`.

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
| Comun | log4j2 y carga de properties. |
| Business | Modelo, aplicación e infraestructura (`*SQLServer`, `LemaClient`, `TicketingClient`, `AgenteIAClient`). |
| ApiFront + WasEAR | REST del Operador. EAR: WAR + Business + Beans + Comun. |
| Web + WebEAR | SPA Vue (UIBaseProject) y servlets de seguridad. EAR: WAR + Comun, sin Business. |
| ApiBack | Endpoints que llama n8n: sondeo, interpretación, derivación. |
| ApiConsumer | Evento de cancelación (RF-10). |
| Migraciones | SQL del esquema. |
| Properties | `{local\|desa\|usua\|prod}#*.properties`, en el disco del servidor (`PATH_PROPERTIES_SERVIDOR`). |
| Server | Scripts wsadmin: datasource SQLPortal y certificado LEMA. |
| Test | JUnit 5 + Mockito, sin WAS. |
| FT | Playwright E2E contra desa/usua. |

Classpath (ya aplicado en RAD): Business → Beans + Comun; Test → Business; Front → Beans + Comun + Business; Web → Comun. `/lib` de WasEAR: Beans, Comun, Business. `/lib` de WebEAR: Comun.

### 4.1 Contrato de clases

| Clase | Métodos |
|---|---|
| `Comunicacion` | `tieneDerivacionAsociada`, `tieneDerivacionExitosa`, `tieneRevisionActiva`, `derivacionActiva`, `contarIntentosReclasificacion`, `registrarDerivacion`, `registrarClasificacion`, `escalarARevision`. Crea `Documento`, `Clasificacion`, `Derivacion` y `Revision`. |
| `Documento` | `verificarIntegridad` |
| `Derivacion` | `esModificableInPlace`, `actualizar` |
| `Revision` | `resolver(Usuario)` |
| `ComunicacionRepository` | `save`, `query`, `siguienteId`, `queryPorIdentificadorDehu`, `queryListado`, `queryRevisionesPendientes` |
| `LemaGateway` | `localiza`, `peticionAcceso`, `consultaAnexos`, `consultaAcusePdf` (implementado por `LemaClient`) |
| `TicketingGateway` | `crearTicket`, `finalizarTicket`, `modificarTicket` |
| `NotificacionGateway` | `soportaCanal`, `ejecutar` |
| `AgenteIAGateway` | `intentarAutocorreccion` |

`ConstantesDominio`:

- Estado de `COMUNICACION`: `pendiente`, `en_proceso`, `en_revision`, `procesada`.
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
- Guardar `concepto`, organismo emisor y `tipoEnvio` al procesar `localiza()`, antes de `peticionAcceso()`: esa respuesta no los devuelve.
- Idempotencia por `identificador` DEHú: un envío ya registrado no se reprocesa.
- Reintentos con backoff ante fallos SOAP, de certificado o de red, sin perder el envío pendiente.
- Lotes de menos de 1000 peticiones por operación LEMA.
- Comunicación (`tipoEnvio` `1`): `peticionAcceso(identificador, codigoOrigen)` en el mismo ciclo. Acceder no tiene efectos jurídicos y no hay acuse.
- Notificación (`tipoEnvio` `2`): no se comparece en el sondeo. `peticionAcceso()` es la comparecencia y puede abrir el plazo de respuesta. El mecanismo está abierto (sección 13).

### RF-02 Almacenamiento (crítico)

- En el mismo ciclo que `peticionAcceso()`, persistir el documento principal y todos los anexos. En notificaciones, también el acuse.
- Anexo con URL directa: descarga HTTP, sin LEMA. Anexo por referencia: `consultaAnexos()`, uno a uno.
- Acuse: `consultaAcusePdf()` con el `csvResguardo` de `peticionAcceso()`. Solo en notificaciones.
- `consultaAnexos()` y `consultaAcusePdf()` solo funcionan durante 24 h desde `peticionAcceso()`. Pasado ese plazo, el documento no se puede recuperar por API. Si la descarga falla, alerta de alta prioridad; no reintentos silenciosos.
- `Documento.verificarIntegridad()`: el SHA-256 del principal debe coincidir con el de DEHú.
- Binarios en almacenamiento de ficheros (`rutaAlmacenamiento`); en BBDD solo metadatos y hash.

### RF-05 Clasificación

- Se clasifica con organismo emisor y concepto de `localiza()`, antes de abrir nada. No se leen principal, anexos ni acuse.
- La categoría es el departamento. No hay catálogo de tipos: `tipoAsignado` y `tipoDetectado` no se rellenan.
- Umbral de confianza configurable. Por debajo: `REVISION` (RF-09); no se abre el envío ni se publica acción automática.
- Pendiente de confirmar con el resto de departamentos.
- Dentro de la DGSFP (DIR3 `E00119006`) el emisor no separa departamentos: Red de Mediación se guía por el área remitente, que no aparece en `localiza()` documentado (sección 13).

### RF-03/04 Extracción y resumen

- OCR/LLM solo del documento principal de las comunicaciones, para el resumen. Anexos y acuse se archivan sin procesar.
- Las notificaciones no se resumen (pendiente de confirmar con el resto de departamentos).
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
- Ticket obligatorio. Email (RF-08.2) deseable: `EmailNotificacionGateway` solo si hay tiempo, pero el Resolver debe admitirlo.
- Idempotencia: si ya hay una `DERIVACION` con `estado=exito` (`tieneDerivacionExitosa()`), no se repite. Un `fallo` sí se reintenta.
- Reclasificación (RF-09.6, RF-10.4, RF-11.5): finalizar el ticket original y crear uno nuevo con `esReclasificacion=true` y `ticketRelacionadoId` = ticket original. No cuenta como duplicado. Ticketing no transfiere de cola (`PATCH` no toca `cola`).

### RF-09 Revisión humana

- Confianza baja → `REVISION` con `resuelto=false`. Ninguna comunicación se pierde entre pasos.
- UI: documento como vista principal y texto extraído como panel auxiliar (diseño abierto).
- Aceptar (09.5) → RF-08 con la clasificación propuesta.
- Reclasificar (09.6) → solo `administrador`; finalizar + crear.
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

### RF-12 Reconciliación (opcional)

Batch con `localizaRealizadas`, que no filtra por fecha. No es consulta interactiva.

---

## 6. Base de datos

SQL Server (`HV_OrganismosPublicos`), JDBC propio (`Database` + `*RepositorySQLServer`). Scripts en `HVOrganismosPublicosMigraciones`.

| Tabla | Relación con `COMUNICACION` | Rol |
|---|---|---|
| `COMUNICACION` | raíz | Un envío DEHú. `estado` interno (no el de DEHú). |
| `DOCUMENTO` | 1:N | `principal` / `anexo` / `acuse` |
| `INTERPRETACION` | 1:1 opcional | Resultado de la IA |
| `TEXTOEXTRAIDO` | 1:1 opcional desde `INTERPRETACION` | Texto completo, con retención configurable |
| `CLASIFICACION` | 1:N, solo inserciones | Historial de clasificaciones |
| `DERIVACION` | 1:N, solo inserciones | Resultado de RF-08 |
| `REVISION` | 1:N opcional | Una fila por escalado |
| `DEPARTAMENTO` | catálogo | Colas y canales |
| `USUARIO` | catálogo | Roles de la app |

Campos:

- **COMUNICACION:** `id` (UUID, PK), `identificador` (índice único), `codigoOrigen`, `concepto`, `organismoEmisorCodigo`, `organismoEmisorNombre`, `tipoEnvio`, `fechaEvento`, `estado`, `fechaIngesta`.
- **DOCUMENTO:** `id`, `comunicacion_id`, `tipo`, `nombre`, `mimeType`, `hashSha256`, `csvResguardo`, `rutaAlmacenamiento`, `fechaDescarga`.
- **INTERPRETACION:** `id`, `comunicacion_id`, `tipoDetectado` (vacío en el MVP), `entidadesExtraidas` (JSON en `nvarchar`), `scoreConfianza`, `fechaProcesado`.
- **TEXTOEXTRAIDO:** `id`, `interpretacion_id`, `textoExtraido`, `fechaCreacion`.
- **CLASIFICACION:** `id`, `comunicacion_id`, `origen` (`ia` | `operador`), `modelo` (solo si hubo LLM), `nIntentos` (`1`, `2`, `3`…; solo si `origen=ia`), `usuario_id` (solo si `origen=operador`), `departamentoAsignado` (obligatorio), `tipoAsignado` (vacío en el MVP), `canalAsignado`, `scoreConfianza`, `resultado`, `fecha`.
- **DERIVACION:** `id`, `comunicacion_id`, `canal`, `identificadorExterno`, `titulo`, `resumen`, `departamento` (FK), `estado` (`exito` | `fallo`), `esReclasificacion`, `ticketRelacionadoId`, `fechaEjecucion`.
- **REVISION:** `id`, `comunicacion_id`, `usuario_id` (nulo hasta resolver), `motivo`, `resuelto`, `fechaEntrada`, `fechaResolucion`.
- **DEPARTAMENTO:** `id`, `nombre`, `colaDestino`, `permiteEmail`, `activo`.
- **USUARIO:** `id` (= `PERSONA.id` de Personas, sin FK física), `rol`, `activo`.

Semilla de `DEPARTAMENTO`: Coordinación DGS, SAC, Red de Mediación, Fiscal, RRHH. Faltan el resto de áreas y las colas reales de Ticketing.

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

Binarios por MTOM. Paginación de `localiza()`: `hayMasResultados`, `totalPag`, `paginaActual`.

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

SMTP corporativo vía `EmailNotificacionGateway`, solo si se implementa RF-08.2.

---

## 8. Endpoints

OpenAPI 3, recursos en camelCase plural, `x-area: hv`, `x-subarea: def`, `x-version: v1`. Prefijo en el gateway: `/api/{front|back|consumer}/hv/def/v1`. Context root en WAS: Front `HVOrganismosPublicos/api/front/v1`, Web `appt/HVOrganismosPublicosWeb`. El navegador nunca ve SOAP ni el certificado.

Errores: `401` no autenticado, `403` sin rol, `404` no existe, `409` regla de negocio (derivación duplicada, revisión ya resuelta, límite de RF-10, comunicación no editable).

### 8.1 Front (Operador) — fase 1

| Método | Path | Rol | RF | Servicio |
|---|---|---|---|---|
| `GET` | `/revisiones` | operador, administrador | 09.1 | `ConsultarRevisionService` |
| `GET` | `/revisiones/{id}` | operador, administrador | 09.2 | `ConsultarRevisionService` (documento, texto extraído, propuesta IA) |
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
| `POST` | `/ciclosIngesta` | 01, 02 | `IngestarComunicacionesService` (sin envíos nuevos → `204`) |
| `POST` | `/interpretaciones` | 03–06 | `RegistrarInterpretacionService` + `ClasificarComunicacionService` + `GenerarContenidoTicketService` |
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
7. RF-11.4/11.5, cuando exista la audiencia back.
8. RF-10 y Consumer, cuando exista el evento de cancelación.
9. Email y RF-12, opcionales.

Fechas: backend y frontend antes del 25 dic 2026; desarrollo y testing hasta el 31 dic; revisión del 4 al 15 ene 2027. Detalle en `Diagramas/Gantt.md`.

---

## 13. Decisiones abiertas

Mientras no se cierren, se implementa lo descrito aquí sin fijar valores en código.

| Tema | Impacto |
|---|---|
| Clasificar sin leer el documento | Falta confirmarlo con el resto de departamentos (RF-05). |
| Área remitente dentro de la DGSFP | Sin ella no se separa Red de Mediación del resto de la DGSFP. Hay que ver de dónde sale sin abrir documentos (RF-05). |
| Comparecencia de notificaciones | Hipótesis: un botón en el frontal HV para el responsable de área, con los datos de `localiza()` y el día 10 desde `fechaPuestaDisposicion`; al pulsarlo se descarga y se genera el ticket sin resumen. Alternativa: comparecer en batch. Falta actor, endpoint y vista hasta que decidan las áreas (RF-01). |
| Resumen de notificaciones | No se hace mientras no se confirme con los departamentos (RF-03/04). |
| Retención de `TEXTOEXTRAIDO` | Valor del parámetro. |
| Audiencia back de Ticketing | Bloquea RF-11.4/11.5. |
| Quién publica el outbox de cancelación en el bus | Bloquea RF-10. |
| Deduplicado por `identificador` DEHú al crear tickets | RF-08.5; preguntar al equipo de Ticketing. |
| Si el umbral de RF-10 lo compara el agente o el servicio | Se implementa lo diagramado: el agente compara y actúa. |
| `codApp` de GestorBackend y grupos de Git | Infra. |
| Colas reales de Ticketing por departamento | Semilla de `DEPARTAMENTO`. |

No republicar código interno de Ticketing (`Estado_Tecnico_Ticketing.md`, `Analisis_Frontend_AYTicketing.md`).
