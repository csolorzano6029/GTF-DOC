# GTF2 — Planificación de Arquitectura y Mapeo de APIs

> Documento de diseño previo a la codificación. Define el mapeo completo de módulos, controladores y endpoints REST a desarrollar en la migración del sistema **GTF (caja chica)** desde el legacy JBoss/JSF hacia el nuevo stack **Java 25 + Spring Boot 4 + Hibernate 7**.

- **Repo destino (backend):** `C:\Users\csolo\Documents\FAVORITA\MIGRACION\gtf-replacements-root` — contiene **administración de reposiciones** y **revisión de reposiciones**. (Habrá un repo por proyecto asignado; este es el de reposiciones.)
- **Proyecto legacy (fuente):** `C:\Users\csolo\Documents\FAVORITA\SOPORTES\gtf-root` (código) / `...\jboss-eap-7.1-GTF` (runtime)
- **Fecha:** 2026-07-06
- **Estado:** Base migrada de la plantilla `gtf-v2-root` → `gtf-replacements-root` (renombrada, compila y pasa tests). Pendiente codificar los módulos.

### Naming y contexto del repo `gtf-replacements-root`

| Aspecto               | Valor                                                                           |
| --------------------- | ------------------------------------------------------------------------------- |
| Módulos Gradle        | `gtf-replacements-{vo,client,core,services}`                                    |
| rootProject.name      | `gtf-replacements-root`                                                         |
| Paquete base          | `ec.com.smx.gtf.replacements`                                                   |
| Clase main            | `GtfReplacementsSpringBootApplication`                                          |
| groupId (Gradle)      | `ec.com.smx.gtf` (se mantiene)                                                  |
| **context-path**      | **`/gtfReplacementsServices`** → base API `/gtfReplacementsServices/api/v1/...` |
| Properties (basename) | `ec.com.smx.gtf.replacements.GtfReplacements`                                   |

> **Patrón de inyección:** controllers con **inyección por constructor**; el campo inyectado **no es `final`** (la clase base de test `MockMvcControllerBaseTest.setUpBase()` lo reinyecta por reflexión) y lleva `@SuppressWarnings("PMD.ImmutableField")`. El constructor llama `super()`.

---

## 1. Stack y contexto

| Aspecto       | Legacy (origen)                     | GTF2 (destino)                                                        |
| ------------- | ----------------------------------- | --------------------------------------------------------------------- |
| Lenguaje      | Java 1.6 → 1.7                      | **Java 25**                                                           |
| Framework     | Spring 3.2.15 + JSF (managed beans) | **Spring Boot 4**                                                     |
| ORM           | Hibernate 5                         | **Hibernate 7** + QueryDSL                                            |
| Empaque       | EAR / WAR                           | Jar Spring Boot                                                       |
| Servidor      | JBoss EAP 7.1                       | **Kubernetes** (contenedor)                                           |
| Seguridad     | Sesión JBoss                        | **Keycloak SSO** (`SecurityKeycloakUtil`)                             |
| Frontend      | JSF (`.xhtml` + DataManagers)       | **N/A** — SPA desarrollada por otro miembro (solo exponemos API REST) |
| Base de datos | Oracle (esquema `KSSEG...`)         | **Igual, sin cambios de esquema**                                     |

### Reglas de código (obligatorias)

- **Código 100 % en inglés** (clases, métodos, variables, endpoints). Se elimina el _spanglish_ del legacy.
- **Comentarios en español** (única excepción).
- **Prohibido** variables de una sola letra (`i`, `j`, etc.). Nombres 100 % descriptivos.
- Se sigue **al pie de la letra el patrón del template** (ver §3).

---

## 2. Arquitectura del proyecto destino (`gtf-v2-root`)

Proyecto Gradle multi-módulo. Cada dominio de negocio se implementa atravesando las 4 capas:

```
gtf-v2-root
├── gtf-v2-root-vo        → VOs / DTOs de transporte (entrada/salida REST). Lombok + validación.
├── gtf-v2-root-client    → Entidades JPA (@Entity), repositorios (interfaces QueryDSL),
│                            interfaces de servicio (IXxxService) y connectors a sistemas externos.
├── gtf-v2-root-core      → Implementación de servicios (@Service), configuración, lógica de negocio.
└── gtf-v2-root-services  → Controllers REST (@RestController) + arranque Spring Boot.
```

### Convenciones del framework (Kruger v3), extraídas del CRUD de ejemplo

