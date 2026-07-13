# BACK-FRONT - Mapeo API vs Frontend

Fecha de corte: 2026-07-13

## 1. Objetivo

Relacionar cada API del backend con su uso en frontend:

- menu u opcion
- pantalla o componente
- boton o accion
- estado actual de implementacion

## 2. Convencion de nombres (ajuste solicitado)

Regla aplicada desde este punto:

- En nombres de metodos Java no se usara prefijo `get`; se usara `find` cuando corresponda.
- En endpoints HTTP, por estandar del proyecto, solo se usan verbos `GET` y `POST`.
- En rutas URL no se usa `get` en el path; se mantiene estilo actual por recurso.

Ejemplos aplicados:

- `getCurrentFundHeader` -> `findCurrentFundHeader`
- `getCurrentFund` -> `findCurrentFund`
- `getByCode` -> `findByCode`
- `getNextReplenishmentId` -> `findNextReplenishmentId`

## 3. Estado de endpoints existentes (antes de P0)

Decision:

- Los endpoints actuales se mantienen y NO se reemplazan.
- Se completan por evolucion incremental para no romper front ni pruebas ya validadas.

Brecha puntual de los endpoints existentes:

| Endpoint actual                   | Estado  | Que falta para quedar completo contra legacy                                                                            |
| --------------------------------- | ------- | ----------------------------------------------------------------------------------------------------------------------- |
| GET /replenishments/current-fund  | Parcial | Incluir validaciones operativas de precheck (bloqueo/alerta por porcentaje, solicitudes pendientes y cheque emitido)    |
| POST /replenishments/findByFilter | Parcial | Forzar filtro por area de trabajo del usuario y mejorar soporte de filtros de fecha/estado como en legacy               |
| GET /replenishments/{id}          | Parcial | Incluir detalle completo de documentos y contexto operativo para edicion/revision                                       |
| POST /replenishments              | Parcial | Aplicar validaciones de detalle legacy (IVA, duplicados, monto maximo, reglas por transaccion, disponibilidad de fondo) |
| POST /replenishments/{id}         | Parcial | Actualizar cabecera en estado pendiente (el detalle se actualiza por endpoints `/details`)                              |

Campos de salida para front en listados/consulta por id:

- `transactionCode`: codigo tecnico del tipo de reposicion (`1`, `2`).
- `replenishmentType`: descripcion legible (`Caja Chica`, `Gastos Personales`).
- `replenishmentStatus`: codigo tecnico de estado (`PEN`, `ENV`, `PAG`, etc.).
- `replenishmentStatusName`: etiqueta legible de estado para UI (`Pendiente`, `Enviado`, `Pagado`, etc.).
- `sendAlert`: bandera de alerta visual de envio (`1`/`0`) para resaltar filas en listado.
- `createdDate` y `replenishmentDate`: equivalentes para fecha de reposicion (legacy `fechaRegistro`).
- `workAreaName`: nombre legible del local para mostrar junto a `workAreaCode` (ejemplo: `186 - EL JARDIN`).
- `responsiblePersonDocument`: numero de documento del responsable para UI (`Registrado por - Numero cedula`), resuelto desde `responsiblePersonId`.

## 4. Matriz BACK-FRONT (Administracion)

Base URL: `/gtfReplacementsServices/api/v1`

| Modulo front                   | Menu/Opcion                   | Pantalla/Componente     | Boton/Accion            | Endpoint backend                                    | Estado backend                        | Uso esperado                                    |
| ------------------------------ | ----------------------------- | ----------------------- | ----------------------- | --------------------------------------------------- | ------------------------------------- | ----------------------------------------------- |
| Administracion de Reposiciones | Reposiciones > Administracion | Lista de reposiciones   | Buscar                  | POST /replenishments/findByFilter                   | Implementado (parcial)                | Cargar grilla con filtros y paginacion          |
| Administracion de Reposiciones | Reposiciones > Administracion | Carga de catalogos UI   | Obtener opciones        | GET /catalogs/{catalogTypeCode}/values              | Implementado                          | Poblar combos transversales (estado, tipo, etc) |
| Administracion de Reposiciones | Reposiciones > Administracion | Cabecera de fondo       | Refrescar cabecera      | GET /replenishments/current-fund?workAreaCode={id}  | Implementado (parcial)                | Mostrar fondo, saldo, alertas y estado actual   |
| Administracion de Reposiciones | Reposiciones > Administracion | Formulario cabecera     | Nueva reposicion        | POST /replenishments                                | Implementado (parcial)                | Crear solicitud en estado pendiente             |
| Administracion de Reposiciones | Reposiciones > Administracion | Formulario cabecera     | Guardar cambios         | POST /replenishments/{replenishmentId}              | Implementado (parcial)                | Editar cabecera en estado pendiente             |
| Administracion de Reposiciones | Reposiciones > Administracion | Vista detalle solicitud | Abrir reposicion        | GET /replenishments/{replenishmentId}               | Implementado (parcial)                | Consultar solo cabecera de reposicion por id    |
| Administracion de Reposiciones | Reposiciones > Administracion | Tabla de detalle        | Cargar detalle          | GET /replenishments/{id}/details                    | Implementado (P0 - corte 5)           | Consultar lineas de detalle por reposicion      |
| Administracion de Reposiciones | Reposiciones > Administracion | Tabla de detalle        | Agregar fila            | POST /replenishments/{id}/details                   | Implementado (P0 - corte 3)           | Crear linea de detalle                          |
| Administracion de Reposiciones | Reposiciones > Administracion | Tabla de detalle        | Editar fila             | POST /replenishments/{id}/details/{detailId}        | Implementado (P0 - corte 4)           | Actualizar linea de detalle                     |
| Administracion de Reposiciones | Reposiciones > Administracion | Tabla de detalle        | Eliminar fila           | POST /replenishments/{id}/details/{detailId}/delete | Pendiente P0                          | Eliminar linea y recalcular totales             |
| Administracion de Reposiciones | Reposiciones > Administracion | Formulario de documento | Validar IVA             | POST /replenishments/validate-vat                   | Implementado (P0 - corte 1)           | Validar reglas IVA legacy                       |
| Administracion de Reposiciones | Reposiciones > Administracion | Formulario de documento | Validar duplicado       | POST /replenishments/validate-duplicate             | Implementado (P0 - corte 2)           | Validar no duplicidad por RUC+documento         |
| Administracion de Reposiciones | Reposiciones > Administracion | Flujo de envio          | Enviar reposicion       | POST /replenishments/{id}/send                      | Pendiente P0                          | Cambiar estado y ejecutar validaciones de envio |
| Administracion de Reposiciones | Reposiciones > Administracion | Flujo de envio          | Seleccionar responsable | GET /replenishments/responsibles                    | Pendiente P1                          | Cargar responsables para pago por ventanilla    |
| Administracion de Reposiciones | Reposiciones > Administracion | Acciones de solicitud   | Anular                  | POST /replenishments/{id}/cancel                    | Pendiente (definir Admin vs Revision) | Anulacion con observacion                       |
| Administracion de Reposiciones | Reposiciones > Administracion | Acciones de solicitud   | Imprimir                | GET /replenishments/{id}/print                      | Pendiente P1                          | Generar comprobante                             |

