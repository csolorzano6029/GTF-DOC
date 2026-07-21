# JIRA - Backend Reposiciones - Plan de Implementacion para Revision

Fecha de corte: 2026-07-21

## 1) Objetivo

Preparar un backlog ejecutable para migrar Revisión de Reposiciones (legacy -> Spring Boot),
evitando trabajo duplicado con lo ya implementado en Administracion.

Este documento deja lista la estructura para Jira con:

1. Epica
2. Historias
3. Tareas
4. Criterios de aceptacion
5. Ejemplos de endpoints (request/response)
6. Matriz de reutilizacion de APIs existentes

## 2) Alcance y decisiones

### 2.1 Alcance funcional

1. Listar reposiciones enviadas para revision.
2. Visualizar cabecera y detalle de la reposicion.
3. Ajustar valores aprobados por detalle.
4. Validar reposicion.
5. Anular reposicion desde contabilidad.
6. Preparar cierre post-validacion (VALIDATED -> PAID/ISSUED).

### 2.2 Decision de alcance vigente

La linea actual del proyecto mantiene foco en flujo Caja Chica (transactionCode=1).
Si negocio decide reabrir Gastos Personales, se crea historia separada para no mezclar
reglas en este paquete.

Resultado de segunda revision funcional legacy:

1. El legacy de Revision si contiene flujos de Gastos Personales (local y exterior).
2. Si se mantiene el alcance solo Caja Chica, esta exclusion debe quedar aprobada en Jira
   de forma explicita para evitar brechas funcionales no trazadas.

Precision funcional cerrada (legacy + migrado):

1. Para capacidades sin diferencia funcional, Revision consume endpoints shared existentes (`/api/v1/replenishments/...`).
2. Solo se crea endpoint en `replenishment-reviews` cuando exista brecha funcional real (reglas distintas, enrichment de respuesta, permisos o contrato Front).
3. El estado revisable operativo para acciones de Revision (editar/validar/anular) es ENVIADA (ENV).

### 2.3 Reglas tecnicas obligatorias

1. QueryDSL + JPA/Hibernate.
2. No introducir SQL manual salvo excepcion aprobada.
3. Contrato de errores: mensaje generico + detalle en errors[] cuando aplique.
4. Cada endpoint nuevo debe registrarse en Bruno y en docs/apis.

## 3) Evidencia legacy usada

### 3.1 Controller de referencia

RevisionReposicionController (legacy) concentra los flujos clave:

1. buscarReposicionesListener
2. visualizarReposicion
3. actualizarReposicion
4. onclickAjusteReposicion
5. onclickValidarReposicion
6. anularReposicion
7. rechazarFilaDetalle / activarFilaDetalle
8. validarRetencion / validarClaveAccesoDocumento / verificaRuc

### 3.2 Transacciones legacy asociadas

1. transAjusteSolicitudReposicion
2. transValidarSolicitudReposicion
3. transAnularSolicitudReposicion

Estas transacciones guian los criterios de aceptacion de las historias E2.

## 4) Matriz de reutilizacion (evitar doble trabajo)

### 4.1 Reutilizacion interna de reglas

| Capacidad interna reutilizable | Uso en Revision                                                | Accion                                           |
| ------------------------------ | -------------------------------------------------------------- | ------------------------------------------------ |
| Validacion de IVA              | Validar ajuste de IVA en detalle aprobado (H3)                 | Reutilizar internamente en servicios             |
| Validacion de duplicidad       | Verificar duplicidad documental en ajustes/retenciones (H3/H8) | Reutilizar internamente en servicios             |
| Resolucion de responsables     | Responsable operativo para flujo de validacion                 | Reutilizar internamente donde aplique            |
| Lectura de cabecera y detalle  | Apertura de solicitud en revision (H2)                         | Reutilizar internamente sin exponer rutas shared |
| Prevalidacion de detalle       | Prevalidar cambios de detalle en modo revision                 | Reutilizar internamente sin duplicar reglas      |

### 4.2 Matriz de consumo para Revision (reuso directo vs endpoint nuevo)

| Capacidad interna (reuso)             | Endpoint actual disponible                                         | Uso en Revision                                         | Decision actual                                              |
| ------------------------------------- | ------------------------------------------------------------------ | ------------------------------------------------------- | ------------------------------------------------------------ |
| Filtros dinamicos de busqueda         | POST /api/v1/replenishments/findByFilter                           | Buscar reposiciones para revision (H1)                  | Reuso directo (sin endpoint nuevo mientras no exista brecha) |
| Lectura de cabecera                   | GET /api/v1/replenishments/{replenishmentId}                       | Visualizar cabecera de reposicion en revision (H2)      | Reuso directo (sin endpoint nuevo mientras no exista brecha) |
| Lectura de detalle                    | GET /api/v1/replenishments/{replenishmentId}/details               | Visualizar detalle de reposicion en revision (H2)       | Reuso directo (sin endpoint nuevo mientras no exista brecha) |
| Consulta de conceptos                 | GET /api/v1/replenishments/billing-concepts                        | Popup de conceptos en revision (H7)                     | Reuso directo (sin endpoint nuevo mientras no exista brecha) |
| Prevalidacion de detalle con cabecera | POST /api/v1/replenishments/{replenishmentId}/details/validate-add | Prevalidar cambios de detalle en modo revision          | Reuso directo (sin endpoint nuevo mientras no exista brecha) |
| Prevalidacion sin cabecera persistida | POST /api/v1/replenishments/details/validate-add                   | Prevalidar detalle cuando no exista cabecera persistida | Reuso directo (sin endpoint nuevo mientras no exista brecha) |

Regla de consumo para H1/H2 (ajustada con evidencia):

