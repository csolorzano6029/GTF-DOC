# Backend Reposiciones — Arquitectura y Mapeo de APIs

> Diseño **backend** acotado a los **dos módulos** del repo `gtf-replacements-root`:
> **1) Administración de Reposiciones** (ingresa el local) y **2) Revisión de Reposiciones** (contabilidad).
> Documento de referencia técnica: convenciones, endpoints, payloads/responses, CRUDs, configuración y despliegue.

- **Repo backend:** `C:\Users\csolo\Documents\FAVORITA\MIGRACION\gtf-replacements-root`
- **Legacy (fuente):** `C:\Users\csolo\Documents\FAVORITA\SOPORTES\gtf-root` (`AdminRembolsoController`, `RevisionReposicionController`)
- **Base API:** `/gtfReplacementsServices/api/v1`
- **Fecha:** 2026-07-06

---

## 1. Stack y naming del repo

| Aspecto              | Valor                                                            |
| -------------------- | ---------------------------------------------------------------- |
| Lenguaje / framework | Java 25 · Spring Boot 4 · Hibernate 7 · QueryDSL                 |
| Módulos Gradle       | `gtf-replacements-{vo,client,core,services}`                     |
| rootProject.name     | `gtf-replacements-root`                                          |
| Paquete base         | `ec.com.smx.gtf.replacements`                                    |
| Clase main           | `GtfReplacementsSpringBootApplication`                           |
| groupId (Gradle)     | `ec.com.smx.gtf`                                                 |
| **context-path**     | **`/gtfReplacementsServices`**                                   |
| Base de datos        | DB2 for i (driver `jt400`) — **sin cambios de esquema**          |
| Seguridad            | Keycloak SSO (realm `CFAVORITA-SSO-INTRANET`)                    |
| Despliegue           | Docker (`eclipse-temurin:25-jdk`) → Kubernetes (namespace `gtf`) |

---

## 2. Arquitectura por capas

```
gtf-replacements-root
├── gtf-replacements-vo        → VOs/DTOs de transporte (Lombok + validación Jakarta)
├── gtf-replacements-client    → Entidades JPA, repositorios QueryDSL, interfaces de servicio, connectors
├── gtf-replacements-core      → Implementación de servicios (@Service), configuración, lógica de negocio
└── gtf-replacements-services  → Controllers REST (@RestController) + arranque Spring Boot
```

### Convenciones (framework Kruger v3)

| Elemento        | Base                                                                                                                                 |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| Controller      | `extends BaseController`, `@RestController`, `@RequestMapping("/api/v1/<recurso>")`, `@Tag` Swagger                                  |
| Respuesta       | `BaseResponseVo<Object>` → `{ code, message, data }`                                                                                 |
| Servicio        | `IBaseService`/`BaseService`, `@Transactional`                                                                                       |
| Repositorio     | `IQueryDslBaseRepository<Entity>`                                                                                                    |
| Entidad         | `extends AbstractBaseAuditableLockingIp<UserView, String>`                                                                           |
| VO              | `extends BaseAuditableVo` (`@Data @SuperBuilder`)                                                                                    |
| Filtro/paginado | `FilterVo { filters, page, size }` → `POST .../findByFilter`                                                                         |
| Seguridad       | `SecurityKeycloakUtil.getCurrentUserLogin()` → `companyCode`                                                                         |
| Mapeo VO↔Entity | `ProjectUtil.convert(...)`                                                                                                           |
| **Inyección**   | **por constructor, campo `final`**; tests de controller con `MockitoExtension` + `standaloneSetup` (sin `MockMvcControllerBaseTest`) |

### Envelope de respuesta

```json
{ "code": 0, "message": "OK", "data": {} }
```

`code=0` éxito · `>0` código de negocio · `data` omitido si null · paginado = `Page<T>` dentro de `data`.

---

## 3. Glosario (Español → Inglés) y estados

| Español                 | Inglés                                                                                                                          |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| Reposición (solicitud)  | `ReplenishmentRequest` / `replenishment`                                                                                        |
| Detalle de reposición   | `ReplenishmentDetail`                                                                                                           |
| Documento               | `documentType`: `INVOICE`, `ELECTRONIC_INVOICE`, `SALES_NOTE`, `MANUAL_RECEIPT`, `ELECTRONIC_WITHHOLDING`, `MANUAL_WITHHOLDING` |
| Fondo / local           | `fund` / `workArea`                                                                                                             |
| Responsable             | `responsible` (`PRINCIPAL` / `SECONDARY`)                                                                                       |
| Concepto de facturación | `billingConcept` (`LC` / `LO`)                                                                                                  |
| Revisión (contabilidad) | `replenishment-review`                                                                                                          |
| Solicitado / Aprobado   | `requestedValue` / `approvedValue`                                                                                              |

