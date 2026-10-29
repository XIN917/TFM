# Criterios de clasificación — DEHú/LEMA

**Fuente única** del reparto por departamento. El resto de documentos cita los IDs de este fichero y no repite las reglas. Las reglas en estado «Prompt» se copian al prompt versionado (v1.0 usa R01–R09), que usa n8n para clasificar, y a `PRD.md` §6.

**Solo criterios generales**, base del prompt zero-shot. Los ejemplos (few-shot y evaluación) no se guardan en este repo.

**Principio:** organismo emisor + concepto de `localiza()` → departamento, **antes de abrir el documento**. Otro metadato previo a la comparecencia (`metadatosPublicos`, etc.) entra solo si una muestra real de LEMA demuestra que discrimina. Sin regla aplicable: sin clasificar; con reglas en conflicto: revisión humana.

## Departamentos de destino

Asesoría Fiscal · Coordinación DGS · SAC · Red de Mediación · Canal Directo · Recursos Humanos · Área de Siniestros.

Pendiente de catálogo: Sucursal Central. Las subáreas (p. ej. de Siniestros) y las unidades internas (p. ej. Servicio Jurídico y de Cumplimiento Normativo, dentro de RRHH) no son departamentos de clasificación.

## Reglas

Estados: **Prompt** (entra en el prompt de clasificación) · **Confirmada** (negocio la confirma; fuera del prompt por el bloqueo indicado) · **Pendiente** (observada; no convertir en regla hasta validarla).

| ID | Organismo emisor | Condición | Departamento | Estado | Bloqueo / nota |
|---|---|---|---|---|---|
| R01 | AEAT | Cualquier concepto | Asesoría Fiscal | Prompt | El concepto suele ser genérico; basta el organismo |
| R02 | Dirección General del Catastro | Cualquier concepto | Asesoría Fiscal | Prompt | |
| R03 | TGSS | Embargo o levantamiento de embargo (incluidas abreviaturas) | Asesoría Fiscal | Prompt | Conciliar con RRHH el destino inicial antes de cambiarla |
| R04 | DGSFP | Inspección, análisis de balances, solvencia o DEC | Coordinación DGS | Prompt | |
| R05 | DGSFP | Modelo 2B o reclamaciones | SAC | Prompt | |
| R06 | DGSFP | Exclusivos o mediación | Red de Mediación | Prompt | |
| R07 | DGSFP | Los metadatos permiten identificar el área remitente Distribución | Red de Mediación | Prompt | Solo aplica si la señal aparece en `localiza()`; comprobar en LEMA dónde llega |
| R08 | Entidad Pública Empresarial red.es | Concepto o referencia de Kit Digital (p. ej. KD) | Canal Directo | Prompt | «Kit Digital» no es literal obligatorio. La referencia de acuerdo sirve además para agrupar envíos (fuera del MVP) |
| R09 | AEPD | Cualquier concepto | Recursos Humanos | Prompt | Lo gestiona el Servicio Jurídico y de Cumplimiento Normativo (unidad de RRHH). Confirmar la cola de Ticketing antes de derivar |
| R10 | TGSS | Conceptos no relacionados con embargo | Recursos Humanos | Confirmada | Faltan ejemplos y excepciones |
| R11 | DGSFP | Corredores | Red de Mediación | Pendiente | Consolidar el criterio observado |
| R12 | Ayuntamientos | Contratos o contratación con Administraciones Públicas | Sucursal Central | Pendiente | Validar con el área |
| R13 | Órganos judiciales (secciones civil, de instrucción o social de los tribunales de instancia) u órganos de gestión de la Administración de Justicia | El concepto identifica un procedimiento judicial (sección, plaza y número de procedimiento) o una actuación procesal (traslado, emplazamiento, desistimiento) | Área de Siniestros | Prompt | Incluye la sección social; RRHH mencionó notificaciones judiciales de empleados sin destino validado |
| R14 | Administraciones autonómicas y sus consejerías | Facturas o recursos sobre facturas, facturas hospitalarias, requerimientos de abono de precio público o citaciones por tráfico | Área de Siniestros | Prompt | |
| R15 | Ayuntamientos u otras administraciones | Daños a bienes públicos | Área de Siniestros | Prompt | |
| R16 | Guardia Civil | Citaciones o diligencias | Área de Siniestros | Prompt | |

## Límites conocidos

- DGSFP no va a un único departamento: decide la materia (R04–R07).
- Siniestros tiene varios tipos (R13–R16), sin subáreas en el MVP. El organismo solo no basta: ayuntamientos y comunidades autónomas también envían contratación (R12), tributos, etc.
- Los conceptos admiten abreviaturas y variación de formato; no se exige igualdad exacta.
- Los DIR3 observados son ejemplos, no clave de la regla.
- Los avisos por correo y los portales propios (p. ej. el de Seguridad Social que usa RRHH) sirven para descubrir reglas; la entrada del clasificador es solo DEHú/LEMA.
- Distinciones que dependen del contenido del documento (p. ej. embargos de empleados frente a mediadores, deudas vinculadas a siniestros) no son reglas por metadatos.
- Área de Siniestros no es un destino por defecto: solo se asigna cuando otra regla lo excluye con claridad (R13–R16); un caso dudoso va a revisión humana, no a Siniestros.

## Cómo añadir o cambiar un criterio

1. Nueva fila con el siguiente ID libre (los IDs no se reutilizan ni se renumeran) o editar la existente. Sin nombres de personas.
2. Rellenar organismo, condición, departamento, estado y bloqueo, en términos generales y comprobando que el dato llega en `localiza()`. Los casos concretos no se añaden aquí.
3. Si hay discrepancia con una regla existente, anotarla en la nota sin sustituir la regla hasta validarla.
4. Si el estado es **Prompt** o pasa a serlo: nueva versión del prompt y cargarla en n8n. `PRD.md` §6 solo cambia si se añade o retira un departamento (ver punto 5); no repite reglas.
5. Si cambia el catálogo de departamentos: actualizar también las categorías del prompt y la semilla de `DEPARTAMENTO` (los nombres deben coincidir).
