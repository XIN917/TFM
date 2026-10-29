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

**Solución:** sustituir la consulta en pantalla por LEMA (servicios web de Gran Destinatario). El sistema sondea DEHú, guarda sus metadatos y clasifica cada envío por departamento sin abrir el documento, descarga y persiste los documentos, y crea un ticket en la cola de Ticketing del departamento. Si la clasificación no alcanza el umbral, la revisa un Operador. Si el departamento cancela el ticket por estar en la cola equivocada, un Agente IA intenta corregirlo, como máximo dos veces.

El nombre `OrganismosPublicos` deja abierta la puerta a otras fuentes, pero el MVP es solo DEHú/LEMA.

### 1.1 Ciclo del MVP

1. Detectar el envío con `localiza()` y guardar los metadatos que `peticionAcceso()` no devuelve.
2. Clasificar por departamento antes de abrir. La señal base es organismo emisor y concepto. No se lee el documento.
3. Según `tipoEnvio`:
   - Comunicación (`1`): descargar en un ciclo, extraer información relevante del documento principal y seguir, sujeto a autorización de tratamiento. Abrirla no tiene efectos jurídicos: se abre antes o después de clasificar según `organismosPublicos.comunicacion.abrirAntesDeClasificar` (RF-01).
   - Notificación (`2`): solo con departamento (clasificada por encima del umbral o asignado por revisión humana, RF-09.5/09.6). Queda sin abrir, a la espera de que un usuario del frontal pulse el botón de descarga (RF-01.5); al pulsarlo, comparece, guarda documento, anexos y acuse, obtiene información relevante del principal si el tratamiento está autorizado y la deja lista para derivar (RF-08).
4. Publicar el evento de comunicación clasificada (RF-07). Pendiente de decisión; mientras tanto, la comunicación queda lista para derivar en BBDD.
5. Ejecutar la acción del canal en un proceso independiente: ticket obligatorio, email deseable (RF-08).
6. Revisión humana (RF-09) y reclasificación por Operador o Agente IA (RF-10).

Generar el evento (RF-07) y ejecutar la acción (RF-08) son pasos separados, para que Ticketing no sea la única salida posible.

### 1.2 Fuera de alcance

- Enviar cualquier cosa a la Administración. LEMA es solo lectura.
- Usar los correos de aviso (sede de la DGSFP o DEHú) como fuente. La fuente es DEHú.
- Automatismos que abran expedientes o actúen sobre otros sistemas de negocio.
- MCP genérico: el Agente IA solo puede finalizar un ticket y crear otro (RF-10).
- Multi-certificado / multi-razón social.
- Buzón interno (RF-08.3; se mantiene el hueco de numeración).
- Aprendizaje continuo del modelo con el feedback del Operador. Persistir ese feedback sí es deseable (RF-09.3).
- RF-12 (`localizaRealizadas` / `consultaRealizadas`): reconciliación y evaluación de la clasificación con `REALIZADAS`, solo si sobra tiempo.
- Gestión de usuarios, roles y permisos: responsabilidad de SeguridadMGS.
- Derivación interna de Área de Siniestros a sus subáreas: sin criterios suficientes para el MVP (RF-04/05).
- Extracción y resumen documental específico para Canal Directo: uso distinto de la interpretación general de RF-05, sin alcance acordado.

---

## 2. Actores y roles

| Actor externo | Qué hace |
|---|---|
| DEHú/LEMA | Fuente SOAP. El sistema llama; LEMA no empuja. |
| Operador / Administrador | Usa el frontal HV: cola de revisión y consulta local (RF-09, RF-11). |
| Departamento | Recibe el ticket en el frontal de Ticketing y consulta el detalle y los documentos en el frontal HV (funcionalidad Consultar), al que lleva el enlace de la descripción del ticket (RF-08). Si cancela un ticket, dispara RF-10. |

Permisos por funcionalidad de SeguridadMGS, asignadas fuera de la aplicación. Se autoriza por funcionalidad en cada endpoint, nunca por nombre de rol. Los perfiles de negocio (Operador, Administrador) son combinaciones de funcionalidades:

| Funcionalidad | Permite |
|---|---|
| Consultar | Entrar al frontal, ver la cola de revisión y la consulta local. Por ahora también pulsar el botón de descarga de una notificación (RF-01.5); separarlo en una funcionalidad propia queda pendiente (§13). |
| Aceptar | Aceptar la propuesta de la IA (RF-09.5). |
| Editar derivación | Modificar el ticket (RF-11.4/11.5). |
| Reclasificar | Elegir otro departamento (RF-09.6). Solo Administrador. |
`USUARIO` solo da trazabilidad: cada resolución de `REVISION` guarda quién la hizo, y el `USUARIO` se da de alta al resolver por primera vez.

---

## 3. Arquitectura

- **Reparto Java / n8n:** n8n programa el sondeo (llama a ApiBack) y aloja los flujos de IA: clasificación e interpretación, con sus prompts y proveedor. Java dirige el flujo y es dueño del estado y de la ventana de 24 h: ingesta, pide a n8n lo que necesita de IA, decide, guarda y deriva. La comparecencia de una notificación ya no la programa n8n: la dispara un usuario desde el frontal (RF-01.5).
- **Estado como cola:** el `estado` de `COMUNICACION` dice en qué punto de la clasificación y la derivación está; qué documentos faltan se deduce de `DOCUMENTO`. Cada ciclo procesa lo pendiente; cada paso se puede repetir sin efectos duplicados, y lo que falla (LEMA, IA) se reintenta en el siguiente ciclo. La derivación inicial (RF-08) no la ejecuta el ciclo: es un proceso independiente (`/derivaciones`) sobre las comunicaciones listas. Las reclasificaciones siguen su propio flujo (RF-08).
- **Eventos:** hacia consumidores externos (RF-07, pendiente de decisión) y desde Ticketing (cancelación, RF-10). Los eventos propios de Ticketing son responsabilidad de ese equipo.
- **Dentro de Java:** hexagonal: `modelo` / `aplicacion` / `infraestructura` / `api`.

```
n8n (programación) ──► ApiBack: /ciclosIngesta, /vigilanciaDescargas (Could), /derivaciones
                                │
DEHú/LEMA ◄── LemaGateway ◄── servicios Java ──► JDBC ──► HV_OrganismosPublicos
(SOAP + WS-Security X.509)      │
                                ├─► ClasificacionIAGateway ──► flujo IA de n8n (prompt + modelo)
                                ├─► InterpretacionIAGateway ──► flujo IA de n8n
                                ├─► EventosGateway ──► bus corporativo (RF-07, pendiente de decisión)
                                └─► NotificacionGateway ──► Ticketing / email (RF-08)

Departamento cancela ticket ─(outbox de Ticketing)─► ApiConsumer ─► AgenteReclasificacionService
                                                                       └─► AgenteIAGateway (MCP, proceso externo)

Operador ──► Vue Web ──► ApiFront (JAX-RS) ──► servicios de revisión/consulta/descarga (RF-01.5)
```

| Bloque | RF | Dónde |
|---|---|---|
| Ingesta y clasificación | 01–06 (07 pendiente) | n8n programa; Java ejecuta y pide la IA a n8n (`ClasificacionIAGateway`, `InterpretacionIAGateway`) |
| Descarga de notificaciones | 01.5 | Bajo demanda del Operador: `ApiFront` + `ComparecerNotificacionService` |
| Vigilancia de descargas incompletas *(Could)* | — | Proceso independiente que programa n8n (`POST /vigilanciaDescargas`): `VigilarDescargasService` |
| Ejecución de canal | 08 | Derivación inicial: proceso independiente que programa n8n (`POST /derivaciones`) sobre las comunicaciones listas: `EjecutarDerivacionService` + `NotificacionGateway` |
| Reclasificación automática | 10 | `ApiConsumer` + `AgenteReclasificacionService` + MCP fuera de WAS |
| Revisión y consulta | 09, 11 | `ApiFront` + `HVOrganismosPublicosWeb` |

### 3.1 Convenciones Java

