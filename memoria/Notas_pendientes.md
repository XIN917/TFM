# Notas pendientes de redactar

Material para capítulos que aún no tienen borrador. Ordenado según la estructura de [`Estructura_Memoria.md`](../Estructura_Memoria.md). Al redactar un apartado, pasar la nota al capítulo y borrarla de aquí.

---

## Criterio de nombres en la memoria (07/10/2026)

Pendiente de confirmar con MGS qué se puede incluir (ver `TODO.md`). Mientras tanto:

- En el texto se habla de **módulos por su función**. La primera vez que aparece la estructura, se define una vez a qué módulo corresponde cada nombre corto, y a partir de ahí se puede usar cualquiera de las dos formas.
- Nombres cortos, sin el prefijo del proyecto: Business, ApiFront, Beans, Comun, EAR, Test, Web.
- Explicar «Beans» y «Comun» al definirlos: en Java, «Beans» se asocia a JavaBeans o CDI.
- Los sistemas corporativos van en genérico: el sistema de tickets corporativo, su proyecto de tests funcionales, el sistema corporativo de autenticación y autorización, la librería corporativa de componentes de interfaz.
- Sí se nombran las tecnologías públicas (Java EE 7, WebSphere, JAX-RS, CDI, SQL Server, Vue 3, Vite, Vitest, Playwright, JUnit), los patrones (arquitectura hexagonal, repositorio, adaptador) y DEHú/LEMA.
- No van en la memoria: código de librerías internas ni del sistema de tickets, URLs o nombres de servidores internos, credenciales ni datos reales de envíos.

Tabla para definir los módulos al principio (borrador):

| Módulo | Nombre | Contenido |
|---|---|---|
| Dominio y aplicación | Business | Modelo de dominio, servicios de aplicación e infraestructura de persistencia |
| Adaptador REST | ApiFront | API del frontal (JAX-RS), contrato OpenAPI y mappers. Se despliega como WAR |
| Compartidos | Beans, Comun | Código común a varios módulos (qué hay exactamente en cada uno: concretar al redactar) |
| Empaquetado | EAR | No contiene código: agrupa el WAR y los JAR para desplegar en WebSphere |
| Tests | Test | Tests unitarios y de integración del backend. No se despliega |
| Frontal | Web | Aplicación de una sola página (Vue 3 + TypeScript + Vite) |

---

## Especificación y diseño: requisitos no funcionales

### Accesibilidad (07/10/2026)

- Hoy no hay ningún requisito de accesibilidad en el PRD ni en la especificación. Hay que tenerla en cuenta siempre, y por eso mismo conviene escribirla como requisito: lo que no está en los requisitos no se diseña, no se prueba y nadie lo revisa. Ejemplo real: al hacer clicable toda la fila de la tabla se rompió el acceso por teclado y nada lo detectó.
- Propuesta de redacción:
  > **RNF-xx Accesibilidad.** El frontal del Operador cumple WCAG 2.1 nivel AA. En particular: todas las funciones se pueden usar solo con el teclado; cada control tiene un nombre accesible; los errores y los cambios de estado (resultados de una búsqueda, error de carga) se anuncian a los lectores de pantalla; y el contraste del texto es de al menos 4,5:1.
  > **Verificación:** tests funcionales automáticos que analizan cada vista con axe (WCAG 2.1 AA) y recorren con el teclado los casos de uso principales, y una revisión manual con un lector de pantalla.
- Justificación para el texto, con cuidado con lo legal:
  - El RD 1112/2018 (EN 301 549 / WCAG) obliga al sector público; la empresa no lo es. La Ley 11/2023 (Directiva Europea de Accesibilidad) se centra en productos y servicios para consumidores; una herramienta interna no lo es. Confirmar con el ponente antes de afirmar algo legal.
  - Lo que sí aplica en una herramienta interna: si la usa un empleado con discapacidad, la empresa debe hacer ajustes razonables. Accesible desde el diseño es más barato que adaptarlo después.
  - Encaja también en «Análisis de sostenibilidad e implicaciones éticas» (dimensión social).
- Pendiente: incorporar el RNF a `Especificacion_Requisitos.md` y al PRD.

---

## Desarrollo de la solución

### Stack tecnológico / Decisiones técnicas: organización en módulos (07/10/2026)

- El backend no es una sola carpeta: es un módulo (proyecto del IDE) por capa, empaquetados en un EAR para WebSphere 9.
- Por qué un módulo por capa: cada módulo tiene su propio classpath, así que **el compilador hace cumplir la dirección de las dependencias** de la arquitectura hexagonal. Business no tiene ApiFront en su classpath, y si importara algo del adaptador no compilaría. En una única carpeta `backend`, esa regla dependería solo de la disciplina.
- Coste: más módulos y más configuración del IDE.
- Es la convención de la empresa (el sistema de tickets corporativo se organiza igual). Equivale a un proyecto Maven o Gradle multimódulo.
- El «backend / frontend» habitual de un proyecto académico corresponde aquí a un nivel superior: el conjunto de módulos Java por un lado y el módulo Web por otro.
- Para el texto, mejor un diagrama de componentes con las capas y las flechas de dependencia que una lista de módulos. Base: `Diagramas/Diagrama_Componentes.md`.

---

## Experimentación y evaluación de la solución