| Elemento        | Clase base / anotación                                                              | Notas                                                                                               |
| --------------- | ----------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| Controller      | `extends BaseController` (`ec.com.kruger.spring.ws.v3.controller`)                  | `@RestController`, `@RequestMapping("/api/v1/<domain>")`, `@Lazy`, `@Validated`, `@Tag` (Swagger).  |
| Respuesta       | `BaseResponseVo<Object>` (`ec.com.kruger.spring.vo.v3.common`)                      | Envelope estándar (ver §4). Se construye con `.builder().data(...).code(...).message(...).build()`. |
| Servicio        | `IBaseService<Entity>` / `BaseService<Entity, IRepo>`                               | Métodos CRUD base + específicos. `@Transactional`.                                                  |
| Repositorio     | `IQueryDslBaseRepository<Entity>` (`ec.com.kruger.spring.orm.v3.repository`)        | Paginación y filtros con QueryDSL.                                                                  |
| Entidad         | `extends AbstractBaseAuditableLockingIp<UserView, String>`                          | Auditoría + optimistic locking. `@Entity(name="TABLA")`, `@Column`.                                 |
| VO              | `extends BaseAuditableVo`                                                           | `@Data @SuperBuilder @AllArgsConstructor @NoArgsConstructor`.                                       |
| Filtro/paginado | `FilterVo` { `Collection<SearchModelDTO> filters`, `Integer page`, `Integer size` } | Endpoint `POST /findByFilter`.                                                                      |
| Seguridad       | `SecurityKeycloakUtil.getCurrentUserLogin()` → `KeycloakUserDTO`                    | De ahí se obtiene `companyID`, usuario, etc.                                                        |
| Mapeo VO↔Entity | `ProjectUtil.convert(source, Target.class)`                                         | Utilitario del proyecto.                                                                            |
| Validación      | `@NotBlankConstraint`, `@Cedula`, `@Size`, `@Email` (kruger + jakarta)              | En los VOs.                                                                                         |

### Contexto de despliegue

- **Context-path del servicio: `/gtfReplacementsServices`** (configurado en `application.yaml` y todos los perfiles). Como habrá **varios microservicios** (uno por proyecto), cada repo lleva su propio context-path; el del legacy era `/gtf`.
- **Base efectiva de las APIs:** `http://host:8080/gtfReplacementsServices/api/v1/<recurso>`.
- Versionado por ruta: prefijo `/api/v1`. Recursos nombrados en **plural** (`replenishments`, `replenishment-reviews`, …). No usar el segmento `inet` salvo recursos provenientes del sistema intranet externo (ej. `companies`).
- Contenedor: `eclipse-temurin:25-jdk`; pipeline `springBootServicePipelineJDK25` (namespace `gtf`).
- Autenticación: token Bearer de Keycloak (realm `CFAVORITA-SSO-INTRANET`, client `BASE-WEB`).

> **Context-roots del legacy** (referencia): `gtf` (app JSF principal), `gtfWs` (web service `gtf-caja-chica-ws`), `TareasProgramadas` (`gtf-stp-ear`). El legacy **no exponía REST**; usaba JSF (`*.jsf`).

---

## 3. Glosario de traducción (Español legacy → Inglés GTF2)

Diccionario obligatorio para nombrar clases, campos y endpoints de forma consistente.