## 4.1 Contrato exacto de campos (legacy vs APIs implementadas)

Fuentes de verdad usadas para este corte:

- Legacy UI listado: `SOPORTES/gtf-root/gtf-web-intranet/src/main/webapp/pages/rembolsos/administracion/adminRembolso.xhtml` y `adminRembolsoCenter.xhtml`.
- Legacy UI cabecera/detalle: `SOPORTES/gtf-root/gtf-web-intranet/src/main/webapp/pages/rembolsos/administracion/nuevoRembolsoCenter.xhtml`.
- Legacy controller/filtros: `SOPORTES/gtf-root/gtf-web-intranet/src/main/java/ec/com/smx/gtf/web/administracion/controller/AdminRembolsoController.java`.
- Contrato real migrado: `gtf-replacements-services/.../ReplenishmentController.java`, `gtf-replacements-core/.../ReplenishmentService.java`, `gtf-replacements-vo/...`.

### 4.1.1 Pantalla legacy y campos que consume

| Pantalla legacy     | Campo visual legacy (binding)                                      | Equivalente en API migrada                                                        | Estado |
| ------------------- | ------------------------------------------------------------------ | --------------------------------------------------------------------------------- | ------ |
| Listado (grilla)    | `id.codigoReposicion`                                              | `replenishmentId`                                                                 | OK     |
| Listado (grilla)    | `fechaRegistro`                                                    | `replenishmentDate` / `createdDate`                                               | OK     |
| Listado (grilla)    | `transaccionSistemaDTO.nombre`                                     | `replenishmentType`                                                               | OK     |
| Listado (grilla)    | `catalogoValorEstadoSolicitudDTO.nombreCatalogoValor`              | `replenishmentStatusName` (manteniendo `replenishmentStatus` como codigo tecnico) | OK     |
| Listado (grilla)    | `valorTotal`                                                       | `requestedTotal`                                                                  | OK     |
| Listado (grilla)    | `alertaEnvioReposicion` (color rojo/normal)                        | `sendAlert`                                                                       | OK     |
| Cabecera visual     | `areTraDepRepDTO.areaTrabajo.codigoReferencia - nombreAreaTrabajo` | `workAreaCode` + `workAreaName`                                                   | OK     |
| Cabecera visual     | `personaResDTO.numeroDocumento`                                    | `responsiblePersonDocument`                                                       | OK     |
| Cabecera visual     | `personaResDTO.nombreCompleto`                                     | `responsiblePersonName`                                                           | OK     |
| Cabecera visual     | `valorSubtotal` / `valorTotal`                                     | `requestedSubtotal` / `requestedTotal`                                            | OK     |
| Fondos area trabajo | `fondoAsignado`, `reposicionesPendientesPago`, `saldoEfectivo`     | `assignedFund`, `pendingPaymentAmount`, `cashBalance`                             | OK     |
| Fondos area trabajo | `totalDocNoEnviados`                                               | No existe en `FundHeaderVo`                                                       | Gap    |
| Detalle visual      | `catalogoValorDocTipDTO.nombreCatalogoValor`                       | `documentType` + `documentTypeName` (cuando existe en catalogo)                   | OK     |
| Detalle visual      | `fechaConcepto`                                                    | `documentDate`                                                                    | OK     |
| Detalle visual      | `numeroRucFactura` / `numeroDocumento`                             | `taxId` / `documentNumber`                                                        | OK     |
| Detalle visual      | `comprobanteVenta`                                                 | `saleReceipt`                                                                     | OK     |
| Detalle visual      | `conceptoAreTraDTO.concepto.nombre`                                | `billingConceptDescription`                                                       | OK     |
| Detalle visual      | `cantidad`                                                         | `quantity`                                                                        | OK     |
| Detalle visual      | `catalogMeasureUnit.nombreCatalogoValor`                           | `measurementUnitName` (con `measurementTypeCode` + `measurementValueCode`)        | OK     |
| Detalle visual      | `nombreArchivo`                                                    | `fileName`                                                                        | OK     |
| Detalle visual      | `valorIva` / `valorDetalleReposicion`                              | `vatValue` / `requestedValue`                                                     | OK     |

