# Checklist de Estado Actual - Jira Backend Reposiciones

Fecha de corte: 2026-07-13

## Fuentes base analizadas

- docs/jira/JIRA-Backend-Reposiciones-Epicas.md (version recalibrada).
- Estado real del codigo en gtf-replacements-root.
- Legacy de administracion/revision en gtf-root.
- Trazabilidad operativa en BITACORA-PROMPTS.md.

## Alcance de este checklist

- Backend de Administracion de Reposiciones (E1).
- Backend de Revision de Reposiciones (E2).

## Criterio de clasificacion

- Completada: existe implementacion y evidencia tecnica verificable.
- Parcial: implementacion avanzada, pero faltan endpoints/reglas/pruebas para cerrar la tarea al 100%.
- Pendiente: no existe implementacion funcional en el repositorio migrado.

## Resumen general

- Total de tareas Jira: 56
- Completadas: 23
- Parciales: 2
- Pendientes: 31

---

## E1 - Administracion de Reposiciones

### E1-H0 - Configuracion base del proyecto y despliegue

| ID tarea | Estado     | Evidencia                                                    | Nota                         |
| -------- | ---------- | ------------------------------------------------------------ | ---------------------------- |
| E1-H0-T1 | Completada | application\*.yaml con context-path /gtfReplacementsServices | Perfiles y contexto activos  |
| E1-H0-T2 | Completada | application\*.yaml con DB2 jt400 + Hikari                    | Conexion parametrizada       |
| E1-H0-T3 | Completada | Configuracion Keycloak por perfil                            | Operativo en alcance backend |
| E1-H0-T4 | Completada | Jenkinsfile + Dockerfile + ci/helm                           | Build/deploy definidos       |
| E1-H0-T5 | Completada | Bitacora y ejecuciones Sonar/PMD                             | Quality gates en verde       |

### E1-H1 - Modelo de datos y persistencia

| ID tarea | Estado     | Evidencia                                        | Nota                       |
| -------- | ---------- | ------------------------------------------------ | -------------------------- |
| E1-H1-T1 | Completada | Entidades de reposicion y detalle en client/core | Mapeo contra tablas legacy |
| E1-H1-T2 | Completada | Repositorios QueryDSL de reposicion/detalle      | Filtro y consultas activas |
| E1-H1-T3 | Completada | ReplenishmentVo, ReplenishmentDetailVo, enums    | Contrato de VO operativo   |
| E1-H1-T4 | Completada | IReplenishmentService + ReplenishmentService     | Servicio base implementado |

### E1-H2 - Consultar cabecera del fondo del local

| ID tarea | Estado     | Evidencia                                         | Nota                             |
| -------- | ---------- | ------------------------------------------------- | -------------------------------- |
| E1-H2-T1 | Completada | ReplenishmentService.findCurrentFundHeader        | Calculo de cabecera implementado |
| E1-H2-T2 | Completada | GET /api/v1/replenishment-management/current-fund | Endpoint expuesto                |
| E1-H2-T3 | Completada | Tests de service/controller                       | Cobertura vigente                |

### E1-H3 - Listar y buscar reposiciones

| ID tarea | Estado     | Evidencia                                | Nota                          |
| -------- | ---------- | ---------------------------------------- | ----------------------------- |
| E1-H3-T1 | Completada | ReplenishmentRepository.findByFilter     | Filtro + paginacion activos   |
| E1-H3-T2 | Completada | POST /api/v1/replenishments/findByFilter | Endpoint disponible           |
| E1-H3-T3 | Completada | Tests de repositorio/controller          | Cobertura de filtros y mapeos |

### E1-H4 - Crear y editar reposicion (cabecera)

| ID tarea | Estado     | Evidencia                                                                    | Nota                                                 |
| -------- | ---------- | ---------------------------------------------------------------------------- | ---------------------------------------------------- |
| E1-H4-T1 | Completada | ReplenishmentService.create + ReplenishmentHeaderValidator.validateForCreate | Regla pendiente por transaccion alineada a legacy    |
| E1-H4-T2 | Completada | GET /api/v1/replenishments/{replenishmentId}                                 | Consulta por id implementada                         |
| E1-H4-T3 | Completada | POST /api/v1/replenishment-management/{replenishmentId}                      | Update en pendiente + limpieza de totales de request |
| E1-H4-T4 | Completada | ReplenishmentServiceTest + ReplenishmentControllerTest                       | Casos de create/read/update en verde                 |

### E1-H5 - Gestion del detalle de documentos

| ID tarea | Estado     | Evidencia                                                     | Nota                                      |
| -------- | ---------- | ------------------------------------------------------------- | ----------------------------------------- |
| E1-H5-T1 | Completada | POST /api/v1/replenishment-management/{id}/details            | Crear detalle implementado                |
| E1-H5-T2 | Completada | POST /api/v1/replenishment-management/{id}/details/{detailId} | Editar detalle implementado               |
| E1-H5-T3 | Pendiente  | No existe endpoint delete detalle                             | Falta /details/{detailId}/delete          |
| E1-H5-T4 | Pendiente  | No existen endpoints de catalogos de detalle                  | Falta /document-types y /billing-concepts |
| E1-H5-T5 | Parcial    | Existen pruebas create/update/findDetails                     | Faltan pruebas de delete y catalogos      |

### E1-H6 - Validaciones de negocio del ingreso

| ID tarea | Estado     | Evidencia                                                           | Nota                                        |
| -------- | ---------- | ------------------------------------------------------------------- | ------------------------------------------- |
| E1-H6-T1 | Completada | POST /api/v1/replenishments/validate-vat                            | IVA validado con maxAllowedVat              |
| E1-H6-T2 | Completada | POST /api/v1/replenishments/validate-duplicate                      | Duplicado por taxId + documentNumber        |
| E1-H6-T3 | Pendiente  | No se encontro regla de valor maximo con excepciones parametrizadas | Brecha funcional legacy                     |
| E1-H6-T4 | Parcial    | Tests para IVA y duplicado en verde                                 | Falta cobertura de valor maximo/excepciones |