| Español (legacy)                            | Inglés (GTF2)                              | Uso                                                                                                                    |
| ------------------------------------------- | ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| Reposición / Reembolso (solicitud)          | **ReplenishmentRequest** / `replenishment` | Entidad y dominio principal                                                                                            |
| Detalle de reposición                       | **ReplenishmentDetail**                    | Línea de documento dentro de la reposición                                                                             |
| Documento (factura, nota de venta, recibo…) | **Document** / `documentType`              | Tipos: `INVOICE`, `ELECTRONIC_INVOICE`, `SALES_NOTE`, `MANUAL_RECEIPT`, `ELECTRONIC_WITHHOLDING`, `MANUAL_WITHHOLDING` |
| Fondo (de local/área)                       | **Fund** / `fund`                          | Fondo de caja chica del local                                                                                          |
| Área de trabajo / local                     | **WorkArea** / `workArea`                  | Local (ej. 186, 153, 101)                                                                                              |
| Funcionario / Responsable                   | **Officer** / `responsible`                | Principal + secundarios                                                                                                |
| Retención (electrónica / manual)            | **Withholding** (`electronic` / `manual`)  | Comprobante de retención                                                                                               |
| Concepto de facturación                     | **BillingConcept** / `billingConcept`      | Tipos `LC` (liquidación compra) / `LO` (liquidación otros)                                                             |
| Estado                                      | **Status**                                 | Máquina de estados de la reposición                                                                                    |
| Ajuste (de fondo / de reposición)           | **Adjustment**                             | Aumentar/disminuir fondo o corregir monto                                                                              |
| Punto de emisión                            | **EmissionPoint**                          | Config requerida en el CIF/SIF                                                                                         |
| Saldo efectivo                              | **cashBalance**                            | Efectivo disponible en caja                                                                                            |
| Fondo asignado                              | **assignedFund**                           | Monto asignado al local                                                                                                |
| Reposiciones pendientes de pago             | **pendingPaymentAmount**                   | —                                                                                                                      |
| Valor máximo (de factura)                   | **maxDocumentValue**                       | Tope por documento                                                                                                     |
| Porcentaje de uso / de alerta               | **usagePercentage** / **alertPercentage**  | Umbrales del fondo                                                                                                     |
| Impuesto al IVA / a la renta                | **vatTax** / **incomeTax**                 | En retenciones                                                                                                         |
| Clave de acceso                             | **accessKey**                              | Comprobante electrónico (SRI)                                                                                          |

### Estados de la reposición (máquina de estados)

```
PENDING → SENT → VALIDATED → PAID → ISSUED → COLLECTED
                                              (CANCELLED en cualquier punto autorizado)
```

| Español        | Inglés            | Descripción                            |
| -------------- | ----------------- | -------------------------------------- |
| Pendiente      | `PENDING`         | Editable por el local.                 |
| Enviado        | `SENT`            | Enviado a contabilidad; no editable.   |
| Validado       | `VALIDATED`       | Revisado y aprobado por contabilidad.  |
| Pago / Emitido | `PAID` / `ISSUED` | Tarea programada emite el pago.        |
| Cobrado        | `COLLECTED`       | El local aceptó/recibió la reposición. |
| Anulado        | `CANCELLED`       | Anulado con observación.               |

---

## 4. Estándar de respuesta JSON (envelope)

Todos los endpoints devuelven el envelope del framework **`BaseResponseVo<T>`**:

```json
{
  "code": 0,
  "message": "OK",
  "data": {}
}
```

| Campo     | Tipo                         | Significado                                                    |
| --------- | ---------------------------- | -------------------------------------------------------------- |
| `code`    | `Integer`                    | `0` = éxito. `>0` = código de negocio (ej. `1` = "ya existe"). |
| `message` | `String`                     | Mensaje legible (errores de validación / negocio).             |
| `data`    | `T` (objeto, lista o `Page`) | Carga útil. Se omite si es `null` (`@JsonInclude(NON_NULL)`).  |

**Respuestas paginadas** (`Page<T>` de Spring Data) van dentro de `data`:

```json
{
  "code": 0,
  "message": "OK",
  "data": {
    "content": [],
    "totalElements": 42,
    "totalPages": 5,
    "number": 0,
    "size": 10
  }
}
```

**Errores de validación / negocio** (ej.):

```json
{
  "code": 400,
  "message": "El valor del IVA ingresado (0.30) supera el máximo permitido (0.26).",
  "data": null
}
```

> Los campos de auditoría (`BaseAuditableVo`) —`companyCode`, `status`, `createdByUser`, `createdDate`, `lastModifiedByUser`, `lastModifiedDate`, `userId`— se heredan en todos los VOs de salida.

---

## 5. Mapa general de módulos a desarrollar

Orden de migración acordado en la reunión (por prioridad). **Se excluye** todo el código de _gastos personales / viáticos / `nuevoReposicionGasto`_ (desarrollado pero nunca usado).