- Paquete raíz: `es.mgs.hv.organismosPublicos`.
- Estereotipos como Ticketing: `@Servicio`, `@Repositorio`, `@Endpoint`, `@Transaccional`. Sin clase base `AggregateRoot` ni eventos de dominio CDI.
- Agregado `ComunicacionAgregado`: la comunicación con sus clasificaciones, derivaciones y revisiones. Las reglas (`tieneRevisionActiva`, `tieneDerivacionExitosa`…) se comprueban sobre él.
- Transacciones cortas. Ninguna llamada HTTP o SOAP (LEMA, IA, Ticketing) dentro de una transacción. Para escribir sobre una comunicación se lee bloqueada (`queryAgregadoParaActualizar`) y se comprueban las reglas antes de escribir; dos operaciones simultáneas sobre la misma comunicación no se pisan.
- Cada `*Service` recibe sus dependencias por constructor con `@Inject`. Los tests lo instancian con `new Servicio(gatewayFalso, repoFalso)`, sin servidor.
- `beans.xml` con `bean-discovery-mode="annotated"` en Business y Comun (`src/META-INF`) y en Front y Web (`WebContent/WEB-INF`).
- `modelo` solo depende de interfaces; las implementaciones están en `infraestructura`.
- Identificadores: `java.util.UUID`, generado por el dominio al crear la entidad, no por la BBDD.
- Patrones: Strategy + Adapter en canales (`NotificacionGateway`), Gateway y Repository (Fowler), `NotificacionGatewayResolver` con `@Inject @Any Instance<NotificacionGateway>`. Canal nuevo = clase nueva, sin `switch` en el servicio.

### 3.2 Qué no crear

- DAO, EJB/EJBClient/Utils, WebService SOAP propio ni EAR batch aparte.
- El sondeo es un endpoint Back que dispara n8n; igual la vigilancia de descargas incompletas (*Could*, `/vigilanciaDescargas`). La comparecencia de una notificación es un endpoint Front que dispara un usuario del frontal, no n8n. Sin timers en el WAR.
- Los prompts y el proveedor de IA no viven en Java ni en BBDD: están en los flujos de n8n.
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
│       │   ├── PeticionNoValidaException.java              # 400; entrada mal formada
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
│           │   ├── ComunicacionAgregado.java
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
│           │   ├── TicketingGestionGateway.java
│           │   ├── NotificacionGateway.java               # UML
│           │   ├── AgenteIAGateway.java                   # UML
│           │   ├── ClasificacionIAGateway.java
│           │   ├── SolicitudClasificacionIa.java
│           │   ├── RespuestaClasificacionIa.java
│           │   ├── InterpretacionIAGateway.java
│           │   ├── EventosGateway.java
│           │   ├── AlmacenDocumentosGateway.java
│           │   └── MailGateway.java
│           ├── aplicacion/
│           │   ├── ingesta/
│           │   │   └── IngestarComunicacionesService.java           # RF-01/02/03
│           │   ├── interpretacion/
│           │   │   ├── InterpretarComunicacionService.java          # RF-04/05
│           │   │   └── RegistrarInterpretacionService.java
│           │   ├── clasificacion/
│           │   │   ├── ClasificarComunicacionService.java           # RF-02: pide la clasificación
│           │   │   ├── RegistrarClasificacionService.java           # RF-02: la guarda
│           │   │   └── GenerarContenidoTicketService.java           # RF-06
│           │   ├── eventos/
│           │   │   └── PublicarComunicacionClasificadaService.java  # RF-07
│           │   ├── comparecencia/
│           │   │   ├── ComparecerNotificacionService.java           # RF-01.5: botón de descarga
│           │   │   └── VigilarDescargasService.java                 # RF-01.5/03, Could: /vigilanciaDescargas
│           │   ├── derivacion/
│           │   │   ├── EjecutarDerivacionService.java               # UML RF-08
│           │   │   ├── IntentoDerivacionService.java                # transacciones de cada intento
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
│               │   └── EventosClient.java                           # RF-07, pendiente de decisión (§7.3)
│               ├── ia/
│               │   ├── ClasificacionIAClient.java                   # flujo IA de n8n
│               │   ├── InterpretacionIAClient.java                  # flujo IA de n8n
│               │   └── AgenteIAClient.java                          # UML MCP
│               └── http/
│                   └── RestClientSupport.java
│
├── HVOrganismosPublicosApiFrontV1WebService/                       # fase 1
│   ├── build.xml                                                    # ant generate
│   ├── WebContent/META-INF/openapi/openapi.yaml                     # contrato
│   ├── gen/                                                        # ant generate; no editar
│   ├── src/es/mgs/hv/organismosPublicos/api/front/v1/
│   │   ├── impl/
│   │   │   ├── RevisionesApiImpl.java                              # UML RevisionController
│   │   │   └── ComunicacionesApiImpl.java                          # UML RegistroController; incluye POST /comunicaciones/{id}/descarga (RF-01.5)
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
│   │   │   ├── ListaComunicaciones/ListaComunicacionesView.vue     # RF-11.1/11.6
│   │   │   ├── ListaNotificaciones/ListaNotificacionesView.vue     # RF-11.6
│   │   │   ├── DetalleComunicacion/DetalleComunicacionView.vue     # RF-11; botón de descarga (RF-01.5) y estado de descarga (RF-11.7)
│   │   │   └── Error/ErrorView.vue
│   │   └── components/
│   │       ├── DocumentoViewer.vue
│   │       ├── TextoExtraidoPanel.vue
│   │       └── EstadoDescargaPanel.vue                             # RF-11.7: estado por documento, intentos, reintentar
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
│   │   │   ├── CiclosIngestaApiImpl.java                           # POST /ciclosIngesta
│   │   │   ├── VigilanciaDescargasApiImpl.java                     # POST /vigilanciaDescargas (Could)
│   │   │   └── DerivacionesApiImpl.java                            # POST /derivaciones
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
| Beans | Excepciones de aplicación (`EntidadNoEncontradaException`, `PeticionNoValidaException`, `ReglaNegocioException`) y DTOs de evento. El 400 es `PeticionNoValidaException` y el 409 `ReglaNegocioException`. Sin SQL ni SOAP. |
| Comun | log4j2, carga de properties, `IntegrationException` y `ObjectMappers`. |
| Business | Modelo, aplicación e infraestructura (`JdbcTemplate`, `*SQLServer`, `LemaClient`, `TicketingClient`, `AgenteIAClient`). |
| ApiFront + WasEAR | REST del Operador, incluida la descarga bajo demanda de notificaciones (RF-01.5). EAR: WAR + Business + Beans + Comun. |
| Web + WebEAR | SPA Vue (UIBaseProject) y servlets de seguridad. EAR: WAR + Comun, sin Business. |
| ApiBack | Endpoints que llama n8n: ciclo de ingesta, derivación y, si se implementa (*Could*), vigilancia de descargas incompletas. |
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
| `ComunicacionAgregado` | `tieneDerivacionAsociada`, `tieneDerivacionExitosa`, `tieneDerivacionPendiente`, `tieneRevisionActiva`, `derivacionActiva`, `ultimaClasificacionIa`, `contarIntentosReclasificacion`, `registrarDerivacion`, `registrarClasificacion`, `escalarARevision`, `resolverRevisionActiva`, `marcarEnProceso`, `marcarProcesada`, `registrarComparecencia` (RF-01.5: persiste `principal`, filas `DOCUMENTO` pendientes y quién/cuándo pulsó el botón), `documentosPendientes` |
| `Documento` | `verificarIntegridad`, `estaPendiente` (`fechaDescarga IS NULL`), `registrarIntento` (incrementa `nIntentos`, guarda `ultimoError`), `registrarDescarga` |
| `Derivacion` | `esModificableInPlace`, `actualizar` |
| `Revision` | `resolver(Usuario)` |
| `ComunicacionRepository` | `saveNueva` (no inserta si el `identificador` ya existe), `query`, `queryListado`, `queryAgregado`, `queryAgregadoParaActualizar` (con bloqueo), `saveAgregado`, `saveResultadoDerivacion`, `queryRevisionesPendientes`, `queryPendientesDeAtencion` (RF-11.8: notificaciones en `en_proceso` sin `principal`, o con `DOCUMENTO` pendiente; ordenable por tiempo restante) |
| `DepartamentoRepository` | `queryColaDestino`, `existsActivo` |
| `LemaGateway` | `localiza`, `peticionAcceso`, `consultaAnexos`, `consultaAcusePdf` (implementado por `LemaClient`) |
| `TicketingGateway` | `crearTicket` |
| `TicketingGestionGateway` | `cerrarPorReclasificacion`, `modificarTicket` (sin implementar hasta que Ticketing lo exponga) |
| `NotificacionGateway` | `soportaCanal`, `ejecutar` |
| `ClasificacionIAGateway` | `clasificar(SolicitudClasificacionIa)` → `RespuestaClasificacionIa` (RF-02) |
| `AgenteIAGateway` | `intentarAutocorreccion` |