### E1-H7 - Retenciones en el ingreso

| ID tarea | Estado    | Evidencia                              | Nota                    |
| -------- | --------- | -------------------------------------- | ----------------------- |
| E1-H7-T1 | Pendiente | Sin implementacion en API migrada      | Spike SRI sigue abierto |
| E1-H7-T2 | Pendiente | Sin endpoint de retencion electronica  | Flujo pendiente         |
| E1-H7-T3 | Pendiente | Sin endpoint de carga XML de retencion | Flujo pendiente         |
| E1-H7-T4 | Pendiente | Sin endpoint de retencion manual       | Flujo pendiente         |
| E1-H7-T5 | Pendiente | Sin suite de pruebas                   | Cobertura pendiente     |

### E1-H8 - Enviar reposicion a contabilidad

| ID tarea | Estado    | Evidencia                                           | Nota                                 |
| -------- | --------- | --------------------------------------------------- | ------------------------------------ |
| E1-H8-T1 | Pendiente | Sin GET /api/v1/replenishments/responsibles         | Consulta de responsables pendiente   |
| E1-H8-T2 | Pendiente | Sin POST /api/v1/replenishment-management/{id}/send | Cambio de estado a enviado pendiente |
| E1-H8-T3 | Pendiente | Sin pruebas de envio                                | Cobertura pendiente                  |

### E1-H9 - Anular e imprimir reposicion

| ID tarea | Estado    | Evidencia                                             | Nota                         |
| -------- | --------- | ----------------------------------------------------- | ---------------------------- |
| E1-H9-T1 | Pendiente | Sin POST /api/v1/replenishment-management/{id}/cancel | Flujo de anulacion pendiente |
| E1-H9-T2 | Pendiente | Sin GET /api/v1/replenishment-management/{id}/print   | Flujo de impresion pendiente |
| E1-H9-T3 | Pendiente | Sin pruebas de anulacion/impresion                    | Cobertura pendiente          |

---

## E2 - Revision de Reposiciones

### E2-H1 - Listar reposiciones enviadas

| ID tarea | Estado    | Evidencia                                            | Nota                           |
| -------- | --------- | ---------------------------------------------------- | ------------------------------ |
| E2-H1-T1 | Pendiente | No existe modulo review en backend migrado           | Repositorio/servicio pendiente |
| E2-H1-T2 | Pendiente | No existe /api/v1/replenishment-reviews/findByFilter | Endpoint pendiente             |
| E2-H1-T3 | Pendiente | Sin pruebas de H1                                    | Cobertura pendiente            |

### E2-H2 - Ver reposicion para revision

| ID tarea | Estado    | Evidencia                                  | Nota                |
| -------- | --------- | ------------------------------------------ | ------------------- |
| E2-H2-T1 | Pendiente | Sin servicio review solicitado vs aprobado | Logica pendiente    |
| E2-H2-T2 | Pendiente | Sin GET /api/v1/replenishment-reviews/{id} | Endpoint pendiente  |
| E2-H2-T3 | Pendiente | Sin pruebas de H2                          | Cobertura pendiente |

### E2-H3 - Ajustar valor aprobado

| ID tarea | Estado    | Evidencia                          | Nota                  |
| -------- | --------- | ---------------------------------- | --------------------- |
| E2-H3-T1 | Pendiente | Sin endpoint de ajuste por detalle | Recalculo pendiente   |
| E2-H3-T2 | Pendiente | Sin notificacion integrada         | Integracion pendiente |
| E2-H3-T3 | Pendiente | Sin pruebas de H3                  | Cobertura pendiente   |

### E2-H4 - Validar reposicion

| ID tarea | Estado    | Evidencia                                            | Nota                       |
| -------- | --------- | ---------------------------------------------------- | -------------------------- |
| E2-H4-T1 | Pendiente | Sin POST /api/v1/replenishment-reviews/{id}/validate | Cambio de estado pendiente |
| E2-H4-T2 | Pendiente | Sin integracion de validaciones externas             | Regla de negocio pendiente |
| E2-H4-T3 | Pendiente | Sin pruebas de H4                                    | Cobertura pendiente        |

### E2-H5 - Anular reposicion desde contabilidad

| ID tarea | Estado    | Evidencia                                          | Nota                |
| -------- | --------- | -------------------------------------------------- | ------------------- |
| E2-H5-T1 | Pendiente | Sin POST /api/v1/replenishment-reviews/{id}/cancel | Flujo pendiente     |
| E2-H5-T2 | Pendiente | Sin pruebas de H5                                  | Cobertura pendiente |

### E2-H6 - Tarea programada VALIDATED -> PAID/ISSUED

| ID tarea | Estado    | Evidencia                            | Nota                     |
| -------- | --------- | ------------------------------------ | ------------------------ |
| E2-H6-T1 | Pendiente | Sin scheduler/job en backend migrado | Implementacion pendiente |
| E2-H6-T2 | Pendiente | Sin bitacora/observabilidad del job  | Monitoreo pendiente      |
| E2-H6-T3 | Pendiente | Sin pruebas de integracion del job   | Cobertura pendiente      |

---

## Observaciones de control para cierre de fase

1. El modulo de Administracion tiene avance real en H2, H3, H4, y parte de H5/H6.
2. El modulo de Revision (E2) no tiene implementacion aun en el backend migrado.
3. El ajuste de historias/tareas ya incorpora reglas legacy para evitar nuevas desalineaciones de contrato.
