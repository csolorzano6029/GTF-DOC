# Checklist de Estado Actual - JIRA Backend Reposiciones

Fecha de corte: 2026-07-10

Fuente base analizada:
- docs/JIRA-Backend-Reposiciones-Epicas.md
- Estado real del codigo en gtf-replacements-root
- Trazabilidad operativa en BITACORA-PROMPTS.md

Alcance de este checklist:
- Solo backend de Administracion de Reposiciones (E1)
- Solo backend de Revision de Reposiciones (E2)

Criterio de clasificacion:
- Completada: existe implementacion y evidencia tecnica verificable.
- Pendiente: no existe implementacion completa o falta evidencia requerida.

Resumen general:
- Total de tareas JIRA identificadas: 56
- Completadas: 19
- Pendientes: 37

## E1 - Administracion de Reposiciones

### E1-H0 - Configuracion base del proyecto y despliegue

| ID tarea | Estado | Evidencia | Nota |
| --- | --- | --- | --- |
| E1-H0-T1 | Completada | application*.yaml con context-path /gtfReplacementsServices | Perfiles de entorno presentes |
| E1-H0-T2 | Completada | application*.yaml con AS400JDBCDriver, Hikari y variables DB_* | Configuracion DB2 parametrizada |
| E1-H0-T3 | Completada | Hay keycloak por perfil (server-url/realm/resource) y se acepta clientId institucional comun por alcance del equipo | Reclasificada como completada por decision funcional del usuario (no aplica gestion con Seguridad en este frente) |
| E1-H0-T4 | Completada | Jenkinsfile, gtf-replacements-services/Dockerfile, ci/helm/values.yaml | Namespace gtf y probes de health definidos |
| E1-H0-T5 | Completada | BITACORA-PROMPTS #63 y #66 | PMD y Sonar reportados en verde |

### E1-H1 - Modelo de datos y persistencia de reposiciones

| ID tarea | Estado | Evidencia | Nota |
| --- | --- | --- | --- |
| E1-H1-T1 | Completada | ReplenishmentEntity, ReplenishmentDetailEntity, IDs compuestos | Mapeo JPA contra tablas legacy |
| E1-H1-T2 | Completada | IReplenishmentRepository, ReplenishmentRepository, IReplenishmentDetailRepository | QueryDSL operativo |
| E1-H1-T3 | Completada | ReplenishmentVo, ReplenishmentDetailVo, FundHeaderVo, ResponsibleVo, enums | VOs y enums creados |
| E1-H1-T4 | Completada | IReplenishmentService y ReplenishmentService | Servicio base implementado |

### E1-H2 - Consultar cabecera del fondo del local

| ID tarea | Estado | Evidencia | Nota |
| --- | --- | --- | --- |
| E1-H2-T1 | Completada | ReplenishmentService.getCurrentFundHeader + WorkAreaFundRepository.findCurrentFundHeader | Calculo de cabecera de fondo implementado |
| E1-H2-T2 | Completada | ReplenishmentController GET /current-fund | Endpoint disponible |
| E1-H2-T3 | Completada | ReplenishmentServiceTest y ReplenishmentControllerTest | Cobertura de pruebas presente |

### E1-H3 - Listar y buscar reposiciones

| ID tarea | Estado | Evidencia | Nota |
| --- | --- | --- | --- |
| E1-H3-T1 | Completada | ReplenishmentRepository.findByFilter | Filtro y paginacion implementados |
| E1-H3-T2 | Completada | ReplenishmentController POST /findByFilter | Endpoint disponible |
| E1-H3-T3 | Completada | ReplenishmentRepositoriesCoverageTest y ReplenishmentControllerTest | Pruebas de flujo y filtros |

### E1-H4 - Crear y editar reposicion (CRUD cabecera)

| ID tarea | Estado | Evidencia | Nota |
| --- | --- | --- | --- |
| E1-H4-T1 | Completada | ReplenishmentService.create + validacion de pendiente por local | Regla principal de creacion aplicada |
| E1-H4-T2 | Completada | ReplenishmentController GET /{replenishmentId} + service.findByCode | Lectura por ID implementada |
| E1-H4-T3 | Completada | ReplenishmentController PUT /{replenishmentId} + validacion estado PENDING/PEN | Actualizacion de cabecera implementada |
| E1-H4-T4 | Completada | ReplenishmentControllerTest y ReplenishmentServiceTest | Pruebas de create/read/update |

### E1-H5 - Gestion del detalle de documentos

| ID tarea | Estado | Evidencia | Nota |
| --- | --- | --- | --- |
| E1-H5-T1 | Pendiente | No existe endpoint POST /{id}/details en controller | Agregar linea no implementado |
| E1-H5-T2 | Pendiente | No existe endpoint PUT /{id}/details/{detailId} | Edicion de detalle no implementada |
| E1-H5-T3 | Pendiente | No existe endpoint DELETE /{id}/details/{detailId} | Eliminacion de detalle no implementada |
| E1-H5-T4 | Pendiente | No existen endpoints /document-types ni /billing-concepts | Catalogos no implementados |
| E1-H5-T5 | Pendiente | No hay suite de pruebas para CRUD detalle y catalogos | Pruebas pendientes |

### E1-H6 - Validaciones de negocio del ingreso