`ConstantesDominio`:

- Estado de `COMUNICACION`: `pendiente` (sin clasificar), `en_revision` (revisión humana abierta), `en_proceso` (con departamento, por IA sobre el umbral o por revisión humana), `procesada` (derivación inicial con `exito`). Transiciones: `pendiente → en_proceso / en_revision`, `en_revision → en_proceso`, `en_proceso → procesada`. Describe la clasificación y la derivación, no la descarga: qué documentos faltan se deduce de `DOCUMENTO` y de `tipoEnvio` (comunicación: la descarga el ciclo, RF-01; notificación: el botón de descarga, RF-01.5). Es el ciclo interno del sistema, no un estado de DEHú.
- Origen de clasificación: `ia`, `operador`.
- Canal: `ticket`, `email`.
- Tipo de documento: `principal`, `anexo`, `acuse`.
- Estado de `DERIVACION`: `en_curso`, `exito`, `fallo`, `desconocido`.
- Motivo de `REVISION`: `umbral` (confianza baja), `sin_criterio` (el modelo no propone departamento), entre otros.

Dependencias de cada servicio:

| Servicio | Interfaces |
|---|---|
| `IngestarComunicacionesService` | `LemaGateway`, `ComunicacionRepository`, `AlmacenDocumentosGateway` |
| `InterpretarComunicacionService` | `InterpretacionIAGateway`, `ComunicacionRepository` |
| `RegistrarInterpretacionService` | `ComunicacionRepository` |
| `ClasificarComunicacionService` | `ComunicacionRepository`, `DepartamentoRepository`, `ClasificacionIAGateway`, `RegistrarClasificacionService` |
| `RegistrarClasificacionService` | `ComunicacionRepository` |
| `GenerarContenidoTicketService` | `ComunicacionRepository` |
| `PublicarComunicacionClasificadaService` | `EventosGateway`, `ComunicacionRepository` |
| `ComparecerNotificacionService` | `LemaGateway`, `ComunicacionRepository`, `AlmacenDocumentosGateway`, `UsuarioRepository` (RF-01.5; reintento acotado de anexo/acuse dentro de la misma petición, con `organismosPublicos.descarga.maxReintentos`) |
| `VigilarDescargasService` | `LemaGateway`, `ComunicacionRepository`, `NotificacionGateway` (Could; `/vigilanciaDescargas`; nunca llama a `peticionAcceso()` por primera vez) |
| `EjecutarDerivacionService` | `IntentoDerivacionService`, `NotificacionGatewayResolver` |
| `IntentoDerivacionService` | `ComunicacionRepository` |
| `NotificacionGatewayResolver` | `@Any Instance<NotificacionGateway>` |
| `AceptarClasificacionService` | `ComunicacionRepository`, `UsuarioRepository` |
| `ReclasificarComunicacionService` | `ComunicacionRepository`, `UsuarioRepository`, `TicketingGateway`, `TicketingGestionGateway` |
| `ConsultarComunicacionesService` | `ComunicacionRepository` |
| `ConsultarRevisionService` | `ComunicacionRepository` |
| `ModificarDerivacionService` | `ComunicacionRepository`, `TicketingGestionGateway` |
| `AgenteReclasificacionService` | `ComunicacionRepository`, `AgenteIAGateway` (`LIMITE_INTENTOS = 2`) |

Un caso de uso con llamada externa entre dos escrituras usa dos servicios: uno que llama (fuera de transacción) y otro con las transacciones (`ClasificarComunicacionService` / `RegistrarClasificacionService`, `EjecutarDerivacionService` / `IntentoDerivacionService`). El interceptor de `@Transaccional` no actúa en llamadas dentro del mismo bean.

---

## 5. Requisitos funcionales

Solo las reglas que el código no puede violar.

### RF-01 Detección y registro inicial

- n8n llama al endpoint Back, que ejecuta `localiza()`. Si no hay envíos nuevos ni pendientes, no se hace nada.
- Se rechaza (y se registra en log) un envío sin `identificador`, sin `fechaPuestaDisposicion` o con `tipoEnvio` desconocido.
- Guardar, al procesar `localiza()` y antes de `peticionAcceso()`, los campos que esa segunda respuesta no devuelve: `concepto`, organismo emisor, organismo emisor raíz, `fechaPuestaDisposicion`, `tipoEnvio`, `vinculo`, titular y `metadatosPublicos`. `metadatosPublicos` se guarda tal como llega; no se interpreta ni entra en la clasificación.
- Idempotencia por `identificador` DEHú: un envío ya registrado no se reprocesa.
- Reintentos con backoff ante fallos SOAP, de certificado o de red, sin perder el envío pendiente.
- Lotes de menos de 1000 peticiones por operación LEMA.
- Comunicación (`tipoEnvio` `1`): `peticionAcceso(identificador, codigoOrigen)`. Acceder no tiene efectos jurídicos y no hay acuse. Cuándo se abre lo decide `organismosPublicos.comunicacion.abrirAntesDeClasificar`: `true`, en el ciclo en que se registra; `false`, cuando tiene departamento (IA por encima del umbral o revisión humana). Si falla, se reintenta en los ciclos siguientes. No cambia RF-02: la clasificación sigue usando solo metadatos.
- Notificación (`tipoEnvio` `2`): el sondeo no comparece. `peticionAcceso()` es la comparecencia y puede abrir el plazo de respuesta. Clasificada por encima del umbral, o con departamento asignado por revisión humana (RF-09.5/09.6), queda en `en_proceso` sin abrir y espera al botón de descarga del frontal (RF-01.5). Por debajo del umbral no se abre.
- La fecha límite de comparecencia se calcula provisionalmente a partir de `fechaPuestaDisposicion` y de las reglas de cómputo previstas. Hasta validar este cálculo con los vencimientos reales y la normativa aplicable, el resultado tendrá carácter orientativo y no se considerará una fecha de vencimiento confirmada por LEMA. No debe utilizarse como único criterio para decidir el momento de acceso a una notificación.

### RF-01.5 Descarga bajo demanda de notificaciones

- `POST /comunicaciones/{id}/descarga` (§8.1), con la funcionalidad Consultar. El frontal exige confirmación explícita de que la acción practica la comparecencia legal y no es reversible.
- Aplica a una notificación en `en_proceso`: clasificada por encima del umbral, o con departamento asignado por `AceptarClasificacionService` (RF-09.5) o `ReclasificarComunicacionService` (RF-09.6). Sin `principal` aún, la llamada comparece; con `principal`, solo reintenta anexos y acuse pendientes.
- `ComparecerNotificacionService`: si la comunicación no tiene ya `principal` descargado, llama a `peticionAcceso()`. Al tener éxito, en una transacción corta persiste el `principal` (con su `csvResguardo`), una fila `DOCUMENTO` pendiente (`fechaDescarga = NULL`) por cada anexo de la respuesta (`urlDirecta` o `referenciaDocumento`, nunca ambos) y, si aplica, por el acuse; registra `usuarioDescarga` y `fechaClicDescarga`. A continuación, en la misma petición, invoca `consultaAnexos()`/`consultaAcusePdf()` para cada fila pendiente, con reintento acotado (`organismosPublicos.descarga.maxReintentos`); cada intento fallido incrementa `DOCUMENTO.nIntentos` y guarda en `ultimoError` el tipo/código de error, nunca el cuerpo de la respuesta. Lo que no se resuelve queda pendiente, visible en RF-11.7.
- Idempotente: si la comunicación ya tiene `principal` persistido, no se repite `peticionAcceso()`; la llamada solo reintenta las filas con `fechaDescarga = NULL` que aún no alcanzaron `maxReintentos`.
- Si `peticionAcceso()` tiene éxito pero la persistencia posterior falla (caída de BBDD a mitad de transacción), la comparecencia ya se produjo sin que quede registro local — incidencia manual; la vía de recuperación exacta está pendiente de definir (§13). Confirmar también si `peticionAcceso()` admite reconsulta sobre el mismo `identificador` (§13).

### RF-02 Clasificación