### 4.1.1.1 Mapeo puntual de filtro/listado (pantalla legacy)

Esta subseccion define el mapeo operativo de la pantalla legacy de Administracion para que Front y Back hablen el mismo idioma funcional.

| Zona pantalla legacy                    | Etiqueta visible front | Campo API migrada                                  | Observacion de integracion                                                               |
| --------------------------------------- | ---------------------- | -------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| Criterios de busqueda                   | Estado reposicion      | `filters[].parameterPattern = replenishmentStatus` | Enviar codigo tecnico de estado (`PEN`, `ENV`, `VAL`, `PAG`, `CHE`, `COB`, `ANU`).       |
| Criterios de busqueda                   | Fecha reposicion Desde | `filters[].parameterPattern = replenishmentDate`   | `replenishmentDate` es equivalente funcional de `createdDate` en backend migrado.        |
| Criterios de busqueda                   | Fecha reposicion Hasta | `filters[].parameterPattern = replenishmentDate`   | Usar el mismo campo base de fecha; la logica de comparador depende del `SearchModelDTO`. |
| Listado de solicitudes (columna grilla) | Nro.                   | N/A                                                | Es numeracion visual de fila; no corresponde a un atributo de dominio en la API.         |
| Listado de solicitudes (columna grilla) | Nro. reposicion        | `replenishmentId`                                  | Identificador de la solicitud.                                                           |
| Listado de solicitudes (columna grilla) | Fecha reposicion       | `replenishmentDate`                                | Alternativamente puede usarse `createdDate` por equivalencia funcional actual.           |
| Listado de solicitudes (columna grilla) | Tipo reposicion        | `replenishmentType`                                | Valor legible (`Caja Chica` / `Gastos Personales`).                                      |
| Listado de solicitudes (columna grilla) | Estado                 | `replenishmentStatusName`                          | Mostrar etiqueta legible; conservar `replenishmentStatus` para logica tecnica/filtros.   |
| Listado de solicitudes (columna grilla) | Total                  | `requestedTotal`                                   | Corresponde al valor total mostrado en legacy para la grilla de administracion.          |

Mapa recomendado de valores para combo "Estado reposicion":

| Valor visible en front | Valor tecnico para filtro `replenishmentStatus` |
| ---------------------- | ----------------------------------------------- |
| Todos                  | No enviar filtro de estado                      |
| Pendiente              | `PEN`                                           |
| Enviado                | `ENV`                                           |
| Validado               | `VAL`                                           |
| Pagado                 | `PAG`                                           |
| Cheque emitido         | `CHE`                                           |
| Cobrado                | `COB`                                           |
| Anulado                | `ANU`                                           |

### 4.1.1.2 Mapeo puntual de formulario de cabecera (crear/editar)

Esta subseccion aterriza el formulario de cabecera de Administracion contra los endpoints `POST /replenishments` y `POST /replenishments/{replenishmentId}`.

| Campo visual en front (legacy)             | Campo API create (`POST`)                           | Campo API update (`POST`)  | Regla backend vigente                                                                                                                                  |
| ------------------------------------------ | --------------------------------------------------- | -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Local (codigo + nombre)                    | `workAreaCode`                                      | `workAreaCode`             | En create es obligatorio. En update puede venir parcial; si no llega, backend conserva valor actual.                                                   |
| Tipo de reposicion                         | `transactionCode`                                   | `transactionCode`          | Opcional; si no llega se aplica default legacy `"1"` (Caja Chica). Valores permitidos: `"1"`, `"2"`.                                                   |
| Observacion                                | `observation`                                       | `observation`              | En create es obligatoria. En update parcial, si no llega, backend conserva observacion actual.                                                         |
| Anticipo                                   | `advanceValue`                                      | `advanceValue`             | Opcional en create/update.                                                                                                                             |
| Responsable (numero de cedula en pantalla) | `responsiblePersonId` o `responsiblePersonDocument` | `responsiblePersonId`      | En create, API acepta id de persona o cedula (`responsiblePersonDocument`) y resuelve el id internamente. En update se mantiene `responsiblePersonId`. |
| Beneficiario                               | `beneficiaryPersonId`                               | `beneficiaryPersonId`      | Obligatorio cuando `transactionCode = "2"` (Gastos Personales). En otros casos es opcional.                                                            |
| Responsable de cheque                      | `checkResponsiblePersonId`                          | `checkResponsiblePersonId` | Opcional; si se envia, debe existir.                                                                                                                   |
| Fecha inicio                               | `startDate`                                         | `startDate`                | Opcional. Si se envia junto con `endDate`, debe cumplirse `startDate <= endDate`.                                                                      |
| Fecha fin                                  | `endDate`                                           | `endDate`                  | Opcional. Si se envia junto con `startDate`, debe cumplirse `startDate <= endDate`.                                                                    |
| Alerta de envio (bandera visual)           | `sendAlert`                                         | `sendAlert`                | Opcional (`"1"`/`"0"` en flujo actual).                                                                                                                |

