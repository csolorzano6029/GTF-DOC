# Gap Analysis Backend Reposiciones (Legacy vs Migrado)

Fecha de corte: 2026-07-10

## 1. Objetivo

Comparar, endpoint por endpoint, el comportamiento esperado en la nueva API con el flujo real legacy para los modulos:

- Administracion de Reposiciones
- Revision de Reposiciones

El foco de esta auditoria es identificar:

- nivel de alineacion real
- validaciones o reglas de negocio faltantes
- ajustes tecnicos concretos para lograr paridad funcional con legacy

## 2. Fuentes auditadas

Legacy (fuente funcional):

- SOPORTES/gtf-root/gtf-web-intranet/src/main/java/ec/com/smx/gtf/web/administracion/controller/AdminRembolsoController.java
- SOPORTES/gtf-root/gtf-web-intranet/src/main/java/ec/com/smx/gtf/web/administracion/controller/RevisionReposicionController.java
- SOPORTES/gtf-root/gtf-web-intranet/src/main/java/ec/com/smx/gtf/web/utils/reposiciones/AdminReposicionesUtils.java
- SOPORTES/gtf-root/gtf-web-intranet/src/main/java/ec/com/smx/gtf/web/utils/retenciones/AdminRetencionUtils.java
- SOPORTES/gtf-root/gtf-modelo/src/main/java/ec/com/smx/fin/gtf/cliente/gestor/impl/ReposicionGestor.java
- SOPORTES/gtf-root/gtf-modelo/src/main/java/ec/com/smx/fin/gtf/cliente/servicio/impl/ReposicionServicio.java

Migrado (estado actual):

- gtf-replacements-root/gtf-replacements-services/src/main/java/ec/com/smx/gtf/replacements/controller/replenishment/ReplenishmentController.java
- gtf-replacements-root/gtf-replacements-core/src/main/java/ec/com/smx/gtf/replacements/replenishment/service/ReplenishmentService.java
- gtf-replacements-root/gtf-replacements-core/src/main/java/ec/com/smx/gtf/replacements/replenishment/repository/ReplenishmentRepository.java
- gtf-replacements-root/gtf-replacements-client/src/main/java/ec/com/smx/gtf/replacements/replenishment/entity/ReplenishmentEntity.java

## 3. Resumen ejecutivo

- Endpoints objetivo evaluados (JIRA): 23
- Alineados al 100% con legacy: 0
- Parciales: 4
- No implementados: 19

Estado general:

- El backend migrado cubre base de cabecera (current-fund, findByFilter, create, update, getById), pero aun no replica reglas clave de negocio legacy en detalle.
- Todo el modulo de Revision de Reposiciones esta pendiente de implementacion.

## 4. Matriz de brecha - Modulo Administracion