### Metodología de pruebas: dónde viven los tests y por qué (07/10/2026)

- **Backend:** tests en un módulo aparte, Test (JUnit 5).
  - Motivo: en el IDE corporativo (RAD/Eclipse, sin Maven), cada módulo se empaqueta entero. Con los tests dentro de Business, acabarían en su JAR desplegado en el servidor, con JUnit como dependencia en producción.
  - Comprobado: el EAR solo contiene el WAR de ApiFront y los JAR de Business, Beans y Comun; Test no se despliega.
  - Los tests usan los mismos paquetes Java que el código, así que acceden a lo que no es `public` igual que en Maven.
  - Es la misma separación que Maven hace con `src/main/java` y `src/test/java`, resuelta con un módulo en vez de una carpeta. No afecta a la arquitectura: es empaquetado, no diseño.
- **Frontal:** tests dentro del propio módulo Web, en `tests/` (reflejando `src/`), con Vitest.
  - Aquí no hay problema de empaquetado: Vite solo incluye en el build lo que se importa desde el punto de entrada, y nadie importa los tests.
- **E2E** (de extremo a extremo, navegador → API → BBDD): en un proyecto aparte, como hace el sistema de tickets corporativo, con Playwright. Prueban el sistema entero y se lanzan contra cualquier entorno cambiando la configuración. Pendientes de que la aplicación tenga roles en el sistema de autorización: **trabajo futuro**.
  - Actualización 07/10/2026: ya hay E2E en local, donde la aplicación funciona sin login. Diez tests de la consulta de envíos (RF-11) contra datos de prueba conocidos: lista, pestañas, filtros por estado y fechas, lista vacía, validación del rango, Limpiar, detalle y vuelta a la pestaña correcta, envío inexistente. Contra los entornos de la empresa siguen esperando a los roles: eso queda como trabajo futuro.
  - Para el texto: los E2E dependen de los datos. Se usa un juego de datos fijo y conocido (ocho envíos), y cada test comprueba identificadores concretos. Pendiente: cubrir la paginación y el error de carga.
- Idea para el texto: cada tipo de test vive donde lo exigen su herramienta de empaquetado y lo que prueba. Los unitarios, junto a su código (en un módulo aparte en el backend por la limitación del IDE); los E2E, fuera, porque prueban todo el sistema.

### Accesibilidad: revisión del frontal (07/10/2026)

- Método: inspección del árbol de accesibilidad y del DOM de la lista de envíos en el navegador; prueba de navegación con teclado.
- Resultado:
  - Cubierto por la librería corporativa de componentes: idioma del documento, encabezado único, regiones de navegación y contenido, etiquetas asociadas a cada filtro, errores de validación enlazados al campo (`aria-describedby`), cabeceras de tabla, pestañas con rol, alertas anunciadas (`aria-live`) y estado como texto, no solo como color.
  - Fallo encontrado y corregido: la fila de la tabla era clicable con ratón pero no recibía foco, así que el detalle no era accesible con el teclado. Solución: el identificador vuelve a ser un enlace, manteniendo el clic en toda la fila.
  - Mejoras añadidas: anuncio para lectores de pantalla del número de resultados, de la lista vacía o del error de carga; idioma del documento sincronizado con el de la interfaz.
  - Limitaciones de la librería corporativa (fuera del alcance; pendiente de comunicarlas al equipo que la mantiene): el botón de calendario sin nombre accesible y los iconos sin ocultar al lector de pantalla.
- Automatización (mismo día): módulo de tests funcionales aparte con Playwright y axe-core, siguiendo el patrón del sistema de tickets corporativo. Recorre la lista (las dos pestañas), los filtros con un error de validación y el detalle, y analiza cada pantalla con las reglas de WCAG 2.1 A y AA; además comprueba que el detalle se abre solo con el teclado. Genera un informe HTML con los fallos de cada pantalla y la traza paso a paso. De momento, estos tests se quedan solo en local, fuera del repositorio.
  - Resultado: sin fallos propios ni de contraste. axe confirmó los fallos de la librería y encontró dos más que la revisión manual no vio (atributos ARIA en un elemento sin rol en el campo de fecha; pestañas sin contenedor `tablist`).
  - Decisión: los fallos conocidos de la librería se registran como anotaciones en el informe, pero no hacen fallar el test. Si no, fallaría siempre y un fallo nuevo pasaría desapercibido.
  - Idea para el texto: lo automático y lo manual se complementan. axe encontró fallos de ARIA que no se ven a simple vista; el fallo de teclado de la fila, en cambio, no lo detecta ninguna regla de axe, y por eso se automatizó como prueba de navegación aparte.
- Idea para el texto: la mayor parte de la accesibilidad la aporta la librería de componentes, pero no basta. Las decisiones de interacción propias (como hacer clicable una fila) pueden romperla, y solo se detecta probando con el teclado.
- Hallazgo relacionado (problemas encontrados): el campo de fecha de la librería corrige solo las fechas imposibles al salir del campo (`30/02/2026` → `02/03/2026`, con un aviso visual breve), porque construye la fecha con `new Date(año, mes, día)` y JavaScript desborda el día al mes siguiente. Ventaja: el modelo nunca tiene una fecha imposible. Inconveniente: cambia lo escrito sin explicarlo. La validación propia queda como segunda barrera.
