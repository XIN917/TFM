# Diagrama de contexto (alto nivel)

*Vista de contexto del sistema: qué entidades rodean la plataforma y qué intercambian con ella, sin entrar en componentes internos (para el detalle interno, ver [Diagrama de componentes](Diagrama_Componentes.md)). Se muestran dos situaciones: la actual (acceso manual) y la futura (con el sistema automatizado + IA).*

`Propuesta de proyecto.md` describe la situación actual de forma genérica ("buzones y sedes electrónicas de organismos públicos"); **Mi Carpeta Ciudadana** es la fuente concreta a la que hoy se accede: es el área personal dentro de DEHú (Dirección Electrónica Habilitada única, el punto único del Estado para notificaciones de cualquier organismo), pensada para acceso manual puntual de un particular — la alternativa para grandes destinatarios como MGS son los servicios web LEMA, que sustituirán este acceso manual por uno automatizado (ver `Estado_del_proyecto.md`, sección 1.1).

**Objetivo concreto**: sustituir el acceso manual y periódico de cada departamento a Mi Carpeta Ciudadana por un acceso automatizado (LEMA) que clasifica cada comunicación con IA y la deriva directamente al departamento correspondiente vía ticket, sin que nadie tenga que revisar el listado completo.

---

## Situación actual

![Diagrama de contexto - situación actual](img/diagrama_contexto_actual.png)

Resumen (detalle completo en `Propuesta de proyecto.md`, sección "Descripción general"): sin sistema intermedio, usuarios de la Empresa MGS acceden manualmente y de forma periódica, revisan todas las comunicaciones pendientes y determinan una por una cuáles son de su departamento — el propio usuario hace de "filtro".

```plantuml
@startuml diagrama_contexto_actual

skinparam backgroundColor White
skinparam defaultFontName Helvetica
skinparam actor {
    BackgroundColor White
    BorderColor Black
}
skinparam component {
    BackgroundColor White
    BorderColor Black
    ArrowColor Black
    FontSize 14
}
skinparam cloud {
    BackgroundColor White
    BorderColor Black
}
skinparam note {
    BackgroundColor #FFF9C4
    BorderColor Black
}
skinparam nodesep 60
skinparam ranksep 60

left to right direction

cloud "Mi Carpeta Ciudadana\n(DEHú)" as Carpeta
component "Empresa MGS\n(usuarios por departamento)" as MGS

Carpeta <-- MGS : accede manualmente\n(certificado digital,\nperiódico)
Carpeta --> MGS : devuelve TODAS las\ncomunicaciones pendientes

note bottom of MGS
  cada usuario revisa el listado
  completo para averiguar si
  alguna comunicación es de
  su departamento
end note

@enduml
```

---

## Situación futura (con Sistema + IA)

![Diagrama de contexto - situación futura](img/diagrama_contexto.png)

El sistema se interpone entre Mi Carpeta Ciudadana y la Empresa MGS: consulta las comunicaciones automáticamente (vía LEMA), las clasifica con IA y deriva a cada departamento solo lo que le corresponde, creando un ticket en el Sistema de Ticketing (canal principal, ver `Estado_del_proyecto.md` sección 4). El Operador gestiona la cola de revisión humana para los casos de baja confianza.

### Elementos

- **Mi Carpeta Ciudadana**: fuente de las comunicaciones (ver explicación arriba); en esta situación el acceso ya es automatizado vía LEMA, no manual.
- **Sistema (IA)**: la plataforma de automatización objeto del TFM — ingesta, OCR/interpretación, clasificación y generación de tickets/notificaciones.
- **Sistema de Ticketing**: sistema interno ya existente en MGS (fuera del alcance del TFM) donde el Sistema (IA) crea el ticket derivado; es el canal por el que la Empresa MGS consulta y gestiona sus comunicaciones asignadas (ver `Diagrama_Componentes.md`).
- **Empresa MGS**: los departamentos internos que reciben ya filtrada y clasificada solo la comunicación que les corresponde (a diferencia de la situación actual, ya no revisan el listado completo).
- **Operador**: rol interno que gestiona la cola de revisión humana y la reclasificación manual (ver `Diagrama_Componentes.md`, bloque "Gestión y Revisión").

```plantuml
@startuml diagrama_contexto

skinparam backgroundColor White
skinparam defaultFontName Helvetica
skinparam actor {
    BackgroundColor White
    BorderColor Black
}
skinparam component {
    BackgroundColor White
    BorderColor Black
    ArrowColor Black
    FontSize 14
}
skinparam cloud {
    BackgroundColor White
    BorderColor Black
}
skinparam nodesep 60
skinparam ranksep 60

left to right direction

cloud "Mi Carpeta Ciudadana\n(DEHú)" as Carpeta
actor "Operador" as Operador
component "Empresa MGS\n(departamentos)" as MGS
component "Sistema de\nTicketing" as Ticketing

component "Sistema\n(IA)" as Sistema

Carpeta --> Sistema : comunicaciones\n(vía LEMA, automático)
Sistema --> Ticketing : crea ticket\nya clasificado
Ticketing --> MGS : consulta / gestiona\nsu cola asignada
MGS --> Ticketing : reporta clasificación\nincorrecta (cancela)
Operador --> Sistema : gestiona / revisa /\nreclasifica
Sistema --> Operador : cola de revisión

@enduml
```