Regla de pendiente por local en create (alineada con legacy):

- Para `transactionCode = "1"` (Caja Chica), si ya existe una reposicion pendiente en el local, el create se bloquea.
- Para `transactionCode = "2"`, el create mantiene compatibilidad con la rama legacy y no bloquea por pendiente.

Campos que NO debe enviar front en cabecera:

- `companyCode` (se toma del token).
- `replenishmentId` en create (en update va por path).
- `replenishmentStatus` (backend controla estado pendiente en create/update cabecera).
- Totales de cabecera (`requestedTotal`, `requestedSubtotal`, `vatTotal`, `approvedTotal`, `approvedVatTotal`), porque se recalculan desde detalle.

### 4.1.1.3 Mapeo puntual de formulario de detalle (crear/editar)

Esta subseccion aterriza el formulario de detalle de Administracion contra los endpoints `POST /replenishments/{replenishmentId}/details` y `POST /replenishments/{replenishmentId}/details/{detailId}`.

| Campo visual en front (legacy) | Campo API create (`POST`) | Campo API update (`POST`) | Regla backend vigente                                                                                          |
| ------------------------------ | ------------------------- | ------------------------- | -------------------------------------------------------------------------------------------------------------- |
| Tipo de documento              | `documentType`            | `documentType`            | Obligatorio en create y update.                                                                                |
| RUC factura                    | `taxId`                   | `taxId`                   | Opcional. Se normaliza con trim. Si se envia junto con `documentNumber`, participa en validacion de duplicado. |
| Numero de documento            | `documentNumber`          | `documentNumber`          | Opcional. Se normaliza con trim. Si se envia junto con `taxId`, participa en validacion de duplicado.          |
| Fecha del documento            | `documentDate`            | `documentDate`            | Obligatorio en create y update. Formato de intercambio actual: epoch numerico (milisegundos).                  |
| Concepto (secuencia)           | `billingConceptSequence`  | `billingConceptSequence`  | Obligatorio en create y update.                                                                                |
| Observacion de detalle         | `observation`             | `observation`             | Opcional. En update parcial, si no llega, backend conserva valor actual.                                       |
| Valor del detalle              | `requestedValue`          | `requestedValue`          | Obligatorio en create y update.                                                                                |
| Valor IVA                      | `vatValue`                | `vatValue`                | Obligatorio en create y update.                                                                                |
| Comprobante de venta           | `saleReceipt`             | `saleReceipt`             | Opcional.                                                                                                      |
| Cantidad                       | `quantity`                | `quantity`                | Opcional.                                                                                                      |
| Tipo unidad de medida          | `measurementTypeCode`     | `measurementTypeCode`     | Opcional.                                                                                                      |
| Unidad de medida               | `measurementValueCode`    | `measurementValueCode`    | Opcional.                                                                                                      |
| Nombre de archivo              | `fileName`                | `fileName`                | Opcional.                                                                                                      |
| Clave de acceso                | `accessKey`               | `accessKey`               | Opcional.                                                                                                      |

Reglas funcionales que aplica backend en detalle:

- Para create y update, la cabecera debe existir y estar en estado pendiente (`PENDING` o `PEN`).
- En update, el detalle debe existir para `replenishmentId + detailId + companyCode`.
- Validaciones de obligatorios: `billingConceptSequence`, `documentType`, `documentDate`, `requestedValue`, `vatValue`.
- Validacion de duplicado por `taxId + documentNumber` cuando ambos datos estan presentes.
- En update, si cambia el identificador documental (`taxId + documentNumber`) se vuelve a validar duplicidad.
- Luego de crear o editar detalle, backend recalcula totales de cabecera (`requestedTotal`, `vatTotal`, `requestedSubtotal`).

Campos que NO debe enviar front en detalle:

- `companyCode` (se toma del token).
- `replenishmentId` en body (se toma del path en create y update).
- `detailId` en body para update (se toma del path `/{detailId}`).
- Campos calculados/aprobados (`approvedValue`, `approvedVatValue`) en flujo de Administracion.

### 4.1.1.4 Mapeo puntual de cabecera de fondo (`GET /replenishments/current-fund`)

Esta subseccion aterriza la tarjeta legacy "Fondos area trabajo" contra la API de cabecera de fondo.

