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

1. El backend actual de `replenishments/findByFilter` ya cubre filtros dinamicos (alias, comparadores y estado "Todos"), por lo que no es obligatorio crear un endpoint nuevo para H1.
2. El endpoint `POST /api/v1/replenishment-reviews/findByFilter` queda como wrapper opcional de contexto (segregacion de dominio/seguridad) y no como requisito tecnico minimo.
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

### 4.1 Reutilizacion directa

| Activo actual                                  | Uso en Revision                                                | Accion                                  |
| ---------------------------------------------- | -------------------------------------------------------------- | --------------------------------------- |
| POST /api/v1/replenishments/validate-vat       | Validar ajuste de IVA en detalle aprobado (H3)                 | Reutilizar directo (sin endpoint nuevo) |
| POST /api/v1/replenishments/validate-duplicate | Verificar duplicidad documental en ajustes/retenciones (H3/H8) | Reutilizar directo (sin endpoint nuevo) |
| GET /api/v1/replenishments/responsibles        | Responsable operativo para flujo de validacion                 | Reutilizar directo (sin endpoint nuevo) |

### 4.2 Reutilizacion condicionada

| Activo actual (interno)                                            | API expuesta en Revision                                                                                                                       | Uso en Revision                                              | Accion                                                         |
| ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ | -------------------------------------------------------------- |
| POST /api/v1/replenishments/findByFilter                           | Opcional: POST /api/v1/replenishment-reviews/findByFilter                                                                                      | Buscar reposiciones para revision (H1) con filtros dinamicos | Reuso directo recomendado; wrapper opcional sin duplicar query |
| GET /api/v1/replenishments/{replenishmentId}                       | Base recomendada: GET /api/v1/replenishments/{replenishmentId} / Opcional: GET /api/v1/replenishment-reviews/{replenishmentId}                 | Visualizar cabecera de reposicion en revision (H2)           | Reuso directo recomendado; wrapper opcional por segregacion    |
| GET /api/v1/replenishments/{replenishmentId}/details               | Base recomendada: GET /api/v1/replenishments/{replenishmentId}/details / Opcional: GET /api/v1/replenishment-reviews/{replenishmentId}/details | Visualizar detalle de reposicion en revision (H2)            | Reuso directo recomendado; wrapper opcional por segregacion    |
| GET /api/v1/replenishments/billing-concepts                        | GET /api/v1/replenishment-reviews/billing-concepts                                                                                             | Popup de conceptos en revision (H7)                          | Reutilizar con wrapper de lectura                              |
| POST /api/v1/replenishments/{replenishmentId}/details/validate-add | POST /api/v1/replenishment-reviews/{replenishmentId}/details/validate-add                                                                      | Prevalidar cambios de detalle en modo revision               | Reutilizar con wrapper de reglas de revision                   |
| POST /api/v1/replenishments/details/validate-add                   | POST /api/v1/replenishment-reviews/details/validate-add                                                                                        | Prevalidar detalle cuando no exista cabecera persistida      | Reutilizar con wrapper de reglas de revision                   |

Regla de consumo para H1/H2 (ajustada con evidencia):

1. H1 puede salir a produccion consumiendo directamente `POST /api/v1/replenishments/findByFilter`.
2. Si se requiere separacion de dominio o seguridad por ruta, se expone `POST /api/v1/replenishment-reviews/findByFilter` como wrapper sin duplicar query.
3. H2 puede salir a produccion consumiendo directamente `GET /api/v1/replenishments/{replenishmentId}` y `GET /api/v1/replenishments/{replenishmentId}/details`.
4. Si se requiere separacion de dominio o seguridad por ruta, se exponen wrappers equivalentes en `/api/v1/replenishment-reviews/...` sin duplicar lectura.
5. En ambos casos se mantiene la regla funcional: acciones de Revision (editar/validar/anular) solo cuando estado = ENVIADA (ENV).
6. Consideracion tecnica: `GET /details` retorna lineas activas de detalle (estado de detalle activo), no aplica filtro por estado de cabecera.
7. Los activos de 4.1 se mantienen como reuso directo (sin endpoint nuevo).

### 4.3 No reutilizar (crear en modulo de revision)

| Activo                                                                            | Motivo                                                                             |
| --------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| POST /api/v1/replenishment-management/{replenishmentId}/send                      | Es flujo administracion -> contabilidad, no flujo contabilidad -> validacion final |
| POST /api/v1/replenishment-management/{replenishmentId}/details/{detailId}/delete | Revision requiere rechazo/activacion de linea, no borrado operativo                |

