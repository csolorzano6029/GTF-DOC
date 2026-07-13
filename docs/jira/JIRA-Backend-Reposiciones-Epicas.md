# Jira - Backend Reposiciones (Epicas, Historias y Tareas)

Version ajustada con enfoque legacy-first y trazabilidad real contra codigo implementado.

- Fecha de corte: 2026-07-13
- Repositorio objetivo: gtf-replacements-root
- Base API: /gtfReplacementsServices/api/v1

---

## 1) Fuentes de verdad usadas para este ajuste

- Codigo backend migrado:
  - gtf-replacements-services/src/main/java/ec/com/smx/gtf/replacements/controller/replenishment/ReplenishmentController.java
  - gtf-replacements-core/src/main/java/ec/com/smx/gtf/replacements/replenishment/service/ReplenishmentService.java
  - gtf-replacements-core/src/main/java/ec/com/smx/gtf/replacements/replenishment/service/ReplenishmentHeaderValidator.java
  - tests de servicio/controller en core y services.
- Legacy funcional:
  - SOPORTES/gtf-root/gtf-web-intranet/src/main/java/ec/com/smx/gtf/web/administracion/controller/AdminRembolsoController.java
  - SOPORTES/gtf-root/gtf-web-intranet/src/main/java/ec/com/smx/gtf/web/administracion/controller/RevisionReposicionController.java
  - pantallas JSF de administracion y revision de reposiciones.
- Contratos documentados:
  - docs/apis/Backend-Reposiciones-Endpoints-Implementados.md
  - docs/apis/BACK-FRONT.md

---

## 2) Estado real de endpoints (hoy)

### 2.1 Administracion - base /replenishments

| Metodo | Ruta                            | Historia | Estado real  |
| ------ | ------------------------------- | -------- | ------------ |
| GET    | /current-fund                   | E1-H2    | Implementado |
| POST   | /findByFilter                   | E1-H3    | Implementado |
| POST   | /validate-vat                   | E1-H6    | Implementado |
| POST   | /validate-duplicate             | E1-H6    | Implementado |
| POST   | /{id}/details                   | E1-H5    | Implementado |
| POST   | /{id}/details/{detailId}        | E1-H5    | Implementado |
| GET    | /{id}/details                   | E1-H5    | Implementado |
| POST   | /                               | E1-H4    | Implementado |
| POST   | /{id}                           | E1-H4    | Implementado |
| GET    | /{id}                           | E1-H4    | Implementado |
| POST   | /{id}/details/{detailId}/delete | E1-H5    | Pendiente    |
| GET    | /document-types                 | E1-H5    | Pendiente    |
| GET    | /billing-concepts               | E1-H5    | Pendiente    |
| GET    | /responsibles                   | E1-H8    | Pendiente    |
| POST   | /{id}/send                      | E1-H8    | Pendiente    |
| POST   | /{id}/cancel                    | E1-H9    | Pendiente    |
| GET    | /{id}/print                     | E1-H9    | Pendiente    |

### 2.2 Revision - base /replenishment-reviews

No existe controller ni servicio implementado para la epica E2 en el repositorio migrado.

---

## 3) Reglas legacy obligatorias (ya confirmadas)

Estas reglas deben quedar en criterios de aceptacion y Definition of Done de Jira:

1. Crear cabecera (POST /replenishments) bloquea pendiente solo para transactionCode=1 (Caja Chica).
2. Si transactionCode=2 (Gastos Personales), el create no se bloquea por pendiente existente.
3. Totales de cabecera enviados manualmente por front no se deben persistir en create/update:
   - requestedTotal, requestedSubtotal, vatTotal, approvedTotal, approvedVatTotal.
4. Update de cabecera solo en estado pendiente (PENDING/PEN).
5. Create/update detalle solo en estado pendiente (PENDING/PEN).
6. Validacion de duplicado se basa en taxId + documentNumber.
7. Validacion IVA debe devolver valid y maxAllowedVat para guiar correccion de front.
8. En flujo de envio legacy (E1-H8) para Caja Chica con pago por ventanilla, se requiere responsable principal seleccionado.
9. En revision legacy (E2), validar/anular se apoya en reglas de observacion, funcionario principal/beneficiario y estado de detalle.

---

## 4) Epica E1 - Administracion de Reposiciones

Objetivo: registrar, editar, validar y enviar reposiciones desde local/departamento con paridad legacy.

### E1-H0 - Configuracion base del proyecto y despliegue

Estado historia: Completada