| Zona pantalla legacy            | Etiqueta visible front    | Campo API migrada      | Observacion de integracion                                                                                  |
| ------------------------------- | ------------------------- | ---------------------- | ----------------------------------------------------------------------------------------------------------- |
| Fondos area trabajo             | Fondo asignado            | `assignedFund`         | Valor monetario del fondo del area de trabajo.                                                              |
| Fondos area trabajo             | Reposiciones pendientes   | `pendingPaymentAmount` | Corresponde al acumulado pendiente de pago del local/departamento.                                          |
| Fondos area trabajo             | Saldo efectivo            | `cashBalance`          | Saldo operativo disponible para nuevas reposiciones.                                                        |
| Fondos area trabajo             | Documentos no enviados    | No expuesto (brecha)   | En legacy se muestra desde `totalDocNoEnviados` de la solicitud en pantalla, no desde la cabecera de fondo. |
| Fondos area trabajo             | Porcentaje de uso         | `usagePercentage`      | Se usa para indicadores visuales de consumo de fondo.                                                       |
| Fondos area trabajo             | Porcentaje de alerta      | `alertPercentage`      | Umbral de alerta para UI.                                                                                   |
| Fondos area trabajo             | Valor maximo documento    | `maxDocumentValue`     | Limite referencial para validaciones de detalle.                                                            |
| Cabecera visual                 | Codigo local/departamento | `workAreaCode`         | Llega como query en el endpoint y se refleja en la respuesta.                                               |
| Cabecera visual                 | Nombre local/departamento | `workAreaName`         | Nombre legible para mostrar junto al codigo del local.                                                      |
| Estado operativo de solicitudes | Estado actual             | `currentStatus`        | Estado calculado de presencia de pendiente para el area consultada.                                         |

Nota de brecha vigente:

- El campo legacy `totalDocNoEnviados` no forma parte del contrato actual de `current-fund`.
- En legacy este valor se inicializa en `0` para nuevo, se recalcula con el total de la solicitud en edicion pendiente y se ajusta durante el calculo de detalle.

### 4.1.1.5 Mapeo puntual de consulta de cabecera (`GET /replenishments/{replenishmentId}`)

Esta subseccion aterriza la consulta de cabecera de una solicitud para pantallas de visualizacion/edicion.

| Zona pantalla legacy | Etiqueta visible front  | Campo API migrada                                 | Observacion de integracion                                                                     |
| -------------------- | ----------------------- | ------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| Cabecera solicitud   | Numero de reposicion    | `replenishmentId`                                 | Identificador principal de la solicitud.                                                       |
| Cabecera solicitud   | Local (codigo + nombre) | `workAreaCode` + `workAreaName`                   | Usar ambos para construir la etiqueta visual del local.                                        |
| Cabecera solicitud   | Tipo de reposicion      | `transactionCode` + `replenishmentType`           | Conservar codigo tecnico y etiqueta legible para UI.                                           |
| Cabecera solicitud   | Estado                  | `replenishmentStatus` + `replenishmentStatusName` | Igual criterio que en listado (codigo + texto).                                                |
| Cabecera solicitud   | Registrado por (cedula) | `responsiblePersonDocument`                       | Campo directo para UI de cabecera.                                                             |
| Cabecera solicitud   | Registrado por (nombre) | `responsiblePersonName`                           | Campo directo para UI de cabecera.                                                             |
| Cabecera solicitud   | Observacion             | `observation`                                     | Texto editable en el formulario de cabecera.                                                   |
| Cabecera solicitud   | Subtotal / IVA / Total  | `requestedSubtotal`, `vatTotal`, `requestedTotal` | Totales de cabecera para visualizacion.                                                        |
| Cabecera solicitud   | Alerta de envio         | `sendAlert`                                       | Bandera visual (`1`/`0`) para resaltar comportamiento de envio.                                |
| Cabecera solicitud   | Documentos no enviados  | `unsentDocumentsTotal`                            | Regla legacy: si estado es `PEN/PENDING`, toma `requestedTotal`; en otros estados retorna `0`. |

Nota operativa:

- Este endpoint devuelve solo cabecera; el detalle enriquecido se consulta en `GET /replenishments/{replenishmentId}/details`.

### 4.1.1.6 Mapeo puntual de consulta de detalle (`GET /replenishments/{replenishmentId}/details`)

Esta subseccion aterriza la tabla de detalle de la solicitud para visualizacion en front.

| Zona pantalla legacy | Etiqueta visible front        | Campo API migrada                                                    | Observacion de integracion                                             |
| -------------------- | ----------------------------- | -------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| Tabla detalle        | Secuencial detalle            | `detailId`                                                           | Identificador de linea de detalle.                                     |
| Tabla detalle        | Tipo documento                | `documentType` + `documentTypeName`                                  | Mostrar `documentTypeName`; usar `documentType` como fallback tecnico. |
| Tabla detalle        | Fecha documento               | `documentDate`                                                       | Formato actual: epoch numerico (milisegundos).                         |
| Tabla detalle        | RUC factura                   | `taxId`                                                              | Puede venir nulo segun tipo de documento/regla de negocio.             |
| Tabla detalle        | Numero documento              | `documentNumber`                                                     | Puede venir nulo segun tipo de documento/regla de negocio.             |
| Tabla detalle        | Comprobante de venta          | `saleReceipt`                                                        | Campo opcional de visualizacion.                                       |
| Tabla detalle        | Concepto                      | `billingConceptSequence` + `billingConceptDescription`               | Mostrar descripcion; usar secuencia como referencia tecnica.           |
| Tabla detalle        | Cantidad                      | `quantity`                                                           | Campo opcional segun tipo de detalle.                                  |
| Tabla detalle        | Unidad de medida              | `measurementTypeCode`, `measurementValueCode`, `measurementUnitName` | Mostrar `measurementUnitName` y conservar codigos para edicion.        |
| Tabla detalle        | Nombre archivo                | `fileName`                                                           | Campo opcional para UI documental.                                     |
| Tabla detalle        | Observacion detalle           | `observation`                                                        | Campo editable/visible por linea.                                      |
| Tabla detalle        | Total (columna legacy)        | `requestedValue`                                                     | Valor economico solicitado por linea.                                  |
| Tabla detalle        | Iva(%) (columna legacy)       | `vatValue`                                                           | En backend representa valor IVA monetario de la linea.                 |
| Tabla detalle        | Valor aprobado / IVA aprobado | `approvedValue`, `approvedVatValue`                                  | Puede usarse en escenarios de revision/consulta avanzada.              |

