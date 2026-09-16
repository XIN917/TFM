# Estructura del repositorio — HVOrganismosPublicos

Módulos Eclipse/WAS a crear en `HV/HVOrganismosPublicos`. Familia **AYTicketing / AYCalendarios** (SQL Server propio, hexagonal, OpenAPI v1), no SISiso/Personas (no hay maestro en CICS).

Alineado con `Estado_del_proyecto.md` §4 y `Diagramas/Diagrama_Clases.md` (`modelo` / `aplicacion` / `infraestructura` / `api`).

---

## Stack (plataforma MGS)

Confirmado por Judit (sept. 2026): el aplicativo **se queda en Java EE 7 sobre WAS 9**, como Ticketing: Java 8, `javax.*`, CDI, JAX-RS, JDBC (sin JPA/Spring). **Jakarta EE** (`jakarta.*`) es más reciente; en los repos clonados no hay WAR así. Desde Desarrollo les gustaría pasar a Jakarta EE con un servidor más moderno, pero depende de Sistemas y no hay fecha. Si eso cambia, se actualiza este documento.

El Agente IA / MCP **no** se implementa como servidor dentro de WAS. En Java solo hay un `AgenteIAGateway`; el agente vive fuera (n8n u otro proceso).

---

## Convención de nombres

| | Valor |
|---|---|
| Prefijo Eclipse | `HVOrganismosPublicos` |
| Paquete Java | `es.mgs.hv.organismosPublicos` |
| OpenAPI | `x-area: hv`, `x-subarea: def`, `x-version: v1` |
| Gateway | `/api/{front\|back}/hv/def/v1` |

`def` = *default* (subárea del API gateway). Si el área HV acaba teniendo más APIs, Infra asignará un código de 3 letras.

---

## Árbol

```
HVOrganismosPublicos/
├── HVOrganismosPublicosBeans
├── HVOrganismosPublicosComun
├── HVOrganismosPublicosBusiness
├── HVOrganismosPublicosApiFrontV1WebService
├── HVOrganismosPublicosApiFrontV1WasEAR
├── HVOrganismosPublicosWeb
├── HVOrganismosPublicosWebEAR
├── HVOrganismosPublicosApiBackV1WebService
├── HVOrganismosPublicosApiBackV1WasEAR
├── HVOrganismosPublicosApiConsumerV1WebService
├── HVOrganismosPublicosApiConsumerV1WasEAR
├── HVOrganismosPublicosMigraciones
├── HVOrganismosPublicosProperties
├── HVOrganismosPublicosServer
├── HVOrganismosPublicosTest
└── HVOrganismosPublicosFT
```

**Orden de construcción (plazo TFM):** no hace falta desplegar los cuatro EAR el primer mes.

| Fase | Módulos |
|---|---|
| 1 — MVP | Beans, Comun, Business, ApiFront + Web, Migraciones, Properties, Server, Test |
| 2 | ApiBack (n8n dispara sondeo LEMA y derivación) |
| 3 | ApiConsumer y FT, cuando exista el evento de cancelación y haya desa |

---

## Qué lleva cada módulo

| Módulo | Facet | Contenido |
|---|---|---|
| **Beans** | `jst.utility` | Ids, excepciones, DTOs de evento. Sin SQL ni SOAP. |
| **Comun** | `jst.utility` | log4j2, carga de properties, utilidades transversales. |
| **Business** | `jst.utility` | Hexagonal: `modelo` / `aplicacion` / `infraestructura` (JdbcTemplate, `LemaGateway`, `TicketingGateway`, `AgenteIAGateway`). |
| **ApiFrontV1WebService** + **WasEAR** | `jst.web` 3.1 + EAR | REST del Operador (RF-09, RF-11). `x-audiencia: front`. OpenAPI + `*ApiImpl`. El EAR empaqueta WebService + Business + Beans + Comun. |
| **Web** + **WebEAR** | `jst.web` + EAR | SPA Vue (UIBaseProject): listado, detalle, revisión. Servlets de entrada/seguridad. El EAR no lleva Business. |
| **ApiBackV1WebService** + **WasEAR** | `jst.web` + EAR | Hooks para n8n/MCP: sondeo LEMA, pipeline, derivación. `x-audiencia: back`. |
| **ApiConsumerV1WebService** + **WasEAR** | `jst.web` + EAR | Consume el evento de cancelación (RF-10) del bus corporativo. |
| **Migraciones** | SQL | Scripts del esquema `HV_OrganismosPublicos` (ER ya cerrado). No es COBOL ni CICS. |
| **Properties** | ficheros | `{desa\|local\|prod\|usua}#…properties` y log4j2. Van al servidor (`PATH_PROPERTIES_SERVIDOR`), no al EAR. |
| **Server** | Jython | `wsadmin`: datasource SQLPortal, certificado LEMA. |
| **Test** | JUnit 4 / Java 8 | Unitarios de `*Service` con Gateways mockeados. Evidencia de evaluación del TFM. |
| **FT** | Playwright | Prueba funcional E2E del Operador contra desa/usua. No sustituye a Test. |

### Tests (no confundir)

- **Test**: un método de negocio, sin WAS ni certificado (p. ej. umbral de confianza, 2 reintentos de RF-10, idempotencia de derivación).
- **FT**: navegador real contra el entorno desplegado.
- El `npm run test` del frontend es `vue-tsc` + eslint, no unitarios de dominio.

Los `*Service` de `aplicacion` llevan **`@Inject` en el constructor** (CDI, `javax.inject`), no solo en campos. En WAS, CDI ensambla las implementaciones reales. En `HVOrganismosPublicosTest` se hace `new Servicio(gatewayFalso, repoFalso)` sin arrancar el servidor. Sin constructor visible, el unitario no es defendible en el TFM.

---

## Qué no crear

| Capa típica MGS | Por qué no |
|---|---|
| **DAO** | El maestro no está en CICS. El JDBC vive en `Business/infraestructura`, como Ticketing. |
| **EJB / EJBClient / Utils** | Nadie consume por IIOP. n8n y Ticketing van por HTTP y eventos. |
| **WebService legacy** | OpenAPI v1 desde el primer día. |
| **BatchV1** (EAR aparte) | El sondeo LEMA es un endpoint Back disparado por n8n (o un timer en el mismo WAR). |

La ventana de 24 h de `consultaAnexos` / `consultaAcusePdf` es irrecuperable: el conector LEMA (RF-01/02) debe persistir en Java (ApiBack), no solo en un flujo n8n.

---

## Dónde cae cada RF

| RF | Dónde |
|---|---|
| RF-01/02 sondeo y descarga LEMA | `infraestructura` `LemaGateway`; disparo desde ApiBack |
| RF-03–06 IA y contenido de ticket | `aplicacion` + gateways OCR/LLM; n8n orquesta, Java persiste |
| RF-07 evento clasificada | publicador en infraestructura hacia AYEventos |
| RF-08 crear ticket / email | Consumer o Back → `EjecutarDerivacionService` |
| RF-09 y RF-11 Operador | ApiFront + Web |
| RF-10 cancelación + MCP | ApiConsumer + `AgenteIAGateway` |