**Estados:** `PENDING → SENT → VALIDATED → PAID → ISSUED → COLLECTED` (+ `CANCELLED`).

---

# MÓDULO 1 — Administración de Reposiciones (local)

**Módulo backend:** `replenishment` · **Base:** `/gtfReplacementsServices/api/v1/replenishments`
**Origen legacy:** `AdminRembolsoController` + `AdminRembolsoDataManager` + `IReposicionServicio`.

## 1.1 CRUD y operaciones

| #   | Método | Ruta                              | Descripción                                              |
| --- | ------ | --------------------------------- | -------------------------------------------------------- |
| 1   | GET    | `/current-fund?workAreaCode=`     | Cabecera del fondo (asignado, saldo, pendientes).        |
| 2   | POST   | `/findByFilter`                   | Búsqueda paginada.                                       |
| 3   | GET    | `/{id}`                           | Ver reposición + detalle.                                |
| 4   | POST   | ``                                | Crear (estado `PENDING`).                                |
| 5   | POST   | `/{id}`                           | Actualizar cabecera.                                     |
| 6   | POST   | `/{id}/details`                   | Agregar línea de documento.                              |
| 7   | POST   | `/{id}/details/{detailId}`        | Editar línea.                                            |
| 8   | POST   | `/{id}/details/{detailId}/delete` | Eliminar línea.                                          |
| 9   | GET    | `/document-types`                 | Catálogo de tipos de documento.                          |
| 10  | GET    | `/billing-concepts?type=LC\|LO`   | Conceptos (consumidos del SIF).                          |
| 11  | POST   | `/validate-vat`                   | Validar/calcular IVA.                                    |
| 12  | POST   | `/validate-duplicate`             | Validar documento no duplicado.                          |
| 13  | GET    | `/responsibles?workAreaCode=`     | Responsables del local.                                  |
| 14  | POST   | `/{id}/send`                      | Enviar a contabilidad (`SENT`).                          |
| 15  | POST   | `/{id}/cancel`                    | Anular.                                                  |
| 16  | GET    | `/{id}/print`                     | PDF de la reposición.                                    |
| 17  | —      | `/withholdings/*`                 | Retenciones electrónica/manual (depende SRI — al final). |

## 1.2 Payloads / Responses

**GET `/current-fund?workAreaCode=186`**

```json
{
  "code": 0,
  "message": "OK",
  "data": {
    "workAreaCode": "186",
    "assignedFund": 250.0,
    "cashBalance": 142.6,
    "pendingPaymentAmount": 107.4,
    "usagePercentage": 60,
    "alertPercentage": 50,
    "maxDocumentValue": 50.0,
    "currentStatus": "PENDING"
  }
}
```

**POST `/findByFilter`**

```json
// request
{ "filters": [ { "field": "status", "operator": "EQUALS", "value": "PENDING" } ], "page": 0, "size": 10 }
// response
{ "code": 0, "message": "OK",
  "data": { "content": [ { "replenishmentId": 4900123, "workAreaCode": "186", "status": "PENDING",
            "requestedTotal": 79.81, "approvedTotal": 0.00, "createdDate": 1751414400000 } ],
            "totalElements": 2, "totalPages": 1, "number": 0, "size": 10 } }
```

**GET `/{id}`**

```json
{
  "code": 0,
  "message": "OK",
  "data": {
    "replenishmentId": 4900123,
    "workAreaCode": "186",
    "status": "PENDING",
    "requestedTotal": 79.81,
    "approvedTotal": 0.0,
    "details": [
      {
        "detailId": 1,
        "documentType": "INVOICE",
        "documentDate": "2026-06-29",
        "ruc": "1790016919001",
        "documentNumber": "001-001-000012345",
        "billingConceptCode": "LC-EDIF",
        "observation": "Prueba 1",
        "vatValue": 0.26,
        "total": 2.0,
        "requestedValue": 2.0,
        "approvedValue": 2.0
      }
    ]
  }
}
```

**POST `` (crear)**

```json
// request
{ "workAreaCode": "186", "identityCard": "1712345678", "description": "Reposición junio" }
// response error (ya existe pendiente)
{ "code": 409, "message": "Ya existe una reposición pendiente para el local.", "data": null }
```

**POST `/{id}/details` (factura)**

```json
// request
{ "documentType": "INVOICE", "documentDate": "2026-06-29", "ruc": "1790016919001",
  "documentNumber": "001-001-000012345", "billingConceptCode": "LC-EDIF",
  "observation": "Prueba 1", "vatValue": 0.26, "total": 2.00 }
// response
{ "code": 0, "message": "OK", "data": { "detailId": 1, "requestedValue": 2.00 } }
```

**POST `/validate-vat`**