### 4.1.1.6.1 Diagnostico de campos que pueden no aparecer en response

Regla general de serializacion vigente:

- La API omite propiedades nulas en JSON (`JsonInclude.NON_NULL` en VO base).

Checklist de los campos reportados como faltantes:

| Campo API                   | Mapeo backend actual                                                     | Cuando puede no aparecer en JSON                                                                                     |
| --------------------------- | ------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------- |
| `saleReceipt`               | Proyeccion directa desde detalle (`COMPROBANTEVENTA`).                   | Cuando el valor viene nulo en la fila de detalle o no fue informado al crear/editar.                                 |
| `billingConceptDescription` | Enriquecido por lookup con `companyCode + billingConceptSequence`.       | Cuando `billingConceptSequence` es nulo, no existe fila en catalogo o la tabla de catalogo no expone columnas texto. |
| `quantity`                  | Proyeccion directa desde detalle (`CANTIDAD`).                           | Cuando no se registra cantidad para esa linea de detalle.                                                            |
| `measurementTypeCode`       | Proyeccion directa desde detalle (`CODTIPUNIMED`).                       | Cuando no se informa unidad/tipo de medida en la linea.                                                              |
| `measurementValueCode`      | Proyeccion directa desde detalle (`CODVALUNIMED`).                       | Cuando no se informa unidad/valor de medida en la linea.                                                             |
| `measurementUnitName`       | Enriquecido por lookup con `measurementTypeCode + measurementValueCode`. | Cuando alguno de los codigos es nulo/vacio, no existe fila en catalogo o la tabla de catalogo no expone descripcion. |
| `fileName`                  | Proyeccion directa desde detalle (`NOMBREARCHIVO`).                      | Cuando no se adjunta nombre de archivo en la linea de detalle.                                                       |

Notas operativas para diagnostico rapido:

1. Si `measurementTypeCode` o `measurementValueCode` no llegan, `measurementUnitName` tampoco se puede resolver.
2. Si `billingConceptSequence` llega nulo, `billingConceptDescription` no se consulta.
3. Si el front no envia estos campos opcionales en `POST /details` o `POST /details/{detailId}`, pueden persistir nulos.

### 4.1.1.7 Mapeo puntual de validacion IVA (`POST /replenishments/validate-vat`)

Esta subseccion aterriza la validacion previa de IVA usada por el formulario de detalle.

| Zona pantalla legacy     | Accion front                 | Campo request API | Campo response API | Observacion de integracion                                                |
| ------------------------ | ---------------------------- | ----------------- | ------------------ | ------------------------------------------------------------------------- |
| Formulario de detalle    | Ingresar total del documento | `documentTotal`   | N/A                | Base para calcular el tope permitido de IVA.                              |
| Formulario de detalle    | Ingresar valor de IVA        | `vatValue`        | N/A                | Valor a validar antes de guardar/editar linea.                            |
| Mensajeria de validacion | Mostrar resultado            | N/A               | `valid`            | `true` permite continuar; `false` bloquea persistencia y muestra mensaje. |
| Mensajeria de validacion | Mostrar limite permitido     | N/A               | `maxAllowedVat`    | Referencia para correccion de IVA cuando excede calculo.                  |
| Mensajeria de validacion | Mostrar mensaje funcional    | N/A               | `message`          | Texto de validacion para feedback directo al usuario.                     |

### 4.1.1.8 Mapeo puntual de validacion de duplicado (`POST /replenishments/validate-duplicate`)

Esta subseccion aterriza la validacion de no duplicidad documental antes de guardar detalle.

| Zona pantalla legacy     | Accion front                 | Campo request API | Campo response API | Observacion de integracion                                     |
| ------------------------ | ---------------------------- | ----------------- | ------------------ | -------------------------------------------------------------- |
| Formulario de detalle    | Ingresar RUC                 | `taxId`           | N/A                | Se evalua en conjunto con `documentNumber`.                    |
| Formulario de detalle    | Ingresar numero de documento | `documentNumber`  | N/A                | Se evalua en conjunto con `taxId`.                             |
| Mensajeria de validacion | Mostrar bandera de duplicado | N/A               | `duplicate`        | `true` indica coincidencia con documento ya registrado.        |
| Mensajeria de validacion | Mostrar si el dato es valido | N/A               | `valid`            | `true` habilita continuar; `false` bloquea y exige correccion. |
| Mensajeria de validacion | Mostrar mensaje funcional    | N/A               | `message`          | Texto de negocio para UI cuando existe/no existe duplicado.    |

### 4.1.2 Contrato por API implementada (entrada/salida exacta)