- Se clasifica antes de abrir nada. La señal base es organismo emisor y concepto de `localiza()`. No se leen principal, anexos ni acuse. Otro metadato de `localiza()` solo entra si una muestra real demuestra que discrimina. El concepto se reconoce por la materia, sin igualdad exacta. Los DIR3 observados no son la clave universal del organismo.
- Resultado: el departamento. No hay catálogo de tipos.
- Se clasifican en cada ciclo los envíos `pendiente` sin clasificación de IA ni revisión abierta: los nuevos y los que fallaron antes.
- `ClasificarComunicacionService` pide la clasificación al flujo de IA de n8n (`ClasificacionIAGateway`), fuera de transacción. Solo envía metadatos: `concepto` y organismo emisor y raíz (código y nombre). El prompt versionado y el modelo están en n8n (reglas en estado «Prompt» de `Criterios_Clasificacion.md`).
- Respuesta (contrato de `ClasificacionIAGateway`): `departamentoId` (`id` de `DEPARTAMENTO`; nulo si ninguna regla aplica), `score` (puntuación de la propuesta; nula si no hay propuesta), `conflicto` (`true` si varias reglas llevan a departamentos distintos) y `modelo`. El formato de salida del prompt y su traducción a este contrato viven en n8n; cambiarlos no toca Java.
- Si la llamada falla o la respuesta no se puede leer, no se guarda nada: el envío sigue `pendiente` y se reintenta en el siguiente ciclo.
- `RegistrarClasificacionService` guarda la `CLASIFICACION` en su transacción, con la comunicación bloqueada; si ya se clasificó, no hace nada.
- Umbral sobre `score` en properties, sin tocar código; su valor se fija tras la experimentación.
- Va a `REVISION` (RF-09), sin abrir el envío ni acción automática:
  - `departamentoId` nulo (motivo `sin_criterio`), sin propuesta: se resuelve reclasificando (RF-09.6);
  - `conflicto` o `score` por debajo del umbral (motivo `umbral`), con la propuesta guardada;
  - `departamentoId` que no existe o no está activo.
- Si no va a revisión, la comunicación pasa a `en_proceso` (RF-01).
- No se clasifica por descarte ni con reglas pendientes.
- Reglas en el prompt y pendientes: §6.

### RF-03 Almacenamiento documental (crítico)

- En el mismo ciclo que `peticionAcceso()`, persistir el documento principal. En comunicaciones, los anexos y el acuse (si aplica) se descargan en ese mismo ciclo automático (RF-01). En notificaciones, `peticionAcceso()` y la persistencia inicial las dispara RF-01.5; anexos y acuse se resuelven desde las filas `DOCUMENTO` pendientes que deja esa misma llamada, con reintento acotado.
- Anexo con URL directa: descarga HTTP, sin LEMA (`DOCUMENTO.urlDirecta`). Anexo por referencia: `consultaAnexos()`, uno a uno (`DOCUMENTO.referenciaDocumento`).
- Acuse: `consultaAcusePdf()` con el `csvResguardo` de `peticionAcceso()`. Solo en notificaciones.
- `consultaAnexos()` y `consultaAcusePdf()` no permiten consultar elementos de notificaciones con más de 1 día de antigüedad (`Especificacion_Requisitos.md`, RF-03). Desde qué momento cuenta ese día (puesta a disposición, `peticionAcceso()`…) está pendiente de verificar con LEMA (§13). Pasado ese plazo, el documento no se puede recuperar por API. Los fallos se reintentan de forma automática y acotada (`organismosPublicos.descarga.maxReintentos`); además de los reintentos, un fallo que persiste genera alerta de alta prioridad, con antelación suficiente para no agotar la ventana — no es un reintento silencioso. Antelación, destinatario y canal de la alerta: §13.
- `Documento.verificarIntegridad()`: el SHA-256 del principal debe coincidir con el de DEHú.
- Binarios en almacenamiento de ficheros (`rutaAlmacenamiento`); en BBDD solo metadatos y hash.
- Guardar en `DOCUMENTO.metadatos` el `documento.metadatos` del principal y el `acusePdf.metadatos` del acuse, tal como llegan y sin interpretar. Si no se guardan al recibirlos, no se pueden recuperar: `consultaRealizadas()` no trae el del documento, y el acuse solo se puede volver a pedir dentro de ese mismo plazo.
- Guardar también `algoritmoHash` y, en cada anexo por referencia, `referenciaDocumento`; en cada anexo por URL directa, `urlDirecta`. El binario va a fichero. No se persiste la paginación de `localiza()`.

### RF-04/05 Obtención de texto e interpretación documental

- Procesamiento textual y semántico del documento principal de las comunicaciones y notificaciones (`tipoEnvio` `1` y `2`), cuando el tratamiento mediante IA externa esté autorizado. Se permite extracción nativa, OCR o procesamiento multimodal directo con modelos de OpenAI o Mistral. Un mismo servicio puede obtener el texto y la interpretación sin requerir dos llamadas independientes. Anexos y acuse se archivan sin procesar.
- La interpretación está orientada a identificar el asunto, las solicitudes o requerimientos, las actuaciones expresas, los plazos indicados y otros datos útiles para el departamento. No se presupone un plazo universal ni se calculan vencimientos jurídicos definitivos con el modelo. La política temporal de sondeo y comparecencia sigue pendiente de cierre.
- Salidas diferenciadas: representación textual conservada en `TEXTOEXTRAIDO`; interpretación estructurada orientada a la tramitación (asunto, requerimientos, actuaciones expresas, plazos documentados, organismo, expediente, importes y partes). Las inferencias se diferencian de los datos explícitos; si falta información se indica. El indicador de confianza de interpretación requiere definición y validación; no asumir probabilidad calibrada del proveedor.
- Mismo patrón que RF-02: `InterpretarComunicacionService` pide la interpretación al flujo de IA de n8n (`InterpretacionIAGateway`), fuera de transacción, y `RegistrarInterpretacionService` persiste `INTERPRETACION` y `TEXTOEXTRAIDO`. Si falla, se reintenta en el siguiente ciclo.

### RF-06 Contenido del ticket

Título, información relevante para la tramitación cuando esté disponible, campos e `identificador` DEHú. Sin adjuntos: `POST /tickets` no tiene campo para ellos; los documentos se consultan en HV, por el enlace de la descripción (RF-08). La derivación sin interpretación por falta de autorización o fallo sigue pendiente de decisión.

### RF-07 Evento

- Requisito: publicar un evento por comunicación con contenido RF-06 completo, en la arquitectura de eventos corporativa, exista o no consumidor. Payload previsto: `comunicacionId`, contenido RF-06, departamento/cola e `identificador` DEHú.
- **Pendiente de decisión:** no está confirmado cómo publicaría Java en el bus ni quién consumiría (§7.3, §13). Hasta decidirlo no se implementa la publicación, y RF-07 no se da por cumplido.
- Lo que sí se implementa es el desacoplamiento de RF-07.2, mediante el estado persistido: al completar RF-06, la comunicación queda lista para derivar en BBDD, y la ingesta no llama al canal. Esto no sustituye a la publicación del evento.
- No hay tabla `EVENTO`: la trazabilidad es `CLASIFICACION` + `DERIVACION`.

### RF-08 Derivación

- La derivación inicial es un proceso independiente de la ingesta: `POST /derivaciones` (§8.2), que programa n8n. Una comunicación está **lista** cuando está en `en_proceso`, tiene guardado el documento principal y no tiene derivación inicial `exito`, `en_curso` ni `desconocido`. Si además hay que esperar al acuse de una notificación, o qué hacer cuando un anexo o el acuse no se consiguen, está pendiente (§13). Ni los ciclos ni la aceptación de una revisión derivan. Las reclasificaciones (RF-09.6, RF-10.4, RF-11.5) siguen el flujo descrito al final de este apartado.
- El canal sale de `Clasificacion.canalAsignado` y del catálogo `DEPARTAMENTO`, no de un `if` por tipo.
- Ticket obligatorio, creado con la API back de Ticketing (`POST /tickets`) en `DEPARTAMENTO.colaDestino`. Email (RF-08.2) no obligatorio, pero planificado justo después del ticket (`EmailNotificacionGateway`). Si no da tiempo, no bloquea el cierre del MVP.
- El ticket no lleva documentos: `POST /tickets` no tiene campo de adjuntos. Los documentos se quedan en HV (RF-03). La descripción lleva el `identificador` DEHú, la referencia de la derivación y, si está configurado, el enlace al detalle de la comunicación en el frontal HV (`organismosPublicos.front.detalle.url` + id de la comunicación).
- Cada intento es una fila `DERIVACION`. Se registra `en_curso` antes de llamar al canal y, después, pasa a:
  - `exito`: la comunicación queda `procesada`;
  - `fallo`: seguro que no se ejecutó (la petición no salió, Ticketing la rechazó con 400, 401, 403 o 404, o falta configuración);
  - `desconocido`: no se sabe si se ejecutó (timeout, 5xx, otros 4xx, respuesta sin id); no se reintenta para no duplicar el ticket.