```json
// request
{ "total": 2.00, "vatValue": 0.30 }
// response
{ "code": 400, "message": "El IVA ingresado supera el máximo calculado.",
  "data": { "total": 2.00, "calculatedVat": 0.26, "maxAllowedVat": 0.26, "valid": false } }
```

**POST `/validate-duplicate`**

```json
// request
{ "companyCode": 1, "ruc": "1790016919001", "documentNumber": "001-001-000012345" }
// response
{ "code": 409, "message": "La factura ingresada se encuentra cancelada.",
  "data": { "duplicate": true, "replenishmentId": 4890010, "workAreaCode": "186", "cancelledDate": "2026-06-29" } }
```

**GET `/document-types`**

```json
{
  "code": 0,
  "message": "OK",
  "data": [
    { "code": "INVOICE", "name": "Factura", "hasVat": true },
    { "code": "SALES_NOTE", "name": "Nota de venta", "hasVat": false },
    { "code": "MANUAL_RECEIPT", "name": "Recibo manual", "hasVat": false }
  ]
}
```

**GET `/responsibles?workAreaCode=186`**

```json
{
  "code": 0,
  "message": "OK",
  "data": [
    {
      "personId": "1712345678",
      "fullName": "Cintia Escobar",
      "type": "PRINCIPAL"
    },
    { "personId": "1798765432", "fullName": "Juan Pérez", "type": "SECONDARY" }
  ]
}
```

**POST `/{id}/send`**

```json
// request
{ "responsiblePersonId": "1712345678" }
// response error (sin responsable)
{ "code": 422, "message": "No existe un responsable asignado para el local.", "data": null }
```

**POST `/{id}/cancel`**

```json
{ "observation": "Documento ingresado por error" }
```

## 1.3 Reglas de negocio

1. **IVA:** ingresado ≤ `total − (total / 1.15)` (permite menor). Solo facturas tienen IVA.
2. **Documento duplicado:** rechazar `companyCode + ruc + documentNumber` ya cancelado.
3. **Valor máximo por documento** con excepciones (estudios de mercado, gas) vía tabla de parámetros.
4. **Una sola `PENDING`** por local a la vez.
5. **Envío requiere responsable** (uno `PRINCIPAL`, varios `SECONDARY`).
6. **Solo `PENDING` es editable.**

---

# MÓDULO 2 — Revisión de Reposiciones (contabilidad)

**Módulo backend:** `replenishment-review` · **Base:** `/gtfReplacementsServices/api/v1/replenishment-reviews`
**Origen legacy:** `RevisionReposicionController`.

## 2.1 CRUD y operaciones

| #   | Método | Ruta                               | Descripción                                               |
| --- | ------ | ---------------------------------- | --------------------------------------------------------- |
| 1   | POST   | `/findByFilter`                    | Listar reposiciones `SENT` por local/fecha.               |
| 2   | GET    | `/{id}`                            | Ver detalle con columnas solicitado/aprobado.             |
| 3   | POST   | `/{id}/details/{detailId}/approve` | Ajustar valor aprobado (+ correo al local).               |
| 4   | POST   | `/{id}/validate`                   | Validar reposición (`VALIDATED`).                         |
| 5   | POST   | `/{id}/cancel`                     | Anular con observación.                                   |
| —   | job    | `VALIDATED → PAID/ISSUED`          | Tarea programada (reemplaza el script manual del legacy). |

## 2.2 Payloads / Responses

**POST `/findByFilter`**

```json
{
  "filters": [
    { "field": "status", "operator": "EQUALS", "value": "SENT" },
    { "field": "workAreaCode", "operator": "EQUALS", "value": "186" }
  ],
  "page": 0,
  "size": 10
}
```

**GET `/{id}`**

```json
{
  "code": 0,
  "message": "OK",
  "data": {
    "replenishmentId": 4900123,
    "workAreaCode": "186",
    "status": "SENT",
    "requestedTotal": 2.0,
    "approvedTotal": 2.0,
    "details": [
      {
        "detailId": 1,
        "documentType": "INVOICE",
        "documentNumber": "001-001-000012345",
        "requestedValue": 0.24,
        "approvedValue": 0.24
      }
    ]
  }
}
```

**POST `/{id}/details/{detailId}/approve`**

```json
// request
{ "approvedValue": 0.14, "observation": "IVA ingresado incorrecto" }
// response
{ "code": 0, "message": "Ajuste realizado. Se notificó al administrador del local.",
  "data": { "detailId": 1, "requestedValue": 0.24, "approvedValue": 0.14 } }
```

**POST `/{id}/validate`**

```json
// response error (punto de emisión)
{
  "code": 422,
  "message": "El usuario no tiene punto de emisión configurado.",
  "data": null
}
```

**POST `/{id}/cancel`**

```json
{ "observation": "Reposición inválida" }
```

## 2.3 Reglas de negocio

