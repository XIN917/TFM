# Diagrama de arquitectura de componentes

*Vista de alto nivel del sistema completo (módulos y su comunicación), complementaria al [Diagrama de clases](Diagrama_Clases.md) — este diagrama muestra el "qué habla con qué" a nivel de sistema; el de clases muestra el detalle interno en capas `modelo`/`aplicacion`/`infraestructura`/`api`.*

---

## Diagrama

![Diagrama de arquitectura de componentes](img/diagrama_componentes.png)

---

## Criterio de agrupación

Los componentes se agrupan en cuatro bloques, cada uno correspondiente a un tramo de requisitos funcionales, siguiendo la misma frontera que ya separa el `Especificacion_Requisitos.md` (sección 1: "generar el evento" vs. "ejecutar la acción") y el `Estado.md` (sección 4, decisión de capas/patrones):

- **Pipeline de Ingesta y Clasificación (RF-01–RF-07)**: sondeo LEMA, clasificación por metadatos de `localiza()` antes de abrir, almacenamiento documental, OCR/interpretación del documento ya descargado, generación del contenido del ticket y publicación del evento.
- **Ejecución de Acciones (RF-08)**: un único componente (Ejecutor/Selector de Canal) que ejecuta la derivación inicial de las comunicaciones listas, en un proceso independiente de la ingesta, y decide el canal de salida (ticket o email) por configuración, no por código específico (RF-08.4).

Ambos bloques se marcan como orquestados por **n8n**, siguiendo la tabla de componentes internos de `Especificacion_Requisitos.md` (sección 2.2): *"Orquestador (n8n u equivalente): coordina el flujo completo (RF-01 a RF-08) y publica/consume eventos"*. Según `PRD.md` §3 y §7.3, n8n programa los ciclos y la derivación; la lógica (derivar, registrar el resultado) está en Java, que n8n invoca (`POST /derivaciones` para RF-08).
- **Reclasificación Automática (RF-10)**: el Agente IA, aislado en su propio bloque porque su alcance de invocación MCP está deliberadamente acotado a una sola acción (finalizar+crear ticket) — no es un agente de propósito general.
- **Gestión y Revisión (RF-09, RF-11)**: el frontal del Operador, la API REST y el módulo de revisión humana, apoyados en la Base de Datos compartida.

## Decisiones de arquitectura reflejadas en el diagrama