| Tipo API          | Endpoint                                                    | Front debe enviar                                                                                                      | Backend recibe hoy                                                                                                                                                                                                                                      | Backend devuelve hoy                                                                                                                                                                                                                                                                                                                                                     | Brecha legacy relevante                                                                                                                           |
| ----------------- | ----------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| Consulta          | `GET /catalogs/{catalogTypeCode}/values`                   | `catalogTypeCode` por path; opcional `onlyActive`, `numericValue` por query                                           | `catalogTypeCode` obligatorio; `onlyActive` opcional (por defecto true); `numericValue` opcional                                                                                                                                                      | `List<CatalogValueVo>`: `catalogTypeCode`, `catalogValueCode`, `catalogValueName`, `shortName`, `state`, `numericValue`, `position`, `parentCatalogTypeCode`, `parentCatalogValueCode`                                                                                                                                                                                 | Sin brecha funcional critica; endpoint transversal para combos de UI                                                                              |
| Consulta          | `GET /replenishments/current-fund`                          | `workAreaCode` (query). En legacy el area viene por sesion.                                                            | `workAreaCode` obligatorio                                                                                                                                                                                                                              | `workAreaCode`, `workAreaName`, `assignedFund`, `cashBalance`, `pendingPaymentAmount`, `usagePercentage`, `alertPercentage`, `maxDocumentValue`, `currentStatus`                                                                                                                                                                                                         | Falta `totalDocNoEnviados` y banderas de precheck (`hasPendingRequest`, `hasIssuedCheck`, bloqueos)                                               |
| Consulta          | `POST /replenishments/findByFilter`                         | `filters[]` (`SearchModelDTO`) + `page` + `size`. Legacy usa estado + fecha desde/hasta.                               | `FilterVo` con `filters`, `page`, `size`                                                                                                                                                                                                                | `Page<ReplenishmentVo>`: `replenishmentId`, `workAreaCode`, `workAreaName`, `transactionCode`, `replenishmentType`, `replenishmentStatus`, `replenishmentStatusName`, `sendAlert`, `requestedTotal`, `unsentDocumentsTotal`, `requestedSubtotal`, `vatTotal`, `approvedVatTotal`, `responsiblePersonId`, `responsiblePersonDocument`, `replenishmentDate`, `createdDate` | No se fuerza area del usuario como en legacy                                                                                                      |
| Consulta          | `GET /replenishments/{replenishmentId}`                     | Path `replenishmentId`                                                                                                 | `replenishmentId` obligatorio                                                                                                                                                                                                                           | Solo cabecera `ReplenishmentVo` (incluye `responsiblePersonName`, `responsiblePersonDocument`, `workAreaName`, `replenishmentStatusName`, `sendAlert`, `unsentDocumentsTotal`)                                                                                                                                                                                           | Sin brecha critica en cabecera                                                                                                                    |
| Consulta          | `GET /replenishments/{replenishmentId}/details`             | Path `replenishmentId`                                                                                                 | `replenishmentId` obligatorio                                                                                                                                                                                                                           | Lista `ReplenishmentDetailVo` enriquecida (`saleReceipt`, `measurementTypeCode`, `measurementValueCode`, `measurementUnitName`, `fileName`, `billingConceptDescription`, `documentTypeName`)                                                                                                                                                                             | `documentTypeName` puede llegar `null` si no existe representacion en catalogo                                                                    |
| Mutacion cabecera | `POST /replenishments`                                      | En legacy se digita cedula (`personaResDTO.numeroDocumento`) + observacion + tipo.                                     | Requiere `workAreaCode`, `observation` y (`responsiblePersonId` o `responsiblePersonDocument`); `beneficiaryPersonId` es obligatorio cuando `transactionCode = "2"`; el resto es opcional.                                                              | `ReplenishmentVo` creado con estado pendiente y enriquecimiento (`workAreaName`, `responsiblePersonDocument`)                                                                                                                                                                                                                                                            | Sin brecha critica en identificacion de responsable para create                                                                                   |
| Mutacion cabecera | `POST /replenishments/{replenishmentId}`                    | Cabecera editable                                                                                                      | `replenishmentId` en path + body parcial permitido; backend conserva valores actuales no enviados y vuelve a validar reglas de cabecera.                                                                                                                | `ReplenishmentVo` actualizado con estado pendiente                                                                                                                                                                                                                                                                                                                       | Sin brecha critica para update de cabecera en este punto                                                                                          |
| Validacion        | `POST /replenishments/validate-vat`                         | `documentTotal`, `vatValue`                                                                                            | Ambos obligatorios                                                                                                                                                                                                                                      | `documentTotal`, `vatValue`, `maxAllowedVat`, `valid`, `message`                                                                                                                                                                                                                                                                                                         | Sin gap funcional critico para regla legacy de IVA                                                                                                |
| Validacion        | `POST /replenishments/validate-duplicate`                   | `taxId`, `documentNumber`                                                                                              | Ambos obligatorios                                                                                                                                                                                                                                      | `taxId`, `documentNumber`, `duplicate`, `valid`, `message`                                                                                                                                                                                                                                                                                                               | Legacy cruza mas contexto de negocio (cancelada/pagada) en algunos flujos                                                                         |
| Mutacion detalle  | `POST /replenishments/{replenishmentId}/details`            | Legacy arma fila con documento, ruc, no documento, fecha, concepto, cantidad, unidad, archivo, observacion, iva, total | Requiere: `billingConceptSequence`, `documentType`, `documentDate`, `requestedValue`, `vatValue`; opcional: `taxId`, `documentNumber`, `observation`, `accessKey`, `quantity`, `saleReceipt`, `measurementTypeCode`, `measurementValueCode`, `fileName` | `ReplenishmentDetailVo` con `detailId`, `replenishmentId` y campos del request                                                                                                                                                                                                                                                                                           | La descripcion legible (`measurementUnitName`, `billingConceptDescription`, `documentTypeName`) se confirma en `GET /replenishments/{id}/details` |
| Mutacion detalle  | `POST /replenishments/{replenishmentId}/details/{detailId}` | Mismos campos de detalle editable                                                                                      | `replenishmentId`, `detailId` path + body `ReplenishmentDetailVo` (mismas reglas de requeridos + opcionales de detalle)                                                                                                                                 | `ReplenishmentDetailVo` actualizado                                                                                                                                                                                                                                                                                                                                      | Sin gap funcional critico para edicion de campos base                                                                                             |

