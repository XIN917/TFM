# Planificación

*Nota: el proyecto se gestiona también como archivo nativo `.gantt` en la herramienta online de Gantt (onlinegantt.com), que conserva la configuración de días laborables/festivos y la jerarquía de subtareas. Este documento (`Gantt.md`) es la versión para incluir en la memoria.*

### 1. Análisis y Diseño — 24/08/2026 a 07/09/2026
- Requisitos y casos de uso — 24/08/2026 a 27/08/2026
- Diagrama de flujo — 27/08/2026 a 28/08/2026
- Diagrama ER — 28/08/2026 a 31/08/2026
- Planificación (Gantt) — 01/09/2026
- Diagrama de clases — 02/09/2026 a 04/09/2026
- Arquitectura de componentes — 04/09/2026 a 07/09/2026

### 2. Desarrollo — 07/09/2026 a 31/12/2026
- Infraestructura — 07/09/2026 a 15/09/2026
- Pipeline — 16/09/2026 a 06/10/2026
- Motor IA (OCR + clasificación) — 07/10/2026 a 26/10/2026
- Integración con Ticketing — 27/10/2026 a 09/11/2026
- Gestión de derivaciones y consultas (Operador) — 10/11/2026 a 01/12/2026
  - Backend — 10/11/2026 a 01/12/2026
  - Frontend — 12/11/2026 a 01/12/2026
- Despliegue continuo a producción — 07/10/2026 a 31/12/2026

### 3. Testing — 05/11/2026 a 31/12/2026
- Pruebas en entorno SE — 05/11/2026 a 18/12/2026
- Validación en entorno PRO (LEMA) — 01/12/2026 a 31/12/2026

### 4. Memoria — 24/08/2026 a 31/12/2026
- Introducción — 24/08/2026 a 18/09/2026
- Gestión del proyecto y planificación — 21/09/2026 a 09/10/2026
- Análisis — 12/10/2026 a 30/10/2026
- Desarrollo — 02/11/2026 a 20/11/2026
- Evaluación — 23/11/2026 a 11/12/2026
- Conclusiones — 14/12/2026 a 31/12/2026

### 5. Revisión Final — 04/01/2027 a 15/01/2027
- Revisión del sistema — 04/01/2027 a 06/01/2027
- Revisión de la memoria — 07/01/2027 a 11/01/2027
- Preparación de la defensa — 12/01/2027 a 15/01/2027

## Diagrama de Gantt

```mermaid
gantt
    title Planificación TFM — Automatización DEHú (MGS)
    dateFormat  YYYY-MM-DD
    axisFormat  %d/%m

    section Análisis y Diseño
    Requisitos y casos de uso              :a1, 2026-08-24, 2026-08-27
    Diagrama de flujo                      :a2, 2026-08-27, 2026-08-28
    Diagrama ER                            :a3, 2026-08-28, 2026-08-31
    Planificación (Gantt)                  :a4, 2026-09-01, 2026-09-01
    Diagrama de clases                     :a5, 2026-09-02, 2026-09-04
    Arquitectura de componentes            :a6, 2026-09-04, 2026-09-07

    section Desarrollo
    Infraestructura                                    :d1, 2026-09-07, 2026-09-15
    Pipeline                                            :d2, 2026-09-16, 2026-10-06
    Motor IA (OCR + clasificación)                      :d3, 2026-10-07, 2026-10-26
    Integración con Ticketing                            :d4, 2026-10-27, 2026-11-09
    Gestión de derivaciones y consultas (Operador)       :d5, 2026-11-10, 2026-12-01
    Backend                                              :d6, 2026-11-10, 2026-12-01
    Frontend                                             :d7, 2026-11-12, 2026-12-01
    Despliegue continuo a producción                    :d8, 2026-10-07, 2026-12-31

    section Testing
    Pruebas en entorno SE           :t1, 2026-11-05, 2026-12-18
    Validación en entorno PRO (LEMA) :t2, 2026-12-01, 2026-12-31

    section Memoria
    Introducción                              :m1, 2026-08-24, 2026-09-18
    Gestión del proyecto y planificación      :m2, 2026-09-21, 2026-10-09
    Análisis                                   :m3, 2026-10-12, 2026-10-30
    Desarrollo                                 :m4, 2026-11-02, 2026-11-20
    Evaluación                                 :m5, 2026-11-23, 2026-12-11
    Conclusiones                               :m6, 2026-12-14, 2026-12-31

    section Revisión Final
    Revisión del sistema       :r1, 2027-01-04, 2027-01-06
    Revisión de la memoria     :r2, 2027-01-07, 2027-01-11
    Preparación de la defensa  :r3, 2027-01-12, 2027-01-15
```