| ID tarea | Estado     | Ajuste aplicado                                       |
| -------- | ---------- | ----------------------------------------------------- |
| E1-H0-T1 | Completada | Context-path y perfiles de entorno activos.           |
| E1-H0-T2 | Completada | Conexion DB2 parametrizada con jt400/Hikari.          |
| E1-H0-T3 | Completada | Configuracion Keycloak por perfil en alcance backend. |
| E1-H0-T4 | Completada | Jenkinsfile + Dockerfile + helm operativos.           |
| E1-H0-T5 | Completada | Sonar/PMD en verde en bitacora del proyecto.          |

### E1-H1 - Modelo de datos y persistencia

Estado historia: Completada

| ID tarea | Estado     | Ajuste aplicado                                             |
| -------- | ---------- | ----------------------------------------------------------- |
| E1-H1-T1 | Completada | Entidades de cabecera/detalle y llaves compuestas mapeadas. |
| E1-H1-T2 | Completada | Repositorios QueryDSL para filtros, consulta y existencias. |
| E1-H1-T3 | Completada | VOs de negocio y enums operativos.                          |
| E1-H1-T4 | Completada | Servicio base y wiring implementado.                        |

### E1-H2 - Consultar cabecera de fondo

Estado historia: Completada

Contrato real:

- Endpoint: GET /replenishments/current-fund
- Query requerida: workAreaCode
- Salida clave: workAreaCode, workAreaName, assignedFund, cashBalance, pendingPaymentAmount, usagePercentage, alertPercentage, maxDocumentValue, currentStatus.

| ID tarea | Estado     | Ajuste aplicado                                  |
| -------- | ---------- | ------------------------------------------------ |
| E1-H2-T1 | Completada | Calculo de cabecera desde repositorio de fondos. |
| E1-H2-T2 | Completada | Endpoint y contrato expuesto en controller.      |
| E1-H2-T3 | Completada | Cobertura en tests de servicio y controller.     |

### E1-H3 - Listar y buscar reposiciones

Estado historia: Completada

Contrato real:

- Endpoint: POST /replenishments/findByFilter
- Request: FilterVo con filters/page/size (SearchModelDTO), no payload simplificado field/operator.
- Response: Page de ReplenishmentVo en envelope BaseResponseVo.

| ID tarea | Estado     | Ajuste aplicado                                         |
| -------- | ---------- | ------------------------------------------------------- |
| E1-H3-T1 | Completada | Filtro/paginacion implementados en repositorio.         |
| E1-H3-T2 | Completada | Endpoint expuesto y validado con front contract actual. |
| E1-H3-T3 | Completada | Pruebas de filtros y mapeos de estado/tipo ejecutadas.  |

### E1-H4 - Crear y editar cabecera

Estado historia: Completada

Criterios ajustados (legacy + codigo actual):

1. Create exige workAreaCode, observation y responsable (ID o documento).
2. transactionCode permitido: 1 o 2; default 1.
3. Si transactionCode=1 y existe pendiente por local, create se bloquea.
4. Si transactionCode=2, create no se bloquea por pendiente.
5. Si transactionCode=2, beneficiaryPersonId es obligatorio.
6. Update solo en estado pendiente.
7. Totales de cabecera enviados por request se ignoran en create/update.

| ID tarea | Estado     | Ajuste aplicado                                                         |
| -------- | ---------- | ----------------------------------------------------------------------- |
| E1-H4-T1 | Completada | Create alineado a regla de pendiente por tipo de transaccion.           |
| E1-H4-T2 | Completada | Consulta por ID implementada con campos de cabecera enriquecidos.       |
| E1-H4-T3 | Completada | Update en pendiente con preservacion de datos y saneamiento de totales. |
| E1-H4-T4 | Completada | Tests de create/read/update en verde.                                   |

### E1-H5 - Gestion del detalle de documentos

Estado historia: Parcial

Criterios vigentes implementados:

1. Create detalle en pendiente con obligatorios: billingConceptSequence, documentType, documentDate, requestedValue, vatValue.
2. Update detalle en pendiente con merge parcial y control de detalle existente.
3. Duplicado por taxId + documentNumber en create/update cuando ambos campos llegan informados.
4. Recalculo de totales de cabecera despues de crear/editar detalle.
5. Consulta de detalle por reposicion implementada (GET /{id}/details).

Pendientes para cerrar la historia segun plan original:

1. Eliminacion de detalle.
2. Catalogos document-types y billing-concepts por endpoint dedicado.