| #   | Prioridad      | Módulo GTF2                           | Origen legacy                                                                            | Rol                                                    | Estado                     |
| --- | -------------- | ------------------------------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------ | -------------------------- |
| 1   | 🔴 **Primero** | **replenishment**                     | `AdminRembolsoController` + `AdminRembolsoDataManager`                                   | Lo que **ingresa el local**                            | A desarrollar              |
| 2   | 🟠 Segundo     | **replenishment-review**              | `RevisionReposicionController`                                                           | Lo que **revisa contabilidad**                         | Pendiente                  |
| 3   | 🟡             | **fund**                              | `AdminFondosController` + `ajusteFondo`                                                  | Administración de fondos del local                     | Pendiente                  |
| 4   | 🟡             | **officer**                           | `AdminFuncionarioAreTraFinController`                                                    | Responsables de caja chica                             | Pendiente                  |
| 5   | 🟢             | **withholding**                       | `AdminRetencionesController` + `AdminReporteRetencionesController` + `gtf-caja-chica-ws` | Retenciones electrónicas/manuales y reporte            | Pendiente (depende de SRI) |
| 6   | ⚪ Último      | **deposit**                           | `AdminDepositoController` + `ReporteDepositoController`                                  | Depósitos ("no da problemas")                          | Pendiente                  |
| —   | Soporte        | **parameter**                         | `GTF_PARAMETRO`                                                                          | Parámetros (ej. conceptos que no validan valor máximo) | ✅ ya en template          |
| —   | Soporte        | **company / person / catalog / file** | connectors                                                                               | Compañía, personas, catálogos, archivos                | ✅ ya en template          |

### Dominios de soporte (compartidos)

- `parameter` — tabla `GTF_PARAMETRO`. Necesario para el control de excepciones de valor máximo (conceptos `estudios de mercado`, `gas/gasolina`). Se recomienda **pantalla/endpoint de administración** (mencionado en la reunión).
- `billing-concept` — proviene del **SIF/CIF** vía connector; tipos `LC` / `LO`.
- `company`, `person`, `catalog`, `file` — ya presentes en el CRUD de ejemplo.

---

## 6. MÓDULO 1 — `replenishment` (Administración de Reposición — Local) 🔴

**Objetivo:** permitir al usuario de local crear, editar, enviar, imprimir y anular sus solicitudes de reposición de caja chica, con todas las validaciones de negocio.

### 6.1 Operaciones de negocio del legacy (origen)

Extraídas de `IReposicionServicio` y `AdminRembolsoController`:

| Operación legacy                           | Descripción                                          |
| ------------------------------------------ | ---------------------------------------------------- |
| `transGuardarOActualizarRembolso`          | Crear / actualizar reposición (estado `PENDING`).    |
| `transEnviarSolicitudReposicion`           | Enviar a contabilidad (valida responsable → `SENT`). |
| `transAnularSolicitudReposicion`           | Anular (con observación).                            |
| `transProcesarImpresionReposicion`         | Generar impresión (PDF FOP).                         |
| `transObtenerReposicionCanceladaCajaChica` | Validar documento duplicado (RUC + N° doc).          |
| `transReposicionFondoDisponible`           | Recalcular saldo/fondo disponible.                   |
| `obtenerRetencionesElectronicasPorFiltros` | Consultar retención electrónica por filtros.         |
| `validarClaveAcceso` (controller)          | Validar clave de acceso contra SRI (solo estado).    |
| `findConceptMeasureByConceptId`            | Conceptos/unidades de medida (SIF).                  |

### 6.2 Endpoints REST a desarrollar

Base: `/api/v1/replenishments`

| #   | Método | Ruta                                           | Descripción                                       | Origen legacy                              |
| --- | ------ | ---------------------------------------------- | ------------------------------------------------- | ------------------------------------------ |
| 1   | `GET`  | `/current-fund?workAreaCode={code}`            | Cabecera del fondo (asignado, saldo, pendientes). | `cargarDatosIniciales`                     |
| 2   | `POST` | `/findByFilter`                                | Búsqueda paginada por estado/fecha.               | `buscarRembolsoListener`                   |
| 3   | `GET`  | `/{replenishmentId}`                           | Ver una reposición y su detalle.                  | `visualizarRembolso`                       |
| 4   | `POST` | ``                                             | Crear reposición (`PENDING`).                     | `transGuardarOActualizarRembolso`          |
| 5   | `POST` | `/{replenishmentId}`                           | Actualizar reposición pendiente.                  | `transGuardarOActualizarRembolso`          |
| 6   | `POST` | `/{replenishmentId}/details`                   | Agregar línea de documento (con validaciones).    | `agregarFilaDetalle`                       |
| 7   | `POST` | `/{replenishmentId}/details/{detailId}`        | Editar línea de documento.                        | `actualizarReposicion`                     |
| 8   | `POST` | `/{replenishmentId}/details/{detailId}/delete` | Eliminar línea.                                   | `eliminarFilaDetalle`                      |
| 9   | `POST` | `/{replenishmentId}/send`                      | Enviar a contabilidad.                            | `onclickEnviarRembolso`                    |
| 10  | `POST` | `/{replenishmentId}/cancel`                    | Anular reposición.                                | `transAnularSolicitudReposicion`           |
| 11  | `GET`  | `/{replenishmentId}/print`                     | Generar PDF de la reposición.                     | `imprimirReposicion`                       |
| 12  | `GET`  | `/responsibles?workAreaCode={code}`            | Responsables del local (principal + secundarios). | `cargarFuncionariosResponsables`           |
| 13  | `POST` | `/validate-duplicate`                          | Validar documento no duplicado.                   | `transObtenerReposicionCanceladaCajaChica` |
| 14  | `POST` | `/validate-vat`                                | Calcular/validar IVA (`total/1.15`).              | control en `agregarFilaDetalle`            |
| 15  | `GET`  | `/document-types`                              | Catálogo de tipos de documento.                   | `verificaTipoDocumento`                    |
| 16  | `GET`  | `/billing-concepts?type={LC\|LO}`              | Conceptos de facturación (SIF).                   | `listarConAreTraFin`                       |

