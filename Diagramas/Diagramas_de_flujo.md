# Diagramas de flujo — Automatización DEHú (MGS)

Cinco flujos independientes (cada uno con su propio inicio/fin, sin referencias cruzadas entre ellos). Código Mermaid listo para pegar en [Mermaid Live Editor](https://mermaid.live) o renderizar directamente.

---

## 1. Sistema (automático) — Detección, sondeo y clasificación

Sondeo periódico de `localiza()`, iteración sobre la lista, obtención de documento/anexos/acuse, OCR, interpretación y clasificación. Termina registrando la comunicación (cola de revisión pendiente si confianza baja; historial de procesadas si confianza alta).

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 600}}}%%
flowchart TD
    start(("Inicio")) --> a1

    subgraph SIS1["Sistema (automático) — Detección y sondeo"]
        direction TB
        a1["Consultar listado de comunicaciones pendientes (localiza())"]
        decHasItems{"¿Hay comunicaciones en la lista?"}
        loopNext["Tomar siguiente comunicación de la lista"]
        a2["Obtener documento y metadatos (peticionAcceso())"]
        aAnexos["Consultar anexos por referencia (consultaAnexos())"]
        aAcuse["Consultar acuse (consultaAcusePdf())"]
        a3["Almacenar documento principal, anexos y acuse"]
        a4["Extraer texto (OCR)"]
        a5["Interpretar contenido"]
        a6["Clasificar comunicación"]
        decLoop{"¿Quedan más en la lista?"}
        pollWait["Fin del ciclo de sondeo (esperar frecuencia configurable)"]

        a1 --> decHasItems
        decHasItems -->|sí| loopNext --> a2 --> aAnexos --> aAcuse --> a3 --> a4 --> a5 --> a6 --> decLoop
        decHasItems -->|no| pollWait
        decLoop -->|sí, quedan mas| loopNext
        decLoop -->|no, lista vacia| pollWait
        pollWait -.->|vuelve a sondear| a1
    end

    a6 -->|"comunicación clasificada"| dec1{"¿Confianza >= umbral?"}
    dec1 -->|sí| b1
    dec1 -->|no| pendReview["Registrar en cola de revisión pendiente"] --> stop2(("Fin"))

    subgraph SIS2["Sistema (automático)"]
        direction TB
        b1["Generar contenido de la notificación"]
        b2["Generar y publicar evento"]
        b1 --> b2
    end

    b2 --> procReg["Registrar en historial de comunicaciones procesadas"] --> stop1(("Fin"))

    style SIS1 fill:#E3F2FD,stroke:#90CAF9,color:#000000,font-weight:bold,font-size:14px
    style SIS2 fill:#E3F2FD,stroke:#90CAF9,color:#000000,font-weight:bold,font-size:14px

    classDef process fill:#FFFFFF,stroke:#B0BEC5,color:#000000;
    classDef decision fill:#FFE0B2,stroke:#FFB74D,color:#000000;
    classDef terminal fill:#37474F,stroke:#263238,color:#ffffff;

    class a1,a2,aAnexos,aAcuse,a3,a4,a5,a6,loopNext,pollWait,b1,b2,pendReview,procReg process
    class dec1,decHasItems,decLoop decision
    class start,stop1,stop2 terminal
```

---

## 2. Departamento + Autocorrección (Agente IA)

El Departamento consulta la notificación asignada; si la clasificación está mal, cancela (lo que genera un evento e incrementa un contador de intentos persistente). El Agente IA escucha ese evento directamente — comprueba el contador *antes* de intentar reclasificar; si ya alcanzó el límite (2), escala sin gastar otro intento.

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 600}}}%%
flowchart TD
    start(("Inicio")) --> d1

    subgraph DEP["Departamento"]
        direction TB
        d1["Consultar notificación asignada"]
        dec3{"¿Clasificación correcta?"}
        d2["Fin del caso"]
        d3["Cancelar y generar evento"]
        d1 --> dec3
        dec3 -->|sí| d2
        dec3 -->|no| d3
    end

    d2 --> stop(("Fin"))
    d3 -->|"+1 intento"| agentEvt

    subgraph AGENT["Sistema — Autocorrección (Agente IA)"]
        direction TB
        agentEvt["Escuchar evento"]
        decAttempts{"¿Contador <= 2?"}
        agentTry["Agente IA intenta autocorregir"]
        decConf{"¿Confianza >= umbral?"}
        agentSuccess["Finalizar ticket y crear uno nuevo en cola correcta"]
        agentEscalate["Escalar a cola de revisión humana"]

        agentEvt --> decAttempts
        decAttempts -->|sí| agentTry --> decConf
        decConf -->|sí| agentSuccess
        decConf -->|no| agentEscalate
        decAttempts -->|no| agentEscalate
    end

    agentSuccess --> stop
    agentEscalate --> stop

    style DEP fill:#E8F5E9,stroke:#A5D6A7,color:#000000,font-weight:bold,font-size:14px
    style AGENT fill:#F3E5F5,stroke:#CE93D8,color:#000000,font-weight:bold,font-size:14px

    classDef process fill:#FFFFFF,stroke:#B0BEC5,color:#000000;
    classDef decision fill:#FFE0B2,stroke:#FFB74D,color:#000000;
    classDef terminal fill:#37474F,stroke:#263238,color:#ffffff;

    class d1,d2,d3,agentEvt,agentTry,agentSuccess,agentEscalate process
    class dec3,decAttempts,decConf decision
    class start,stop terminal
```

