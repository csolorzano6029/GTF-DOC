# Jira — Backend Reposiciones (Épicas, Historias y Tareas)

> Desglose para registrar en Jira la migración **backend** del proyecto de Reposiciones (`gtf-replacements-root`).
> **2 épicas:** (1) Administración de Reposiciones y (2) Revisión de Reposiciones.
> Cada épica → historias de usuario → tareas, con criterios de aceptación, endpoints (payload/response), CRUD, configuración y despliegue.

- **Repo:** `gtf-replacements-root` · **Base API:** `/gtfReplacementsServices/api/v1`
- **Stack:** Java 25 · Spring Boot 4 · Hibernate 7 · QueryDSL · Keycloak SSO · DB2 for i (driver `jt400`) · Kubernetes.
- **Fecha:** 2026-07-06

---

## Convenciones comunes (aplican a todas las historias)

- **Envelope de respuesta** `BaseResponseVo<T>`: `{ "code": <int>, "message": <string>, "data": <T> }` (éxito `code=0`; `data` se omite si es null).
- **Auth:** token Bearer Keycloak (realm `CFAVORITA-SSO-INTRANET`). `companyCode` se toma de `SecurityKeycloakUtil.getCurrentUserLogin()`.
- **Paginado:** `POST .../findByFilter` con `FilterVo { filters:[...], page, size }`; `data` = `Page<T>`.
- **Estados de reposición:** `PENDING → SENT → VALIDATED → PAID → ISSUED → COLLECTED` (+ `CANCELLED`).
- **Inyección:** por constructor, campo `final`; tests de controller con `MockitoExtension` + `standaloneSetup` (no `MockMvcControllerBaseTest`).
- **Definition of Done (global):** código en inglés (comentarios en español), sin variables de 1 letra; PMD nivel 2 limpio; tests unitarios + integración; Swagger anotado; build `gradlew build` OK con JDK 25.

### Nomenclatura de IDs (reemplazar por claves reales de Jira)
`E1` = Épica Administración · `E2` = Épica Revisión · `E1-H1` = historia · `E1-H1-T1` = tarea.

---

# ÉPICA E1 — Administración de Reposiciones

**Épica:** *Como usuario de local (caja), quiero registrar, editar, enviar, anular e imprimir mis reposiciones de caja chica, para solicitar el reembolso de gastos del local.*

**Módulo backend:** `replenishments` · **Base:** `/gtfReplacementsServices/api/v1/replenishments`

---

## E1-H0 — Configuración base del proyecto y despliegue *(fundacional, compartida con E2)*

**Historia:** *Como equipo, quiero el proyecto configurado y desplegable, para construir sobre una base estable.*

**Criterios de aceptación**
- `context-path = /gtfReplacementsServices` en todos los perfiles (`application*.yaml`).
- Datasource DB2 for i (`jt400`) parametrizado por perfil (DESARROLLO, PRUEBAS, CALIDAD, PRODUCCIÓN).
- Seguridad Keycloak (clientId por ambiente) operativa; endpoints protegidos con Bearer.
- Swagger disponible en `/gtfReplacementsServices/swagger-ui/index.html`; Actuator health en `/gtfReplacementsServices/actuator/health`.
- Pipeline Jenkins (`springBootServicePipelineJDK25`, namespace `gtf`) y `Dockerfile` (`eclipse-temurin:25-jdk`) construyen la imagen; despliegue en Kubernetes.

**Tareas**
- E1-H0-T1: Configurar perfiles y `context-path`; variables por ambiente.
- E1-H0-T2: Configurar datasource DB2/i (`jt400`) + pool Hikari; validar conexión.
- E1-H0-T3: Configurar clientId Keycloak por ambiente (solicitud KEY) y filtros de seguridad.
- E1-H0-T4: Revisar `Dockerfile`, `Jenkinsfile` y manifiestos K8s (namespace `gtf`, probes health).
- E1-H0-T5: Verificar SonarQube y quality gate (PMD nivel 2, cobertura JaCoCo).