- Reintentos: un `fallo` se reintenta en la siguiente ejecución de `/derivaciones`, como máximo `maxReintentos` veces tras el intento inicial (2 por defecto, 3 intentos en total; `0` los desactiva). Se cuentan las filas `DERIVACION` iniciales (`esReclasificacion=0`) de la comunicación; no hace falta otra columna. Al llegar al límite, deja de intentarse y queda para conciliación. Es independiente del límite de RF-10.
- Con una derivación `exito` no se repite (`tieneDerivacionExitosa()`). Con una `en_curso` o `desconocido` no se lanza otra (`tieneDerivacionPendiente()`). La aplicación nunca cambia por su cuenta un `en_curso` ni un `desconocido`, y las filas de `DERIVACION` no se borran.
- La respuesta de `/derivaciones` resume lo derivado, lo fallido y lo pendiente de conciliar (`desconocido`, `en_curso` que no termina, límite alcanzado), que también va al log.
- Conciliación manual, excepcional, sin API ni funcionalidad del frontal. La ejecuta una persona con permiso en Ticketing y de escritura en BBDD, con una incidencia que registra quién, cuándo, qué comunicación y por qué.
  1. Comprobar en el frontal de Ticketing si el ticket existe, por la referencia de la derivación y el `identificador` DEHú que lleva la descripción.
  2. Si existe, registrar `exito` con su código.
  3. Si no existe, solo se crea uno si consta que el intento anterior terminó sin crearlo (para un `en_curso`, que la llamada terminó: log del intento o reinicio de la aplicación). Se crea con el contenido RF-06 completo: título y resumen de la fila `DERIVACION`, `identificador` DEHú, referencia de la derivación y enlace al detalle en HV, como lo crea la aplicación. Si no consta, la derivación queda como está, pendiente de conciliación.
  4. El registro va en una sola transacción, con la comunicación bloqueada como lo hace la aplicación (`UPDLOCK`) y comprobando de nuevo el estado. Un `desconocido` o `en_curso` pasa a `exito` si el ticket existía, o a `fallo` si se creó uno nuevo; en ese caso, y cuando se parte de un `fallo`, se añade una fila `exito` con el código del nuevo ticket. La comunicación pasa a `procesada`. Si la comprobación no cuadra, no se escribe nada. El índice único impide dos derivaciones iniciales `exito`.
  - El SQL de este procedimiento (README de `HVOrganismosPublicosMigraciones`) no está validado: verificarlo en desa es requisito previo a producción (§13).
- Si Ticketing ofrece búsqueda por referencia externa (§7.2), `desconocido` podrá conciliarse sin intervención manual.
- Reclasificación (RF-09.6, RF-10.4, RF-11.5): finalizar el ticket original y crear uno nuevo con `esReclasificacion=true` y `ticketRelacionadoId` = ticket original. No cuenta como duplicado. Ticketing no transfiere de cola (`PATCH` no toca `cola`).

### RF-09 Revisión humana

- Confianza baja → `REVISION` con `resuelto=false`. Ninguna comunicación se pierde entre pasos.
- UI (09.2): metadatos de `localiza()` y clasificación propuesta. No se abre el documento para decidir el departamento.
- Aceptar (09.5) fija el departamento; no deriva. La comunicación pasa a `en_proceso`. Si no tiene `principal`, una comunicación la descarga el siguiente ciclo (RF-01); una notificación no se abre aquí: espera al botón de descarga del frontal (RF-01.5). Cuando queda lista, la deriva el proceso de RF-08.
- Reclasificar (09.6) → funcionalidad Reclasificar (Administrador). Elige el departamento; finalizar + crear si ya hay ticket. Si no hay ticket todavía, pasa a `en_proceso` igual que RF-09.5. Es la única forma de resolver una revisión sin propuesta (`sin_criterio`), así que es necesaria para vaciar la cola.
- Feedback (09.3): deseable, asíncrono, no bloquea el cierre.

### RF-10 Agente IA

- Entrada: evento de cancelación del ticket (outbox de Ticketing).
- Intentos = filas `CLASIFICACION` con `origen=ia` y `nIntentos>1`. Límite: 2 (`LIMITE_INTENTOS`), independiente de los reintentos de RF-08.
- En el límite: escalar a RF-09 sin llamar al agente.
- Con margen: `AgenteIAGateway.intentarAutocorreccion`. El agente propone; si supera el umbral, él mismo finaliza + crea (`AgenteIAClient` usa `TicketingGateway`). Si no, escala a revisión.
- No hay bucle interno: el siguiente intento llega con la siguiente cancelación.

### RF-11 Consulta local

- Nunca llama a DEHú.
- Editable si y solo si existe al menos una `DERIVACION` (`tieneDerivacionAsociada()`), no según `tipoEnvio`. En revisión pendiente, solo lectura.
- 11.4, mismo departamento: `modificarTicket` in-place (depende de la audiencia back de Ticketing).
- 11.5, cambio de departamento: finalizar + crear, como en 09.6.
- 11.7 Estado de descarga en el detalle: por documento (principal, anexos, acuse), descargado o pendiente, `nIntentos` y `ultimoError`. «Reintentar» vuelve a llamar a `POST /comunicaciones/{id}/descarga` (idempotente, RF-01.5).
- 11.8 Vista de pendientes de atención: notificaciones en `en_proceso` sin `principal` y envíos con `DOCUMENTO` pendiente, ordenados por el tiempo que queda de la ventana de RF-03 o del plazo de comparecencia (`queryPendientesDeAtencion`).

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
| `DERIVACION` | 1:N, una fila por intento | Resultado de RF-08 |
| `REVISION` | 1:N opcional | Una fila por escalado |
| `DEPARTAMENTO` | catálogo | Colas y canales |
| `USUARIO` | catálogo | Quién resuelve revisiones (trazabilidad) |
| `REALIZADAS` | ninguna | Envíos ya realizados para evaluar la clasificación (RF-12.4–12.6). Fuera del flujo |

Campos:

- **COMUNICACION:** `id` (UUID, PK), `identificador` (índice único), `codigoOrigen`, `concepto`, `organismoEmisorCodigo`, `organismoEmisorNombre`, `organismoEmisorRaizCodigo`, `organismoEmisorRaizNombre`, `fechaPuestaDisposicion` (obligatorio; la consulta filtra y ordena por él), `tipoEnvio`, `vinculo`, `titularNombre`, `titularNif`, `fechaEvento`, `estado` (interno), `metadatosPublicos` (base64 de `localiza()`, `nvarchar(max)`; no se interpreta ni entra en el prompt), `fechaIngesta`, `usuarioDescarga` (FK `USUARIO`, nullable; quién pulsó el botón de descarga — RF-01.5), `fechaClicDescarga` (nullable; cuándo lo pulsó).
- **DOCUMENTO:** `id`, `comunicacion_id`, `tipo`, `nombre`, `mimeType`, `hashSha256`, `algoritmoHash`, `csvResguardo` (el del documento en `peticionAcceso()`), `referenciaDocumento` (anexo por referencia), `urlDirecta` (anexo por URL directa; exactamente uno de los dos no nulo en una fila `anexo`), `metadatos` (`documento.metadatos` o `acusePdf.metadatos` en base64, `nvarchar(max)`; vacío en anexos; no se interpreta), `rutaAlmacenamiento`, `fechaDescarga` (nullable; `NULL` = pendiente de descarga o en reintento), `nIntentos` (nullable; solo filas `anexo`/`acuse`; cuenta los intentos de `consultaAnexos()`/`consultaAcusePdf()`), `ultimoError` (nullable; tipo/código del último fallo, nunca el cuerpo de la respuesta — evita guardar datos personales). Índice recomendado: `(comunicacion_id, tipo)`.
- **INTERPRETACION:** `id`, `comunicacion_id`, `entidadesExtraidas` (JSON en `nvarchar`), `scoreConfianza`, `fechaProcesado`.
- **TEXTOEXTRAIDO:** `id`, `interpretacion_id`, `textoExtraido`, `fechaCreacion`.
- **CLASIFICACION:** `id`, `comunicacion_id`, `origen` (`ia` | `operador`), `modelo` (solo si hubo LLM; lo envía n8n), `nIntentos` (`1`, `2`, `3`…; solo si `origen=ia`), `usuario_id` (solo si `origen=operador`), `departamentoAsignado` (obligatorio), `canalAsignado`, `scoreConfianza` (nulo si no hay propuesta), `resultado` (sin uso), `fecha`.
- **DERIVACION:** `id`, `comunicacion_id`, `canal`, `identificadorExterno`, `titulo`, `resumen`, `departamento` (FK), `estado` (`en_curso` | `exito` | `fallo` | `desconocido`; la aplicación solo lo actualiza desde `en_curso`), `esReclasificacion`, `ticketRelacionadoId`, `fechaEjecucion` (inicio del intento). Índice único **filtrado** sobre (`comunicacion_id`, `canal`) con `estado IN ('en_curso','desconocido','exito')` y `esReclasificacion=0` (`UX_DERIVACION_inicial`): impide una segunda derivación inicial mientras otra está en curso, sin resolver o con éxito, pero permite varias filas `fallo` de reintento. Las filas no se borran.
- **REVISION:** `id`, `comunicacion_id`, `usuario_id` (nulo hasta resolver), `motivo`, `resuelto`, `fechaEntrada`, `fechaResolucion`.
- **DEPARTAMENTO:** `id`, `nombre`, `colaDestino`, `permiteEmail`, `activo`.
- **USUARIO:** `id` (= `PERSONA.id` de Personas, sin FK física), `rol` (sin uso: los permisos están en SeguridadMGS), `activo`.
- **REALIZADAS:** `id`, `identificador` (único; nunca presente en `COMUNICACION`), `codigoOrigen`, `concepto`, `organismoEmisorCodigo`, `organismoEmisorNombre`, `organismoEmisorRaizCodigo`, `organismoEmisorRaizNombre`, `fechaPuestaDisposicion`, `tipoEnvio`, `vinculo`, `titularNombre`, `titularNif`, `estadoDehu`, `referenciaPdfAcuse`, `csvResguardo` (del envío), `rutaDocumento`, `hashSha256`, `algoritmoHash` (solo si se descargó el principal), `departamentoEsperado` (FK, etiquetado), `departamentoPropuesto` (FK), `scoreConfianza`, `modelo`, `fechaClasificacion`, `fechaCarga`.

Semilla de `DEPARTAMENTO`. El prompt de clasificación de n8n devuelve uno de estos `id`, que son también las categorías que admite el prompt; las reglas que decide cada `id` solo viven en el prompt de n8n, descritas en `Criterios_Clasificacion.md` — **Java no las necesita ni las consulta.** Faltan las colas reales de Ticketing. Los DIR3 son ejemplos observados (`E00119006` DGSFP, `EA0028512` AEAT, `E00127105` Catastro, `EA0042298` TGSS); no son la clave de la regla.

| `id` | Departamento |
|---|---|
| `ASESORIA_FISCAL` | Asesoría Fiscal |
| `COORDINACION_DGS` | Coordinación DGS |
| `SAC` | SAC |
| `RED_MEDIACION` | Red de Mediación |
| `CANAL_DIRECTO` | Canal Directo |
| `RECURSOS_HUMANOS` | Recursos Humanos |
| `AREA_SINIESTROS` | Área de Siniestros |
| `SUCURSAL_CENTRAL` | Sucursal Central — **pendiente de confirmación de negocio** (ver `Criterios_Clasificacion.md`); no incluir en las categorías del prompt ni dar de alta en la migración hasta confirmarlo |

Detalle de reglas, estado y bloqueos: `Criterios_Clasificacion.md`.

Un cambio de reglas es una nueva versión del prompt en n8n, sin tocar Java ni BBDD. Un departamento nuevo es una fila en una migración más el `enum` del prompt.

Agrupar varias notificaciones por referencia de acuerdo es una necesidad identificada en Canal Directo y no es requisito del MVP.

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

- HTTP / OpenAPI, audiencia back. Se crea el ticket en la cola `DEPARTAMENTO.colaDestino`.
- Hoy la audiencia back solo crea tickets y consulta tipos. Finalizar por reclasificación y `modificarTicket` (`TicketingGestionGateway`) esperan a que Ticketing los exponga.
- Buscar un ticket por referencia externa permitiría resolver `desconocido` sin intervención manual (RF-08). Depende de Ticketing.
- El cliente tiene timeouts de conexión y lectura acotados. Los valores recomendados dependen de Ticketing.
- Los eventos que genere Ticketing al crear o modificar tickets, y su publicación, son responsabilidad de ese equipo.
- Las cancelaciones no salen por webhook sino por un outbox interno.

### 7.3 n8n y bus de eventos

- n8n programa el sondeo y la derivación: llama a `POST /ciclosIngesta` y `POST /derivaciones` (§8.2), con independencia entre ambas. La frecuencia vive en n8n. La comparecencia de una notificación ya no la programa n8n: la dispara un usuario desde el frontal (`POST /comunicaciones/{id}/descarga`, §8.1, RF-01.5).
- *(Could, fuera del MVP, se implementa si da tiempo)* n8n puede programar además `POST /vigilanciaDescargas` (§8.2): reintenta anexos/acuse pendientes y avisa si se acerca la ventana de 24 h o el vencimiento de comparecencia sin descargar. Nunca llama a `peticionAcceso()` por primera vez.
- n8n aloja los flujos de IA de clasificación e interpretación. Java los llama por HTTP (`ClasificacionIAClient`, `InterpretacionIAClient`), siempre fuera de transacción. Cambiar de proveedor o de prompt se hace en n8n.
- Publicación de RF-07: pendiente de decisión. Kafka es el sistema de eventos corporativo, pero no está confirmado cómo publicaría Java (`EventosGateway`), ni el topic, ni quién consumiría. No se implementa hasta que Infra lo confirme (§13).

### 7.4 Agente IA / MCP

Proceso fuera de WAS, accedido por `AgenteIAGateway`. Su única herramienta MCP es finalizar + crear ticket. El procesamiento documental de la ingesta es una capacidad distinta del agente de reclasificación.

### 7.5 Seguridad

- Frontal: entrada por el portal corporativo y servlet de seguridad, como Ticketing Web.
- ApiFront: `GestorBackendFilter` y funcionalidades de SeguridadMGS declaradas en cada endpoint (§2). 401 sin usuario, 403 sin funcionalidad. Business no comprueba permisos.
- ApiBack: credencial de servicio para n8n; mecanismo pendiente de Infra.
- Identidad: el `codPersona` de la sesión es el `USUARIO.id`.

### 7.6 Correo

SMTP corporativo vía `EmailNotificacionGateway` (RF-08.2).

---

## 8. Endpoints

OpenAPI 3, recursos en camelCase plural, `x-area: hv`, `x-subarea: def`, `x-version: v1`. Prefijo en el gateway: `/api/{front|back|consumer}/hv/def/v1`. Context root en WAS: Front `HVOrganismosPublicos/api/front/v1`, Web `appt/HVOrganismosPublicosWeb`. El navegador nunca ve SOAP ni el certificado.

Errores: `400` petición no válida, `401` no autenticado, `403` sin funcionalidad, `404` no existe, `409` regla de negocio (derivación duplicada, revisión ya resuelta, límite de RF-10, comunicación no editable).

### 8.1 Front (Operador) — fase 1