- **Desacople Pipeline ↔ Ejecución.** Es la decisión de escalabilidad marcada como principio de diseño (no solo objetivo) en los RNF: permite añadir un canal de salida nuevo, o una fuente de entrada distinta de DEHú, sin tocar RF-01–RF-07. `Publicador de Eventos` y `Ejecutor de Acciones` no se conectan directamente. Mientras la publicación de RF-07 esté pendiente de decisión, el desacople lo da el estado persistido: el pipeline deja la comunicación lista en la Base de Datos y el Ejecutor, programado aparte por n8n, la deriva. Eso no cumple la publicación del evento, que sigue abierta (`PRD.md` RF-07).
- **`n8n` y el `Bus de Eventos Corporativo` se dibujan como dos elementos separados, deliberadamente.** `Especificacion_Requisitos.md` (nota de diseño de RF-07) los menciona juntos como una única infraestructura ya existente ("n8n + ecosistema de eventos corporativo"), pero `TODO.md` deja anotado, sin resolver, si Ticketing publica el evento de cancelación (RF-10.1) a través del propio n8n o de un bus corporativo distinto y suscribible por separado. Mientras no se confirme con el equipo de Infraestructura/Ticketing, se representa el bus como la infraestructura de mensajería (por eso es una nube, igual que DEHú/LEMA) y n8n como el motor que publica y consume contra ella — es la lectura que menos presupone.
- **El Agente IA de Reclasificación (RF-10) escucha el mismo bus**, no una llamada directa de Ticketing — reutiliza el patrón de evento ya definido en RF-07, tal como indica la nota de diseño de RF-10.1.
- **El MCP se representa como una única flecha etiquetada** ("finalizar + crear ticket vía MCP") en lugar de como un componente de integración genérico, para dejar visualmente claro que el alcance del MVP es esa acción concreta, no una capa de invocación de acciones arbitraria (ver sección 1 y RF-10, nota técnica).
- **Convención de línea discontinua vs. continua.** El diagrama usa la línea discontinua (`..>`) exclusivamente para una dependencia que el diseño da por hecha pero que todavía no está confirmada del lado externo (falta una API, falta identificar un proceso). No se usa para bifurcaciones condicionales del propio flujo (p. ej. "confianza baja" o "escala a revisión humana"), que son continuas porque son un comportamiento ya cerrado del diseño, simplemente condicional. Solo hay tres flechas discontinuas en todo el diagrama: las dos siguientes y la publicación del evento de RF-07 (`Publicador de Eventos` → `Bus de Eventos Corporativo`), pendiente de que Infra confirme cómo se publicaría.
- **La modificación de tickets desde la API REST (RF-11.4/RF-11.5)** se dibuja con flecha discontinua hacia `Sistema de Ticketing` — la línea discontinua marca que esta dependencia está aún condicionada a que exista la API "audiencia back" de Ticketing (ver `Especificacion_Requisitos.md`, sección 6, tabla de riesgos), a diferencia del resto de integraciones ya cerradas.
- **El evento de cancelación (`Sistema de Ticketing` → `Bus de Eventos Corporativo`, RF-10.1) también se dibuja discontinuo, por el mismo motivo.** Un análisis del sistema de Ticketing confirmó que no realiza ninguna notificación de red directa (ni webhook ni cola de mensajes) ante un cambio de estado de ticket — el mecanismo real es interno a Ticketing. Que ese cambio interno llegue finalmente al bus de eventos corporativo depende de una pieza intermedia aún no identificada, ajena a Ticketing. La línea discontinua es intencionadamente la misma que la de "modificar ticket": ambas representan integraciones que forman parte del diseño pero no están confirmadas al 100%.
- **Autenticación contra el Directorio de Personal** aparece como dependencia de la API REST, reflejando el RNF de control de acceso (roles `operador`/`administrador`).
- **Base de Datos como componente único compartido** entre Pipeline, Ejecución y Gestión — coherente con el ER ya cerrado (`ER_Explicacion.md`) y con la ausencia deliberada de una tabla de eventos propia (la trazabilidad la cubren `CLASIFICACION` y `DERIVACION`).
- **Nota de maquetación**: `nodesep 60` (frente a los 30 iniciales) no es arbitrario — sin él, PlantUML/Graphviz aprieta demasiado las etiquetas de los paquetes contiguos (`Ejecución de Acciones`, `Gestión y Revisión`) y el trazo de alguna flecha acaba atravesando el texto por la mitad. Si se vuelve a tocar el layout, revisar primero si las etiquetas siguen legibles antes de simplificar el `skinparam`.

## Correspondencia RF → componente

| Componente | RF cubiertos |
|---|---|
| Conector LEMA | RF-01 |
| Repositorio Documental | RF-02 |
| Motor de IA (OCR + Interpretación) | RF-03, RF-04 |
| Motor de Clasificación | RF-05 |
| Generador de Ticket | RF-06 |
| Publicador de Eventos | RF-07 |
| Ejecutor / Selector de Canal | RF-08 |
| Módulo de Revisión Humana | RF-09 |
| Agente IA de Reclasificación | RF-10 |
| API REST + Frontal Web | RF-11, RNF control de acceso |

## Código PlantUML