**Configuración / Despliegue**
```yaml
server:
  port: 8080
  servlet:
    context-path: /gtfReplacementsServices
spring:
  application:
    name: gtf-replacements-services
```
Despliegue: imagen Docker → K8s (namespace `gtf`), profile activo por `SPRING_PROFILES_ACTIVE`.

---

## E1-H1 — Modelo de datos y persistencia de reposiciones

**Historia:** *Como desarrollador, quiero las entidades, repositorios y VOs de reposición y su detalle, para soportar las operaciones de negocio sobre las tablas existentes (sin cambios de esquema).*

**Criterios de aceptación**
- Entidades JPA mapeadas a las tablas legacy (`@Entity(name="...")`, `@Column`), sin alterar el esquema DB2/i.
- Repositorios `IQueryDslBaseRepository` con métodos de consulta/paginado.
- VOs de transporte (`ReplenishmentVo`, `ReplenishmentDetailVo`, `FundHeaderVo`, `ResponsibleVo`) con validaciones Jakarta.
- Mapeo VO↔Entity con `ProjectUtil.convert`.

**Tareas**
- E1-H1-T1: Entidades `ReplenishmentEntity`, `ReplenishmentDetailEntity` (+ IDs compuestos si aplica).
- E1-H1-T2: Repositorios QueryDSL (findByFilter, findByCode, exist).
- E1-H1-T3: VOs + enums `ReplenishmentStatus`, `DocumentType`.
- E1-H1-T4: Servicio base (`IReplenishmentService`/`ReplenishmentService`) y configuración de escaneo de entidades.

---

## E1-H2 — Consultar cabecera del fondo del local

**Historia:** *Como usuario de local, quiero ver el estado de mi fondo (asignado, saldo, pendientes), para saber cuánto puedo reponer.*

**Criterios de aceptación**
- Devuelve fondo asignado, saldo efectivo, reposiciones pendientes, % de uso/alerta y valor máximo.
- `workAreaCode` obligatorio; error controlado si el local no existe.

**API**
```
GET /gtfReplacementsServices/api/v1/replenishments/current-fund?workAreaCode=186
```
**Response**
```json
{
  "code": 0, "message": "OK",
  "data": {
    "workAreaCode": "186",
    "assignedFund": 250.00,
    "cashBalance": 142.60,
    "pendingPaymentAmount": 107.40,
    "usagePercentage": 60,
    "alertPercentage": 50,
    "maxDocumentValue": 50.00,
    "currentStatus": "PENDING"
  }
}
```

**Tareas:** E1-H2-T1 servicio de cálculo del fondo · E1-H2-T2 endpoint + Swagger · E1-H2-T3 tests.

---

## E1-H3 — Listar y buscar reposiciones

**Historia:** *Como usuario de local, quiero buscar mis reposiciones por estado y fecha, para consultarlas y abrirlas.*

**Criterios de aceptación**
- Paginado y filtrable por estado, fechas y local.
- Devuelve totales solicitado/aprobado y estado por fila.

**API**
```
POST /gtfReplacementsServices/api/v1/replenishments/findByFilter
```
**Payload**
```json
{ "filters": [ { "field": "status", "operator": "EQUALS", "value": "PENDING" } ], "page": 0, "size": 10 }
```
**Response**
```json
{
  "code": 0, "message": "OK",
  "data": {
    "content": [ { "replenishmentId": 4900123, "workAreaCode": "186", "status": "PENDING", "requestedTotal": 79.81, "approvedTotal": 0.00, "createdDate": 1751414400000 } ],
    "totalElements": 2, "totalPages": 1, "number": 0, "size": 10
  }
}
```

**Tareas:** E1-H3-T1 repositorio findByFilter · E1-H3-T2 endpoint · E1-H3-T3 tests.

---

## E1-H4 — Crear y editar reposición (CRUD cabecera)

**Historia:** *Como usuario de local, quiero crear y editar una reposición en estado pendiente, para ir registrando mis documentos.*

