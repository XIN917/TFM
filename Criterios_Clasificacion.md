# Criterios actuales de clasificación — DEHú/LEMA

**Criterio base:** organismo emisor + concepto → departamento. La clasificación se plantea antes de abrir el documento, con los metadatos de `localiza()`. La disponibilidad de datos adicionales todavía debe verificarse con respuestas reales de LEMA.

## Reglas confirmadas

| Organismo emisor | Condición | Departamento |
|---|---|---|
| AEAT | Cualquier concepto | Asesoría Fiscal |
| Dirección General del Catastro | Cualquier concepto | Asesoría Fiscal |
| TGSS | Embargo o levantamiento de embargo, incluidas abreviaturas | Asesoría Fiscal |
| DGSFP | Inspección, análisis de balances, solvencia o DEC | Coordinación DGS |
| DGSFP | Modelo 2B o reclamaciones | SAC |
| DGSFP | Exclusivos o mediación | Red de Mediación |
| DGSFP | Área remitente = Distribución | Red de Mediación |
| Entidad Pública Empresarial red.es | Concepto o referencia relacionados con Kit Digital (p. ej., KD) | Canal Directo |
| AEPD | Cualquier concepto | Servicio Jurídico |

**Nota:** La regla DGSFP + Distribución está confirmada por negocio, pero todavía no se sabe si LEMA proporcionará ese dato o uno equivalente.

## Criterios identificados pendientes de validación o precisión

| Organismo emisor | Condición | Destino identificado | Pendiente |
|---|---|---|---|
| TGSS | Notificaciones no relacionadas con embargo | Recursos Humanos | Validación directa con RRHH; reparto indicado por Asesoría Fiscal |
| DGSFP | Corredores | Red de Mediación | Consolidar el criterio observado |
| Ayuntamientos | Contratos o contratación con Administraciones Públicas | Sucursal Central / Administraciones Públicas | Validación con el área |
| Guardia Civil, juzgados, Conselleria de Sanidad, ayuntamientos | Casuísticas relacionadas con siniestros, daños, facturas hospitalarias, citaciones o diligencias judiciales | Área de Siniestros | Precisar y validar las reglas por organismo y concepto |

## Límites conocidos

- DGSFP no se asigna a un único departamento: depende de la materia o de una señal adicional confirmada.
- No se puede asignar automáticamente toda comunicación de un ayuntamiento, juzgado u otro organismo a Siniestros.
- Los conceptos pueden contener abreviaturas o variaciones de formato (p. ej., `LEV.EMB.` y `Modelo 2B.xml`).
- Los valores DIR3 observados en los ejemplos no son reglas universales confirmadas.
- Los avisos por correo han servido para identificar criterios, pero la entrada prevista del sistema es LEMA; todavía falta comprobar la correspondencia con datos reales de `localiza()`.