1. H1 y H2 consumen por defecto endpoints shared (`/api/v1/replenishments/...`) porque la logica actual ya cubre ambos escenarios.
2. H1 usa `POST /api/v1/replenishments/findByFilter`.
3. H2 usa `GET /api/v1/replenishments/{replenishmentId}` y `GET /api/v1/replenishments/{replenishmentId}/details`.
4. Si aparece brecha funcional real (filtros extra, enrichment, permisos o contrato distinto), se crea endpoint en `/api/v1/replenishment-reviews/...` como subtarea tecnica.
5. La regla funcional se mantiene: acciones de Revision (editar/validar/anular) solo cuando estado = ENVIADA (ENV).
6. Consideracion tecnica: `GET /details` retorna lineas activas de detalle (estado de detalle activo), no aplica filtro por estado de cabecera.
7. No duplicar query ni paginacion si se habilita wrapper de Revision.

### 4.3 No reutilizar (crear en modulo de revision)

| Activo                                                                            | Motivo                                                                             |
| --------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| POST /api/v1/replenishment-management/{replenishmentId}/send                      | Es flujo administracion -> contabilidad, no flujo contabilidad -> validacion final |
| POST /api/v1/replenishment-management/{replenishmentId}/details/{detailId}/delete | Revision requiere rechazo/activacion de linea, no borrado operativo                |

### 4.4 Trazabilidad de historias E2 vs tipo de API

| Historia | Endpoint(s) objetivo                                                                                                                                                                                  | Tipo de API                           | Referencia |
| -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------- | ---------- |
| E2-H1    | POST /api/v1/replenishments/findByFilter                                                                                                                                                              | Reuso directo de API shared           | 4.2        |
| E2-H2    | GET /api/v1/replenishments/{replenishmentId}; GET /api/v1/replenishments/{replenishmentId}/details                                                                                                    | Reuso directo de API shared           | 4.2        |
| E2-H3    | POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/adjust                                                                                                                        | Nueva API + validaciones reutilizadas | 4.1        |
| E2-H4    | POST /api/v1/replenishment-reviews/{replenishmentId}/validate                                                                                                                                         | Nueva API                             | 4.3        |
| E2-H5    | POST /api/v1/replenishment-reviews/{replenishmentId}/cancel                                                                                                                                           | Nueva API                             | 4.3        |
| E2-H6    | Scheduler interno (sin endpoint publico)                                                                                                                                                              | Nuevo componente                      | 4.3        |
| E2-H7    | GET /api/v1/replenishments/billing-concepts; POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/adjust-concept                                                                   | Mixta: shared (GET) + nueva (POST)    | 4.2 + 4.3  |
| E2-H8    | POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/withholding/validate; POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/withholding/validate-access-key | Nueva API                             | 4.3        |
| E2-H9    | POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/support-file                                                                                                                  | Nueva API                             | 4.3        |
| E2-H10   | Decision funcional en Jira (sin endpoint directo)                                                                                                                                                     | Sin API directa                       | N/A        |

## 5) Epica y backlog Jira listo para asignar

## E2 - Revision de Reposiciones

Descripcion: Implementar el modulo backend de revision contable de reposiciones enviadas,
con paridad funcional legacy y reutilizacion de APIs ya construidas en administracion.

Regla de backlog (ajustada):

1. Este backlog de implementacion incluye solo desarrollo backend nuevo.
2. H1 (filtro) y H2 (lectura por id) se ejecutan por reuso directo de endpoints shared actuales, sin desarrollo backend nuevo por defecto.
3. Si en H1/H2 aparece brecha funcional real, se crea subtarea tecnica para endpoint de `replenishment-reviews`.
4. Las APIs de H3 a H9 siguen como foco de desarrollo nuevo en Revision.

### E2-H1 - Consulta de reposiciones para revision (filtros y paginacion)

Endpoint objetivo:

1. POST /api/v1/replenishments/findByFilter

Tipo de API en este plan:

1. Reuso directo de API shared existente.
2. Endpoint nuevo en `replenishment-reviews` solo si se detecta brecha funcional comprobada.

Tareas de desarrollo backend:

1. Validar que `POST /api/v1/replenishments/findByFilter` cubre filtros de Revision (area, estado y fechas).
2. Confirmar normalizacion de alias/comparadores (`replenishmentStatus`, `replenishmentDate`) sin cambios de query.
3. Verificar forzado de contexto de compania/area segun sesion.
4. Si se detecta brecha funcional, abrir subtarea para endpoint `POST /api/v1/replenishment-reviews/findByFilter` delegando a la misma logica.
5. Pruebas de contrato y regresion de consumo Front Revision.

Criterios de consumo/QA:

1. Debe filtrar por codigo area, nombre area, estado y rango de fechas.
2. Debe soportar filtros dinamicos con alias legacy (`replenishmentStatus`, `replenishmentDate`) y manejo de estado "Todos".
3. Estado revisable operativo para acciones (editar/validar/anular): solo ENVIADA (ENV).
4. Debe mantener envelope BaseResponseVo.

### E2-H2 - Apertura de reposicion para revision (cabecera y detalle)

Endpoint objetivo:

1. GET /api/v1/replenishments/{replenishmentId}
2. GET /api/v1/replenishments/{replenishmentId}/details

Tipo de API en este plan:

1. Reuso directo de APIs shared de lectura.
2. Endpoint nuevo en `replenishment-reviews` solo si se detecta brecha funcional comprobada.

Tareas de desarrollo backend:

1. Validar que la lectura shared de cabecera/detalle cubre datos requeridos para Revision.
2. Confirmar que los flags operativos para acciones de Revision se pueden derivar sin cambiar contrato.
3. Si aparece brecha funcional (campos faltantes o reglas de permisos), abrir subtarea para endpoints de `replenishment-reviews` sin duplicar logica.
4. Pruebas de contrato y escenarios funcionales.