**Criterios de aceptación**
- No permite crear si ya existe una reposición `PENDING` (mensaje claro).
- Si existe `PAID/ISSUED`, indica el monto a reponer (para el modal del front).
- Solo se puede editar en estado `PENDING`.
- Al crear, estado inicial `PENDING`.

**API (CRUD)**
```
POST /gtfReplacementsServices/api/v1/replenishments          (crear)
GET  /gtfReplacementsServices/api/v1/replenishments/{id}      (leer)
PUT  /gtfReplacementsServices/api/v1/replenishments/{id}      (actualizar cabecera)
```
**Payload (POST)**
```json
{ "workAreaCode": "186", "identityCard": "1712345678", "description": "Reposición junio" }
```
**Response (GET {id})**
```json
{
  "code": 0, "message": "OK",
  "data": {
    "replenishmentId": 4900123, "workAreaCode": "186", "status": "PENDING",
    "requestedTotal": 79.81, "approvedTotal": 0.00,
    "details": [ { "detailId": 1, "documentType": "INVOICE", "documentNumber": "001-001-000012345", "total": 2.00 } ]
  }
}
```
**Response (regla pendiente existente)**
```json
{ "code": 409, "message": "Ya existe una reposición pendiente para el local.", "data": null }
```

**Tareas:** E1-H4-T1 crear (+regla pendiente) · E1-H4-T2 leer · E1-H4-T3 actualizar · E1-H4-T4 tests.

---

## E1-H5 — Gestión del detalle de documentos

**Historia:** *Como usuario de local, quiero agregar, editar y eliminar líneas de documento (factura, factura electrónica, nota de venta, recibo), para componer la reposición.*

**Criterios de aceptación**
- Tipos soportados: `INVOICE`, `ELECTRONIC_INVOICE`, `SALES_NOTE`, `MANUAL_RECEIPT` (retenciones en E1-H7).
- IVA solo aplica a facturas; nota de venta y recibo sin IVA.
- Concepto según catálogo (`LC`/`LO`).
- Recalcula el total solicitado de la reposición.

**API (CRUD detalle)**
```
POST   /gtfReplacementsServices/api/v1/replenishments/{id}/details
PUT    /gtfReplacementsServices/api/v1/replenishments/{id}/details/{detailId}
DELETE /gtfReplacementsServices/api/v1/replenishments/{id}/details/{detailId}
```
**Payload (POST factura)**
```json
{
  "documentType": "INVOICE", "documentDate": "2026-06-29", "ruc": "1790016919001",
  "documentNumber": "001-001-000012345", "billingConceptCode": "LC-EDIF",
  "observation": "Prueba 1", "vatValue": 0.26, "total": 2.00
}
```
**Response OK**
```json
{ "code": 0, "message": "OK", "data": { "detailId": 1, "requestedValue": 2.00 } }
```

### Catálogos (soporte del detalle)
```
GET /gtfReplacementsServices/api/v1/replenishments/document-types
GET /gtfReplacementsServices/api/v1/replenishments/billing-concepts?type=LC
```
**Response document-types**
```json
{ "code": 0, "message": "OK", "data": [ { "code": "INVOICE", "name": "Factura", "hasVat": true }, { "code": "SALES_NOTE", "name": "Nota de venta", "hasVat": false } ] }
```

**Tareas:** E1-H5-T1 agregar línea · E1-H5-T2 editar · E1-H5-T3 eliminar · E1-H5-T4 catálogos (document-types, billing-concepts desde SIF) · E1-H5-T5 tests.

---

## E1-H6 — Validaciones de negocio del ingreso

**Historia:** *Como negocio, quiero validar IVA, documentos duplicados y valor máximo, para evitar errores y fraudes.*

**Criterios de aceptación**
- **IVA:** el IVA ingresado debe ser ≤ `total − (total / 1.15)`; devuelve el máximo permitido.
- **Duplicado:** rechaza si (`companyCode` + `ruc` + `documentNumber`) ya fue cancelado; informa local y fecha.
- **Valor máximo:** rechaza si supera `maxDocumentValue`, salvo conceptos exceptuados (estudios de mercado, gas) definidos en la tabla de parámetros.

