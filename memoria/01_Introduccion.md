# Introducción

Este Trabajo de Fin de Máster se centra en el diseño e implementación de un sistema que automatiza la recepción y el tratamiento de las notificaciones y comunicaciones electrónicas que la Administración Pública española dirige a la empresa aseguradora MGS, Seguros y Reaseguros, S.A. Ambas se depositan en la Dirección Electrónica Habilitada única (DEHú), que es el sistema oficial a través del cual los organismos públicos notifican electrónicamente a empresas y ciudadanos.

Hoy, el personal de la empresa accede a ellas de forma manual, una a una, desde el portal de DEHú y mediante certificado digital. El proyecto sustituye este acceso manual por una conexión automatizada a través de LEMA, la modalidad de acceso a DEHú pensada para organizaciones con un gran volumen de envíos (*Grandes Destinatarios*) que necesitan automatizar su consulta [1].

El sistema automatiza tres pasos que hoy son manuales: la detección y descarga de cada nuevo envío desde DEHú, la interpretación de su contenido mediante OCR y modelos de lenguaje (LLM), y su derivación al departamento correspondiente a través del sistema de Ticketing interno de la empresa.

## Contexto

La relación entre las empresas y la Administración Pública española se canaliza, cada vez de forma más generalizada, a través de medios electrónicos. El artículo 14.2 de la Ley 39/2015, de 1 de octubre, del Procedimiento Administrativo Común de las Administraciones Públicas [2], obliga a las personas jurídicas a relacionarse electrónicamente con las Administraciones Públicas.

El Real Decreto 203/2021, de 30 de marzo, por el que se aprueba el Reglamento de actuación y funcionamiento del sector público por medios electrónicos [3], regula el sistema de notificaciones y comunicaciones electrónicas del que forma parte la Dirección Electrónica Habilitada única (DEHú): una plataforma que centraliza en un único buzón, por destinatario, las notificaciones y comunicaciones de los organismos públicos que se han ido adhiriendo progresivamente a ella (miles de administraciones y organismos, en crecimiento continuo) [4].

No es, sin embargo, el único canal de notificación electrónica existente en la Administración española: determinados organismos mantienen sistemas de notificación propios, independientes de DEHú.

En DEHú conviven dos tipos de envío con naturaleza jurídica distinta: las *notificaciones*, que trasladan formalmente un acto administrativo y producen efectos jurídicos, y las *comunicaciones*, de carácter meramente informativo, sin plazo de lectura ni efectos jurídicos derivados del acceso a su contenido [5]. Salvo que se indique lo contrario, en este trabajo se emplea el término *comunicación* en sentido amplio para referirse a cualquier envío recibido a través de DEHú, distinguiendo ambos tipos cuando la diferencia sea relevante.

Recibir a tiempo estos envíos no es una cuestión meramente operativa, especialmente en el caso de las notificaciones. Para los sujetos obligados a relacionarse electrónicamente con la Administración, una notificación se entiende practicada en el momento en que se accede a su contenido y, si no se accede, se considera rechazada una vez transcurridos diez días naturales desde su puesta a disposición, continuando el procedimiento (art. 43.2 de la Ley 39/2015 [2]). A partir de ese momento pueden empezar a correr los plazos propios de cada actuación —atender un requerimiento, presentar alegaciones, interponer un recurso— [5], cuyo incumplimiento puede tener consecuencias jurídicas y económicas directas para la organización.

En MGS Seguros, el acceso a estas comunicaciones se realiza actualmente de forma manual: distintos usuarios de diferentes áreas acceden periódicamente a DEHú —a diferencia del acceso automatizado que ofrece LEMA— empleando certificado digital, habitualmente a través del enlace directo incluido en el correo de aviso de cada envío; también es posible llegar a DEHú desde el apartado *Mis Notificaciones* de Mi Carpeta Ciudadana, un portal ciudadano que, entre otros trámites, ofrece un acceso directo a DEHú con el usuario ya identificado [6]. Los usuarios revisan los envíos pendientes, que el portal presenta en apartados separados para notificaciones y comunicaciones [5]; ninguno de ellos distingue por departamento, de modo que es el propio usuario quien actúa como filtro, examinando una a una todas las comunicaciones para determinar cuáles corresponden a su área, descargando la documentación asociada y trasladando manualmente esa información a los circuitos de trabajo internos correspondientes.

Este modelo presenta varias limitaciones relevantes. Por un lado, supone una carga operativa recurrente sobre tareas de bajo valor añadido, con el consiguiente riesgo de retrasos o comunicaciones no atendidas a tiempo. Por otro lado, el acceso manual requiere que el certificado digital con el que se accede a DEHú esté disponible en los puestos de trabajo de los usuarios, lo que incrementa la superficie de exposición y dificulta la trazabilidad de los accesos. Además, la identificación del área destinataria y la naturaleza de cada comunicación depende del criterio manual de la persona que la revisa, sin un registro sistemático que permita auditar después cómo se gestionó cada caso (Figura 1).

