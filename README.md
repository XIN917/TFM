# TFM — Automatización de comunicaciones con la administración pública (DEHú)

Trabajo de Final de Máster: diseño de una plataforma para automatizar la recepción y
tramitación de comunicaciones electrónicas de la administración pública (DEHú/LEMA)
mediante IA (OCR + LLM), integrada con el sistema interno de ticketing de la empresa
colaboradora (MGS Seguros).

Este repositorio reúne toda la documentación de análisis y diseño, diagramas y anexos
generados durante el desarrollo del TFM, que después se formalizarán en la memoria
final.

## Contenido

| Documento | Descripción |
|---|---|
| [Propuesta de proyecto.md](Propuesta%20de%20proyecto.md) | Propuesta inicial: motivación, objetivos y alcance |
| [Especificacion_Requisitos.md](Especificacion_Requisitos.md) | Especificación funcional completa (requisitos RF-01 a RF-11) |
| [DEHu_Campos_Respuesta_Servicios.md](DEHu_Campos_Respuesta_Servicios.md) | Referencia de campos y servicios web LEMA/DEHú |
| [Estado_del_proyecto.md](Estado_del_proyecto.md) | Estado actual, contexto y decisiones cerradas |
| [TODO.md](TODO.md) | Tareas pendientes y preguntas abiertas |
| [Referencias.md](Referencias.md) | Listado de referencias utilizadas |
| [Estructura_Memoria.md](Estructura_Memoria.md) | Estructura orientativa de la memoria final del TFM |

### Diagramas ([`Diagramas/`](Diagramas/))

| Documento | Descripción |
|---|---|
| [Casos_de_Uso.md](Diagramas/Casos_de_Uso.md) | Diagrama y descripciones textuales de casos de uso |
| [ER_Explicacion.md](Diagramas/ER_Explicacion.md) | Modelo entidad-relación y explicación de cada tabla |
| [Diagrama_Clases.md](Diagramas/Diagrama_Clases.md) | Diagrama de clases (capas modelo/aplicación/infraestructura/api) |
| [Diagramas_de_flujo.md](Diagramas/Diagramas_de_flujo.md) | Flujos de proceso principales |
| [Gantt.md](Diagramas/Gantt.md) | Planificación temporal del proyecto |

Los diagramas están escritos en Mermaid/PlantUML. Los de Mermaid se renderizan
directamente en GitHub o en [Mermaid Live Editor](https://mermaid.live); los de PlantUML
(p. ej. [Casos_de_Uso.md](Diagramas/Casos_de_Uso.md), [Diagrama_Clases.md](Diagramas/Diagrama_Clases.md))
se pueden renderizar pegando el código en [PlantText](https://www.planttext.com) o en el
[PlantUML Web Server](https://www.plantuml.com/plantuml). Las imágenes ya renderizadas
están en [`Diagramas/img/`](Diagramas/img/).

## Estado

Documento vivo, en desarrollo activo. Ver [Estado_del_proyecto.md](Estado_del_proyecto.md)
para el contexto y las decisiones más recientes, y [TODO.md](TODO.md) para lo pendiente.