**APIs**
```
POST /gtfReplacementsServices/api/v1/replenishments/validate-vat
POST /gtfReplacementsServices/api/v1/replenishments/validate-duplicate
```
**Payload/Response validate-vat**
```json
// request
{ "total": 2.00, "vatValue": 0.30 }
// response
{ "code": 400, "message": "El IVA ingresado supera el máximo calculado.", "data": { "calculatedVat": 0.26, "maxAllowedVat": 0.26, "valid": false } }
```
**Response validate-duplicate**
```json
{ "code": 409, "message": "La factura ingresada se encuentra cancelada.", "data": { "duplicate": true, "replenishmentId": 4890010, "workAreaCode": "186", "cancelledDate": "2026-06-29" } }
```

**Tareas:** E1-H6-T1 validación IVA · E1-H6-T2 validación duplicado · E1-H6-T3 valor máximo + parámetros de excepción · E1-H6-T4 tests de reglas.

---

## E1-H7 — Retenciones en el ingreso (electrónica y manual)

**Historia:** *Como usuario de local, quiero registrar retenciones electrónicas y manuales dentro de la reposición, para incluir los comprobantes de retención.*

> ⚠️ **Dependencia SRI:** el SRI bloqueó el servicio web que devolvía los datos (ahora solo estado). Coordinar diseño con **Frank**. Esta historia va **al final** de la épica.

**Criterios de aceptación**
- Retención electrónica: buscar por RUC/doc; validar clave de acceso (solo estado: AUTHORIZED/NOT_AUTHORIZED/PROCESSING/CANCELLED); permitir subir XML.
- Retención manual: registrar RUC/cédula, total, impuesto al IVA e impuesto a la renta.

**APIs**
```
GET  /gtfReplacementsServices/api/v1/withholdings/electronic?ruc=&documentNumber=
POST /gtfReplacementsServices/api/v1/withholdings/electronic/access-key/validate
POST /gtfReplacementsServices/api/v1/withholdings/electronic/xml        (multipart)
POST /gtfReplacementsServices/api/v1/withholdings/manual
```
**Response electronic (validada)**
```json
{ "code": 0, "message": "OK", "data": { "ruc": "1790016919001", "documentNumber": "001-001-000000123", "sriStatus": "AUTHORIZED", "vatTax": 0.05, "incomeTax": 0.02 } }
```
**Payload manual**
```json
{ "ruc": "1790016919001", "documentNumber": "001-001-000000123", "vatTax": 0.05, "incomeTax": 0.02, "total": 0.07 }
```

**Tareas:** E1-H7-T1 (spike con Frank) alternativa al servicio SRI · E1-H7-T2 buscar/validar retención electrónica · E1-H7-T3 carga XML · E1-H7-T4 retención manual · E1-H7-T5 tests.

---

## E1-H8 — Enviar reposición a contabilidad

**Historia:** *Como usuario de local, quiero enviar la reposición a contabilidad, para que sea revisada.*

**Criterios de aceptación**
- Requiere responsable asignado (uno `PRINCIPAL`, varios `SECONDARY`); si no hay, error `422`.
- Cambia estado a `SENT`; no editable después.

**APIs**
```
GET  /gtfReplacementsServices/api/v1/replenishments/responsibles?workAreaCode=186
POST /gtfReplacementsServices/api/v1/replenishments/{id}/send
```
**Payload send**
```json
{ "responsiblePersonId": "1712345678" }
```
**Response sin responsable**
```json
{ "code": 422, "message": "No existe un responsable asignado para el local.", "data": null }
```

**Tareas:** E1-H8-T1 consulta responsables · E1-H8-T2 envío + validación · E1-H8-T3 tests.

---

## E1-H9 — Anular e imprimir reposición