Criterios de consumo/QA:

1. Debe exponer montos solicitados y montos aprobables por linea.
2. Debe indicar si la reposicion se puede validar o anular (regla de acciones por estado ENVIADA).
3. El endpoint de detalle debe retornar lineas activas de detalle.
4. La lectura no debe restringirse por estado especifico de cabecera; la restriccion aplica a acciones de Revision.
5. No debe romper contratos vigentes de administracion.

### E2-H3 - Ajuste contable por detalle con recalculo de totales

Endpoint objetivo:

1. POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/adjust

Alcance:

1. Endpoint exclusivo de Revision.
2. No aplica como endpoint de Administracion.

Tareas:

| ID       | Tarea                                                                 | Resultado esperado       |
| -------- | --------------------------------------------------------------------- | ------------------------ |
| E2-H3-T1 | Definir ReviewDetailAdjustVo (entrada/salida)                         | Contrato claro de ajuste |
| E2-H3-T2 | Implementar validaciones (IVA, duplicado, observacion cuando aplique) | Ajuste validado          |
| E2-H3-T3 | Persistir valor aprobado y recalcular totales aprobados de cabecera   | Totales consistentes     |
| E2-H3-T4 | Implementar rechazo/activacion de linea sin borrar detalle            | Paridad legacy           |
| E2-H3-T5 | Tests unitarios e integracion                                         | Cobertura del flujo      |

Criterios de aceptacion:

1. Si cambia valor aprobado, debe guardar trazabilidad de observacion.
2. Debe validar IVA con la misma regla de administracion.
3. Debe mantener detalle historico (sin hard delete).

### E2-H4 - Validacion contable de reposicion revisada

Endpoint objetivo:

1. POST /api/v1/replenishment-reviews/{replenishmentId}/validate

Alcance:

1. Endpoint exclusivo de Revision.
2. No aplica como endpoint de Administracion.

Tareas:

| ID       | Tarea                                                                              | Resultado esperado             |
| -------- | ---------------------------------------------------------------------------------- | ------------------------------ |
| E2-H4-T1 | Implementar ReviewService.validateReview                                           | Flujo de validacion central    |
| E2-H4-T2 | Integrar reglas de precondicion de detalle (lineas activas, montos, observaciones) | Bloqueo de errores funcionales |
| E2-H4-T3 | Cambiar estado a VALIDATED y registrar auditoria                                   | Estado contable actualizado    |
| E2-H4-T4 | Tests de escenarios OK/error                                                       | Contrato estable               |

Criterios de aceptacion:

1. Solo permite validar reposiciones en estado ENVIADO.
2. Si hay linea rechazada sin justificacion, rechaza la validacion.
3. Si valida exitosamente, deja estado en VALIDATED.

### E2-H5 - Anulacion de reposicion en flujo de revision

Endpoint objetivo:

1. POST /api/v1/replenishment-reviews/{replenishmentId}/cancel

Alcance:

1. Endpoint exclusivo de Revision.
2. No aplica como endpoint de Administracion.

Tareas:

| ID       | Tarea                                              | Resultado esperado    |
| -------- | -------------------------------------------------- | --------------------- |
| E2-H5-T1 | Definir ReviewCancelVo con observacion obligatoria | Contrato funcional    |
| E2-H5-T2 | Implementar transicion de estado a ANULADO         | Estado consistente    |
| E2-H5-T3 | Registrar auditoria y motivo                       | Trazabilidad completa |
| E2-H5-T4 | Tests de negocio y contrato                        | Sin regresiones       |

Criterios de aceptacion:

1. Observacion de anulacion es obligatoria.
2. Solo se anula si el estado es anulable desde revision.
3. La respuesta debe detallar motivo en errors[] cuando falle.

### E2-H6 - Cierre automatico post-validacion (scheduler)

Componente objetivo:

1. Scheduler interno (sin endpoint publico obligatorio)

Tareas:

| ID       | Tarea                                                | Resultado esperado      |
| -------- | ---------------------------------------------------- | ----------------------- |
| E2-H6-T1 | Implementar job que procese VALIDATED -> PAID/ISSUED | Flujo automatico activo |
| E2-H6-T2 | Agregar logs funcionales y metricas de ejecucion     | Observabilidad minima   |
| E2-H6-T3 | Pruebas de integracion del job                       | Confiabilidad operativa |

Criterios de aceptacion:

1. Debe procesar solo registros elegibles y activos.
2. Debe registrar cantidad procesada y errores por corrida.
3. Debe ser idempotente ante reintentos.

### E2-H7 - Gestion de conceptos en detalle de revision

Endpoint objetivo:

1. GET /api/v1/replenishment-reviews/billing-concepts
2. POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/adjust-concept

Tipo de API en este plan:

1. `GET /billing-concepts` es endpoint de Revision con reuso interno de logica.
2. `POST /adjust-concept` es API nueva de Revision.

Alcance:

1. El flujo de `adjust-concept` es exclusivo de Revision.
2. No aplica como endpoint de Administracion.

Tareas:

| ID       | Tarea                                                                               | Resultado esperado                      |
| -------- | ----------------------------------------------------------------------------------- | --------------------------------------- |
| E2-H7-T1 | Integrar consulta de conceptos por area/transaccion/tipo documento                  | Popup funcional con datos consistentes  |
| E2-H7-T2 | Permitir actualizar `secConAreTra` y descripcion de concepto en detalle de revision | Paridad funcional de cambio de concepto |
| E2-H7-T3 | Validar impacto en totales/observacion cuando el cambio modifica aprobacion         | Sin inconsistencias contables           |
| E2-H7-T4 | Tests de servicio y controller para seleccion/cambio de concepto                    | Cobertura de regresion                  |