| ID tarea | Estado | Evidencia | Nota |
| --- | --- | --- | --- |
| E1-H6-T1 | Pendiente | No existe endpoint /validate-vat | Validacion IVA pendiente |
| E1-H6-T2 | Pendiente | No existe endpoint /validate-duplicate | Validacion de duplicado pendiente |
| E1-H6-T3 | Pendiente | No hay logica de valor maximo + excepciones por parametro | Regla de tope pendiente |
| E1-H6-T4 | Pendiente | No hay pruebas de reglas H6 | Suite pendiente |

### E1-H7 - Retenciones en el ingreso

| ID tarea | Estado | Evidencia | Nota |
| --- | --- | --- | --- |
| E1-H7-T1 | Pendiente | No existe evidencia de spike tecnico con definicion SRI/Frank en repo | Dependencia externa sin cerrar |
| E1-H7-T2 | Pendiente | No existe modulo/endpoints withholdings electronicas | Flujo pendiente |
| E1-H7-T3 | Pendiente | No existe endpoint de carga XML de retencion | Flujo pendiente |
| E1-H7-T4 | Pendiente | No existe endpoint para retencion manual | Flujo pendiente |
| E1-H7-T5 | Pendiente | No hay pruebas de retenciones | Suite pendiente |

### E1-H8 - Enviar reposicion a contabilidad

| ID tarea | Estado | Evidencia | Nota |
| --- | --- | --- | --- |
| E1-H8-T1 | Pendiente | No existe endpoint GET /responsibles en controller actual | Consulta responsables pendiente |
| E1-H8-T2 | Pendiente | No existe endpoint POST /{id}/send | Envio a contabilidad pendiente |
| E1-H8-T3 | Pendiente | No hay pruebas de envio y validacion 422 | Suite pendiente |

### E1-H9 - Anular e imprimir reposicion

| ID tarea | Estado | Evidencia | Nota |
| --- | --- | --- | --- |
| E1-H9-T1 | Pendiente | No existe endpoint POST /{id}/cancel en controller actual | Anulacion pendiente |
| E1-H9-T2 | Pendiente | No existe endpoint GET /{id}/print | Impresion PDF pendiente |
| E1-H9-T3 | Pendiente | No hay pruebas para anulacion/impresion | Suite pendiente |

## E2 - Revision de Reposiciones

### E2-H1 - Listar reposiciones enviadas

| ID tarea | Estado | Evidencia | Nota |
| --- | --- | --- | --- |
| E2-H1-T1 | Pendiente | No existe modulo repository/service de replenishment-reviews | Filtro de revision pendiente |
| E2-H1-T2 | Pendiente | No existe controller para /api/v1/replenishment-reviews/findByFilter | Endpoint pendiente |
| E2-H1-T3 | Pendiente | No hay pruebas de H1 | Suite pendiente |

### E2-H2 - Ver reposicion para revision

| ID tarea | Estado | Evidencia | Nota |
| --- | --- | --- | --- |
| E2-H2-T1 | Pendiente | No existe servicio de revision solicitado vs aprobado | Logica pendiente |
| E2-H2-T2 | Pendiente | No existe GET /api/v1/replenishment-reviews/{id} | Endpoint pendiente |
| E2-H2-T3 | Pendiente | No hay pruebas de H2 | Suite pendiente |

### E2-H3 - Ajustar valor aprobado

| ID tarea | Estado | Evidencia | Nota |
| --- | --- | --- | --- |
| E2-H3-T1 | Pendiente | No existe endpoint de aprobacion por detalle | Ajuste y recalculo pendientes |
| E2-H3-T2 | Pendiente | No existe integracion de notificacion por correo | Notificacion pendiente |
| E2-H3-T3 | Pendiente | No hay pruebas de H3 | Suite pendiente |

### E2-H4 - Validar reposicion

| ID tarea | Estado | Evidencia | Nota |
| --- | --- | --- | --- |
| E2-H4-T1 | Pendiente | No existe endpoint POST /{id}/validate en revision | Cambio de estado pendiente |
| E2-H4-T2 | Pendiente | No hay verificacion de punto de emision CIF/SIF | Integracion pendiente |
| E2-H4-T3 | Pendiente | No hay pruebas de H4 | Suite pendiente |

### E2-H5 - Anular reposicion desde contabilidad

| ID tarea | Estado | Evidencia | Nota |
| --- | --- | --- | --- |
| E2-H5-T1 | Pendiente | No existe endpoint POST /{id}/cancel en revision | Flujo pendiente |
| E2-H5-T2 | Pendiente | No hay pruebas de H5 | Suite pendiente |

### E2-H6 - Tarea programada VALIDATED -> PAID/ISSUED

| ID tarea | Estado | Evidencia | Nota |
| --- | --- | --- | --- |
| E2-H6-T1 | Pendiente | No existe scheduler/job de transicion de estados en codigo actual | Implementacion pendiente |
| E2-H6-T2 | Pendiente | No existe bitacora tecnica/monitoreo del job | Observabilidad pendiente |
| E2-H6-T3 | Pendiente | No hay pruebas de integracion del job | Suite pendiente |

## Observaciones de control para cierre de Fase 1

- En el codigo actual solo existen endpoints de E1-H2, E1-H3 y E1-H4 dentro de ReplenishmentController.
- No se encontraron controllers ni servicios para Replenishment Review (E2).
- No se encontraron endpoints de detalle, validaciones H6, retenciones H7, envio H8 ni anulacion/impresion H9.
- Existe evidencia de calidad tecnica reciente (PMD y Sonar en verde) en la bitacora.

Este checklist queda como base de priorizacion para la siguiente fase (Gap Analysis Legacy vs nuevos endpoints).