**Historia:** *Como usuario de local, quiero anular una reposición e imprimir su comprobante, para corregir errores y tener soporte físico.*

**Criterios de aceptación**
- Anular requiere observación; refleja estado `CANCELLED`.
- Impresión genera PDF de la reposición.

**APIs**
```
POST /gtfReplacementsServices/api/v1/replenishments/{id}/cancel
GET  /gtfReplacementsServices/api/v1/replenishments/{id}/print
```
**Payload cancel**
```json
{ "observation": "Documento ingresado por error" }
```

**Tareas:** E1-H9-T1 anular · E1-H9-T2 impresión (reporte FOP) · E1-H9-T3 tests.

---

# ÉPICA E2 — Revisión de Reposiciones

**Épica:** *Como usuario de contabilidad, quiero revisar, ajustar, validar o anular las reposiciones enviadas por los locales, para aprobar el reembolso correcto.*

**Módulo backend:** `replenishment-reviews` · **Base:** `/gtfReplacementsServices/api/v1/replenishment-reviews`

---

## E2-H1 — Listar reposiciones enviadas

**Historia:** *Como contabilidad, quiero listar las reposiciones en estado enviado por local y fecha, para revisarlas.*

**Criterios de aceptación**
- Filtra por local, fecha y estado `SENT`; paginado.

**API**
```
POST /gtfReplacementsServices/api/v1/replenishment-reviews/findByFilter
```
**Payload**
```json
{ "filters": [ { "field": "status", "operator": "EQUALS", "value": "SENT" }, { "field": "workAreaCode", "operator": "EQUALS", "value": "186" } ], "page": 0, "size": 10 }
```

**Tareas:** E2-H1-T1 repositorio filtro · E2-H1-T2 endpoint · E2-H1-T3 tests.

---

## E2-H2 — Ver reposición para revisión (solicitado vs aprobado)

**Historia:** *Como contabilidad, quiero ver el detalle con lo solicitado y lo aprobado, para comparar contra los documentos físicos.*

**Criterios de aceptación**
- Cada línea muestra `requestedValue` y `approvedValue`; totales de ambos.

**API**
```
GET /gtfReplacementsServices/api/v1/replenishment-reviews/{id}
```
**Response**
```json
{
  "code": 0, "message": "OK",
  "data": {
    "replenishmentId": 4900123, "workAreaCode": "186", "status": "SENT",
    "requestedTotal": 2.00, "approvedTotal": 2.00,
    "details": [ { "detailId": 1, "documentType": "INVOICE", "documentNumber": "001-001-000012345", "requestedValue": 0.24, "approvedValue": 0.24 } ]
  }
}
```

**Tareas:** E2-H2-T1 servicio de revisión · E2-H2-T2 endpoint · E2-H2-T3 tests.

---

## E2-H3 — Ajustar valor aprobado

**Historia:** *Como contabilidad, quiero ajustar el valor aprobado de una línea con una observación, para corregir montos; el sistema notifica al local.*

**Criterios de aceptación**
- Actualiza `approvedValue` y observación; envía correo de alerta al administrador del local.
- Recalcula el total aprobado.

**API**
```
PUT /gtfReplacementsServices/api/v1/replenishment-reviews/{id}/details/{detailId}/approve
```
**Payload**
```json
{ "approvedValue": 0.14, "observation": "IVA ingresado incorrecto" }
```
**Response**
```json
{ "code": 0, "message": "Ajuste realizado. Se notificó al administrador del local.", "data": { "detailId": 1, "requestedValue": 0.24, "approvedValue": 0.14 } }
```

**Tareas:** E2-H3-T1 ajuste + recálculo · E2-H3-T2 notificación por correo · E2-H3-T3 tests.

---

## E2-H4 — Validar reposición

**Historia:** *Como contabilidad, quiero validar la reposición, para aprobarla y que continúe al pago.*

**Criterios de aceptación**
- Cambia estado a `VALIDATED`.
- Requiere **punto de emisión** configurado en el CIF; si falta, error controlado (probar con usuario Cintia/Cristina Escobar).