| Método | Path | Funcionalidad | RF | Servicio |
|---|---|---|---|---|
| `GET` | `/revisiones` | Consultar | 09.1 | `ConsultarRevisionService` (query: `tipoEnvio`, fechas sobre `fechaPuestaDisposicion`, paginación; orden por entrada en revisión) |
| `GET` | `/revisiones/{id}` | Consultar | 09.2 | `ConsultarRevisionService` (metadatos de `localiza()` y propuesta de departamento; el documento solo si ya consta abierto) |
| `GET` | `/revisiones/{id}/documentos/{documentoId}` | Consultar | 09.2 | binario local |
| `POST` | `/revisiones/{id}/aceptacion` | Aceptar | 09.5 | `AceptarClasificacionService` |
| `POST` | `/revisiones/{id}/reclasificacion` | Reclasificar | 09.6 | `ReclasificarComunicacionService` (body: `departamento`) |
| `GET` | `/comunicaciones` | Consultar | 11.1, 11.6 | `ConsultarComunicacionesService` (query: `tipoEnvio`, `estado`, fechas, paginación; el frontend presenta dos listados independientes) |
| `GET` | `/comunicaciones/{id}` | Consultar | 11.1–11.3 | detalle con `editable` |
| `GET` | `/comunicaciones/{id}/documentos/{documentoId}` | Consultar | 11 | binario local |
| `PATCH` | `/comunicaciones/{id}` | Editar derivación | 11.4 / 11.5 | `ModificarDerivacionService` (body: `titulo`, `resumen`, `departamentoDestino`) |
| `GET` | `/departamentos` | Consultar | 09.6, 11.5 | `DEPARTAMENTO` activos |
| `POST` | `/comunicaciones/{id}/descarga` | Consultar | 01.5 | `ComparecerNotificacionService`; solo si `tipoEnvio=2` y `estado=en_proceso`; si ya tiene `principal`, solo reintenta anexos y acuse pendientes. El frontal confirma antes de llamar que la acción practica la comparecencia legal |

### 8.2 Back (n8n) — fase 2

Credencial de servicio, no de Operador.

| Método | Path | RF | Servicio |
|---|---|---|---|
| `POST` | `/ciclosIngesta` | 01–06 | `localiza()` y registro de envíos nuevos (`IngestarComunicacionesService`); clasificación de los pendientes pidiéndola al flujo de IA de n8n (`ClasificarComunicacionService`). Comunicación: `peticionAcceso()`, documentos, interpretación si está autorizada y RF-06, en el momento que fija `abrirAntesDeClasificar` (RF-01); también descarga las comunicaciones en `en_proceso` sin `principal` (aceptadas en revisión o con descarga fallida antes). Notificación sobre el umbral: queda en `en_proceso` sin abrir, a la espera del botón de descarga del frontal (RF-01.5). Por debajo del umbral: revisión; la notificación no se abre. Responde un resumen del ciclo; sin nada que procesar → `204` |
| `POST` | `/vigilanciaDescargas` *(Could, fuera del MVP)* | 01.5/03 | `VigilarDescargasService`: reintenta `anexo`/`acuse` con `fechaDescarga = NULL` y dentro del límite de intentos; avisa si se acerca la ventana de 24 h o el vencimiento de comparecencia sin descargar. Nunca llama a `peticionAcceso()` por primera vez |
| `POST` | `/derivaciones` | 08 | Deriva las comunicaciones listas (`EjecutarDerivacionService`), con idempotencia y el límite de reintentos (RF-08). Independiente de los ciclos: no llama a LEMA. Responde un resumen: derivadas, fallidas y pendientes de conciliar; sin nada que derivar → `204` |

Cada paso es idempotente y se puede relanzar: lo que falla en un ciclo se reintenta en el siguiente. La derivación inicial se reintenta en la siguiente ejecución de `/derivaciones`, con límite (RF-08). La extracción/OCR y la interpretación pueden resolverse en una misma llamada multimodal; se conserva `TEXTOEXTRAIDO`.

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
| `/comunicaciones` | `ListaComunicacionesView` (filtro `tipoEnvio=1`) | `GET /comunicaciones` |
| `/notificaciones` | `ListaNotificacionesView` | `GET /comunicaciones?tipoEnvio=2` |
| `/comunicaciones/:id` | `DetalleComunicacionView` | `GET /comunicaciones/{id}`, `PATCH` si `editable` |

---

## 9. Requisitos no funcionales

| Área | Implicación |
|---|---|
| Seguridad | Certificado solo en el servidor, con alerta de caducidad. Los logs no vuelcan el certificado. |
| Protección de datos e IA externa | No enviar documentos reales a OpenAI o Mistral sin validación corporativa de Seguridad y Protección de Datos. No puede descartarse la presencia de datos personales a partir de `localiza()`. Condiciones contractuales, retención y transferencias por confirmar; alternativa operativa sin autorización pendiente. |
| Accesibilidad | WCAG 2.1 AA como objetivo de calidad del frontal, sin presumir obligación legal específica para esta herramienta interna. |
| Conservación legal | Documentos oficiales y trazabilidad de recepción a ticket. |
| Resiliencia | Una caída de SE/PRO no pierde envíos ya detectados. |
| Rendimiento | `TEXTOEXTRAIDO` fuera de los listados. |
| Auditabilidad | Log del resultado de cada llamada SOAP y de cada clasificación con su score; también de qué usuario pulsó el botón de descarga de cada notificación y cuándo (RF-01.5). |
| Extensibilidad | Reglas de reparto = nueva versión del prompt en n8n. Proveedor o modelo de IA = cambio en el flujo de n8n. Departamento = fila de `DEPARTAMENTO` + `enum` del prompt. Canal = otra `NotificacionGateway`. Otra fuente distinta de DEHú = otro adaptador de entrada como `LemaGateway`. |
| Entornos | local / desa / usua / prod, y SE / PRO en LEMA. |

---

## 10. Configuración y operación

Properties (`{entorno}#organismosPublicos.properties`, patrón Ticketing):

- LEMA: URL/WSDL de SE y PRO, NIF titular, timeouts, tamaño de lote. La frecuencia de sondeo vive en n8n.
- Clasificación: umbral sobre `score`.
- URLs de los flujos de IA de n8n (clasificación e interpretación).
- Días de retención de `TEXTOEXTRAIDO`.
- Ticketing: tipo y subtipo de ticket. La URL del back va por `resource-env-ref` del WAS, como en Ticketing.
- Enlace al detalle en el ticket (RF-08): `organismosPublicos.front.detalle.url`, prefijo al que se añade el id de la comunicación; vacío, sin enlace.
- Apertura de comunicaciones (RF-01): `organismosPublicos.comunicacion.abrirAntesDeClasificar` (valor por defecto pendiente, §13).
- URL del Agente IA.
- Derivación (RF-08): `organismosPublicos.derivacion.maxReintentos` (2; `0` desactiva los reintentos). El límite de RF-10 no es una propiedad.
- Descarga de notificaciones (RF-01.5): `organismosPublicos.descarga.maxReintentos` (valor por defecto pendiente de experimentación — ver `TODO.md`; `0` desactiva los reintentos, igual que en RF-08).
- `PATH_PROPERTIES_SERVIDOR` (resource-env-ref en WAS).

Las properties se leen al crear los servicios (`@ApplicationScoped`): un cambio de valor requiere reiniciar la aplicación en WAS. No hay recarga dinámica.

El datasource SQLPortal se obtiene por JNDI de WAS, no por propiedad.

`HVOrganismosPublicosServer` configura el datasource e instala el certificado para WS-Security.

Binarios: ruta en el servidor o volumen. FILESTREAM no, salvo que se decida con Infra.

`.gitignore` del repo HV: `.metadata/`, compilados, `tsclient/`, `node_modules`, Playwright, `.env.*`. Se versionan `.project`, `.classpath` y `.settings`.

---

## 11. Pruebas

| Capa | Qué demuestra |
|---|---|
| `HVOrganismosPublicosTest` (JUnit 5) | Regla de decisión de RF-02 (sin propuesta, `conflicto`, umbral), reintento sin guardar si falla la IA, límite de 2 en RF-10, estados de derivación (`en_curso`, `fallo`, `desconocido`), límite de reintentos de RF-08 (incluido `0`), un `exito` registrado a mano bloquea cualquier nueva derivación inicial, `/derivaciones` independiente de LEMA, finalizar + crear frente a in-place, integridad del hash, `tieneDerivacionExitosa` frente a `tieneDerivacionAsociada`, idempotencia del botón de descarga (RF-01.5: una segunda pulsación no repite `peticionAcceso()`), reintento acotado de anexo/acuse (incluido `0`), trazabilidad de `usuarioDescarga`/`fechaClicDescarga`. Gateways simulados, incluido el flujo de IA. |
| `npm run test` (Vue) | `vue-tsc` + eslint. |
| FT Playwright | Flujos del Operador contra desa/usua. |
| SE LEMA | Llamadas SOAP reales, con certificado de pruebas y alta de Gran Destinatario. |

