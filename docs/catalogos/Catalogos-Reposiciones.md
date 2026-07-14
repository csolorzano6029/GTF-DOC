# Catalogos - Reposiciones

Fecha de corte: 2026-07-13

## 1. Objetivo

Documentar los catalogos funcionales que intervienen en Reposiciones (legacy y migrado),
con su codigo, uso y reglas de paridad.

## 2. Catalogos tipo detectados en propiedades legacy

Fuente base: GTFAplicacion.properties.

| Codigo catalogo tipo | Nombre funcional legacy    |
| -------------------- | -------------------------- |
| 300                  | estadoDeposito             |
| 301                  | formaDeposito              |
| 302                  | conceptoDeposito           |
| 303                  | estadoRembolso             |
| 304                  | catalogoConceptos          |
| 305                  | doucumentoTipoRembolso     |
| 306                  | movimientoRembolso         |
| 307                  | tipoViaje                  |
| 327                  | catalogoConceptosDocumento |
| 28                   | impuestoTipoRetencion      |
| 19                   | dinero                     |

Nota:

- Este bloque es inventario inicial de codigos tipo.
- Cada catalogo se ira ampliando con sus catalogo valor y campos API impactados.

## 3. Catalogo 305 - Tipos de documento de reposicion

- Codigo de catalogo tipo: `305`
- Nombre funcional: Tipos de documento de reposicion
- Regla legacy: el valor enviado/guardado es el codigo de catalogo valor.
- Regla de etiqueta: el nombre visible se toma de `nombreCatalogoValor` en BD.

### 2.1 Codigos funcionales identificados

| documentType (codigo) | Clave funcional legacy | documentTypeName (etiqueta)                  |
| --------------------- | ---------------------- | -------------------------------------------- |
| REN                   | reciboManual           | Sale de catalogo 305 (`nombreCatalogoValor`) |
| NOV                   | notaVenta              | Sale de catalogo 305 (`nombreCatalogoValor`) |
| RET                   | retencion              | Sale de catalogo 305 (`nombreCatalogoValor`) |
| FAC                   | factura                | Sale de catalogo 305 (`nombreCatalogoValor`) |
| FEL                   | factura.electronica    | Sale de catalogo 305 (`nombreCatalogoValor`) |
| RTM                   | retencion.manual       | Sale de catalogo 305 (`nombreCatalogoValor`) |
| RTE                   | retencion.electronica  | Sale de catalogo 305 (`nombreCatalogoValor`) |

Nota:

- En entorno real, la etiqueta exacta depende de los valores activos en BD para catalogo 305.
- Ejemplo confirmado: `REN` -> `Recibo manual`.

## 4. Valores relacionados

| Valor | Significado         | Observacion                                            |
| ----- | ------------------- | ------------------------------------------------------ |
| -1    | No asignado         | Legacy lo excluye al cargar opciones de tipo documento |
| NA    | No asignado (texto) | Valor de apoyo para estados/no asignado                |

## 5. Paridad en backend migrado

1. `documentType` se retorna como codigo (ejemplo: `REN`).
2. `documentTypeName` se resuelve por lookup al catalogo tipo `305` usando el codigo.
3. No hay hardcode de equivalencia `codigo -> nombre` para este campo.
4. Si el codigo no existe en catalogo 305, `documentTypeName` puede venir nulo.

## 6. Fuentes revisadas

- `gtf-root/gtf-web-intranet/src/main/resources/ec/com/smx/gtf/web/commons/resources/GTFWeb.properties`
- `gtf-root/gtf-modelo/src/main/resources/ec/com/smx/fin/gtf/cliente/commons/resources/GTFAplicacion.properties`
- `gtf-root/gtf-modelo/src/main/java/ec/com/smx/fin/gtf/cliente/commons/util/GTFConstantes.java`
- `gtf-root/gtf-web-intranet/src/main/java/ec/com/smx/gtf/web/utils/reposiciones/AdminReposicionesUtils.java`
- `gtf-root/gtf-web-intranet/src/main/webapp/pages/rembolsos/administracion/nuevoRembolsoCenter.xhtml`
- `gtf-replacements-root/gtf-replacements-core/src/main/java/ec/com/smx/gtf/replacements/replenishment/service/ReplenishmentService.java`
- `gtf-replacements-root/gtf-replacements-core/src/main/java/ec/com/smx/gtf/replacements/replenishment/repository/ReplenishmentDetailRepository.java`
- `gtf-replacements-root/gtf-replacements-core/src/main/java/ec/com/smx/gtf/replacements/replenishment/service/ReplenishmentDetailDisplayMapper.java`

## 7. Proximo uso

Este archivo queda como inventario incremental. Cada nuevo catalogo identificado se agrega con:

- codigo catalogo tipo
- codigos catalogo valor relevantes
- campo API impactado
- origen legacy
- regla de paridad migrada

## 8. Catalogos legacy para frontend via endpoint /catalogs/{catalogTypeCode}/values

Objetivo de esta seccion:

- Listar que catalogos del flujo legacy de Reposiciones se pueden consultar con el endpoint generico de catalogos.
- Entregar al frontend el codigo de catalogo tipo y su referencia funcional.

### 8.1 Catalogos con uso directo en pantalla (legacy)

| catalogTypeCode | Nombre funcional legacy | Referencia para frontend                                          | Uso en legacy (pantalla/flujo)                |
| --------------- | ----------------------- | ----------------------------------------------------------------- | --------------------------------------------- |
| 303             | estadoRembolso          | Estados de reposicion (filtro de busqueda y estado de solicitud)  | Combo Estado en Administracion y Revision     |
| 305             | doucumentoTipoRembolso  | Tipos de documento de detalle (REN, NOV, RET, FAC, FEL, RTM, RTE) | Combo Tipo documento en detalle de reposicion |
| 304             | catalogoConceptos       | Categorias de conceptos para popup de seleccion de concepto       | Popup de conceptos por area/transaccion       |
| 28              | impuestoTipoRetencion   | Tipos de impuesto para retencion (ej. IVA, REN)                   | Popup Validar retencion                       |

### 8.2 Catalogos referenciales en reposiciones (no siempre cargados como combo directo)

| catalogTypeCode | Nombre funcional legacy    | Referencia para frontend                      | Observacion de uso                                                                                              |
| --------------- | -------------------------- | --------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| 306             | movimientoRembolso         | Tipos de movimiento/transaccion de reposicion | Se usa como contexto transaccional; en legacy se resuelve principalmente por transaccion del sistema            |
| 307             | tipoViaje                  | Tipo de viaje (gastos personales)             | Se registra en detalle (`codCatTipViaTip`), pero no se identifico carga directa de combo catalogo en este flujo |
| 327             | catalogoConceptosDocumento | Relacion concepto-documento                   | Definido en propiedades; sin consumo directo identificado en pantallas/controladores de reposiciones revisados  |

Notas para consumo frontend:

1. Base endpoint: `/gtfReplacementsServices/api/v1/catalogs/{catalogTypeCode}/values`.
2. Para paridad con legacy, priorizar `onlyActive=true` en consultas de catalogos.
3. En `305`, excluir `-1` (No asignado) cuando se requiera replicar el comportamiento legacy del combo de tipo documento.