---

## 3. Usuario/Operador

Gestión manual de comunicaciones de baja confianza: aceptar la propuesta de la IA o reclasificar. Ambas rutas generan un evento que, en paralelo (sin bloquear el cierre del caso), retroalimenta al modelo IA.

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 600}}}%%
flowchart TD
    start(("Inicio")) --> c1

    subgraph OP["Usuario/Operador"]
        direction TB
        c1["Consultar comunicación pendiente"]
        dec2{"¿Propuesta correcta?"}
        c2["Aceptar clasificación propuesta"]
        c4["Reclasificar comunicación"]
        c3["Generar evento"]
        c1 --> dec2
        dec2 -->|sí| c2 --> c3
        dec2 -->|no| c4 --> c3
    end

    c3 --> stop(("Fin"))
    c3 -.-> fb["Retroalimentación al modelo IA"]

    style OP fill:#FFF9C4,stroke:#FFE082,color:#000000,font-weight:bold,font-size:14px

    classDef process fill:#FFFFFF,stroke:#B0BEC5,color:#000000;
    classDef decision fill:#FFE0B2,stroke:#FFB74D,color:#000000;
    classDef terminal fill:#37474F,stroke:#263238,color:#ffffff;

    class c1,c2,c4,c3,fb process
    class dec2 decision
    class start,stop terminal
```

---

## 4. Consulta local (Operador)

Consulta bajo demanda de registros ya guardados, sin invocar a DEHú. Las comunicaciones son de solo lectura; las notificaciones (tickets) permiten modificación.

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 600}}}%%
flowchart TD
    ostart(("Inicio")) --> o1

    subgraph OPCONSULTA["Usuario/Operador"]
        direction TB
        o1["Consultar registro local"]
    end

    o1 --> o2

    subgraph SISCONSULTA["Sistema (automático)"]
        direction TB
        o2["Buscar en repositorio local"]
        o3["Mostrar resultados"]
        o2 --> o3
    end

    o3 -->|comunicación| stop1(("Fin"))
    o3 -->|notificación| o4

    subgraph OPMOD["Usuario/Operador"]
        direction TB
        o4["Modificar registro"]
    end

    o4 --> stop2(("Fin"))

    style OPCONSULTA fill:#FFF9C4,stroke:#FFE082,color:#000000,font-weight:bold,font-size:14px
    style SISCONSULTA fill:#E3F2FD,stroke:#90CAF9,color:#000000,font-weight:bold,font-size:14px
    style OPMOD fill:#FFF9C4,stroke:#FFE082,color:#000000,font-weight:bold,font-size:14px

    classDef process fill:#FFFFFF,stroke:#B0BEC5,color:#000000;
    classDef decision fill:#FFE0B2,stroke:#FFB74D,color:#000000;
    classDef terminal fill:#37474F,stroke:#263238,color:#ffffff;

    class o1,o2,o3,o4 process
    class ostart,stop1,stop2 terminal
```

---

## 5. Reconciliación periódica *(condicionado a tiempo disponible)*

Proceso batch independiente que compara `localizaRealizadas()` contra el repositorio local, para detectar posibles fallos silenciosos del flujo principal. No forma parte del alcance comprometido del MVP.

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 600}}}%%
flowchart TD
    rstart(("Inicio batch<br>(independiente)")) --> r1

    subgraph BATCH["Sistema (proceso batch de reconciliación)"]
        direction TB
        r1["Consultar listado de comunicaciones realizadas (localizaRealizadas())"]
        r2["Filtrar por fecha"]
        r3["Consultar registro local"]
        decR1{"¿Existe en local?"}
        r4["Consultar detalle (consultaRealizadas())"]
        r5["Guardar registro"]

        r1 --> r2 --> r3 --> decR1
        decR1 -->|sí| rstop
        decR1 -->|no| r4 --> r5 --> rstop
    end

    rstop(("Fin batch"))

    style BATCH fill:#E0F7FA,stroke:#4DD0E1,color:#000000,font-weight:bold,font-size:15px

    classDef processBatch fill:#B2EBF2,stroke:#4DD0E1,color:#000000;
    classDef decision fill:#FFE0B2,stroke:#FFB74D,color:#000000;
    classDef terminalBatch fill:#00695C,stroke:#004D40,color:#ffffff;

    class r1,r2,r3,r4,r5 processBatch
    class decR1 decision
    class rstart,rstop terminalBatch
```