| ID tarea | Estado     | Ajuste aplicado                                                           |
| -------- | ---------- | ------------------------------------------------------------------------- |
| E1-H5-T1 | Completada | Crear detalle implementado.                                               |
| E1-H5-T2 | Completada | Editar detalle implementado.                                              |
| E1-H5-T3 | Pendiente  | Eliminar detalle no implementado aun.                                     |
| E1-H5-T4 | Pendiente  | Endpoints de catalogos no implementados aun.                              |
| E1-H5-T5 | Parcial    | Hay tests de create/update/findDetails; faltan tests de delete/catalogos. |

### E1-H6 - Validaciones de negocio del ingreso

Estado historia: Parcial

Implementado:

1. validate-vat con reglas de negativo, total requerido, IVA >= total, IVA > maxAllowedVat.
2. validate-duplicate por taxId + documentNumber.

Pendiente:

1. Regla de valor maximo por documento con excepciones parametrizadas (legacy).

| ID tarea | Estado     | Ajuste aplicado                                                               |
| -------- | ---------- | ----------------------------------------------------------------------------- |
| E1-H6-T1 | Completada | Endpoint y logica IVA implementados.                                          |
| E1-H6-T2 | Completada | Endpoint y logica de duplicado implementados.                                 |
| E1-H6-T3 | Pendiente  | Tope maximo + conceptos exceptuados no implementado aun en API.               |
| E1-H6-T4 | Parcial    | Tests existentes para IVA/duplicado; falta cobertura de regla de tope maximo. |

### E1-H7 - Retenciones en ingreso

Estado historia: Pendiente

Ajuste de definicion para Jira:

1. Mantener dependencia externa SRI y decision funcional con negocio.
2. Separar subtareas de retencion electronica, manual y carga XML.
3. Incluir explicitamente mapeo de campos de retencion segun flujos legacy de administracion/revision.

| ID tarea | Estado    | Ajuste aplicado                                        |
| -------- | --------- | ------------------------------------------------------ |
| E1-H7-T1 | Pendiente | Spike funcional/tecnico con reglas definitivas de SRI. |
| E1-H7-T2 | Pendiente | Flujo retencion electronica pendiente.                 |
| E1-H7-T3 | Pendiente | Carga/validacion XML pendiente.                        |
| E1-H7-T4 | Pendiente | Flujo retencion manual pendiente.                      |
| E1-H7-T5 | Pendiente | Suite de pruebas pendiente.                            |

### E1-H8 - Enviar reposicion a contabilidad

Estado historia: Pendiente

Ajuste de criterios con legacy:

1. Endpoint de responsables debe recuperar candidatos por area de trabajo.
2. Para Caja Chica + pago por ventanilla, envio requiere funcionario principal seleccionado.
3. Al enviar, estado pasa a enviado y alerta de envio se inactiva.

| ID tarea | Estado    | Ajuste aplicado                                    |
| -------- | --------- | -------------------------------------------------- |
| E1-H8-T1 | Pendiente | GET /responsibles pendiente.                       |
| E1-H8-T2 | Pendiente | POST /{id}/send pendiente con validaciones legacy. |
| E1-H8-T3 | Pendiente | Pruebas de envio/errores pendientes.               |

### E1-H9 - Anular e imprimir reposicion

Estado historia: Pendiente

Ajuste de criterios con legacy:

1. Anulacion requiere observacion de validacion.
2. Impresion genera comprobante PDF de reposicion.

| ID tarea | Estado    | Ajuste aplicado                            |
| -------- | --------- | ------------------------------------------ |
| E1-H9-T1 | Pendiente | POST /{id}/cancel pendiente.               |
| E1-H9-T2 | Pendiente | GET /{id}/print pendiente.                 |
| E1-H9-T3 | Pendiente | Pruebas de anulacion/impresion pendientes. |

---

## 5) Epica E2 - Revision de Reposiciones

Objetivo: revisar, ajustar, validar o anular solicitudes enviadas.

Estado epica: Pendiente (sin implementacion en gtf-replacements-root)

### E2-H1 - Listar reposiciones enviadas

Estado historia: Pendiente

| ID tarea | Estado    | Ajuste aplicado                                         |
| -------- | --------- | ------------------------------------------------------- |
| E2-H1-T1 | Pendiente | Repositorio/servicio de revision pendiente.             |
| E2-H1-T2 | Pendiente | Endpoint /replenishment-reviews/findByFilter pendiente. |
| E2-H1-T3 | Pendiente | Suite de pruebas pendiente.                             |

### E2-H2 - Ver reposicion para revision