Criterios de aceptacion:

1. Debe permitir seleccion de concepto por fila de detalle en modo revision.
2. Debe registrar la actualizacion de concepto en la misma trazabilidad de ajuste.
3. Debe conservar compatibilidad con reglas de validacion existentes de detalle.

### E2-H8 - Validacion de retenciones y documentos de respaldo

Endpoint objetivo:

1. POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/withholding/validate
2. POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/withholding/validate-access-key

Alcance:

1. Endpoints exclusivos de Revision.
2. No aplican como endpoints de Administracion.

Tareas:

| ID       | Tarea                                                                   | Resultado esperado           |
| -------- | ----------------------------------------------------------------------- | ---------------------------- |
| E2-H8-T1 | Modelar contrato para retencion manual/electronica en revision          | Entrada/salida unificada     |
| E2-H8-T2 | Implementar validacion de retencion manual (conceptos e impuestos)      | Flujo manual cubierto        |
| E2-H8-T3 | Implementar validacion de retencion electronica por clave de acceso/XML | Flujo electronico cubierto   |
| E2-H8-T4 | Integrar reglas de no duplicidad y contexto bloqueado para retenciones  | Paridad legacy en bloqueos   |
| E2-H8-T5 | Tests de escenarios OK/error para manual/electronica                    | Cobertura funcional completa |

Criterios de aceptacion:

1. Debe soportar retencion manual y retencion electronica.
2. Debe devolver errores funcionales detallados cuando falle validacion documental.
3. Debe persistir conceptos de retencion validados en el detalle de revision.

### E2-H9 - Gestion de archivo de soporte y regla de huella de carbono

Endpoint objetivo:

1. POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/support-file

Alcance:

1. Endpoint exclusivo de Revision.
2. No aplica como endpoint de Administracion.

Tareas:

| ID       | Tarea                                                              | Resultado esperado             |
| -------- | ------------------------------------------------------------------ | ------------------------------ |
| E2-H9-T1 | Implementar carga/actualizacion de archivo soporte por linea       | Soporte documental por detalle |
| E2-H9-T2 | Integrar validacion de huella (archivo obligatorio cuando aplique) | Regla legacy cubierta          |
| E2-H9-T3 | Implementar limpieza/control de archivos reemplazados              | Sin residuos de archivo        |
| E2-H9-T4 | Tests de flujo manual/electronico y reemplazo de archivo           | Cobertura de regresion         |

Criterios de aceptacion:

1. Si el concepto requiere huella, el archivo soporte es obligatorio.
2. Debe soportar flujo de reemplazo de archivo y trazabilidad del cambio.
3. Debe fallar con mensaje funcional cuando falte soporte requerido.

### E2-H10 - Cierre de alcance: Gastos Personales (decision y trazabilidad)

Componente objetivo:

1. Decision funcional en Jira + task tecnico de implementacion o exclusion.

Tareas:

| ID        | Tarea                                                                                  | Resultado esperado         |
| --------- | -------------------------------------------------------------------------------------- | -------------------------- |
| E2-H10-T1 | Confirmar con negocio si Revision migrara Gastos Personales (local/exterior)           | Alcance cerrado y aprobado |
| E2-H10-T2 | Si SI aplica: crear historias tecnicas para ciudad/pais/moneda/tasa y totales exterior | Backlog completo de gastos |
| E2-H10-T3 | Si NO aplica: registrar exclusion formal y actualizar criterios de aceptacion de E2    | Trazabilidad de exclusion  |

Criterios de aceptacion:

1. Debe existir decision explicita y aprobada en Jira.
2. No debe quedar ambiguedad de alcance en flujos de Revision.

## 6) Ejemplos de endpoint, request y logica de negocio

### 6.1 Buscar en revision

Metodo y ruta:

1. POST /api/v1/replenishments/findByFilter

Request ejemplo:

```json
{
  "filters": [
    { "field": "workAreaCode", "operator": "EQUALS", "value": "101" },
    { "field": "replenishmentStatus", "operator": "EQUALS", "value": "ENV" }
  ],
  "page": 0,
  "size": 20
}
```

Regla principal:

1. Debe soportar filtros dinamicos de legacy (estado/fecha/area) reutilizando la logica actual.
2. Las acciones de Revision (editar/validar/anular) solo aplican para estado ENVIADA (ENV).
3. Si no existe brecha funcional, Front Revision consume shared; si aparece brecha, se habilita endpoint de Revision delegando internamente sin duplicar query.

### 6.2 Ajustar detalle aprobado

Metodo y ruta:

1. POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/adjust

Request ejemplo:

```json
{
  "approvedValue": 120.45,
  "approvedVatValue": 18.07,
  "validationObservation": "Ajuste por validacion de soporte"
}
```

Reglas principales:

1. Validar IVA con la misma formula de validate-vat.
2. Si approvedValue difiere del solicitado, exigir observacion.
3. Recalcular totales aprobados de cabecera.

### 6.3 Validar reposicion

Metodo y ruta:

1. POST /api/v1/replenishment-reviews/{replenishmentId}/validate

Request ejemplo:

```json
{
  "reviewObservation": "Revision contable completada"
}
```

Reglas principales:

1. Solo estado ENVIADO.
2. Debe existir al menos una linea activa y validable.
3. Cambiar a VALIDATED en transaccion atomica.

### 6.4 Anular reposicion

Metodo y ruta:

1. POST /api/v1/replenishment-reviews/{replenishmentId}/cancel

Request ejemplo:

```json
{
  "cancelObservation": "Soporte no cumple politica contable"
}
```

Reglas principales:

1. cancelObservation obligatoria.
2. Solo estados anulables.
3. Registrar auditoria de anulacion.

### 6.5 Validar retencion en revision

Metodo y ruta:

1. POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/withholding/validate

Request ejemplo:

```json
{
  "documentType": "RTM",
  "taxId": "1790012345001",
  "documentNumber": "001-001-000123456",
  "withholdingTotal": 12.55,
  "taxItems": [
    { "taxCode": "REN", "withheldValue": 10.0 },
    { "taxCode": "IVA", "withheldValue": 2.55 }
  ]
}
```

Reglas principales:

1. Validar consistencia de sumatoria de impuestos retenidos.
2. Validar no duplicidad y contexto permitido para retencion.
3. Persistir conceptos retenidos en el detalle validado.

### 6.6 Cargar archivo de soporte en revision

Metodo y ruta:

1. POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/support-file

Request ejemplo:

```json
{
  "fileName": "factura-001-001-000123456.pdf",
  "fileExtension": "pdf",
  "accessKey": null
}
```

Reglas principales:

1. Si el concepto requiere huella, el archivo es obligatorio.
2. Si existe archivo previo y se reemplaza, registrar limpieza/control del anterior.
3. Conservar trazabilidad del cambio en la linea de detalle.

### 6.7 Prevalidar detalle en revision (reuso actual)

Metodo y ruta:

1. POST /api/v1/replenishments/{replenishmentId}/details/validate-add
2. POST /api/v1/replenishments/details/validate-add

Request ejemplo (sin cabecera persistida):

```json
{
  "workAreaCode": 101,
  "transactionCode": "1",
  "currentRequestedTotal": 40.0,
  "detail": {
    "documentType": "FAC",
    "taxId": "1790012345001",
    "documentNumber": "001-001-000123456",
    "requestedValue": 20.0,
    "vatValue": 2.61
  }
}
```

Reglas principales:

1. Revision reutiliza estas rutas shared mientras no exista brecha funcional.
2. Si se requiere regla exclusiva de Revision, se habilita wrapper en `replenishment-reviews` delegando validaciones existentes.
3. Evitar duplicar reglas de prevalidacion entre shared y Revision.

## 7) Ubicacion esperada en Bruno (plan de implementacion)

Rutas base de coleccion:

1. Reuso de endpoints shared: GTF-DOC/REPLACEMENTS
2. Endpoints nuevos de Revision: GTF-DOC/REPLACEMENTS-REVIEWS

Ubicacion sugerida para cada endpoint segun estrategia vigente:

1. Shared (REPLACEMENTS/BUSCAR): Buscar reposiciones, obtener reposicion por id, listar conceptos.
2. Shared (REPLACEMENTS/VALIDAR): Prevalidar detalle con y sin cabecera.
3. Revision (REPLACEMENTS-REVIEWS/ACTUALIZAR): Ajustar detalle aprobado, validar reposicion, anular reposicion, ajustar concepto, retenciones, archivo soporte.

Hint de busqueda rapida en Bruno:

1. Buscar por ruta parcial shared: /api/v1/replenishments/
2. Buscar por ruta parcial revision: /api/v1/replenishment-reviews/

## 8) Plan de ejecucion incremental para el desarrollador asignado

1. Pre-sprint: cerrar verificacion funcional de H1/H2 sobre shared y registrar brechas (si existen).
2. Sprint 0 (solo si aplica): wrappers de Revision para H1/H2 por brecha validada.
3. Sprint 1: E2-H3 (ajustes de detalle + recalculo).
4. Sprint 2: E2-H4 y E2-H5 (validar + anular).
5. Sprint 3: E2-H6 (job de cierre + observabilidad).
6. Sprint 4: E2-H7 y E2-H8 (conceptos + retenciones).
7. Sprint 5: E2-H9 y E2-H10 (archivo/huella + cierre de alcance Gastos Personales).

## 9) Definicion de Done (global)

1. Endpoint implementado con QueryDSL + JPA/Hibernate.
2. Pruebas de servicio y controller en verde.
3. PMD/Sonar sin regresiones del modulo.
4. Endpoint documentado en Bruno.
5. Endpoint registrado en docs/apis.
6. Metodo + ruta + ubicacion Bruno informados al equipo.

## 10) Resultado de segunda revision de flujos legacy

Resumen ejecutivo:

1. Si hay hallazgos nuevos: el plan inicial no cubria completamente conceptos, retenciones,
   archivo/huella ni la decision formal de alcance de Gastos Personales.
2. El backlog E2 queda actualizado con historias E2-H7 a E2-H10 para cerrar esas brechas.
3. Se confirma estrategia hibrida: H1/H2 y consultas equivalentes reutilizan shared; endpoints de `replenishment-reviews` se crean para capacidades propias de Revision o por brecha funcional validada.

Checklist de brechas detectadas:

| Flujo legacy revisado                             | Estado en plan anterior | Estado actual      |
| ------------------------------------------------- | ----------------------- | ------------------ |
| Popup y cambio de concepto por linea              | Parcial                 | Cubierto en E2-H7  |
| Retencion manual/electronica/XML                  | No cubierto             | Cubierto en E2-H8  |
| Archivo soporte + huella de carbono               | No cubierto             | Cubierto en E2-H9  |
| Definicion formal de alcance de Gastos Personales | Ambiguo                 | Cubierto en E2-H10 |

## 11) Texto listo para copiar y pegar en Jira

### E2-H1 - Consulta de reposiciones para revision (filtros y paginacion)

Descripcion sugerida para la historia:

Como usuario del flujo de Revision, necesito consultar reposiciones con filtros dinamicos
por area, estado y fechas, para identificar rapidamente los casos que requieren gestion
contable. Esta historia se ejecuta por reuso directo del endpoint shared existente y solo
requiere desarrollo backend nuevo si se detecta una brecha funcional real.