### 4.4 Trazabilidad de historias E2 vs tipo de API

| Historia | Endpoint(s) objetivo                                                                                                                                                                                  | Tipo de API                           | Referencia |
| -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------- | ---------- |
| E2-H1    | POST /api/v1/replenishments/findByFilter (base) / opcional wrapper POST /api/v1/replenishment-reviews/findByFilter                                                                                    | Reuso directo con wrapper opcional    | 4.2        |
| E2-H2    | GET /api/v1/replenishments/{replenishmentId}; GET /api/v1/replenishments/{replenishmentId}/details (base) / wrappers opcionales en /replenishment-reviews                                             | Reuso directo con wrapper opcional    | 4.2        |
| E2-H3    | POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/adjust                                                                                                                        | Nueva API + validaciones reutilizadas | 4.1        |
| E2-H4    | POST /api/v1/replenishment-reviews/{replenishmentId}/validate                                                                                                                                         | Nueva API                             | 4.3        |
| E2-H5    | POST /api/v1/replenishment-reviews/{replenishmentId}/cancel                                                                                                                                           | Nueva API                             | 4.3        |
| E2-H6    | Scheduler interno (sin endpoint publico)                                                                                                                                                              | Nuevo componente                      | 4.3        |
| E2-H7    | GET /api/v1/replenishment-reviews/billing-concepts; POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/adjust-concept                                                            | Mixta: wrapper (GET) + nueva (POST)   | 4.2 + 4.3  |
| E2-H8    | POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/withholding/validate; POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/withholding/validate-access-key | Nueva API                             | 4.3        |
| E2-H9    | POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/support-file                                                                                                                  | Nueva API                             | 4.3        |
| E2-H10   | Decision funcional en Jira (sin endpoint directo)                                                                                                                                                     | Sin API directa                       | N/A        |

## 5) Epica y backlog Jira listo para asignar

## E2 - Revision de Reposiciones

Descripcion: Implementar el modulo backend de revision contable de reposiciones enviadas,
con paridad funcional legacy y reutilizacion de APIs ya construidas en administracion.

Regla de backlog (ajustada):

1. Este backlog de implementacion incluye solo desarrollo backend nuevo.
2. H1 (filtro) y H2 (lectura por id) quedan como consumo/reuso de endpoints ya existentes,
   por lo que no generan tareas de desarrollo en este paquete base.
3. Las APIs de H3 a H9 son exclusivas de Revision (`/api/v1/replenishment-reviews/...`)
   y no aplican como endpoints de Administracion.

### E2-H1 - Buscar reposiciones enviadas para revision

Endpoint objetivo:

1. Base recomendada: POST /api/v1/replenishments/findByFilter
2. Opcional por segregacion de contexto: POST /api/v1/replenishment-reviews/findByFilter

Tipo de API en este plan:

1. Reuso directo del endpoint existente con reglas funcionales de Revision.
2. Wrapper opcional solo si se requiere separacion por ruta/permisos.

Tareas de desarrollo backend:

1. No aplica en este backlog base (reuso de endpoint existente).
2. Si se aprueba wrapper de contexto, abrir historia tecnica separada fuera del paquete de APIs nuevas.

Criterios de consumo/QA:

1. Debe filtrar por codigo area, nombre area, estado y rango de fechas.
2. Debe soportar filtros dinamicos con alias legacy (`replenishmentStatus`, `replenishmentDate`) y manejo de estado "Todos".
3. Estado revisable operativo para acciones (editar/validar/anular): solo ENVIADA (ENV).
4. Debe mantener envelope BaseResponseVo.

### E2-H2 - Visualizar reposicion para revision

Endpoint objetivo:

1. Base recomendada: GET /api/v1/replenishments/{replenishmentId}
2. Base recomendada: GET /api/v1/replenishments/{replenishmentId}/details
3. Opcional por segregacion de contexto: GET /api/v1/replenishment-reviews/{replenishmentId}
4. Opcional por segregacion de contexto: GET /api/v1/replenishment-reviews/{replenishmentId}/details

Tipo de API en este plan:

1. Reuso directo recomendado de los GET actuales.
2. Wrapper de lectura opcional si se aprueba segregacion por ruta/permisos.

Tareas de desarrollo backend:

1. No aplica en este backlog base (reuso de endpoints existentes).
2. Si se aprueba wrapper de contexto, abrir historia tecnica separada fuera del paquete de APIs nuevas.

Criterios de consumo/QA:

1. Debe exponer montos solicitados y montos aprobables por linea.
2. Debe indicar si la reposicion se puede validar o anular (regla de acciones por estado ENVIADA).
3. El endpoint de detalle debe retornar lineas activas de detalle.
4. La lectura no debe restringirse por estado especifico de cabecera; la restriccion aplica a acciones de Revision.
5. No debe romper contratos vigentes de administracion.

### E2-H3 - Ajustar valores aprobados en detalle

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

### E2-H4 - Validar reposicion

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

### E2-H5 - Anular reposicion desde revision

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

### E2-H6 - Cierre post-validacion (job)

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

### E2-H7 - Gestion de conceptos en revision (popup + cambio de concepto)

Endpoint objetivo:

1. GET /api/v1/replenishment-reviews/billing-concepts
2. POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/adjust-concept

Tipo de API en este plan:

1. `GET /billing-concepts` es reutilizacion condicionada (wrapper sobre `replenishments`).
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

### E2-H8 - Validacion de retenciones y documentos en revision

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

### E2-H9 - Gestion de archivo soporte y regla de huella de carbono en revision

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

### E2-H10 - Decision de alcance sobre Gastos Personales (control de brecha)

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

1. Base recomendada: POST /api/v1/replenishments/findByFilter
2. Opcional por contexto: POST /api/v1/replenishment-reviews/findByFilter

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
3. Si se habilita ruta `replenishment-reviews`, debe ser wrapper sin duplicar query.

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

### 6.7 Prevalidar detalle en revision (wrapper)

Metodo y ruta:

1. POST /api/v1/replenishment-reviews/{replenishmentId}/details/validate-add
2. POST /api/v1/replenishment-reviews/details/validate-add

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

1. El Front de Revision llama siempre rutas `/replenishment-reviews/...`.
2. El wrapper reutiliza internamente validaciones de `replenishments`.
3. El wrapper agrega reglas propias de Revision sin requerir parametro extra de contexto.

## 7) Ubicacion esperada en Bruno (plan de implementacion)

Ruta base de coleccion:

1. GTF-DOC/REPLACEMENTS

Ubicacion sugerida para cada endpoint de revision (nuevo o wrapper):

1. BUSCAR/7) Buscar reposiciones en revision.bru
2. BUSCAR/8) Obtener reposicion en revision por id.bru
3. BUSCAR/9) Listar conceptos en revision.bru
4. ACTUALIZAR/5) Prevalidar detalle en revision con cabecera.bru
5. ACTUALIZAR/6) Prevalidar detalle en revision sin cabecera.bru
6. ACTUALIZAR/7) Ajustar detalle aprobado en revision.bru
7. ACTUALIZAR/8) Validar reposicion en revision.bru
8. ACTUALIZAR/9) Anular reposicion en revision.bru
9. ACTUALIZAR/10) Ajustar concepto en revision.bru
10. ACTUALIZAR/11) Validar retencion en revision.bru
11. ACTUALIZAR/12) Validar retencion por clave acceso en revision.bru
12. ACTUALIZAR/13) Cargar archivo soporte en revision.bru

Hint de busqueda rapida en Bruno:

1. Buscar por prefijo: Revision
2. Buscar por ruta parcial: /replenishment-reviews/

## 8) Plan de ejecucion incremental para el desarrollador asignado

1. Pre-sprint: confirmar consumo de H1/H2 por reuso (sin desarrollo backend nuevo).
2. Sprint 1: E2-H3 (ajustes de detalle + recalculo).
3. Sprint 2: E2-H4 y E2-H5 (validar + anular).
4. Sprint 3: E2-H6 (job de cierre + observabilidad).
5. Sprint 4: E2-H7 y E2-H8 (conceptos + retenciones).
6. Sprint 5: E2-H9 y E2-H10 (archivo/huella + cierre de alcance Gastos Personales).

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
3. Se confirma que los endpoints publicos de Revision deben exponerse bajo `/replenishment-reviews/...`
   y reutilizar internamente logica de `replenishments` mediante wrapper.

Checklist de brechas detectadas:

| Flujo legacy revisado                             | Estado en plan anterior | Estado actual      |
| ------------------------------------------------- | ----------------------- | ------------------ |
| Popup y cambio de concepto por linea              | Parcial                 | Cubierto en E2-H7  |
| Retencion manual/electronica/XML                  | No cubierto             | Cubierto en E2-H8  |
| Archivo soporte + huella de carbono               | No cubierto             | Cubierto en E2-H9  |
| Definicion formal de alcance de Gastos Personales | Ambiguo                 | Cubierto en E2-H10 |