Estado historia: Pendiente

| ID tarea | Estado    | Ajuste aplicado                                     |
| -------- | --------- | --------------------------------------------------- |
| E2-H2-T1 | Pendiente | Logica solicitado vs aprobado pendiente.            |
| E2-H2-T2 | Pendiente | Endpoint GET /replenishment-reviews/{id} pendiente. |
| E2-H2-T3 | Pendiente | Suite de pruebas pendiente.                         |

### E2-H3 - Ajustar valor aprobado

Estado historia: Pendiente

Ajuste con legacy:

1. Si cambia valor aprobado, exigir observacion de validacion en escenarios de rechazo/ajuste.
2. Recalcular subtotal/IVA/total aprobado.

| ID tarea | Estado    | Ajuste aplicado                                 |
| -------- | --------- | ----------------------------------------------- |
| E2-H3-T1 | Pendiente | Endpoint de ajuste por detalle pendiente.       |
| E2-H3-T2 | Pendiente | Notificacion (si aplica por negocio) pendiente. |
| E2-H3-T3 | Pendiente | Pruebas de recalculo y validacion pendientes.   |

### E2-H4 - Validar reposicion

Estado historia: Pendiente

Ajuste con legacy:

1. Para Caja Chica se valida funcionario principal.
2. Para Gastos Personales se valida beneficiario.
3. Si falla validacion externa de facturacion, retornar mensaje funcional de rechazo.

| ID tarea | Estado    | Ajuste aplicado                                 |
| -------- | --------- | ----------------------------------------------- |
| E2-H4-T1 | Pendiente | POST /{id}/validate pendiente.                  |
| E2-H4-T2 | Pendiente | Integraciones de validacion externa pendientes. |
| E2-H4-T3 | Pendiente | Pruebas de validacion pendiente.                |

### E2-H5 - Anular reposicion desde contabilidad

Estado historia: Pendiente

| ID tarea | Estado    | Ajuste aplicado                                 |
| -------- | --------- | ----------------------------------------------- |
| E2-H5-T1 | Pendiente | POST /{id}/cancel pendiente en modulo revision. |
| E2-H5-T2 | Pendiente | Pruebas de anulacion pendiente.                 |

### E2-H6 - Job VALIDATED -> PAID/ISSUED

Estado historia: Pendiente

| ID tarea | Estado    | Ajuste aplicado                               |
| -------- | --------- | --------------------------------------------- |
| E2-H6-T1 | Pendiente | Scheduler de transicion de estados pendiente. |
| E2-H6-T2 | Pendiente | Bitacora/observabilidad del job pendiente.    |
| E2-H6-T3 | Pendiente | Pruebas de integracion del job pendientes.    |

---

## 6) Contrato de campos para Jira (evitar nuevas desalineaciones)

### 6.1 Headers (create/update)

Request recomendado para create/update:

- workAreaCode (obligatorio)
- observation (obligatorio)
- transactionCode (opcional, default 1, valores permitidos 1/2)
- responsiblePersonId o responsiblePersonDocument (obligatorio para create)
- beneficiaryPersonId (obligatorio cuando transactionCode=2)
- checkResponsiblePersonId (opcional)
- startDate/endDate (opcional, startDate <= endDate)
- sendAlert (opcional)

No enviar en cabecera:

- companyCode
- replenishmentStatus
- requestedTotal/requestedSubtotal/vatTotal/approvedTotal/approvedVatTotal

### 6.2 Details (create/update)

Request minimo requerido:

- billingConceptSequence
- documentType
- documentDate
- requestedValue
- vatValue

Request opcional:

- taxId, documentNumber, observation, saleReceipt, accessKey, quantity, measurementTypeCode,
  measurementValueCode, fileName.

No enviar en detalle:

- companyCode
- detailId en create

### 6.3 Validaciones

- validate-vat:
  - request: documentTotal, vatValue
  - response clave: valid, maxAllowedVat
- validate-duplicate:
  - request: taxId, documentNumber
  - response clave: duplicate, valid

---

## 7) Orden de ejecucion recomendado (proxima iteracion)

1. Cerrar E1-H5 (delete + catalogos + pruebas faltantes).
2. Cerrar E1-H6-T3 (tope maximo + excepciones por parametros + pruebas).
3. Implementar E1-H8 (responsibles y send) con reglas legacy de ventanilla.
4. Implementar E1-H9 (cancel + print).
5. Iniciar E2 por H1/H2/H3, luego H4/H5 y finalmente H6.