Subtareas sugeridas (descripcion lista para Jira):

1. E2-H1-T1 - Validar contrato funcional del endpoint shared: Verificar que `POST /api/v1/replenishments/findByFilter` cubre el payload y la respuesta esperada por Front Revision, sin cambios de contrato.
2. E2-H1-T2 - Validar filtros y alias legacy: Confirmar soporte de `replenishmentStatus`, `replenishmentDate`, operadores dinamicos y estado "Todos" sin duplicar query.
3. E2-H1-T3 - Validar contexto de seguridad: Confirmar que compania y area se fuerzan por sesion y no por valores manipulables desde request.
4. E2-H1-T4 - Pruebas de regresion H1: Ejecutar pruebas de contrato/servicio para garantizar que no se rompe consumo de Administracion ni de Revision.
5. E2-H1-T5 (condicional) - Wrapper de Revision para filtro: Solo si hay brecha validada, exponer `POST /api/v1/replenishment-reviews/findByFilter` delegando a la misma logica.

### E2-H2 - Apertura de reposicion para revision (cabecera y detalle)

Descripcion sugerida para la historia:

Como usuario del flujo de Revision, necesito abrir una reposicion y sus lineas de detalle
para revisar montos, documentos y estado operativo antes de validar o anular. Esta historia
usa reuso directo de endpoints shared y solo pasa a wrapper de Revision si se confirma una
brecha funcional.

Subtareas sugeridas (descripcion lista para Jira):

1. E2-H2-T1 - Validar cabecera para Revision: Confirmar que `GET /api/v1/replenishments/{replenishmentId}` entrega datos requeridos para decision contable.
2. E2-H2-T2 - Validar detalle para Revision: Confirmar que `GET /api/v1/replenishments/{replenishmentId}/details` devuelve lineas activas con informacion suficiente para revision.
3. E2-H2-T3 - Validar reglas operativas de accion: Confirmar calculo/derivacion de banderas de accion (editable/validable/anulable) sin romper contrato vigente.
4. E2-H2-T4 - Pruebas de contrato y regresion H2: Cubrir escenarios de lectura existentes y consumo de Front Revision.
5. E2-H2-T5 (condicional) - Wrapper de Revision para apertura: Solo si hay brecha validada, exponer wrappers en `/api/v1/replenishment-reviews/...` delegando logica existente.

### E2-H3 - Ajuste contable por detalle con recalculo de totales

Descripcion sugerida para la historia:

Como usuario de Revision, necesito ajustar valores aprobados por linea de detalle con
trazabilidad y recalculo inmediato de totales, para reflejar la validacion contable real
sin perder historial ni consistencia transaccional.

Subtareas sugeridas (descripcion lista para Jira):

1. E2-H3-T1 - Definir contrato de ajuste: Crear VO/DTO de entrada y salida para ajuste de detalle con validaciones de campos obligatorios.
2. E2-H3-T2 - Implementar validaciones contables: Aplicar reglas de IVA, duplicidad, observacion obligatoria y restricciones de negocio por linea.
3. E2-H3-T3 - Persistir ajuste y recalcular cabecera: Guardar valor aprobado y recalcular totales aprobados de manera atomica.
4. E2-H3-T4 - Implementar rechazo/reactivacion de linea: Cubrir cambio de estado de detalle sin borrado fisico para mantener trazabilidad.
5. E2-H3-T5 - Pruebas completas H3: Implementar pruebas unitarias e integracion para escenarios OK/error y regresion.

### E2-H4 - Validacion contable de reposicion revisada

Descripcion sugerida para la historia:

Como usuario de Revision, necesito validar formalmente una reposicion revisada para moverla
al estado contable correspondiente, garantizando precondiciones funcionales y auditoria
consistente del proceso.

Subtareas sugeridas (descripcion lista para Jira):

1. E2-H4-T1 - Implementar servicio de validacion: Construir flujo de validacion central con orquestacion de reglas de negocio.
2. E2-H4-T2 - Implementar precondiciones de validacion: Verificar lineas activas, montos consistentes, observaciones requeridas y estado permitido.
3. E2-H4-T3 - Persistir cambio de estado y auditoria: Cambiar a `VALIDATED` (codigo legacy `VAL`) cuando corresponda y registrar trazabilidad de usuario/fecha.
4. E2-H4-T4 - Pruebas funcionales H4: Cubrir escenarios de exito y rechazo con mensajes funcionales claros en `errors[]`.

### E2-H5 - Anulacion de reposicion en flujo de revision

Descripcion sugerida para la historia:

Como usuario de Revision, necesito anular una reposicion con motivo obligatorio para cortar
el flujo de aprobacion cuando existan inconsistencias, manteniendo auditoria completa y
reglas de transicion de estado.

Subtareas sugeridas (descripcion lista para Jira):

1. E2-H5-T1 - Definir contrato de anulacion: Crear VO/DTO con observacion obligatoria y validaciones de entrada.
2. E2-H5-T2 - Implementar reglas de anulacion: Validar estados anulables y restricciones funcionales antes de ejecutar la accion.
3. E2-H5-T3 - Persistir anulacion y auditoria: Cambiar estado a anulado, registrar motivo y metadatos de trazabilidad.
4. E2-H5-T4 - Pruebas funcionales H5: Cubrir anulacion correcta, anulacion rechazada y contrato de errores.

### E2-H6 - Cierre automatico post-validacion (scheduler)

Descripcion sugerida para la historia:

Como equipo backend, necesitamos un cierre automatico de reposiciones validadas para
ejecutar el paso contable final sin intervencion manual, con control de idempotencia,
observabilidad y manejo de errores por corrida.