1. Dos columnas por línea: **solicitado (`requestedValue`)** vs **aprobado (`approvedValue`)**.
2. **Ajuste** dispara **correo de alerta** al administrador del local.
3. **Validar** requiere **punto de emisión** configurado en el CIF (pruebas con usuario Cintia/Cristina Escobar).
4. Tras `VALIDATED`, un **job programado** pasa a `PAID/ISSUED` (idempotente, con bitácora).

---

## 4. Base de datos (AS/400 · DB2 for i)

El sistema corre sobre **AS/400 (IBM i) → DB2 for i**, driver **`jt400`**, host **`bddpreprod`** (preproducción). Datasources definidos en el JBoss legacy (`standalone.xml`):

| JNDI                               | Usuario / esquema | Dominio                           |
| ---------------------------------- | ----------------- | --------------------------------- |
| `java:jboss/datasources/ExampleDS` | H2 memoria (`sa`) | Default JBoss — no se usa         |
| `java:/jdbc/smxcorp`               | `smxcorp`         | Corporativo                       |
| **`java:/jdbc/smxfinanc`**         | **`smxfinanc`**   | **Financiero (GTF/reposiciones)** |
| `java:/jdbc/smxsic`                | `smxsic`          | SIC                               |
| `java:/jdbc/smxmbase`              | `smxmbase`        | Maestro base                      |
| `java:/jdbc/smxfact`               | `smxfact`         | Facturación (SIF)                 |
| `java:/jdbc/smxjde`                | `smxjde` (XA)     | JDE                               |

### Datasource de reposiciones

- **Administración y revisión de reposiciones usan un único datasource: `jdbc/smxfinanc`** (esquema **SMXFINANC**). No hay datasource por módulo.
- Dialecto Hibernate legacy: `DB2i7Dialect`.
- **Tablas de reposiciones** (esquema SMXFINANC, prefijo `SFGTFT`):
  - `SFGTFTSOLICITUDREPOSICION` (cabecera)
  - `SFGTFTDETALLEREPOSICION` (detalle)
  - `SFGTFTREPOSICIONESTADO` (estados)
  - `SFGTFTSOLICITUDDOCUMENTO` / `SFGTFTSOLREPDOC` (documentos)
- **Accesos cruzados:** aunque la conexión es solo `smxfinanc`, se mapean entidades de otros dominios que viven en otras librerías (**SMXCORP** corporativo, **SMXFACT** facturación/SIF, **SMXJDE** JDE, auditoría, mensajería). Como no hay `schema`/`catalog` explícito en las entidades ni propiedad `libraries` en la URL, se resuelven por la **lista de librerías del perfil `smxfinanc`** en el AS/400.
  - Administración → lee corporativo (locales, personas, catálogos) y facturación/SIF (conceptos LC/LO).
  - Revisión → además facturación/SIF (punto de emisión, validación) y JDE (pago/cheque).

> **En `gtf-replacements-root`:** configurar el datasource `jt400` con el perfil **financiero (smxfinanc)** como conexión principal (DB2 for i) y garantizar que la **lista de librerías** incluya SMXCORP/SMXFACT/SMXJDE para los accesos cruzados, o **calificar el esquema por entidad**.

---

## 5. Configuración y despliegue

**Perfiles / context-path** (`application*.yaml`):

```yaml
server:
  port: 8080
  servlet:
    context-path: /gtfReplacementsServices
spring:
  application:
    name: gtf-replacements-services
```

- **Datasource DB2/i** (`jt400`) parametrizado por ambiente (DESARROLLO, PRUEBAS, CALIDAD, PRODUCCIÓN) + pool Hikari.
- **Keycloak:** clientId por ambiente; endpoints protegidos con Bearer.
- **Swagger:** `/gtfReplacementsServices/swagger-ui/index.html` · **Actuator health:** `/gtfReplacementsServices/actuator/health`.
- **Despliegue:** `Dockerfile` (`eclipse-temurin:25-jdk`) → imagen → **Kubernetes** (namespace `gtf`), profile por `SPRING_PROFILES_ACTIVE`. Pipeline Jenkins `springBootServicePipelineJDK25`.
- **Calidad:** PMD nivel 2, cobertura JaCoCo, SonarQube.

---

## 6. Dependencias externas / consideraciones

- **SIF/CIF:** conceptos de facturación, punto de emisión (validación al validar la reposición).
- **SRI:** servicio de retenciones **cortado**; el ingreso de retención electrónica (Módulo 1) queda al final, a coordinar con Frank.
- **Responsables:** consumidos para el envío; su administración vive en otro proyecto.
- **Correo:** notificación al local en el ajuste (Módulo 2).

---

_Documento acotado a los módulos de reposiciones. El desglose para Jira está en `JIRA-Backend-Reposiciones-Epicas.md` (misma carpeta)._