### 4.1.3 Campos obligatorios actuales por endpoint (sin ambiguedad)

| Endpoint                                                    | Campos obligatorios                                                                                                                        |
| ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `GET /replenishments/current-fund`                          | `workAreaCode` (query)                                                                                                                     |
| `POST /replenishments/findByFilter`                         | Ninguno en body (si no llegan filtros, lista por compania)                                                                                 |
| `GET /replenishments/{replenishmentId}`                     | `replenishmentId` (path)                                                                                                                   |
| `GET /replenishments/{replenishmentId}/details`             | `replenishmentId` (path)                                                                                                                   |
| `POST /replenishments`                                      | `workAreaCode`, `observation`, (`responsiblePersonId` o `responsiblePersonDocument`); `beneficiaryPersonId` cuando `transactionCode = "2"` |
| `POST /replenishments/{replenishmentId}`                    | `replenishmentId` (path). El body puede ser parcial; se revalidan reglas de cabecera al consolidar con valores actuales.                   |
| `POST /replenishments/validate-vat`                         | `documentTotal`, `vatValue`                                                                                                                |
| `POST /replenishments/validate-duplicate`                   | `taxId`, `documentNumber`                                                                                                                  |
| `POST /replenishments/{replenishmentId}/details`            | `replenishmentId` (path), `billingConceptSequence`, `documentType`, `documentDate`, `requestedValue`, `vatValue`                           |
| `POST /replenishments/{replenishmentId}/details/{detailId}` | `replenishmentId` (path), `detailId` (path), `billingConceptSequence`, `documentType`, `documentDate`, `requestedValue`, `vatValue`        |

### 4.1.4 Acuerdo de integracion para evitar doble trabajo

1. Para listado, el frontend debe usar `replenishmentStatusName` como etiqueta visual, conservar `replenishmentStatus` como codigo tecnico y usar `sendAlert` para resalte cuando venga `1`.
2. Para create de cabecera, frontend puede enviar `responsiblePersonId` o `responsiblePersonDocument` (cedula); el backend resuelve el id cuando llega por documento. En update se mantiene `responsiblePersonId`.
3. Para detalle, frontend debe usar `saleReceipt`, `measurementTypeCode`, `measurementValueCode` y `fileName` en altas/ediciones; para visualizacion legible usar `measurementUnitName` y `billingConceptDescription` desde `GET /replenishments/{id}/details`.
4. `GET /replenishments/{id}` devuelve solo cabecera con `responsiblePersonName`; el detalle enriquecido se consulta en `GET /replenishments/{id}/details`.

## 5. Matriz BACK-FRONT (Revision)

Base URL: `/gtfReplacementsServices/api/v1`

| Modulo front             | Menu/Opcion             | Pantalla/Componente           | Boton/Accion            | Endpoint backend                                            | Estado backend | Uso esperado                           |
| ------------------------ | ----------------------- | ----------------------------- | ----------------------- | ----------------------------------------------------------- | -------------- | -------------------------------------- |
| Revision de Reposiciones | Reposiciones > Revision | Lista de solicitudes enviadas | Buscar                  | POST /replenishment-reviews/findByFilter                    | Pendiente P0   | Listar solicitudes para contabilidad   |
| Revision de Reposiciones | Reposiciones > Revision | Pantalla de revision          | Abrir solicitud         | GET /replenishment-reviews/{id}                             | Pendiente P0   | Ver solicitado vs aprobado             |
| Revision de Reposiciones | Reposiciones > Revision | Tabla de revision detalle     | Aprobar/ajustar detalle | POST /replenishment-reviews/{id}/details/{detailId}/approve | Pendiente P0   | Ajustar valor aprobado con observacion |
| Revision de Reposiciones | Reposiciones > Revision | Acciones de revision          | Validar solicitud       | POST /replenishment-reviews/{id}/validate                   | Pendiente P0   | Validar contablemente y pasar estado   |
| Revision de Reposiciones | Reposiciones > Revision | Acciones de revision          | Anular solicitud        | POST /replenishment-reviews/{id}/cancel                     | Pendiente P0   | Anulacion contable con observacion     |

## 6. Criterio para arrancar P0

Para iniciar P0 sin romper lo ya estable:

1. Mantener endpoints implementados y agregar capacidades faltantes por capas.
2. Implementar primero CRUD de detalle y validaciones de detalle (IVA/duplicado).
3. Luego implementar flujo de envio en Administracion.
4. Finalmente habilitar modulo base de Revision (listado, consulta, aprobar, validar, anular).