Subtareas sugeridas (descripcion lista para Jira):

1. E2-H6-T1 - Implementar scheduler de cierre: Procesar transicion `VALIDATED (VAL) -> PAID/ISSUED (PAG/CHE)` sobre registros elegibles.
2. E2-H6-T2 - Garantizar idempotencia del job: Evitar dobles cierres ante reintentos o ejecuciones solapadas.
3. E2-H6-T3 - Implementar observabilidad minima: Registrar metricas, volumen procesado y errores funcionales por ejecucion.
4. E2-H6-T4 - Pruebas de integracion H6: Validar flujo completo del job, incluyendo reintentos y manejo de fallos.

### E2-H7 - Gestion de conceptos en detalle de revision

Descripcion sugerida para la historia:

Como usuario de Revision, necesito consultar y cambiar el concepto contable de una linea
de detalle para reflejar correctamente la naturaleza del gasto y mantener consistencia
con reglas de validacion y recalculo.

Subtareas sugeridas (descripcion lista para Jira):

1. E2-H7-T1 - Integrar consulta de conceptos: Reusar endpoint de conceptos para poblar popup segun area/transaccion/tipo documento.
2. E2-H7-T2 - Implementar cambio de concepto por linea: Exponer endpoint de ajuste de concepto en contexto de Revision.
3. E2-H7-T3 - Validar impacto funcional del cambio: Recalcular y mantener trazabilidad cuando el cambio afecta aprobacion.
4. E2-H7-T4 - Pruebas funcionales H7: Cubrir seleccion/cambio de concepto y escenarios de incompatibilidad.

### E2-H8 - Validacion de retenciones y documentos de respaldo

Descripcion sugerida para la historia:

Como usuario de Revision, necesito validar retenciones manuales y electronicas para asegurar
consistencia tributaria y documental de cada detalle, incluyendo validaciones por clave de
acceso y controles de duplicidad.

Subtareas sugeridas (descripcion lista para Jira):

1. E2-H8-T1 - Definir contratos de retencion: Modelar DTOs para validacion manual/electronica con estructura unificada.
2. E2-H8-T2 - Implementar validacion de retencion manual: Validar impuestos, montos y reglas funcionales del escenario manual.
3. E2-H8-T3 - Implementar validacion electronica documental: Validar clave de acceso/XML y consistencia de datos tributarios.
4. E2-H8-T4 - Integrar reglas de contexto y no duplicidad: Aplicar controles de seguridad y duplicidad en flujo de Revision.
5. E2-H8-T5 - Pruebas funcionales H8: Cubrir escenarios OK/error para manual y electronica.

### E2-H9 - Gestion de archivo de soporte y regla de huella de carbono

Descripcion sugerida para la historia:

Como usuario de Revision, necesito cargar y reemplazar archivo de soporte por linea de detalle
para cumplir politicas de respaldo documental y la regla de huella de carbono cuando el
concepto lo exija.

Subtareas sugeridas (descripcion lista para Jira):

1. E2-H9-T1 - Implementar endpoint de archivo soporte: Habilitar carga/actualizacion de soporte documental por detalle.
2. E2-H9-T2 - Implementar manejo de reemplazo de archivo: Controlar limpieza de archivo previo y trazabilidad de reemplazo.
3. E2-H9-T3 - Aplicar regla de huella de carbono: Exigir archivo cuando el concepto lo requiera y validar cumplimiento.
4. E2-H9-T4 - Pruebas funcionales H9: Cubrir carga inicial, reemplazo, validaciones y errores de negocio.

### E2-H10 - Cierre de alcance: Gastos Personales (decision y trazabilidad)

Descripcion sugerida para la historia:

Como responsable funcional/tactico, necesito cerrar formalmente el alcance de Gastos
Personales para evitar ambiguedades en el backlog, definiendo si se implementa en esta
epica o si se excluye con aprobacion explicita de negocio.

Subtareas sugeridas (descripcion lista para Jira):

1. E2-H10-T1 - Mesa de definicion con negocio: Confirmar decision funcional (incluir/excluir) sobre Gastos Personales local y exterior.
2. E2-H10-T2 (si aplica) - Descomponer backlog tecnico: Crear historias y tareas para ciudad, pais, moneda, tasa y totales de exterior.
3. E2-H10-T3 (si no aplica) - Formalizar exclusion: Registrar exclusion en Jira y actualizar criterios de aceptacion de la epica E2.
4. E2-H10-T4 - Actualizar trazabilidad de alcance: Reflejar decision final en documentos funcionales y matriz de cobertura.

## 12) Contexto operativo obligatorio para nuevo desarrollador

## 12.1 Fuente de verdad de estados (NO inventados)

Cuando en este plan se usa `VALIDATED`, corresponde al estado legacy `VAL` (etiqueta `VALIDADO`).
No es un estado inventado; es una representacion legible usada en el contrato tecnico y documental.

Evidencia legacy (archivo de recursos):

1. `codigo.estado.reposicion.pendiente = PEN`
2. `codigo.estado.reposicion.enviado = ENV`
3. `codigo.estado.reposicion.validado = VAL`
4. `codigo.estado.reposicion.chequeEmitido = CHE`
5. `codigo.estado.reposicion.pagado = PAG`

Evidencia migrado (mapeo tecnico):

1. `STATUS_CODE_SENT = ENV`
2. `STATUS_CODE_VALIDATED = VAL`
3. `STATUS_CODE_PAID = PAG`
4. `STATUS_CODE_ISSUED = CHE`
5. Enum de dominio: `PENDING, SENT, VALIDATED, PAID, ISSUED, COLLECTED, CANCELLED`

Regla para historias y QA:

1. En texto funcional se puede usar `Validado/VALIDATED`.
2. En persistencia/filtro legacy usar codigo `VAL`.
3. En evidencias de prueba, reportar ambos: nombre legible y codigo.

## 12.2 Alcance real ya implementado (evitar retrabajo)

Antes de abrir desarrollo nuevo, validar si la capacidad ya existe en shared:

1. `POST /api/v1/replenishments/findByFilter`
2. `GET /api/v1/replenishments/{replenishmentId}`
3. `GET /api/v1/replenishments/{replenishmentId}/details`
4. `GET /api/v1/replenishments/billing-concepts`
5. `POST /api/v1/replenishments/{replenishmentId}/details/validate-add`
6. `POST /api/v1/replenishments/details/validate-add`

Decision vigente:

1. H1/H2 consumen estos endpoints por defecto.
2. Solo crear wrapper en `replenishment-reviews` cuando exista brecha funcional validada.

## 12.3 Mapa tecnico exacto de archivos a tocar

Para historias con backend nuevo (H3-H9), usar este mapa minimo:

1. Capa controller:
  `gtf-replacements-services/src/main/java/ec/com/smx/gtf/replacements/controller/replenishment-reviews/ReplenishmentReviewController.java`
2. Capa servicio/reglas:
  `gtf-replacements-core/src/main/java/ec/com/smx/gtf/replacements/replenishment/service/`
3. Capa repositorio/querydsl:
  `gtf-replacements-core/src/main/java/ec/com/smx/gtf/replacements/replenishment/repository/`
4. Capa contratos VO/DTO:
  `gtf-replacements-vo/src/main/java/ec/com/smx/gtf/replacements/vo/replenishment/`
5. Tests controller:
  `gtf-replacements-services/src/test/java/ec/com/smx/gtf/replacements/controller/`
6. Tests servicio/repositorio:
  `gtf-replacements-core/src/test/java/ec/com/smx/gtf/replacements/replenishment/`
7. Documentacion obligatoria:
  Bruno en `GTF-DOC/REPLACEMENTS-REVIEWS` + registro en `docs/apis`.

## 12.4 Formato de subtarea Jira (plantilla obligatoria)

Usar esta plantilla para cada subtarea tecnica:

1. Objetivo tecnico:
  describir exactamente la regla o capacidad a implementar.
2. Entradas:
  payload, parametros, estado inicial y rol esperado.
3. Salidas:
  respuesta esperada, cambio de estado y side effects.
4. Archivos a tocar:
  listar rutas exactas (controller/service/repository/vo/test).
5. Criterios de aceptacion:
  casos OK + casos de error + contrato `errors[]`.
6. Evidencia de cierre:
  pruebas ejecutadas, request Bruno y actualizacion de docs/apis.

## 12.5 Checklist de ejecucion por historia (para onboarding)

Aplicar en cada historia E2-H1..E2-H10:

1. Confirmar si es reuso o desarrollo nuevo.
2. Confirmar estados permitidos (nombre legible + codigo legacy).
3. Confirmar validaciones obligatorias de negocio.
4. Implementar pruebas antes de cerrar subtarea.
5. Publicar evidencia en Bruno y `docs/apis`.
6. Registrar avance en este documento y en bitacora de prompts.

## 12.6 Matriz historia -> que tocar exactamente

| Historia | Que hacer exactamente | Archivos minimos a tocar | Resultado esperado |
| -------- | --------------------- | ------------------------ | ------------------ |
| E2-H1 | Validar reuso de filtro shared y abrir wrapper solo si hay gap | `ReplenishmentController` (lectura), tests de contrato, Bruno shared | Evidencia de consumo H1 sin regresion |
| E2-H2 | Validar reuso de lectura shared (cabecera/detalle) y flags operativos | `ReplenishmentController` (lectura), tests de lectura, Bruno shared | Evidencia de apertura para Revision |
| E2-H3 | Crear endpoint de ajuste por detalle + reglas + recalculo | `ReplenishmentReviewController`, servicio de revision, repositorio de detalle, VO ajuste, tests | Ajuste persistido con auditoria y recalculo |
| E2-H4 | Crear endpoint de validar reposicion revisada | `ReplenishmentReviewController`, servicio de validacion, tests de estado | Transicion a `VAL` con reglas de precondicion |
| E2-H5 | Crear endpoint de anular reposicion en revision | `ReplenishmentReviewController`, servicio de anulacion, VO anulacion, tests | Anulacion con motivo obligatorio y auditoria |
| E2-H6 | Implementar scheduler de cierre contable | job/scheduler en core, servicio de cierre, pruebas de integracion | Transicion `VAL -> PAG/CHE` idempotente |
| E2-H7 | Reusar consulta de conceptos + crear cambio de concepto | endpoint shared de conceptos + endpoint nuevo en revision, servicio de ajuste concepto, tests | Cambio de concepto trazable por linea |
| E2-H8 | Crear validaciones de retencion manual/electronica | endpoints de revision, VO retencion, servicio de validacion, tests | Retencion validada con control documental |
| E2-H9 | Crear carga/reemplazo de archivo soporte | endpoint de revision, servicio de archivos, regla huella, tests | Soporte persistido con validaciones |
| E2-H10 | Cerrar decision de alcance de Gastos Personales | Jira + documentacion funcional + trazabilidad en este MD | Decision formal aprobada y sin ambiguedad |

Notas operativas para esta matriz:

1. Cuando la fila diga "reuso", no abrir desarrollo nuevo si no existe brecha funcional demostrada.
2. Cuando la fila diga "endpoint nuevo", incluir siempre: controller + servicio + test + Bruno + docs/apis.
3. Todo cambio de estado debe reportar codigo legacy y nombre legible en evidencia de QA.