```plantuml
@startuml arquitectura_componentes_DEHu

skinparam backgroundColor White
skinparam defaultFontName Helvetica
skinparam component {
    BackgroundColor White
    BorderColor Black
    ArrowColor Black
    FontSize 13
}
skinparam interface {
    BackgroundColor White
    BorderColor Black
}
skinparam actor {
    BackgroundColor White
    BorderColor Black
}
skinparam database {
    BackgroundColor White
    BorderColor Black
}
skinparam cloud {
    BackgroundColor White
    BorderColor Black
}
skinparam package {
    BackgroundColor<<pipeline>> #E3F2FD
    BackgroundColor<<ejecucion>> #FFF3E0
    BackgroundColor<<gestion>> #FFF9C4
    BackgroundColor<<agente>> #F3E5F5
    BorderColor Black
    FontStyle bold
}
skinparam wrapWidth 180
skinparam nodesep 60
skinparam ranksep 55

left to right direction

together {
  cloud "DEHú / LEMA" as DEHu
  actor "Operador /\nAdministrador" as Operador
  actor "Departamento" as Departamento
}

package "Sistema de Automatización DEHú" {

  package "Pipeline de Ingesta y Clasificación\n//(n8n)//" as Pipeline <<pipeline>> {
    [Conector LEMA\n(SOAP + WS-Security)] as ConectorLema
    [Repositorio\nDocumental] as RepoDoc
    [Motor de IA\n(OCR + Interpretación)] as MotorIA
    [Motor de\nClasificación] as Clasificador
    [Generador de\nTicket] as GenTicket
    [Publicador de\nEventos] as PubEventos

    ConectorLema --> RepoDoc
    RepoDoc --> MotorIA
    MotorIA --> Clasificador
    Clasificador --> GenTicket : confianza alta
    GenTicket --> PubEventos
  }

  package "Ejecución de Acciones\n//(n8n)//" as Ejecucion <<ejecucion>> {
    [Ejecutor / Selector\nde Canal] as Ejecutor
  }

  package "Reclasificación Automática" as AgentePkg <<agente>> {
    [Agente IA de\nReclasificación] as AgenteIA
  }

  package "Gestión y Revisión" as Gestion <<gestion>> {
    [Frontal Web\n(Operador)] as Frontend
    [API REST] as API
    [Módulo de Revisión\nHumana] as Revision
    database "Base de\nDatos" as BD
  }
}

cloud "Bus de Eventos\nCorporativo" as Bus
[Sistema de\nTicketing] as Ticketing
[Servidor de\nCorreo] as Correo
[Directorio de\nPersonal] as Directorio

' --- Ingesta ---
DEHu --> ConectorLema : SOAP 1.1 / WS-Security\n(X.509)

' --- Revisión humana cuando falta confianza ---
Clasificador --> Revision : confianza baja

' --- Publicación y consumo desacoplado del evento ---
PubEventos ..> Bus : publica evento\ncomunicación clasificada\n(pendiente de decisión)
GenTicket --> BD : deja lista\npara derivar
BD --> Ejecutor : comunicaciones listas\n(n8n → POST /derivaciones)

' --- Ejecución de la acción ---
Ejecutor --> Ticketing : crear ticket (REST)
Ejecutor --> Correo : enviar email
Ejecutor --> BD : registra resultado

' --- Reclasificación automática ---
Departamento --> Ticketing : consulta / cancela\nnotificación
Ticketing ..> Bus : evento de\ncancelación
Bus --> AgenteIA : consume evento
AgenteIA --> Ticketing : finalizar + crear ticket\n(**vía MCP**)
AgenteIA --> Revision : escala a revisión\nhumana

' --- Frontal y backend de gestión ---
Operador --> Frontend
Frontend --> API : HTTP / REST
API --> Revision : aceptar / reclasificar
API --> BD : consulta local
API ..> Ticketing : modificar ticket
API --> Directorio : autenticación
Revision --> BD

@enduml
```

## Pendiente

- Infra: si Java puede publicar RF-07 en el bus corporativo (Kafka) y cómo. Hasta entonces la flecha `PubEventos ..> Bus` sigue discontinua. La resiliencia ante caídas de Ticketing no requiere cola intermedia: la llamada es síncrona y un fallo se reintenta en la siguiente ejecución, con límite (`PRD.md` RF-08).
- Cuando se cierre la API "audiencia back", pasar la flecha `API ..> Ticketing` de discontinua a continua y documentar el contrato real.
- Valorar si el Motor de IA (OCR + Interpretación) debe representarse como dos componentes separados si en el desarrollo se confirma que se externalizan como módulo de agentes IA independiente (mencionado como decisión de arquitectura pendiente en `Especificacion_Requisitos.md`, sección 2.2).