| Endpoint objetivo                                   | Flujo legacy de referencia                                                                                                           | Estado migrado  | Alineacion  | Brecha detectada                                                                                                                                                                 | Ajuste tecnico requerido                                                                                                                                                  |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ | --------------- | ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| GET /replenishments/current-fund                    | AdminRembolsoController.inicializaFondosAreaNuevo + validacionesPopUpInicial + validarFondoAsignado                                  | Implementado    | Parcial     | Legacy no solo consulta fondo: valida cheque emitido, solicitudes pendientes y reglas de bloqueo/alerta de porcentaje antes de habilitar flujo                                   | Incluir endpoint de precheck operacional (o ampliar current-fund) con flags: hasPendingRequest, hasIssuedCheck, allowCreate, alertThresholdReached, blockThresholdReached |
| POST /replenishments/findByFilter                   | AdminRembolsoController.buscarRembolsoListener                                                                                       | Implementado    | Parcial     | Legacy fuerza codigoAreaTrabajo del usuario y estado activo; ademas combina rango de fechas con estado                                                                           | Aplicar filtro por workArea del usuario autenticado por defecto y soportar rango fecha desde/hasta con semantica legacy                                                   |
| GET /replenishments/{id}                            | AdminRembolsoController.visualizarRembolso + AdminReposicionesUtils.obtenerDetallesReposicion                                        | Implementado    | Parcial     | Migrado retorna cabecera sin detalle completo ni contexto de fondos/responsables; legacy carga detalle y estructura para operacion                                               | Extender respuesta para incluir details y metadatos operativos minimos (totales, estado por linea, contexto de edicion)                                                   |
| POST /replenishments                                | AdminRetencionUtils.guardarRembolsoGeneral + AdminReposicionesUtils.validacionRestriccionesReposicionLocal                           | Implementado    | Parcial     | Legacy valida detalle, RUC, duplicados, IVA, monto maximo, saldo disponible, porcentaje de uso/alerta y reglas por transaccion; migrado valida solo pending/workArea/responsable | Incorporar pipeline de validaciones de cabecera+detalle antes de guardar; rechazar create incompleto segun reglas legacy                                                  |
| POST /replenishments/{id}                           | AdminRembolsoController.editarRembolso + AdminReposicionesUtils.editarReposicion + ReposicionGestor.guardarOActualizarReposicion     | Implementado    | Parcial     | Legacy contempla recalculo de totales y persistencia de cambios en detalle; migrado solo actualiza cabecera                                                                      | Agregar actualizacion transaccional de detalle y recalculo integral (requestedSubtotal, vatTotal, requestedTotal)                                                         |
| POST /replenishments/{id}/details                   | AdminRembolsoController.agregarFila / agregarFilaDetalle                                                                             | No implementado | No alineado | CRUD de detalle no expuesto en API                                                                                                                                               | Implementar creacion de linea con validaciones por tipo documento/transaccion                                                                                             |
| POST /replenishments/{id}/details/{detailId}        | AdminRembolsoController.agregarFila (edicion) + editarRetencion                                                                      | No implementado | No alineado | Ajuste de linea y retenciones no existe en migrado                                                                                                                               | Implementar edicion de linea con control de campos sensibles y revalidacion                                                                                               |
| POST /replenishments/{id}/details/{detailId}/delete | AdminRembolsoController.eliminarFilaDetalle                                                                                          | No implementado | No alineado | No hay eliminacion de detalle ni recalculo posterior                                                                                                                             | Implementar baja logica/fisica segun estado y recalculo de totales                                                                                                        |
| GET /replenishments/document-types                  | AdminReposicionesUtils.obtenerCollectionItemTipoDocumento                                                                            | No implementado | No alineado | Catalogo requerido por flujo de ingreso no existe                                                                                                                                | Exponer catalogo de tipos de documento con metadatos hasVat                                                                                                               |
| GET /replenishments/billing-concepts                | AdminReposicionesUtils.construyeEstructuraConceptos                                                                                  | No implementado | No alineado | Sin catalogo de conceptos por area/trx/tipo documento                                                                                                                            | Exponer endpoint con filtro por area/trx/tipoDocumento                                                                                                                    |
| POST /replenishments/validate-vat                   | AdminReposicionesUtils.validarExistenciaIva                                                                                          | No implementado | No alineado | Falta regla exacta: IVA > 0, IVA < total, IVA <= total - (total\*100/(100+15))                                                                                                   | Implementar servicio de validacion VAT con respuesta calculada y mensaje legacy                                                                                           |
| POST /replenishments/validate-duplicate             | AdminReposicionesUtils.validacionRepetidosDetalleRucNoDocumento + AdminRetencionUtils.validarRetencionRevision + validarPagoRevision | No implementado | No alineado | Sin validacion de duplicidad en solicitud activa/cancelada/pagada                                                                                                                | Implementar validacion transversal por (compania, ruc, documento[, comprobante]) y devolver contexto de conflicto                                                         |
| GET /replenishments/responsibles                    | AdminRembolsoController.cargarFuncionariosResponsables + AdminFuncionariosUtils.obtenerFuncionariosResponsablesPagoCheque            | No implementado | No alineado | Sin obtencion de responsables para envio                                                                                                                                         | Implementar endpoint y diferenciar principal/secundario                                                                                                                   |
| POST /replenishments/{id}/send                      | AdminRembolsoController.onclickEnviarRembolso + enviarReposicion + ReposicionGestor.enviarSolicitudReposicion                        | No implementado | No alineado | Falta cambio de estado a ENVIADA, validacion de funcionario para cheque por ventanilla y movimientos financieros                                                                 | Implementar envio con validaciones de responsable y actualizacion transaccional de estado/movimientos                                                                     |
| POST /replenishments/{id}/cancel                    | En legacy operativo la anulacion formal se ejecuta en RevisionReposicionController.anularReposicion (contabilidad), no desde Admin   | No implementado | No alineado | Hay diferencia de diseno funcional respecto a JIRA                                                                                                                               | Definir politica final: anular en Admin o solo en Revision; si se mantiene en Admin, replicar reglas de observacion y movimiento ajuste                                   |
| GET /replenishments/{id}/print                      | AdminRembolsoController.imprimirReposicion + ReposicionGestor.procesarImpresionReposicion                                            | No implementado | No alineado | No existe impresion PDF en migrado                                                                                                                                               | Implementar generacion de reporte (HTML/PDF) con detalle local/exterior y datos de cabecera                                                                               |
| GET/POST /withholdings/\*                           | AdminRembolsoController.validarIngresoRetencion/validarClaveAcceso + AdminRetencionUtils.\*                                          | No implementado | No alineado | Todo flujo de retencion manual/electronica/clave acceso/XML esta ausente                                                                                                         | Implementar modulo withholding aislado con endpoints de validacion SRI/repositorio y reglas de conceptos                                                                  |

## 5. Matriz de brecha - Modulo Revision

