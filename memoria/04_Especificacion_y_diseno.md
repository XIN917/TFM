# Especificación y diseño de la solución

Este capítulo presenta la especificación y el diseño de la solución propuesta para automatizar la recepción y el tratamiento de las comunicaciones procedentes de DEHú. En primer lugar, se delimitan los actores y componentes que participan en el sistema y se establecen sus requisitos funcionales y no funcionales. A continuación, se describen los casos de uso y los principales flujos de funcionamiento, para posteriormente presentar el modelo de datos y la arquitectura adoptada. Finalmente, se detallan las principales decisiones de diseño y se delimita el alcance de la solución.

## Actores y componentes del sistema

Antes de especificar los requisitos del sistema, es necesario delimitar su frontera e identificar tanto los actores externos que interactúan con él como los principales componentes que intervienen en su funcionamiento. Esta distinción permite separar las responsabilidades propias del sistema de aquellas correspondientes a servicios corporativos o externos.

### Actores externos

Se consideran actores aquellas entidades externas a la frontera del sistema que participan en alguno de sus procesos. Se han identificado tres actores principales:

- **DEHú/LEMA.** Constituye la fuente externa de las notificaciones y comunicaciones electrónicas procedentes de las Administraciones Públicas. El sistema interactúa con LEMA mediante los servicios web disponibles para Grandes Destinatarios, que permiten localizar las comunicaciones y acceder posteriormente a su contenido y a la documentación asociada.

- **Operador/Administrador.** Representa al personal interno que utiliza el frontal de la aplicación para consultar los registros almacenados y gestionar aquellos casos que requieren intervención humana. Se distinguen dos niveles de autorización. El Operador puede consultar los casos pendientes y aceptar una clasificación propuesta, mientras que el Administrador dispone, además, de permisos para corregir la asignación del departamento.

- **Departamento.** Representa al área de negocio destinataria de una comunicación una vez clasificada y derivada. El Departamento no trabaja directamente sobre el frontal desarrollado en este proyecto, sino sobre el sistema corporativo de tickets. Si detecta que una comunicación ha sido asignada incorrectamente, esta situación puede iniciar el proceso de reclasificación previsto por el sistema.

El certificado digital utilizado para la autenticación frente a LEMA no se considera un actor, ya que no interactúa con el sistema de forma autónoma, sino que constituye un mecanismo técnico de autenticación empleado en la integración con DEHú/LEMA.

### Componentes principales

Desde un punto de vista funcional, el sistema puede dividirse en cuatro bloques principales.

El primero comprende la **recepción y el procesamiento de las comunicaciones**. Incluye la integración con DEHú/LEMA, la persistencia de los datos y documentos obtenidos y los mecanismos de inteligencia artificial utilizados para su clasificación e interpretación. La clasificación del departamento se realiza antes de acceder al contenido del documento, utilizando como criterios base determinados metadatos proporcionados por LEMA, principalmente el organismo emisor y el concepto. El diseño permite incorporar otros metadatos disponibles antes de la comparecencia si posteriormente se comprueba que aportan información útil para determinar el departamento destinatario.

El segundo bloque comprende la **derivación del resultado**. Una vez que el sistema dispone de la información necesaria, se genera un evento que permite desacoplar el procesamiento de la comunicación de las acciones posteriores. El canal obligatorio en el MVP es el sistema corporativo de tickets, aunque la arquitectura contempla la posibilidad de incorporar otros canales, como el correo electrónico.

El tercer bloque comprende la **gestión y revisión humana**. Mediante un frontal propio, el Operador puede consultar los registros almacenados y revisar aquellos casos cuya clasificación automática no alcanza el nivel de confianza requerido. Esta intervención permite mantener supervisión humana sobre aquellas decisiones que el sistema no puede resolver con suficiente fiabilidad.

Finalmente, el sistema contempla un bloque de **reclasificación automática** para aquellos casos en los que un departamento detecta que una comunicación ya derivada ha sido asignada incorrectamente. En este proceso, un agente de inteligencia artificial intenta determinar una nueva clasificación y, si alcanza el nivel de confianza establecido, se procede a corregir la derivación. Si el sistema no puede resolver el caso con suficiente fiabilidad, este se deriva a revisión humana.

Esta división describe las principales responsabilidades funcionales del sistema sin entrar todavía en su implementación concreta. La estructura interna de los componentes, sus dependencias y las decisiones arquitectónicas adoptadas se desarrollan posteriormente en el apartado dedicado a la arquitectura.