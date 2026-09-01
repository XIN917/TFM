# Propuesta de proyecto – Automatización de comunicaciones con la administración pública

## Descripción general del proyecto

El proyecto tiene como objetivo diseñar e implementar una plataforma de automatización para la gestión de comunicaciones electrónicas con organismos de la administración pública, integrando dichas comunicaciones dentro de los circuitos operativos internos de la compañía.

Actualmente, distintos usuarios de diferentes áreas acceden manualmente a buzones y sedes electrónicas de organismos públicos utilizando certificados digitales instalados localmente para consultar notificaciones, descargar documentación y revisar comunicaciones pendientes. El objetivo del proyecto es evolucionar este modelo hacia una arquitectura automatizada, centralizada y orientada a eventos.

La solución propuesta se basaría en la integración vía APIs con organismos externos para la obtención de comunicaciones y documentos electrónicos, permitiendo posteriormente su tratamiento automatizado dentro del ecosistema interno de aplicaciones de la compañía.

El flujo general planteado sería:

1.  Conexión e integración con APIs externas de organismos públicos.

2.  Descarga y recopilación automatizada de comunicaciones y documentación asociada.

3.  Interpretación y clasificación automática de la información recibida.

4.  Generación de eventos dentro de la arquitectura corporativa.

5.  Enrutado automático hacia las colas de trabajo correspondientes dentro del sistema de ticketing.

6.  Ejecución de automatismos o acciones sobre sistemas internos cuando el contexto lo permita.

Adicionalmente, el proyecto incorporaría capacidades de inteligencia artificial y automatización documental mediante OCR y modelos LLM para interpretar el contenido de documentos no estructurados y ayudar en la toma de decisiones automáticas dentro de determinados flujos.

Por ejemplo:

- Identificación automática de demandas o comunicaciones judiciales.

- Localización de expedientes ya existentes relacionados con la comunicación.

- Apertura automática de expedientes cuando no existan previamente.

- Clasificación y derivación hacia áreas concretas (siniestros, laboral, comercial por concursos públicos, etc.).

- Lanzamiento de automatismos corporativos en función del contenido interpretado.

El proyecto se apoyaría además en el ecosistema de eventos y automatización ya existente en la compañía, integrándose con plataformas de automatización de flujos y orquestación de procesos.

# Ganancia y aplicabilidad para la empresa

La implantación de este sistema supondría una mejora operativa significativa en distintos ámbitos de la organización.

## Automatización de tareas manuales

Actualmente, gran parte de la gestión de comunicaciones administrativas requiere intervención manual de usuarios de distintas áreas, que deben:

- acceder a buzones y sedes electrónicas,

- revisar comunicaciones pendientes,

- descargar documentación,

- identificar el área destinataria,

- y trasladar manualmente la información a los circuitos internos correspondientes.

La automatización de este proceso permitiría reducir de forma considerable la carga operativa y el tiempo dedicado a tareas repetitivas de bajo valor añadido.

## Mejora de seguridad y control de accesos

En el modelo actual, determinados accesos requieren certificados digitales instalados en equipos de usuarios concretos, asociados además a personas apoderadas de la organización.

La centralización y automatización del acceso permitiría:

- reducir la exposición de certificados digitales,

- minimizar accesos manuales desde múltiples puestos,

- mejorar la trazabilidad de operaciones,

- y reforzar el modelo de seguridad y control sobre comunicaciones oficiales.

## Reducción de tiempos de gestión

La identificación automática del tipo de comunicación y su derivación directa hacia las colas de trabajo adecuadas permitiría disminuir significativamente los tiempos de reacción y tramitación.

Además, en determinados casos, el sistema podría desencadenar acciones automáticas sobre aplicaciones internas, acelerando todavía más la gestión operativa.

## Escalabilidad y reutilización

La arquitectura planteada permitiría incorporar progresivamente nuevos organismos, nuevos tipos de comunicaciones y nuevos automatismos, reutilizando componentes comunes de integración, clasificación y orquestación.

Asimismo, el modelo orientado a eventos facilita su integración con otros proyectos corporativos actuales y futuros.

## Aplicación real sobre necesidades existentes

El proyecto responde a una necesidad real ya identificada dentro de la organización y alineada con líneas de evolución tecnológica actualmente en marcha:

- arquitectura orientada a eventos,

- automatización de procesos,

- centralización de acciones,

- integración API-first,

- e incorporación de capacidades de inteligencia artificial aplicada a negocio.

Por tanto, el resultado del trabajo tendría aplicabilidad directa dentro de un entorno empresarial real.

# Componentes técnicos y tecnologías involucradas

El proyecto combina distintos componentes tecnológicos que abarcan integración, arquitectura orientada a eventos, inteligencia artificial y automatización de procesos.

## Integración vía APIs

- Consumo de APIs de organismos externos para la obtención de comunicaciones y documentos.

- Integración con APIs internas corporativas para la gestión de expedientes y procesos.

- Gestión de autenticación segura y trazabilidad de llamadas.

## Arquitectura orientada a eventos

- Generación de eventos a partir de comunicaciones recibidas desde sistemas externos.

- Publicación de eventos dentro del ecosistema interno.

- Consumo de eventos por distintos sistemas de forma desacoplada.

- Enrutado de información hacia colas de trabajo del sistema de ticketing según tipología y contexto.

## Inteligencia artificial y procesamiento documental

- Uso de OCR para extracción de texto desde documentos no estructurados.

- Aplicación de modelos de IA para interpretación semántica del contenido.

- Identificación de entidades relevantes (tipo de comunicación, expediente, área afectada).

- Detección de posibles acciones a ejecutar en base al contenido analizado.

- Clasificación automática de comunicaciones (siniestros, laboral, fiscal, etc.).

## Automatización de procesos (n8n)

- Uso de **n8n** como motor de automatización y orquestación de flujos.

- Definición de workflows que se activan a partir de eventos generados en el sistema.

- Ejecución de secuencias automáticas de acciones sobre distintos sistemas internos.

- Coordinación de tareas entre sistemas de ticketing, APIs y servicios internos.

- Capacidad de extender los flujos de forma incremental según nuevos casos de uso.

## Integración de agentes y MCP (Model Context Protocol)

- Uso de **MCP** como capa de integración entre modelos de inteligencia artificial y APIs corporativas.

- Exposición de capacidades del sistema (APIs internas) a agentes IA de forma estructurada.

- Permitir que los agentes puedan no solo interpretar información, sino también **invocar acciones concretas**.

- Integración de MCP dentro de los flujos de n8n para habilitar automatismos guiados por IA.

- Exploración de decisiones semi-autónomas basadas en contexto documental y reglas de negocio.

## Diseño funcional y aterrizaje técnico

Además de la implementación técnica, el proyecto implicaría:

- análisis funcional de procesos,

- definición de flujos de actuación,

- diseño de automatismos,

- identificación de casos de uso,

- y coordinación entre distintos componentes tecnológicos.

Esto aporta una dimensión adicional respecto a desarrollos puramente acotados a tareas técnicas concretas, permitiendo trabajar también capacidades de arquitectura y diseño de soluciones.