> Las retenciones (endpoints 17+) se detallan en el **Módulo 5**, pero el módulo 1 las consume al agregar un detalle de tipo retención.

### 6.3 Ejemplos de estructuras JSON

#### `GET /api/v1/replenishments/current-fund?workAreaCode=186`

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

#### `POST /api/v1/replenishments/findByFilter`

Request:

```json
{
  "filters": [{ "field": "status", "operator": "EQUALS", "value": "PENDING" }],
  "page": 0,
  "size": 10
}
```

Response:

```json
{
  "code": 0,
  "message": "OK",
  "data": {
    "content": [
      {
        "replenishmentId": 4900123,
        "workAreaCode": "186",
        "status": "PENDING",
        "createdDate": 1751414400000,
        "requestedTotal": 79.81,
        "approvedTotal": 0.0
      }
    ],
    "totalElements": 2,
    "totalPages": 1,
    "number": 0,
    "size": 10
  }
}
```

#### `GET /api/v1/replenishments/{replenishmentId}`

```json
{
  "code": 0,
  "message": "OK",
  "data": {
    "replenishmentId": 4900123,
    "workAreaCode": "186",
    "status": "PENDING",
    "responsible": {
      "personId": "1712345678",
      "fullName": "Cintia Escobar",
      "type": "PRINCIPAL"
    },
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
        "billingConceptType": "LC",
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

#### `POST /api/v1/replenishments/{replenishmentId}/details` (agregar factura)

Request:

```json
{
  "documentType": "INVOICE",
  "documentDate": "2026-06-29",
  "ruc": "1790016919001",
  "documentNumber": "001-001-000012345",
  "billingConceptCode": "LC-EDIF",
  "observation": "Prueba 1",
  "vatValue": 0.26,
  "total": 2.0
}
```

Response OK:

```json
{ "code": 0, "message": "OK", "data": { "detailId": 1, "requestedValue": 2.0 } }
```

Response error (IVA):

```json
{
  "code": 400,
  "message": "El valor del IVA (0.30) supera el máximo permitido (0.26).",
  "data": null
}
```

#### `POST /api/v1/replenishments/validate-duplicate`

Request:

```json
{
  "companyCode": 1,
  "ruc": "1790016919001",
  "documentNumber": "001-001-000012345"
}
```

Response (duplicado encontrado):

```json
{
  "code": 409,
  "message": "La factura ingresada se encuentra cancelada.",
  "data": {
    "duplicate": true,
    "replenishmentId": 4890010,
    "workAreaCode": "186",
    "cancelledDate": "2026-06-29"
  }
}
```

#### `POST /api/v1/replenishments/validate-vat`

Request:

```json
{ "total": 2.0, "vatValue": 0.3 }
```

Response:

```json
{
  "code": 400,
  "message": "El IVA ingresado supera el máximo calculado.",
  "data": {
    "total": 2.0,
    "calculatedVat": 0.26,
    "maxAllowedVat": 0.26,
    "valid": false
  }
}
```

#### `POST /api/v1/replenishments/{replenishmentId}/send`

Request:

```json
{ "responsiblePersonId": "1712345678" }
```

Response error (sin responsable):

```json
{
  "code": 422,
  "message": "No existe un responsable asignado para el local.",
  "data": null
}
```

#### `GET /api/v1/replenishments/responsibles?workAreaCode=186`

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

### 6.4 Reglas de negocio a preservar (Módulo 1)

1. **Validación de IVA:** el IVA ingresado debe ser **≤** `total − (total / 1.15)`. Se permite menor (facturas con ítems gravados y no gravados). Error si es mayor.
2. **Documento duplicado:** rechazar si (`companyCode` + `ruc` + `documentNumber`) ya fue cancelado en otra reposición/fecha.
3. **Valor máximo por documento:** rechazar si supera `maxDocumentValue`, **excepto** conceptos configurados en `GTF_PARAMETRO` (estudios de mercado, gas/gasolina).
4. **% de fondo:** alertar/bloquear al superar `usagePercentage` (60 %) y avisar en `alertPercentage` (50 %).
5. **IVA por tipo de documento:** solo `INVOICE` / `ELECTRONIC_INVOICE` tienen IVA. `SALES_NOTE` y `MANUAL_RECEIPT` no.
6. **Una sola pendiente:** no se puede crear una reposición nueva si existe una en estado `PENDING`.
7. **Envío requiere responsable:** debe existir un responsable asignado (uno `PRINCIPAL`, varios `SECONDARY`).
8. **Solo `PENDING` es editable.**

---

## 7. MÓDULO 2 — `replenishment-review` (Revisión — Contabilidad) 🟠

**Objetivo:** contabilidad revisa lo enviado por el local, compara _solicitado vs aprobado_, ajusta, valida o anula.

Base: `/api/v1/replenishment-reviews`

| Método | Ruta                                            | Descripción                                    | Origen                            |
| ------ | ----------------------------------------------- | ---------------------------------------------- | --------------------------------- |
| `POST` | `/findByFilter`                                 | Listar reposiciones `SENT` por local/fecha.    | `RevisionReposicionController`    |
| `GET`  | `/{replenishmentId}`                            | Ver detalle con columnas _requested/approved_. | `visualizarRevision`              |
| `POST` | `/{replenishmentId}/details/{detailId}/approve` | Ajustar valor aprobado (envía mail al local).  | `transAjusteSolicitudReposicion`  |
| `POST` | `/{replenishmentId}/validate`                   | Validar reposición (`VALIDATED`).              | `transValidarSolicitudReposicion` |
| `POST` | `/{replenishmentId}/cancel`                     | Anular con observación.                        | `transAnularSolicitudReposicion`  |

**JSON de ajuste** — `POST .../details/{detailId}/approve`:

```json
{ "approvedValue": 0.14, "observation": "IVA ingresado incorrecto" }
```

```json
{
  "code": 0,
  "message": "Ajuste realizado. Se notificó al administrador del local.",
  "data": { "detailId": 1, "requestedValue": 0.24, "approvedValue": 0.14 }
}
```

**Regla clave:** al validar requiere **punto de emisión** configurado en el CIF (usar usuario _Cintia/Cristina Escobar_ en pruebas). Tras `VALIDATED`, una **tarea programada** pasa a `PAID/ISSUED` (hoy rota → se corre script manual; en GTF2 reimplementar como job).

---

## 8. MÓDULO 3 — `fund` (Administración de Fondos)

Base: `/api/v1/funds`

| Método | Ruta                          | Descripción                                              | Origen                  |
| ------ | ----------------------------- | -------------------------------------------------------- | ----------------------- |
| `GET`  | `/{workAreaCode}`             | Configuración del fondo del local.                       | `AdminFondosController` |
| `POST` | ``                            | Crear fondo (local nuevo).                               | `transGuardarFondoArea` |
| `POST` | `/{workAreaCode}`             | Editar % uso, % alerta, valor máximo, forma de depósito. | `transGuardarFondoArea` |
| `POST` | `/{workAreaCode}/adjustments` | Ajuste (+/–) del fondo (ej. temporada navideña).         | `ajusteFondo`           |
| `GET`  | `/{workAreaCode}/adjustments` | Historial de ajustes.                                    | —                       |

```json
{
  "code": 0,
  "message": "OK",
  "data": {
    "workAreaCode": "186",
    "assignedFund": 700.0,
    "usagePercentage": 60,
    "alertPercentage": 50,
    "maxDocumentValue": 50.0,
    "depositType": "DEPOSIT"
  }
}
```

**Regla:** ni la reposición ni la disminución de fondo pueden superar el `cashBalance` (validación del legacy v1.10.3).

---

## 9. MÓDULO 4 — `officer` (Responsables de Caja Chica)

Base: `/api/v1/officers`

| Método | Ruta                   | Descripción             | Origen                                |
| ------ | ---------------------- | ----------------------- | ------------------------------------- |
| `GET`  | `?workAreaCode={code}` | Responsables del local. | `AdminFuncionarioAreTraFinController` |
| `POST` | ``                     | Asignar responsable.    | —                                     |
| `POST` | `/{officerId}`         | Editar responsable.     | —                                     |
| `POST` | `/{officerId}/delete`  | Quitar responsable.     | —                                     |

**Regla:** **un solo** `PRINCIPAL` por local; varios `SECONDARY`.

---

## 10. MÓDULO 5 — `withholding` (Retenciones) 🟢

Base: `/api/v1/withholdings`

| Método | Ruta                               | Descripción                                  | Origen                                     |
| ------ | ---------------------------------- | -------------------------------------------- | ------------------------------------------ |
| `GET`  | `/electronic?ruc=&documentNumber=` | Buscar retención electrónica en repositorio. | `obtenerRetencionesElectronicasPorFiltros` |
| `POST` | `/electronic/validate`             | Validar retención electrónica.               | `validarIngresoRetencion`                  |
| `POST` | `/electronic/access-key/validate`  | Validar clave de acceso (SRI → solo estado). | `validarClaveAcceso`                       |
| `POST` | `/electronic/xml`                  | Subir XML descargado del SRI.                | `obtenerArchivoXml`                        |
| `POST` | `/manual`                          | Registrar retención manual (IVA + renta).    | `agregarImpuestoRetencionManual`           |
| `GET`  | `/report`                          | Reporte de retenciones.                      | `AdminReporteRetencionesController`        |

```json
{
  "code": 0,
  "message": "OK",
  "data": {
    "ruc": "1790016919001",
    "documentNumber": "001-001-000000123",
    "accessKey": "2706202601...",
    "sriStatus": "AUTHORIZED",
    "vatTax": 0.05,
    "incomeTax": 0.02
  }
}
```

> ⚠️ **Bloqueador conocido:** el SRI **cortó el servicio web** que devolvía los datos de la retención; ahora solo devuelve estado (`AUTHORIZED`, `NOT_AUTHORIZED`, `PROCESSING`, `CANCELLED`). Diseño a coordinar con **Frank**. **Este módulo va al final.**

---

## 11. MÓDULO 6 — `deposit` (Depósitos) ⚪

Base: `/api/v1/deposits`. CRUD estándar + reporte (`AdminDepositoController`, `ReporteDepositoController`). Última prioridad; "no ha dado problemas".

---

## 12. Pendientes técnicos transversales

| #   | Pendiente                                                            | Impacto en GTF2                                              |
| --- | -------------------------------------------------------------------- | ------------------------------------------------------------ |
| 1   | Tarea programada `VALIDATED → PAID/ISSUED` rota (hoy script manual). | Reimplementar como job programado.                           |
| 2   | Servicios del SRI cortados.                                          | Rediseñar retenciones (con Frank).                           |
| 3   | Desfase de centavos (SIF valida solo el total).                      | Automatizar ajuste de centavos en validación.                |
| 4   | Punto de emisión en CIF.                                             | Verificar configuración por usuario.                         |
| 5   | Connectors a SIF/CIF, auditoría, impresión (FOP), file-manager.      | Confirmar versiones compatibles con Java 25 / Spring Boot 4. |

---

## 13. Definición de "listo para codificar" (Módulo 1)

- [x] Template base disponible y patrón entendido.
- [x] Reglas de negocio del Módulo 1 documentadas (§6.4).
- [x] Endpoints y JSON del Módulo 1 definidos (§6.2, §6.3).
- [ ] Confirmar disponibilidad de librerías internas (SIF, auditoría, impresión) para Java 25.
- [ ] Entorno para levantar/comparar contra el legacy (JBoss de Carlos).
- [ ] Aprobación del glosario de traducción (§3) por el equipo.

---

_Documento vivo. Se actualiza a medida que se aborda cada módulo. Los ejemplos JSON de los módulos 2–6 se ampliarán al iniciar su desarrollo._
