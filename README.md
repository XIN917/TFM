# TFM — Automatización de comunicaciones con la administración pública (DEHú)

Trabajo de Final de Máster: diseño de una plataforma para automatizar la recepción y
tramitación de comunicaciones electrónicas de la administración pública (DEHú/LEMA)
mediante IA (OCR + LLM), integrada con el sistema interno de ticketing de la empresa
colaboradora (MGS Seguros).

Este repositorio reúne la documentación de análisis y diseño del TFM (diagramas, requisitos, memoria en borrador). Convenciones de redacción y confidencialidad: [Estado.md](Estado.md) (apartado «Convenciones del repo»).

## Contenido

| Documento | Descripción |
|---|---|
| [Propuesta de proyecto.md](Propuesta%20de%20proyecto.md) | Propuesta inicial: motivación, objetivos y alcance |
| [Especificacion_Requisitos.md](Especificacion_Requisitos.md) | Especificación funcional (RF-01 a RF-12) |
| [Criterios_Clasificacion.md](Criterios_Clasificacion.md) | Fuente única de reglas de reparto (IDs R01…, estado y bloqueos) |
| [DEHu_Campos_Respuesta_Servicios.md](DEHu_Campos_Respuesta_Servicios.md) | Campos y servicios web LEMA/DEHú |
| [Estado.md](Estado.md) | Contexto, decisiones cerradas, convenciones del repo |
| [TODO.md](TODO.md) | Tareas pendientes y preguntas abiertas |
| [Estructura_Memoria.md](Estructura_Memoria.md) | Estructura de la memoria final del TFM |
| [PRD.md](PRD.md) | Producto e implementación (copia de trabajo hacia `HVOrganismosPublicos`) |
| [`docs/`](docs/) | Guía DEHú (PDF), instrucciones certificado LEMA (docx) |

### Memoria ([`memoria/`](memoria/))

Borradores de la memoria, un fichero por capítulo. Los documentos de análisis (requisitos, diagramas, TODO) se quedan en la raíz.

| Documento | Descripción |
|---|---|
| [01_Introduccion.md](memoria/01_Introduccion.md) | Capítulo Introducción |
| [Referencias.md](memoria/Referencias.md) | Listado de referencias, numeradas por orden de mención en la memoria |

### Diagramas ([`Diagramas/`](Diagramas/))

| Documento | Descripción |
|---|---|
| [Diagrama_Contexto.md](Diagramas/Diagrama_Contexto.md) | Diagrama de contexto de alto nivel. |
| [Casos_de_Uso.md](Diagramas/Casos_de_Uso.md) | Diagrama y descripciones textuales de casos de uso |
| [ER_Explicacion.md](Diagramas/ER_Explicacion.md) | Modelo entidad-relación y explicación de cada tabla |
| [Diagrama_Clases.md](Diagramas/Diagrama_Clases.md) | Diagrama de clases (capas modelo/aplicación/infraestructura/api) |
| [Diagrama_Componentes.md](Diagramas/Diagrama_Componentes.md) | Diagrama de arquitectura de componentes |
| [Diagramas_de_flujo.md](Diagramas/Diagramas_de_flujo.md) | Flujos de proceso principales |
| [Gantt.md](Diagramas/Gantt.md) | Planificación temporal del proyecto |

Los diagramas están escritos en Mermaid/PlantUML. Los de Mermaid se renderizan
directamente en GitHub o en [Mermaid Live Editor](https://mermaid.ai/live); los de PlantUML
(p. ej. [Casos_de_Uso.md](Diagramas/Casos_de_Uso.md), [Diagrama_Clases.md](Diagramas/Diagrama_Clases.md))
se pueden renderizar pegando el código en [PlantText](https://www.planttext.com) o en el
[PlantUML Web Server](https://www.plantuml.com/plantuml). Las imágenes ya renderizadas
están en [`Diagramas/img/`](Diagramas/img/).

## Estado

Documento vivo, en desarrollo activo. Ver [Estado.md](Estado.md)
para el contexto y las decisiones más recientes, y [TODO.md](TODO.md) para lo pendiente.