**API**
```
POST /gtfReplacementsServices/api/v1/replenishment-reviews/{id}/validate
```
**Response error (punto de emisión)**
```json
{ "code": 422, "message": "El usuario no tiene punto de emisión configurado.", "data": null }
```

**Tareas:** E2-H4-T1 validación + cambio de estado · E2-H4-T2 verificación de punto de emisión (integración SIF/CIF) · E2-H4-T3 tests.

---

## E2-H5 — Anular reposición desde contabilidad

**Historia:** *Como contabilidad, quiero anular una reposición con observación, para descartar solicitudes inválidas.*

**API**
```
POST /gtfReplacementsServices/api/v1/replenishment-reviews/{id}/cancel
```
**Payload**
```json
{ "observation": "Reposición inválida" }
```

**Tareas:** E2-H5-T1 anular · E2-H5-T2 tests.

---

## E2-H6 — Tarea programada: VALIDATED → PAID/ISSUED

**Historia:** *Como sistema, quiero un job programado que pase las reposiciones validadas a pago/emitido, para reemplazar el script manual del legacy.*

**Criterios de aceptación**
- Job periódico que transforma `VALIDATED → PAID/ISSUED`.
- Registra bitácora del proceso; idempotente ante reintentos.
- Reemplaza el script manual actual (bug del legacy).

**Configuración**
- Scheduler (cron) configurable por ambiente; feature flag para habilitar/deshabilitar.

**Tareas:** E2-H6-T1 job de transición de estados · E2-H6-T2 logging/monitoreo · E2-H6-T3 pruebas de integración del job.

---

## Resumen de endpoints

### E1 — Administración (`/gtfReplacementsServices/api/v1/replenishments`)
| Método | Ruta | Historia |
|--------|------|----------|
| GET | `/current-fund` | E1-H2 |
| POST | `/findByFilter` | E1-H3 |
| POST | `` | E1-H4 |
| GET | `/{id}` | E1-H4 |
| PUT | `/{id}` | E1-H4 |
| POST | `/{id}/details` | E1-H5 |
| PUT | `/{id}/details/{detailId}` | E1-H5 |
| DELETE | `/{id}/details/{detailId}` | E1-H5 |
| GET | `/document-types` | E1-H5 |
| GET | `/billing-concepts` | E1-H5 |
| POST | `/validate-vat` | E1-H6 |
| POST | `/validate-duplicate` | E1-H6 |
| GET | `/responsibles` | E1-H8 |
| POST | `/{id}/send` | E1-H8 |
| POST | `/{id}/cancel` | E1-H9 |
| GET | `/{id}/print` | E1-H9 |
| GET/POST | `/withholdings/electronic*`, `/withholdings/manual` | E1-H7 |

### E2 — Revisión (`/gtfReplacementsServices/api/v1/replenishment-reviews`)
| Método | Ruta | Historia |
|--------|------|----------|
| POST | `/findByFilter` | E2-H1 |
| GET | `/{id}` | E2-H2 |
| PUT | `/{id}/details/{detailId}/approve` | E2-H3 |
| POST | `/{id}/validate` | E2-H4 |
| POST | `/{id}/cancel` | E2-H5 |
| (job) | `VALIDATED → PAID/ISSUED` | E2-H6 |

---

## Orden sugerido de ejecución
1. **E1-H0** (config/despliegue) → **E1-H1** (modelo).
2. E1-H2, E1-H3, E1-H4 (fondo, listado, CRUD cabecera).
3. E1-H5, E1-H6 (detalle + validaciones).
4. E1-H8, E1-H9 (enviar, anular/imprimir).
5. E1-H7 (retenciones, depende de SRI — al final).
6. **E2** completa (revisión) una vez estabilizada la administración; E2-H6 (job) al cierre.

*Doc de apoyo para carga en Jira. Detalle técnico completo de payloads/reglas en `GTF2-Arquitectura-y-Mapeo-APIs.md`.*