La evidencia para el TFM son los tests de `HVOrganismosPublicosTest`, no los E2E.

### 11.2 Evaluación de precisión de la clasificación

El desarrollo y las pruebas del MVP usan datos sintéticos. La precisión de la clasificación (RF-02) se evalúa aparte, con al menos 100 casos reales aportados por las áreas de negocio mediante el proceso interno de solicitud de datos (pendiente de confirmar — ver `TODO.md`), limitados a metadatos (organismo emisor, concepto, departamento real) sin documentos ni datos de cliente. Al medir la precisión, se cuenta aparte la tasa de error específica de Área de Siniestros (ver `Criterios_Clasificacion.md`).

### 11.1 Criterios de aceptación

| RF | Criterio |
|---|---|
| 01 | Ninguna comunicación pendiente permanece sin detectar más de N minutos (SLA a definir); ninguna se procesa dos veces. |
| 02 | Toda comunicación recibe una propuesta de departamento con indicador de confianza basada en organismo emisor y concepto, o se deriva a revisión humana sin acceder al documento. |
| 03 | Tras el acceso, se conservan el documento principal, los anexos y, en notificaciones, el acuse cuando exista, respetando la ventana de recuperación y la integridad documental; lo que falla se reintenta de forma acotada y queda visible como pendiente (RF-11.7) si no se resuelve. |
| 04 | El documento principal de cada comunicación o notificación procesada dispone de representación textual verificable, con independencia del mecanismo de extracción. |
| 05 | La interpretación produce información estructurada útil para la tramitación, distinguiendo los requerimientos y plazos expresos de las inferencias y señalando datos no determinados. |
| 06 | El contenido generado es válido según el esquema de la API de Ticketing, sin intervención manual, en el 100% de los casos clasificados con confianza suficiente. |
| 07 | Toda comunicación que complete RF-06 genera exactamente un evento publicado, verificable con independencia del resultado de RF-08. **Pendiente:** depende de la decisión de publicación (§13). El desacoplamiento por estado no cumple este criterio. |
| 08 | Toda comunicación clasificada con éxito produce exactamente una acción por el canal ticket, en un tiempo máximo definido por SLA, o queda señalada para conciliación manual (`desconocido`, `en_curso` que no termina o límite de reintentos). Nunca dos tickets iniciales. El email (RF-08.2) no es requisito de cierre. |
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
4. `LemaGateway`, RF-01/02/03, alerta de 24 h y ApiBack.
5. RF-04 a RF-06 y n8n. RF-07, cuando se decida la publicación (§13).
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
| Tratamiento de documentos mediante IA externa | Validación con Seguridad y Protección de Datos antes de procesar documentos reales; determinar alternativa cuando no se autorice o falle la interpretación. |
| Interpretación específica de Canal Directo | La guía de Kit Digital sirve como material de dominio; confirmar con el área qué fases, requerimientos y actuaciones necesita identificar. No implica un catálogo cerrado para el MVP. |
| Subdepartamentos de Siniestros | Clasificación interna documental fuera del MVP hasta disponer de criterios suficientes. |
| Cobertura LEMA para destinatarios personas físicas | Comprobar durante integración si los envíos cuyos avisos recibe MGS pero que identifican a una persona física como destinataria aparecen con la autorización de Gran Destinatario de MGS. No ampliar por ello el alcance a Notific@. |
| Vencimiento de acceso | Contrastar fechas reales con `localiza()` y validar el cómputo jurídico; no confundir estimaciones con vencimientos confirmados ni con plazos de actuación. |
| Cómo aparece Distribución en `localiza()` | R07 solo se dispara si el dato llega en `concepto`, `metadatosPublicos` u otro metadato previo a comparecer (§6). |
| Reglas sin activar | R10–R12 (TGSS salvo embargo, Corredores, Sucursal Central) hasta validarlas (§6). |
| Interpretación de notificaciones | Incluida en el alcance funcional, condicionada a la autorización de tratamiento mediante IA externa. |
| Retención de `TEXTOEXTRAIDO` | Valor del parámetro. |
| Procesamiento documental OpenAI/Mistral | Seleccionar modelo y API; probar PDFs digitales, escaneados y mixtos, transcripción completa, coste, latencia y calidad. No se presupone OCR independiente ni confianza calibrada. |
| Audiencia back de Ticketing | Bloquea RF-11.4/11.5. |
| Quién publica el outbox de cancelación en el bus | Bloquea RF-10. |
| Buscar o deduplicar tickets por referencia externa | Resolver `desconocido` sin intervención manual y RF-08.5. Depende de Ticketing. |
| Umbral de RF-02 | Valor del umbral sobre `score`; se fija tras la experimentación (§10). Las puntuaciones del prompt son autodeclaradas y sin calibrar. |
| Alta de `SAC` y `RECURSOS_HUMANOS` en `DEPARTAMENTO` | Migración con los `id` de §6 y sus colas. |
| Credencial de servicio de ApiBack | Infra: cómo autentica el gateway a n8n. |
| Publicación de RF-07 | Requisito sin cumplir. Infra: si Java puede publicar en Kafka y cómo, topic, garantías y consumidores. Después, decidir si entra en el MVP y si RF-08 pasa a consumir el evento. |
| SQL de la conciliación manual de RF-08 | No validado. Verificarlo en desa (bloqueo, comprobación previa, alta de `exito`, paso a `procesada`, índice único) antes de producción. |
| Timeouts del cliente de Ticketing | Valores recomendados por Ticketing. |
| Si el umbral de RF-10 lo compara el agente o el servicio | Se implementa lo diagramado: el agente compara y actúa. |
| Grupos de acceso de Git | Infra. |
| Servidor SMTP y remitente del email | Infra. Bloquea probar RF-08.2 contra correo real; el código va por properties. |
| Colas reales de Ticketing por departamento | Semilla de `DEPARTAMENTO`. |
| Alcance real de `localizaRealizadas()` | Cuántos devuelve, qué periodo cubre (el portal solo enseña 30 días), si el tipo `1` trae comunicaciones, si aparece al momento de comparecer y si trae `metadatosPublicos`. Condiciona RF-12.4. |
| Aviso de vencimiento de comparecencia o de la ventana de 24 h sin descargar (RF-01.5) | Destinatario, canal y antelación sin definir; la Guía no fija explícitamente este plazo operativo. Mientras tanto, cubierto por la vista de pendientes (RF-11.8) y el reintento manual, no por un aviso proactivo. |
| `peticionAcceso()` con éxito y persistencia posterior fallida (RF-01.5) | Incidencia manual; la vía de recuperación exacta está pendiente de definir. |
| Si `peticionAcceso()` admite reconsulta sobre el mismo `identificador` | Relevante para `VigilarDescargasService` y para recuperar la lista de anexos si la primera respuesta no llegó a persistirse. Verificar en el entorno de pruebas de LEMA. |
| Valor por defecto de `organismosPublicos.descarga.maxReintentos` | Se fija tras observar fallos reales de `consultaAnexos()`/`consultaAcusePdf()`. |
| Funcionalidad propia para la descarga (RF-01.5) | Hoy basta con Consultar: cualquier usuario del frontal puede comparecer. Decidir con la empresa si se restringe; hacerlo es una funcionalidad nueva en SeguridadMGS y cambiar la anotación del endpoint. |
| `/vigilanciaDescargas` | *Could*, fuera del MVP; se implementa si el calendario lo permite. |
| Ventana de `consultaAnexos()`/`consultaAcusePdf()` | Desde qué momento cuenta el día de antigüedad (puesta a disposición, `peticionAcceso()`…). Verificar con acceso a LEMA. Condiciona RF-03, la alerta y RF-11.8. |
| Derivar sin acuse o con un anexo sin descargar | Si la notificación espera al acuse para estar lista (RF-08) y qué hacer cuando un anexo o el acuse no se consiguen. |
| Valor por defecto de `abrirAntesDeClasificar` | Abrir las comunicaciones al registrarlas o solo con departamento (RF-01). |
| Enlace al detalle desde el ticket | Si un enlace directo a `/comunicaciones/{id}` abre sesión o debe pasar por el JumpTo del portal; valor de `organismosPublicos.front.detalle.url` por entorno. |
| Qué ve el departamento en HV | Con Consultar ve todas las comunicaciones y la cola de revisión; no hay filtro por departamento. Decidir si basta o hace falta restringirlo (funcionalidad nueva). |