![Diagrama de contexto — situación actual](../Diagramas/img/diagrama_contexto_actual.png)

**Figura 1.** Situación actual: acceso manual a DEHú.

## Motivación

Este proyecto surge de una necesidad real ya identificada dentro de la organización, alineada además con líneas de evolución tecnológica que la compañía tiene actualmente en marcha: una arquitectura orientada a eventos, la automatización de procesos mediante herramientas como n8n, la integración API-first entre sistemas y la incorporación progresiva de capacidades de inteligencia artificial aplicadas a procesos de negocio.

DEHú, a través de su modalidad de acceso para Grandes Destinatarios (LEMA), ofrece un conjunto de servicios web que permiten automatizar la detección y obtención de comunicaciones sin depender de la interacción manual con su portal, lo que abre la posibilidad de sustituir el proceso actual por un flujo automatizado y centralizado.

Combinando esta integración con técnicas de reconocimiento óptico de caracteres (OCR) y modelos de lenguaje (LLM) capaces de interpretar el contenido no estructurado de los documentos recibidos, es posible no solo automatizar la recepción, sino también la clasificación de cada comunicación y su derivación al departamento correspondiente a través del sistema de Ticketing ya utilizado internamente por la compañía.

A nivel personal, este trabajo responde también a un interés auténtico por las aplicaciones prácticas de la inteligencia artificial y por la automatización de procesos con herramientas como n8n, así como al deseo de aprender y poner en práctica conocimientos nuevos en un proyecto real. A ello se suma que la oportunidad de desarrollarlo en colaboración con MGS Seguros, dentro de una relación laboral remunerada con la empresa, surgió en el momento adecuado.

## Objetivos

El objetivo general de este trabajo es diseñar e implementar un sistema que automatice la recepción y el tratamiento de las comunicaciones electrónicas procedentes de la Administración Pública a través de LEMA[^1], aplicando inteligencia artificial para interpretar y clasificar su contenido, y que integre el resultado de dicha clasificación con el sistema interno de Ticketing de MGS Seguros, reduciendo la intervención manual actualmente necesaria y mejorando los tiempos de reacción y la trazabilidad del proceso.

De este objetivo general se derivan los siguientes objetivos específicos:

- Sustituir el acceso manual a DEHú mediante certificado digital por una integración automatizada con las APIs de LEMA, capaz de detectar y descargar por sí misma las comunicaciones y la documentación asociada.
- Interpretar y clasificar automáticamente el contenido de cada comunicación mediante inteligencia artificial (OCR y modelos de lenguaje), identificando su naturaleza, el organismo emisor y el área de negocio a la que corresponde.
- Generar eventos dentro de la arquitectura orientada a eventos ya existente en la compañía, de modo que la clasificación de una comunicación quede desacoplada de la acción concreta que se ejecuta sobre ella.
- Enrutar automáticamente cada comunicación clasificada hacia la cola de trabajo correspondiente del sistema de Ticketing interno —o, como canal alternativo, notificarla por correo electrónico—, evitando el traslado manual de información entre sistemas.
- Ofrecer un mecanismo de revisión humana para los casos en los que la clasificación automática no alcance el nivel de confianza necesario, explorando además que un agente de inteligencia artificial, apoyado en el protocolo MCP, pueda corregir sus propios errores de clasificación sin depender de la disponibilidad de un operador.
- Sentar una base escalable y reutilizable, alineada con las líneas de evolución tecnológica ya en marcha en la compañía (arquitectura orientada a eventos, automatización de procesos con n8n, integración API-first), que permita incorporar en el futuro nuevos organismos, tipos de comunicaciones y automatismos adicionales.

Estos objetivos se persiguen dentro del alcance de un Producto Mínimo Viable (MVP) centrado en el ciclo de recepción de comunicaciones; los límites concretos de dicho alcance se detallan en el apartado "Alcance y límites del diseño" del capítulo de especificación y diseño de la solución.

---

**Referencias citadas en este capítulo** (ver [`Referencias.md`](./Referencias.md) para el detalle completo):

1. Guía de integración para Grandes Destinatarios (AEAD, 2026, v3.0)
2. Ley 39/2015, arts. 14.2 y 43.2 (BOE-A-2015-10565)
3. Real Decreto 203/2021 (BOE-A-2021-5032)
4. DEHú continúa su crecimiento... (IT User, 2023)
5. Preguntas frecuentes — DEHú (dehu.redsara.es)
6. Mi Carpeta Ciudadana: Mis Notificaciones (carpetaciudadana.gob.es)

[^1]: DEHú es el sistema; LEMA es la forma automática de acceder a él.