| Endpoint objetivo                                           | Flujo legacy de referencia                                                                                       | Estado migrado  | Alineacion  | Brecha detectada                                                                             | Ajuste tecnico requerido                                                                                                   |
| ----------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- | --------------- | ----------- | -------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| POST /replenishment-reviews/findByFilter                    | RevisionReposicionController.buscarReposicionesListener                                                          | No implementado | No alineado | Sin listado para contabilidad (filtro por local, nombre, estado, fechas)                     | Crear controller/service/repository de revision con filtro paginado y estado ENVIADA por defecto                           |
| GET /replenishment-reviews/{id}                             | RevisionReposicionController.visualizarReposicion + AdminReposicionesUtils.inicializaValorAprobado               | No implementado | No alineado | Sin vista de comparacion solicitado vs aprobado                                              | Implementar DTO de revision con requested/approved en cabecera y detalle                                                   |
| POST /replenishment-reviews/{id}/details/{detailId}/approve | RevisionReposicionController.valorTotalFila + rechazarFilaDetalle + activarFilaDetalle + onclickAjusteReposicion | No implementado | No alineado | No existe ajuste de linea con observacion obligatoria cuando hay diferencia                  | Implementar ajuste por detalle con validaciones: observacion requerida, no negativos, tope por documento y recalculo total |
| POST /replenishment-reviews/{id}/validate                   | RevisionReposicionController.onclickValidarReposicion + ReposicionGestor.validarSolicitudReposicion              | No implementado | No alineado | Falta validacion final (principal/beneficiario), generacion de liquidacion y estado VALIDADA | Implementar validacion con precondiciones y trazabilidad de errores funcionales                                            |
| POST /replenishment-reviews/{id}/cancel                     | RevisionReposicionController.anularReposicion + ReposicionGestor.anularSolicitudReposicion                       | No implementado | No alineado | Sin anulacion contable con observacion y ajuste de movimientos                               | Implementar anulacion transaccional con observacion obligatoria                                                            |
| Job VALIDATED -> PAID/ISSUED                                | ReposicionGestor.procesarVistaPagoReposiciones + registrarChequeEmitido + reposicionFondoDisponible              | No implementado | No alineado | No hay proceso automatizado de transicion de estados ni conciliacion con JDE                 | Implementar scheduler idempotente con bitacora y control de reintentos                                                     |

## 6. Reglas legacy criticas aun no replicadas

1. Validacion de IVA:

- valorIva >= 0
- valorIva < totalDocumento
- valorIva <= total - (total\*100/(100+15))

2. Validacion de documento duplicado:

- no repetir RUC + numeroDocumento en la misma solicitud
- rechazar documento ya asociado a reposicion cancelada/pagada segun contexto

3. Validacion por monto maximo de documento:

- aplicable por defecto
- excepciones por conceptos parametrizados (no aplica validacion)

4. Validacion por disponibilidad operativa del fondo:

- bloqueo por porcentaje de uso
- alerta por porcentaje de alerta
- control de saldo efectivo

5. Validacion de envio:

- para caja chica con pago por ventanilla exige responsable encontrado
- cambia estado a ENVIADA y registra movimiento

6. Ajuste en revision:

- observacion obligatoria cuando se modifica valor aprobado
- recalculo de totales y control de no sobregiro
- notificacion por correo al local/administrador

7. Validacion final (contabilidad):

- requiere principal o beneficiario segun tipo de transaccion
- genera liquidacion de facturacion
- cambia a VALIDADA solo si validacion de facturacion es exitosa

## 7. Ajustes recomendados por prioridad

### Prioridad P0 (paridad minima para evitar regresion)

1. Implementar CRUD de detalle (H5 completo) con recalculo de totales.
2. Implementar validate-vat y validate-duplicate con logica legacy.
3. Implementar send de reposicion con validaciones de responsable y cambio de estado.
4. Implementar modulo de revision base: findByFilter, getById, approve, validate, cancel.

### Prioridad P1 (consistencia operativa)

1. Implementar catalogos document-types y billing-concepts.
2. Implementar responsibles.
3. Exponer print PDF.
4. Incorporar notificacion por correo en ajuste de revision.

### Prioridad P2 (cierre funcional completo)

1. Implementar flujo de retenciones (manual/electronica, XML, clave acceso).
2. Implementar scheduler VALIDATED -> PAID/ISSUED con conciliacion JDE.
3. Ajustar endpoint current-fund/precheck para incluir banderas operativas legacy.

## 8. Conclusiones

- El migrado actual resuelve una base de cabecera util, pero no alcanza aun equivalencia funcional con legacy para Administracion y Revision.
- La brecha mayor esta en detalle documental, validaciones transaccionales y todo el circuito de revision contable.
- Para lograr comportamiento 1:1 con legacy y evitar regresiones de negocio, el siguiente paso debe enfocarse en P0, luego P1 y finalmente P2.
