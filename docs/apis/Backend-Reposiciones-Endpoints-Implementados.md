# Backend Reposiciones - Endpoints Implementados

## Proposito

Documentar los endpoints ya implementados del modulo de reposiciones con un formato estandar:

- Metodo HTTP
- URL
- Parametros
- Ejemplo de request
- Ejemplo de response

Nota de alcance documental:

- Este archivo define el contrato tecnico de APIs (request/response).
- El mapeo campo API <-> campo visual del frontend legacy se mantiene en `docs/BACK-FRONT.md` (seccion 4.1.1.1) para evitar duplicidad y desalineaciones.

Este archivo se actualiza cada vez que se crea o modifica un endpoint.

## Alcance

- Modulo: replenishments + catalogs (generico)
- Base URL aplicacion: /gtfReplacementsServices
- Base path controllers: /api/v1/replenishments (compartido), /api/v1/replenishment-management (administracion), /api/v1/replenishment-reviews (revision) y /api/v1/catalogs

## Convenciones de respuesta

Envelope de salida:

```json
{
  "code": 200,
  "message": "OK",
  "data": {}
}
```

Notas:

- El front valida primero `code`.
- En los endpoints actuales de `replenishments`, el backend retorna HTTP 200 y `code: 200` tambien para escenarios de validacion funcional.
- El resultado funcional se interpreta por `message` y por presencia/ausencia de `data`.
- Si no existe data para un escenario, el campo `data` se omite en la respuesta.

---

## 1) Obtener cabecera de fondo

### Resumen

- Nombre: Obtener cabecera de fondo por area de trabajo
- Metodo: GET
- URL: /gtfReplacementsServices/api/v1/replenishment-management/current-fund

### Parametros

| Tipo  | Nombre       | Requerido | Tipo dato | Descripcion               |
| ----- | ------------ | --------- | --------- | ------------------------- |
| Query | workAreaCode | Si        | Integer   | Codigo de area de trabajo |

### Ejemplo de request

```http
GET /gtfReplacementsServices/api/v1/replenishment-management/current-fund?workAreaCode=186
```

### Ejemplo de response con data

```json
{
  "code": 200,
  "message": "OK",
  "data": {
    "workAreaCode": "186",
    "workAreaName": "EL JARDIN",
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

### Ejemplo de response sin data

```json
{
  "code": 200,
  "message": "OK"
}
```

---

## 2) Buscar reposiciones por filtro

### Resumen

- Nombre: Buscar reposiciones por filtro con paginacion
- Metodo: POST
- URL: /gtfReplacementsServices/api/v1/replenishments/findByFilter
- URL completa sugerida: {{host}}/gtfReplacementsServices/api/v1/replenishments/findByFilter

### Parametros

| Tipo | Nombre  | Requerido | Tipo dato | Descripcion                       |
| ---- | ------- | --------- | --------- | --------------------------------- |
| Body | filters | No        | Array     | Reglas de filtro (puede ser null) |
| Body | page    | No        | Integer   | Numero de pagina (default 0)      |
| Body | size    | No        | Integer   | Tamanio de pagina (default 10)    |

### Campos de filtro disponibles

Actualmente el backend permite filtrar por los campos proyectados en la consulta:

- replenishmentId
- workAreaCode
- transactionCode
- replenishmentType
- replenishmentStatus
- observation
- requestedTotal
- requestedSubtotal
- advanceValue
- vatTotal
- approvedVatTotal
- responsiblePersonId
- beneficiaryPersonId
- checkResponsiblePersonId
- startDate
- endDate
- createdDate
- replenishmentDate

Notas importantes:

- companyCode no se envia en filters: se toma del token de Keycloak.
- Cada item de filters usa estructura SearchModelDTO (alias/parameterPattern/comparatorTypeEnum/parameterValue).
- Si `alias` viene nulo, el backend usa `parameterPattern` como nombre de campo.
- Comparador recomendado para igualdad con el componente estandar: `Igual`.
- `transactionCode` es el valor tecnico persistido (`1`, `2`), mientras que `replenishmentType` es el valor legible para front (`Caja Chica`, `Gastos Personales`).
- `replenishmentStatus` mantiene el codigo tecnico de estado (`PEN`, `ENV`, `PAG`, etc.) y `replenishmentStatusName` expone la etiqueta legible para UI.
- `sendAlert` expone la bandera de alerta visual legacy (`1`/`0`) para resaltar filas en la grilla.
- `replenishmentDate` es equivalente a `createdDate` y corresponde a la fecha de reposicion usada en legacy (fechaRegistro).

### Operadores recomendados por tipo de campo

| Tipo de campo | Operadores recomendados            | Campos sugeridos                                                                                                  |
| ------------- | ---------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| Texto         | Igual, Like                        | transactionCode, replenishmentStatus, observation                                                                 |
| Numerico      | Igual, MayorQue, MenorQue          | replenishmentId, workAreaCode, responsiblePersonId, requestedTotal, requestedSubtotal, vatTotal, approvedVatTotal |
| Fecha         | Igual, MayorQue, MenorQue, Between | startDate, endDate, createdDate                                                                                   |

Nota:

- En el estado actual del backend, el operador validado funcionalmente para igualdad es `Igual` (componentes legacy) y se mantiene compatibilidad con `EQUALS/EQUAL/EQ`.

### Ejemplo de request payload

```json
{
  "filters": [
    {
      "alias": null,
      "parameterPattern": "workAreaCode",
      "dataType": "integer",
      "parameterValue": 156,
      "parameterValues": [],
      "selectedOption": "uv",
      "operatorOR": false,
      "comparatorTypeEnum": "Igual",
      "insensitive": false,
      "rangeBean": null
    },
    {
      "alias": null,
      "parameterPattern": "replenishmentStatus",
      "dataType": "string",
      "parameterValue": "PAG",
      "parameterValues": [],
      "selectedOption": "uv",
      "operatorOR": false,
      "comparatorTypeEnum": "Igual",
      "insensitive": false,
      "rangeBean": null
    }
  ],
  "page": 0,
  "size": 1
}
```

### Payloads de prueba listos (Postman)

#### Caso A: Debe retornar resultados

```json
{
  "filters": [
    {
      "alias": null,
      "parameterPattern": "workAreaCode",
      "dataType": "integer",
      "parameterValue": 156,
      "parameterValues": [],
      "selectedOption": "uv",
      "operatorOR": false,
      "comparatorTypeEnum": "Igual",
      "insensitive": false,
      "rangeBean": null
    },
    {
      "alias": null,
      "parameterPattern": "replenishmentStatus",
      "dataType": "string",
      "parameterValue": "PAG",
      "parameterValues": [],
      "selectedOption": "uv",
      "operatorOR": false,
      "comparatorTypeEnum": "Igual",
      "insensitive": false,
      "rangeBean": null
    }
  ],
  "page": 0,
  "size": 10
}
```

#### Caso B: No existe (debe retornar pagina vacia)

```json
{
  "filters": [
    {
      "alias": null,
      "parameterPattern": "workAreaCode",
      "dataType": "integer",
      "parameterValue": 1561231,
      "parameterValues": [],
      "selectedOption": "uv",
      "operatorOR": false,
      "comparatorTypeEnum": "Igual",
      "insensitive": false,
      "rangeBean": null
    },
    {
      "alias": null,
      "parameterPattern": "replenishmentStatus",
      "dataType": "string",
      "parameterValue": "PAG1123",
      "parameterValues": [],
      "selectedOption": "uv",
      "operatorOR": false,
      "comparatorTypeEnum": "Igual",
      "insensitive": false,
      "rangeBean": null
    }
  ],
  "page": 0,
  "size": 10
}
```

#### Caso C: Filtro por responsiblePersonId

```json
{
  "filters": [
    {
      "alias": null,
      "parameterPattern": "responsiblePersonId",
      "dataType": "long",
      "parameterValue": 73446,
      "parameterValues": [],
      "selectedOption": "uv",
      "operatorOR": false,
      "comparatorTypeEnum": "Igual",
      "insensitive": false,
      "rangeBean": null
    }
  ],
  "page": 0,
  "size": 10
}
```

Notas de compatibilidad:

- El backend acepta `comparatorTypeEnum` en formato legacy (`Igual`) y tambien `EQUALS`, `EQUAL`, `EQ`, `EQUAL_COMPARATOR` para igualdad.
- Si `alias` viene `null`, se usa `parameterPattern`.
- Los nombres recomendados para `parameterPattern` son los campos listados en "Campos de filtro disponibles".

### Ejemplo de response

```json
{
  "code": 200,
  "message": "OK",
  "data": {
    "content": [
      {
        "replenishmentId": 100,
        "replenishmentStatus": "PEN",
        "replenishmentStatusName": "Pendiente",
        "sendAlert": "1",
        "unsentDocumentsTotal": 250.0,
        "responsiblePersonId": 72643,
        "responsiblePersonDocument": "1712345678",
        "workAreaCode": 186,
        "workAreaName": "EL JARDIN"
      }
    ],
    "empty": false,
    "first": true,
    "last": true,
    "number": 0,
    "numberOfElements": 1,
    "pageable": {
      "offset": 0,
      "pageNumber": 0,
      "pageSize": 10,
      "paged": true,
      "sort": {
        "empty": true,
        "sorted": false,
        "unsorted": true
      },
      "unpaged": false
    },
    "size": 10,
    "sort": {
      "empty": true,
      "sorted": false,
      "unsorted": true
    },
    "totalElements": 1,
    "totalPages": 1
  }
}
```

### Response cuando no existen resultados (200)

```json
{
  "code": 200,
  "message": "OK",
  "data": {
    "content": [],
    "empty": true,
    "first": true,
    "last": true,
    "number": 0,
    "numberOfElements": 0,
    "pageable": {
      "offset": 0,
      "pageNumber": 0,
      "pageSize": 10,
      "paged": true,
      "sort": {
        "empty": true,
        "sorted": false,
        "unsorted": true
      },
      "unpaged": false
    },
    "size": 10,
    "sort": {
      "empty": true,
      "sorted": false,
      "unsorted": true
    },
    "totalElements": 0,
    "totalPages": 0
  }
}
```

---

## 3) Validar IVA de detalle

### Resumen

- Nombre: Validar IVA de detalle de reposicion
- Metodo: POST
- URL: /gtfReplacementsServices/api/v1/replenishments/validate-vat
- URL completa sugerida: {{host}}/gtfReplacementsServices/api/v1/replenishments/validate-vat

### Request payload (minimo recomendado)

```json
{
  "documentTotal": 115.0,
  "vatValue": 15.0
}
```

Reglas aplicadas (alineadas al legacy):

- IVA no puede ser negativo.
- El total del documento es obligatorio.
- IVA debe ser menor al total del documento.
- IVA debe ser menor o igual al maximo calculado: `total - ((total * 100) / (100 + 15))`.

### Response valida (200)

```json
{
  "code": 200,
  "message": "OK",
  "data": {
    "documentTotal": 115.0,
    "vatValue": 15.0,
    "maxAllowedVat": 15.0,
    "valid": true
  }
}
```

### Response invalida por limite calculado (200)

```json
{
  "code": 200,
  "message": "El iva ingresado es mayor al calculado. Por favor revise y vuelva a ingresarlo.",
  "data": {
    "documentTotal": 115.0,
    "vatValue": 20.0,
    "maxAllowedVat": 15.0,
    "valid": false
  }
}
```

### Response invalida por IVA requerido (200)

```json
{
  "code": 200,
  "message": "El valor del iva es requerido."
}
```

### Response invalida por total requerido (200)

```json
{
  "code": 200,
  "message": "El valor total es requerido.",
  "data": {
    "vatValue": 10.0,
    "valid": false
  }
}
```

### Response invalida por IVA negativo (200)

```json
{
  "code": 200,
  "message": "El valor del iva no puede ser negativo.",
  "data": {
    "documentTotal": 115.0,
    "vatValue": -1.0,
    "valid": false
  }
}
```

### Response invalida por IVA mayor o igual al total (200)

```json
{
  "code": 200,
  "message": "El valor del iva no puede ser mayor o igual al total.",
  "data": {
    "documentTotal": 50.0,
    "vatValue": 50.0,
    "valid": false
  }
}
```

---

## 4) Obtener cabecera de reposicion por id

### Resumen

- Nombre: Obtener cabecera de reposicion por identificador
- Metodo: GET
- URL: /gtfReplacementsServices/api/v1/replenishments/{replenishmentId}
- URL completa sugerida: {{host}}/gtfReplacementsServices/api/v1/replenishments/1

### Parametros

| Tipo | Nombre          | Requerido | Tipo dato | Descripcion                    |
| ---- | --------------- | --------- | --------- | ------------------------------ |
| Path | replenishmentId | Si        | Long      | Identificador de la reposicion |

### Ejemplo de request

```http
GET /gtfReplacementsServices/api/v1/replenishments/1
```

### Ejemplo de response

```json
{
  "code": 200,
  "message": "OK",
  "data": {
    "approvedVatTotal": 4.64,
    "createdDate": 1352739835523,
    "observation": "CARGA INICIAL DE CAJA CHICA",
    "replenishmentId": 1,
    "replenishmentDate": 1352739835523,
    "replenishmentType": "Caja Chica",
    "replenishmentStatus": "PAG",
    "replenishmentStatusName": "Pagado",
    "sendAlert": "0",
    "requestedSubtotal": 173.6,
    "requestedTotal": 178.24,
    "unsentDocumentsTotal": 0.0,
    "responsiblePersonId": 72643,
    "responsiblePersonDocument": "1712345678",
    "responsiblePersonName": "JUAN PEREZ",
    "transactionCode": "1",
    "vatTotal": 4.82,
    "workAreaCode": 156,
    "workAreaName": "EL JARDIN"
  }
}
```

Notas:

- `responsiblePersonName` se resuelve desde catalogo de personas por `responsiblePersonId`.
- `unsentDocumentsTotal` aplica regla legacy: si la reposicion esta en estado `PEN/PENDING`, devuelve `requestedTotal`; en otros estados devuelve `0`.
- El detalle ya no se retorna en este endpoint para evitar consultas largas/complejas.
- Para obtener lineas de detalle usar `GET /api/v1/replenishments/{replenishmentId}/details`.

### Response cuando no existe la reposicion (200)

```json
{
  "code": 200,
  "message": "OK"
}
```

## 4.1) Obtener detalle de reposicion por id

### Resumen

- Nombre: Obtener lineas de detalle de una reposicion
- Metodo: GET
- URL: /gtfReplacementsServices/api/v1/replenishments/{replenishmentId}/details
- URL completa sugerida: {{host}}/gtfReplacementsServices/api/v1/replenishments/1/details

### Parametros

| Tipo | Nombre          | Requerido | Tipo dato | Descripcion                    |
| ---- | --------------- | --------- | --------- | ------------------------------ |
| Path | replenishmentId | Si        | Long      | Identificador de la reposicion |

### Ejemplo de request

```http
GET /gtfReplacementsServices/api/v1/replenishments/1/details
```

### Ejemplo de response

```json
{
  "code": 200,
  "message": "OK",
  "data": [
    {
      "detailId": 10,
      "replenishmentId": 1,
      "documentType": "REN",
      "documentTypeName": "Retencion",
      "taxId": "1790016919001",
      "documentNumber": "001-001-000012345",
      "documentDate": 1783684800000,
      "observation": "Detalle de prueba",
      "requestedValue": 115.0,
      "approvedValue": 115.0,
      "vatValue": 15.0,
      "approvedVatValue": 15.0,
      "saleReceipt": "FACTURA",
      "accessKey": "1234567890",
      "quantity": 1,
      "measurementTypeCode": 911,
      "measurementValueCode": "UND",
      "measurementUnitName": "Unidad",
      "fileName": "factura-001.pdf",
      "billingConceptSequence": 1,
      "billingConceptDescription": "ALIMENTACION"
    }
  ]
}
```

Notas:

- `measurementUnitName` se resuelve desde catalogo corporativo (`CODTIPUNIMED` + `CODVALUNIMED`).
- `billingConceptDescription` se resuelve desde concepto de area de trabajo (`SECCONARETRA`).
- `documentTypeName` se intenta resolver desde catalogo legacy de tipos de documento (tipo 305).
- Si no existe nombre en catalogo para el codigo (`documentType`), `documentTypeName` llega en `null` y el front puede resolverlo con su propio diccionario.

---

## 5) Crear reposicion (cabecera)

### Resumen

- Nombre: Crear reposicion en estado pendiente
- Metodo: POST
- URL: /gtfReplacementsServices/api/v1/replenishment-management
- URL completa sugerida: {{host}}/gtfReplacementsServices/api/v1/replenishment-management

### Request payload minimo (obligatorio base)

```json
{
  "workAreaCode": 186,
  "responsiblePersonId": 72643,
  "observation": "Reposicion inicial"
}
```

Este payload representa el minimo requerido para create de cabecera.

Alternativa cuando front solo dispone de cedula del responsable:

```json
{
  "workAreaCode": 186,
  "responsiblePersonDocument": "1712345678",
  "observation": "Reposicion inicial"
}
```

### Request payload (ampliado de cabecera, soportado hoy)

```json
{
  "workAreaCode": 186,
  "responsiblePersonId": 72643,
  "responsiblePersonDocument": "1712345678",
  "transactionCode": "1",
  "observation": "Reposicion inicial",
  "advanceValue": 10.5,
  "beneficiaryPersonId": 81234,
  "checkResponsiblePersonId": 94567,
  "startDate": "2026-07-01T00:00:00Z",
  "endDate": "2026-07-31T23:59:59Z",
  "sendAlert": "0"
}
```

Campos de cabecera procesados actualmente en create:

- workAreaCode (obligatorio)
- responsiblePersonId (obligatorio cuando no llega responsiblePersonDocument)
- responsiblePersonDocument (obligatorio cuando no llega responsiblePersonId)
- transactionCode
- observation
- advanceValue
- beneficiaryPersonId
- checkResponsiblePersonId
- startDate
- endDate
- sendAlert

Reglas de obligatoriedad en create de cabecera:

- Obligatorios siempre: `workAreaCode`, `observation` y (`responsiblePersonId` o `responsiblePersonDocument`).
- Obligatorio condicional: `beneficiaryPersonId` cuando `transactionCode = "2"`.
- Opcionales: `transactionCode` (si no llega se aplica default legacy `"1"`), `advanceValue`, `checkResponsiblePersonId`, `startDate`, `endDate`, `sendAlert`.

Notas:

- `companyCode` se toma del token Keycloak.
- El backend fuerza `replenishmentStatus = "PENDING"` (estado pendiente) al crear.
- Si llega `responsiblePersonDocument`, backend intenta resolver `responsiblePersonId` en catalogo de personas antes de validar.
- Si `transactionCode` no llega, backend aplica default legacy `"1"`.
- Regla de pendiente por local (alineada con legacy): el bloqueo por pendiente aplica cuando `transactionCode = "1"` (Caja Chica).
- Para `transactionCode = "2"`, el create mantiene compatibilidad con flujo legacy y no bloquea por pendiente existente.

Campos que no aplican a create de cabecera (son de detalle):

- taxId / numeroRucFactura
- documentNumber
- documentType
- documentDate
- requestedValue / vatValue (de linea)

Aclaracion sobre "RUC corporacion restringido":

- Esa regla pertenece al flujo de detalle legacy (validacion de numeroRucFactura), no a la cabecera.
- En el backend migrado actual, esa validacion especifica aun no esta aplicada como regla dedicada en create/update de detalle.

Aclaracion sobre totales en create de cabecera:

- `requestedTotal`, `requestedSubtotal`, `vatTotal`, `approvedTotal` y `approvedVatTotal` no deben enviarse en create de cabecera.
- En el flujo actual los totales se inicializan con defaults y se recalculan desde endpoints de detalle.

### Response exitosa (200)

```json
{
  "code": 200,
  "message": "Creado",
  "data": {
    "workAreaCode": 186,
    "workAreaName": "EL JARDIN",
    "responsiblePersonId": 72643,
    "responsiblePersonDocument": "1712345678",
    "observation": "Reposicion inicial",
    "replenishmentStatus": "PENDING"
  }
}
```

### Response cuando ya existe pendiente en el local (200, Caja Chica)

```json
{
  "code": 200,
  "message": "Ya existe una reposicion pendiente para el local."
}
```

### Response cuando faltan responsiblePersonId y responsiblePersonDocument (200)

```json
{
  "code": 200,
  "message": "El campo responsiblePersonId o responsiblePersonDocument es obligatorio para crear la reposicion."
}
```

### Response cuando responsiblePersonDocument no existe (200)

```json
{
  "code": 200,
  "message": "No existe una persona con el responsiblePersonDocument enviado."
}
```

### Response cuando workAreaCode es invalido para la compania (200)

```json
{
  "code": 200,
  "message": "El workAreaCode enviado no existe o no pertenece a la compania."
}
```

### Response cuando responsiblePersonId no existe (200)

```json
{
  "code": 200,
  "message": "El responsiblePersonId enviado no existe."
}
```

---

## 6) Actualizar reposicion (cabecera)

### Resumen

- Nombre: Actualizar cabecera de reposicion
- Metodo: POST
- URL: /gtfReplacementsServices/api/v1/replenishment-management/{replenishmentId}
- URL completa sugerida: {{host}}/gtfReplacementsServices/api/v1/replenishment-management/1

### Parametros

| Tipo | Nombre          | Requerido | Tipo dato | Descripcion                                 |
| ---- | --------------- | --------- | --------- | ------------------------------------------- |
| Path | replenishmentId | Si        | Long      | Identificador de la reposicion a actualizar |

### Request payload

```json
{
  "workAreaCode": 186,
  "transactionCode": "1",
  "observation": "Actualizacion cabecera",
  "advanceValue": 10.5,
  "responsiblePersonId": 72643,
  "beneficiaryPersonId": 81234,
  "checkResponsiblePersonId": 94567,
  "startDate": "2026-07-01T00:00:00Z",
  "endDate": "2026-07-31T23:59:59Z"
}
```

Campos de cabecera soportados actualmente en update:

- workAreaCode
- transactionCode
- observation
- advanceValue
- responsiblePersonId
- beneficiaryPersonId
- checkResponsiblePersonId
- startDate
- endDate

Notas:

- `replenishmentId` se toma del path (`/{replenishmentId}`), no del body.
- `companyCode` se toma del token (Keycloak), no del body.
- El backend conserva/normaliza `replenishmentStatus = "PENDING"` durante la actualizacion.

Reglas de negocio aplicadas:

- Si la reposicion no existe, retorna `code: 200` con mensaje `No existe la reposicion solicitada.`.
- Si la reposicion existe y su estado no es pendiente (`PENDING`), retorna `code: 200` con mensaje `Solo se puede editar una reposicion en estado pendiente.`.
- Si aplica la actualizacion, retorna `code: 200` con mensaje `OK` y se conserva estado pendiente (`PENDING`).

### Response exitosa (200)

```json
{
  "code": 200,
  "message": "OK",
  "data": {
    "replenishmentId": 1,
    "workAreaCode": 186,
    "workAreaName": "EL JARDIN",
    "responsiblePersonId": 72643,
    "responsiblePersonDocument": "1712345678",
    "observation": "Actualizacion cabecera",
    "replenishmentStatus": "PENDING"
  }
}
```

### Response no encontrado (200)

```json
{
  "code": 200,
  "message": "No existe la reposicion solicitada."
}
```

### Response conflicto por estado invalido (200)

```json
{
  "code": 200,
  "message": "Solo se puede editar una reposicion en estado pendiente."
}
```

---

## 7) Validar documento duplicado

### Resumen

- Nombre: Validar documento duplicado en detalle de reposicion
- Metodo: POST
- URL: /gtfReplacementsServices/api/v1/replenishments/validate-duplicate
- URL completa sugerida: {{host}}/gtfReplacementsServices/api/v1/replenishments/validate-duplicate

### Request payload

```json
{
  "taxId": "1790016919001",
  "documentNumber": "001-001-000012345"
}
```

Notas:

- `companyCode` se toma del token Keycloak.
- La validacion se ejecuta por `companyCode + taxId + documentNumber`.
- El backend normaliza `taxId` y `documentNumber` con trim antes de validar.

### Response cuando no existe duplicado (200)

```json
{
  "code": 200,
  "message": "OK",
  "data": {
    "taxId": "1790016919001",
    "documentNumber": "001-001-000012345",
    "duplicate": false,
    "valid": true
  }
}
```

### Response cuando existe duplicado (200)

```json
{
  "code": 200,
  "message": "Ya se encuentra ingresado un documento con caracteristicas similares.",
  "data": {
    "taxId": "1790016919001",
    "documentNumber": "001-001-000012345",
    "duplicate": true,
    "valid": false
  }
}
```

---

## 8) Validar pre-check de nuevo

### Resumen

- Nombre: Validar reglas previas para habilitar nuevo flujo de reposicion
- Metodo: POST
- URL: /gtfReplacementsServices/api/v1/replenishment-management/validate-new
- URL completa sugerida: {{host}}/gtfReplacementsServices/api/v1/replenishment-management/validate-new

### Request payload

```json
{
  "workAreaCode": 186,
  "transactionCode": "1"
}
```

Notas:

- `companyCode` se toma del token Keycloak.
- `workAreaCode` es obligatorio para la validacion.
- `transactionCode` soportado: `1` (Caja Chica) y `2` (Gastos Personales).
- Si `transactionCode` no llega, el backend aplica por default `1`.
- La respuesta incluye `transactionSelectionRequired` para indicar si front debe solicitar seleccion explicita de tipo de transaccion.
- La respuesta incluye `supportedTransactionCodes` para poblar el selector (`["1", "2"]`).
- La respuesta incluye `issuedCheckReplenishment` para reflejar condicion de cheque emitido bloqueante.

Reglas de negocio aplicadas:

- Si `workAreaCode` no existe para la compania, retorna validacion no aprobada.
- Si existe reposicion en estado cheque emitido para el local, bloquea el flujo de nuevo.
- Si `transactionCode = 1` y ya existe reposicion pendiente por local, bloquea el flujo de nuevo.
- Si `transactionCode = 1` y no existe fondo asignado, marca que se requiere carga inicial de fondos.
- Si `transactionCode = 2`, permite continuar aunque exista reposicion pendiente.

### Response valida (200)

```json
{
  "code": 200,
  "message": "OK",
  "data": {
    "workAreaCode": 186,
    "transactionCode": "2",
    "transactionSelectionRequired": false,
    "supportedTransactionCodes": ["1", "2"],
    "issuedCheckReplenishment": false,
    "pendingReplenishment": true,
    "initialFundLoadRequired": false,
    "valid": true
  }
}
```

### Response invalida por cheque emitido en el local (200)

```json
{
  "code": 200,
  "message": "Existen reposiciones con cheque emitido para el local. Regularice antes de crear una nueva reposicion.",
  "data": {
    "workAreaCode": 186,
    "transactionCode": "1",
    "transactionSelectionRequired": false,
    "supportedTransactionCodes": ["1", "2"],
    "issuedCheckReplenishment": true,
    "pendingReplenishment": false,
    "initialFundLoadRequired": false,
    "valid": false
  }
}
```

### Response invalida por reposicion pendiente en Caja Chica (200)

```json
{
  "code": 200,
  "message": "Ya existe una reposicion pendiente para el local.",
  "data": {
    "workAreaCode": 186,
    "transactionCode": "1",
    "transactionSelectionRequired": false,
    "supportedTransactionCodes": ["1", "2"],
    "issuedCheckReplenishment": false,
    "pendingReplenishment": true,
    "initialFundLoadRequired": false,
    "valid": false
  }
}
```

### Response invalida por carga inicial requerida (200)

```json
{
  "code": 200,
  "message": "El local requiere carga inicial de fondos antes de crear una reposicion.",
  "data": {
    "workAreaCode": 186,
    "transactionCode": "1",
    "transactionSelectionRequired": false,
    "supportedTransactionCodes": ["1", "2"],
    "issuedCheckReplenishment": false,
    "pendingReplenishment": false,
    "initialFundLoadRequired": true,
    "valid": false,
    "fundHeader": {
      "workAreaCode": "186",
      "workAreaName": "EL JARDIN",
      "assignedFund": 0.0
    }
  }
}
```

### Response invalida por transactionCode no soportado (200)

```json
{
  "code": 200,
  "message": "El campo transactionCode enviado no es valido. Use 1 para Caja Chica o 2 para Gastos Personales.",
  "data": {
    "workAreaCode": 186,
    "transactionCode": "9",
    "transactionSelectionRequired": false,
    "supportedTransactionCodes": ["1", "2"],
    "issuedCheckReplenishment": false,
    "pendingReplenishment": false,
    "initialFundLoadRequired": false,
    "valid": false
  }
}
```

### Response valida cuando no se envia transactionCode (200)

```json
{
  "code": 200,
  "message": "OK",
  "data": {
    "workAreaCode": 186,
    "transactionCode": "1",
    "transactionSelectionRequired": true,
    "supportedTransactionCodes": ["1", "2"],
    "issuedCheckReplenishment": false,
    "pendingReplenishment": false,
    "initialFundLoadRequired": false,
    "valid": true
  }
}
```

---

## 8.1) Obtener conceptos de facturacion por contexto de reposicion

### Resumen

- Nombre: Obtener conceptos de facturacion agrupados por categoria para el popup de Concepto pago
- Metodo: GET
- URL: /gtfReplacementsServices/api/v1/replenishments/billing-concepts
- URL completa sugerida: {{host}}/gtfReplacementsServices/api/v1/replenishments/billing-concepts

### Parametros

| Tipo  | Nombre          | Requerido | Tipo dato | Descripcion                                                                              |
| ----- | --------------- | --------- | --------- | ---------------------------------------------------------------------------------------- |
| Query | workAreaCode    | Si        | Integer   | Codigo de area de trabajo para filtrar conceptos configurados                            |
| Query | transactionCode | No        | String    | Tipo de transaccion (`1` caja chica, `2` gastos personales). Si no llega, aplica `1`     |
| Query | documentType    | Si        | String    | Tipo de documento del detalle (`FAC`, `FEL`, `NOV`, etc.) para resolver tipo de concepto |

### Regla de tipo de concepto

- Para `FAC`, `FEL` y `NOV`, el backend filtra tipo de concepto `LC` (liquidacion compra).
- Para el resto de documentos, el backend filtra tipo de concepto `LO` (liquidacion otros).
- Si `documentType` no llega, el backend responde `200` con `data: []` y mensaje funcional:
  `El campo documentType es obligatorio para consultar conceptos de pago.`

### Ejemplo de request

```http
GET /gtfReplacementsServices/api/v1/replenishments/billing-concepts?workAreaCode=186&transactionCode=1&documentType=FAC
```

### Ejemplo de response (200)

```json
{
  "code": 200,
  "message": "OK",
  "data": [
    {
      "categoryCode": "CAT01",
      "categoryName": "MOVILIZACION",
      "concepts": [
        {
          "billingConceptSequence": 101,
          "billingConceptCode": 2001,
          "billingConceptName": "PEAJE",
          "billingConceptDescription": "PEAJE VIA TERRESTRE",
          "accountNumber": "510101",
          "categoryCode": "CAT01"
        }
      ]
    }
  ]
}
```

Notas:

- `companyCode` no se envia por query; se toma del token de Keycloak.
- Si el contexto no es valido (local inexistente o transaccion no soportada), responde `200` con `data: []`.
- La agrupacion mantiene el orden del catalogo de categorias de conceptos (`304`) cuando exista informacion.

---

## 8.2) Prevalidar fila de detalle con cabecera persistida

### Resumen

- Nombre: Validar reglas antes de agregar fila de detalle (con cabecera existente)
- Metodo: POST
- URL: /gtfReplacementsServices/api/v1/replenishments/{replenishmentId}/details/validate-add
- URL completa sugerida: {{host}}/gtfReplacementsServices/api/v1/replenishments/{replenishmentId}/details/validate-add

### Parametros

| Tipo | Nombre          | Requerido | Tipo dato                          | Descripcion                                                            |
| ---- | --------------- | --------- | ---------------------------------- | ---------------------------------------------------------------------- |
| Path | replenishmentId | Si        | Long                               | Identificador de cabecera ya creada en estado pendiente                |
| Body | request         | Si        | ReplenishmentDetailAddValidationVo | Objeto con `detail` (obligatorio) y `currentRequestedTotal` (opcional) |

### Request payload (ejemplo)

```json
{
  "detail": {
    "documentType": "FAC",
    "taxId": "1790016919001",
    "documentNumber": "001-001-000012345",
    "documentDate": 1783684800000,
    "billingConceptSequence": 1,
    "requestedValue": 20.0,
    "vatValue": 3.0
  },
  "currentRequestedTotal": 40.0
}
```

Notas:

- Si `currentRequestedTotal` no llega, backend calcula el total actual desde detalle persistido de la reposicion.
- Requiere que la cabecera exista y este en estado pendiente.
- Reutiliza reglas de detalle: obligatorios, IVA, duplicado y maximo por documento.
- Para `transactionCode = "1"` aplica reglas operativas de fondo: saldo disponible, bloqueo por porcentaje y alerta preventiva.
- Cuando falla una regla de negocio, `message` retorna un texto generico y el detalle de errores se entrega en `errors` (nivel raiz de `BaseResponseVo`).

Semantica de `data`:

- `valid = true`: la fila cumple reglas y se puede agregar.
- `valid = false`: existe al menos una regla incumplida; revisar `errors[]`.
- `blockedByAvailableBalance = true`: el total proyectado (`currentRequestedTotal + requestedValue`) supera el saldo disponible (`cashBalance`).
- `blockedByUsagePercentage = true`: el total proyectado supera el umbral maximo de consumo (`usagePercentage`) sobre el fondo asignado.
- `sendAlert = true`: no bloquea; es advertencia de envio recomendado porque se supero el umbral de alerta (`alertPercentage`) sin reglas bloqueantes.

Orden recomendado en front:

1. Si `valid = false`, bloquear agregado y mostrar `errors[]`.
2. Si `blockedByAvailableBalance` o `blockedByUsagePercentage` estan en `true`, tratar como bloqueo de negocio.
3. Si `valid = true` y `sendAlert = true`, permitir agregar fila y mostrar advertencia visual.

### Response valida (200)

```json
{
  "code": 200,
  "message": "OK",
  "errors": [],
  "data": {
    "sendAlert": true,
    "blockedByAvailableBalance": false,
    "blockedByUsagePercentage": false,
    "valid": true
  }
}
```

### Response invalida por reposicion no encontrada (200)

```json
{
  "code": 200,
  "message": "La fila de detalle presenta errores de validacion.",
  "errors": ["No existe la reposicion solicitada."],
  "data": {
    "sendAlert": false,
    "blockedByAvailableBalance": false,
    "blockedByUsagePercentage": false,
    "valid": false
  }
}
```

### Response invalida por estado de cabecera (200)

```json
{
  "code": 200,
  "message": "La fila de detalle presenta errores de validacion.",
  "errors": [
    "Solo se puede agregar detalle cuando la reposicion esta en estado pendiente."
  ],
  "data": {
    "sendAlert": false,
    "blockedByAvailableBalance": false,
    "blockedByUsagePercentage": false,
    "valid": false
  }
}
```

---

## 8.3) Prevalidar fila de detalle sin cabecera persistida

### Resumen

- Nombre: Validar reglas antes de agregar fila de detalle (sin cabecera creada)
- Metodo: POST
- URL: /gtfReplacementsServices/api/v1/replenishments/details/validate-add
- URL completa sugerida: {{host}}/gtfReplacementsServices/api/v1/replenishments/details/validate-add

### Parametros

| Tipo | Nombre  | Requerido | Tipo dato                                   | Descripcion                                                                           |
| ---- | ------- | --------- | ------------------------------------------- | ------------------------------------------------------------------------------------- |
| Body | request | Si        | ReplenishmentDetailDraftValidationRequestVo | Objeto con contexto minimo de cabecera (`workAreaCode`, `transactionCode`) y `detail` |

### Request payload (ejemplo)

```json
{
  "workAreaCode": 186,
  "transactionCode": "1",
  "detail": {
    "documentType": "FAC",
    "taxId": "1790016919001",
    "documentNumber": "001-001-000012345",
    "documentDate": 1783684800000,
    "billingConceptSequence": 1,
    "requestedValue": 20.0,
    "vatValue": 3.0
  },
  "currentRequestedTotal": 40.0
}
```

Notas:

- `workAreaCode` es obligatorio.
- `transactionCode` es opcional; si no llega se asume `"1"`.
- Si llega `transactionCode` debe ser `"1"`.
- En `detail` son obligatorios: `billingConceptSequence`, `documentType`, `documentDate`, `requestedValue` y `vatValue`.
- No requiere `replenishmentId`; pensado para filas temporales en UI antes de crear cabecera.
- Valida duplicado por `companyCode + taxId + documentNumber` contra datos existentes.
- El endpoint aplica Bean Validation de entrada (`@Valid`): errores de campos requeridos/vacios retornan HTTP 400 antes de ejecutar reglas de negocio.
- En errores de negocio, `message` es generico y los mensajes puntuales se entregan en `errors` (nivel raiz de `BaseResponseVo`).

Semantica de `data`:

- `valid = true`: la fila cumple reglas y se puede agregar.
- `valid = false`: existe al menos una regla incumplida; revisar `errors[]`.
- `blockedByAvailableBalance = true`: el total proyectado (`currentRequestedTotal + requestedValue`) supera el saldo disponible (`cashBalance`).
- `blockedByUsagePercentage = true`: el total proyectado supera el umbral maximo de consumo (`usagePercentage`) sobre el fondo asignado.
- `sendAlert = true`: no bloquea; es advertencia de envio recomendado porque se supero el umbral de alerta (`alertPercentage`) sin reglas bloqueantes.

Regla de alcance:

- Las banderas `blockedByAvailableBalance`, `blockedByUsagePercentage` y `sendAlert` aplican cuando `transactionCode = "1"` (Caja Chica).
- Si `transactionCode` no es soportado, el endpoint retorna error funcional y `valid = false`.

Orden recomendado en front:

1. Si `valid = false`, bloquear agregado y mostrar `errors[]`.
2. Si `blockedByAvailableBalance` o `blockedByUsagePercentage` estan en `true`, tratar como bloqueo de negocio.
3. Si `valid = true` y `sendAlert = true`, permitir agregar fila y mostrar advertencia visual.

### Response valida (200)

```json
{
  "code": 200,
  "message": "OK",
  "errors": [],
  "data": {
    "sendAlert": true,
    "blockedByAvailableBalance": false,
    "blockedByUsagePercentage": false,
    "valid": true
  }
}
```

### Response invalida de entrada por campos requeridos o formato (400)

```json
{
  "status": 400,
  "error": "Bad Request"
}
```

Casos tipicos que devuelven 400 en este endpoint:

- `workAreaCode` nulo.
- `detail` nulo.
- `detail.documentType` vacio.
- `detail.documentDate` nulo.
- `detail.requestedValue` nulo.
- `detail.vatValue` nulo.

### Response invalida de negocio (200)

Cuando el request es valido a nivel de estructura, pero falla una o varias reglas de negocio (IVA, maximo por documento, porcentaje, saldo, duplicado o `transactionCode` no soportado), el endpoint mantiene `code: 200`, retorna `message` generico y acumula el detalle en `errors` (nivel raiz).

Ejemplo:

```json
{
  "code": 200,
  "message": "La fila de detalle presenta errores de validacion.",
  "errors": [
    "El valor de su detalle excede el saldo disponible del fondo de caja chica.",
    "La presente reposicion sobrepaso el 60% del fondo asignado de $200.00. Favor realice el envio de la reposicion."
  ],
  "data": {
    "sendAlert": false,
    "blockedByAvailableBalance": true,
    "blockedByUsagePercentage": true,
    "valid": false
  }
}
```

---

## 9) Crear detalle (ruta consolidada)

### Nota de contrato

El endpoint `POST /gtfReplacementsServices/api/v1/replenishment-management/{replenishmentId}/details` se consolidó para recibir lotes de detalle (array JSON).

Para request/response completos del contrato vigente, usar la sección siguiente.

---

## 10) Crear lineas de detalle en batch

### Resumen

- Nombre: Crear multiples lineas de detalle para una reposicion en una sola solicitud
- Metodo: POST
- URL: /gtfReplacementsServices/api/v1/replenishment-management/{replenishmentId}/details
- URL completa sugerida: {{host}}/gtfReplacementsServices/api/v1/replenishment-management/{replenishmentId}/details

### Parametros

| Tipo | Nombre          | Requerido | Tipo dato      | Descripcion                                |
| ---- | --------------- | --------- | -------------- | ------------------------------------------ |
| Path | replenishmentId | Si        | Long           | Identificador de la reposicion de cabecera |
| Body | request         | Si        | Array<Detalle> | Coleccion de lineas de detalle a registrar |

### Request payload (ejemplo)

```json
[
  {
    "documentType": "FAC",
    "taxId": "1790016919001",
    "documentNumber": "001-001-000012346",
    "documentDate": 1783684800000,
    "observation": "Detalle batch 1",
    "requestedValue": 115.0,
    "vatValue": 15.0,
    "billingConceptSequence": 1
  },
  {
    "documentType": "FAC",
    "taxId": "1790016919001",
    "documentNumber": "001-001-000012347",
    "documentDate": 1783684800000,
    "observation": "Detalle batch 2",
    "requestedValue": 120.0,
    "vatValue": 15.65,
    "billingConceptSequence": 1
  }
]
```

### Payloads de prueba listos (Front/Postman)

#### Caso A: Lote valido (2 lineas)

```json
[
  {
    "documentType": "FAC",
    "taxId": "1790016919001",
    "documentNumber": "001-001-000012346",
    "documentDate": 1783684800000,
    "observation": "Detalle batch 1",
    "requestedValue": 115.0,
    "vatValue": 15.0,
    "billingConceptSequence": 1
  },
  {
    "documentType": "FAC",
    "taxId": "1790016919001",
    "documentNumber": "001-001-000012347",
    "documentDate": 1783684800000,
    "observation": "Detalle batch 2",
    "requestedValue": 120.0,
    "vatValue": 15.65,
    "billingConceptSequence": 1
  }
]
```

#### Caso B: Duplicado dentro del mismo lote (debe bloquear)

```json
[
  {
    "documentType": "FAC",
    "taxId": "1790016919001",
    "documentNumber": "001-001-000012345",
    "documentDate": 1783684800000,
    "observation": "Detalle duplicado 1",
    "requestedValue": 115.0,
    "vatValue": 15.0,
    "billingConceptSequence": 1
  },
  {
    "documentType": "FAC",
    "taxId": "1790016919001",
    "documentNumber": "001-001-000012345",
    "documentDate": 1783684800000,
    "observation": "Detalle duplicado 2",
    "requestedValue": 115.0,
    "vatValue": 15.0,
    "billingConceptSequence": 1
  }
]
```

Notas:

- `companyCode` se toma del token Keycloak.
- Se aplican las mismas validaciones de negocio del endpoint unitario por cada linea del lote.
- El backend valida todo el lote antes de persistir; si una linea falla, no se persiste ninguna.
- Se valida duplicado dentro del mismo request batch por `taxId + documentNumber`.
- Si el lote es valido, el backend persiste todas las lineas y recalcula cabecera una sola vez al final.

### Response exitosa (200)

```json
{
  "code": 200,
  "message": "OK",
  "data": [
    {
      "detailId": 200,
      "replenishmentId": 1,
      "documentType": "FAC",
      "taxId": "1790016919001",
      "documentNumber": "001-001-000012346",
      "billingConceptSequence": 1,
      "requestedValue": 115.0,
      "vatValue": 15.0
    },
    {
      "detailId": 201,
      "replenishmentId": 1,
      "documentType": "FAC",
      "taxId": "1790016919001",
      "documentNumber": "001-001-000012347",
      "billingConceptSequence": 1,
      "requestedValue": 120.0,
      "vatValue": 15.65
    }
  ]
}
```

### Response validacion funcional (200)

```json
{
  "code": 200,
  "message": "Debe enviar al menos un detalle para registrar."
}
```

```json
{
  "code": 200,
  "message": "Ya se encuentra ingresado un documento con caracteristicas similares."
}
```

---

## 11) Editar linea de detalle

### Resumen

- Nombre: Editar linea de detalle para una reposicion
- Metodo: POST
- URL: /gtfReplacementsServices/api/v1/replenishment-management/{replenishmentId}/details/{detailId}
- URL completa sugerida: {{host}}/gtfReplacementsServices/api/v1/replenishment-management/{replenishmentId}/details/{detailId}

### Parametros

| Tipo | Nombre          | Requerido | Tipo dato | Descripcion                                       |
| ---- | --------------- | --------- | --------- | ------------------------------------------------- |
| Path | replenishmentId | Si        | Long      | Identificador de la reposicion de cabecera        |
| Path | detailId        | Si        | Long      | Identificador de la linea de detalle a actualizar |

### Request payload

```json
{
  "documentType": "FAC",
  "taxId": "1790016919001",
  "documentNumber": "001-001-000099999",
  "documentDate": 1783684800000,
  "observation": "Detalle editado",
  "saleReceipt": "FACTURA",
  "quantity": 1,
  "measurementTypeCode": 911,
  "measurementValueCode": "UND",
  "fileName": "factura-001.pdf",
  "requestedValue": 115.0,
  "vatValue": 15.0,
  "billingConceptSequence": 1
}
```

Notas:

- `companyCode` se toma del token Keycloak.
- El backend solo permite editar detalle cuando la cabecera existe y esta en estado pendiente (`PENDING`/`PEN`).
- El detalle a editar debe existir en estado activo para la compania y la reposicion.
- Si cambia `taxId + documentNumber`, se valida no duplicidad por `companyCode + taxId + documentNumber`.
- Al guardar, el backend recalcula en cabecera: `requestedTotal`, `vatTotal` y `requestedSubtotal`.
- Los campos opcionales de detalle (`saleReceipt`, `measurementTypeCode`, `measurementValueCode`, `fileName`) se pueden actualizar en el mismo request.

Reglas de negocio aplicadas:

- Si la reposicion no existe, retorna `code: 200` con mensaje `No existe la reposicion solicitada.`.
- Si la reposicion no esta en estado editable, retorna `code: 200` con mensaje `Solo se puede editar detalle cuando la reposicion esta en estado pendiente.`.
- Si el detalle no existe, retorna `code: 200` con mensaje `No existe el detalle solicitado.`.
- Si el documento ya existe, retorna `code: 200` con mensaje `Ya se encuentra ingresado un documento con caracteristicas similares.`.
- Si la validacion pasa, retorna `code: 200`, `message: "OK"` y el detalle actualizado en `data`.

### Response exitosa (200)

```json
{
  "code": 200,
  "message": "OK",
  "data": {
    "detailId": 10,
    "replenishmentId": 1,
    "documentType": "FAC",
    "taxId": "1790016919001",
    "documentNumber": "001-001-000099999",
    "saleReceipt": "FACTURA",
    "quantity": 1,
    "measurementTypeCode": 911,
    "measurementValueCode": "UND",
    "fileName": "factura-001.pdf",
    "billingConceptSequence": 1,
    "requestedValue": 115.0,
    "vatValue": 15.0
  }
}
```

### Response validacion funcional (200)

```json
{
  "code": 200,
  "message": "No existe el detalle solicitado."
}
```

---

## 12) Obtener valores de catalogo generico

### Resumen

- Nombre: Listar valores de catalogo por tipo
- Metodo: GET
- URL: /gtfReplacementsServices/api/v1/catalogs/{catalogTypeCode}/values
- URL completa sugerida: {{host}}/gtfReplacementsServices/api/v1/catalogs/303/values

### Parametros

| Tipo  | Nombre          | Requerido | Tipo dato | Descripcion                                                          |
| ----- | --------------- | --------- | --------- | -------------------------------------------------------------------- |
| Path  | catalogTypeCode | Si        | Integer   | Codigo del tipo de catalogo (ejemplo: 303 para estado de reposicion) |
| Query | onlyActive      | No        | Boolean   | Si es true o null, filtra solo valores activos                       |
| Query | numericValue    | No        | Long      | Filtra por valor numerico exacto del catalogo (`VALORNUMERICO`)      |

### Ejemplo de request

```http
GET /gtfReplacementsServices/api/v1/catalogs/303/values?onlyActive=true
```

### Ejemplo de request con filtro numerico

```http
GET /gtfReplacementsServices/api/v1/catalogs/303/values?onlyActive=true&numericValue=1
```

### Ejemplo de response (200)

```json
{
  "code": 200,
  "message": "OK",
  "data": [
    {
      "catalogTypeCode": 303,
      "catalogValueCode": "PEN",
      "catalogValueName": "Pendiente",
      "shortName": "PEND",
      "state": "ACT",
      "numericValue": null,
      "position": 1,
      "parentCatalogTypeCode": null,
      "parentCatalogValueCode": null
    }
  ]
}
```

### Ejemplo de response sin resultados (200)

```json
{
  "code": 200,
  "message": "OK",
  "data": []
}
```

Notas:

- Endpoint transversal reutilizable para cualquier modulo del sistema.
- Cuando `onlyActive` no se envia, el backend aplica filtro de activos por defecto.
- Estados considerados activos en catalogo: `ACT`, `1`, `A`.
- El endpoint se usa, entre otros casos, para resolver etiquetas legibles de estado en reposiciones.

---

## 13) Obtener responsables para envio

### Resumen

- Nombre: Obtener responsables por area de trabajo
- Metodo: GET
- URL: /gtfReplacementsServices/api/v1/replenishments/responsibles
- URL completa sugerida: {{host}}/gtfReplacementsServices/api/v1/replenishments/responsibles?workAreaCode=186

### Parametros

| Tipo  | Nombre       | Requerido | Tipo dato | Descripcion                                                                |
| ----- | ------------ | --------- | --------- | -------------------------------------------------------------------------- |
| Query | workAreaCode | Si        | Integer   | Codigo de area de trabajo para cargar responsables de pago por ventanilla. |

### Ejemplo de request

```http
GET /gtfReplacementsServices/api/v1/replenishments/responsibles?workAreaCode=186
```

### Ejemplo de response con data (200)

```json
{
  "code": 200,
  "message": "OK",
  "data": [
    {
      "personId": 72643,
      "fullName": "NOMBRE RESPONSABLE",
      "type": "PRINCIPAL"
    }
  ]
}
```

### Ejemplo de response sin data (200)

```json
{
  "code": 200,
  "message": "OK"
}
```

Notas:

- `companyCode` no se envia: se toma del token.
- El endpoint se usa en el flujo de envio para poblar el selector de responsable.
- La clasificacion del responsable se expone en `type` (por ejemplo `PRINCIPAL` o `SECUNDARIO`).

---

## 14) Crear reposicion con detalles (endpoint principal)

### Resumen

- Nombre: Crear cabecera y lineas de detalle en una sola API
- Metodo: POST
- URL: /gtfReplacementsServices/api/v1/replenishment-management
- URL completa sugerida: {{host}}/gtfReplacementsServices/api/v1/replenishment-management

### Parametros

| Tipo | Nombre             | Requerido | Tipo dato | Descripcion                                                            |
| ---- | ------------------ | --------- | --------- | ---------------------------------------------------------------------- |
| Body | campos de cabecera | Si        | Object    | Campos de cabecera en raiz del body (mismas reglas de create cabecera) |
| Body | details            | Si        | Array     | Lineas de detalle (mismas reglas de create details batch)              |

### Request payload (ejemplo)

```json
{
  "workAreaCode": 105,
  "transactionCode": "1",
  "responsiblePersonDocument": "1708682446",
  "observation": "Creacion atomica cabecera + detalle",
  "details": [
    {
      "documentType": "FAC",
      "taxId": "1790016919001",
      "documentNumber": "001-001-000012346",
      "documentDate": 1783684800000,
      "observation": "Detalle atomico 1",
      "requestedValue": 50.0,
      "vatValue": 6.52,
      "billingConceptSequence": 1
    },
    {
      "documentType": "FAC",
      "taxId": "1790016919001",
      "documentNumber": "001-001-000012347",
      "documentDate": 1783684800000,
      "observation": "Detalle atomico 2",
      "requestedValue": 49.0,
      "vatValue": 6.39,
      "billingConceptSequence": 1
    }
  ]
}
```

### Response exitosa (200)

```json
{
  "code": 200,
  "message": "Creado",
  "data": {
    "replenishmentId": 39987,
    "workAreaCode": 105,
    "details": [
      {
        "detailId": 200,
        "replenishmentId": 39987,
        "documentType": "FAC",
        "taxId": "1790016919001",
        "documentNumber": "001-001-000012346",
        "billingConceptSequence": 1,
        "requestedValue": 50.0,
        "vatValue": 6.52
      },
      {
        "detailId": 201,
        "replenishmentId": 39987,
        "documentType": "FAC",
        "taxId": "1790016919001",
        "documentNumber": "001-001-000012347",
        "billingConceptSequence": 1,
        "requestedValue": 49.0,
        "vatValue": 6.39
      }
    ]
  }
}
```

### Response validacion funcional (200)

```json
{
  "code": 200,
  "message": "No se pudo completar la creacion. Revise los errores reportados.",
  "errors": [
    "Fila 1: El detalle que se va a ingresar ya se encuentra ingresado, verifique por favor."
  ]
}
```

Notas:

- Se reutiliza el endpoint principal de creacion (`POST /api/v1/replenishment-management`).
- El endpoint ejecuta create de cabecera y create batch de detalle en una sola transaccion.
- Si falla cualquier validacion de detalle, se revierte tambien la cabecera (sin cabeceras huerfanas).
- El contrato de errores funcionales mantiene `code: 200` con `errors[]` para consistencia con el flujo de management.

---

## Regla de actualizacion

Por cada endpoint nuevo se debe agregar en este archivo:

1. Metodo HTTP
2. URL completa
3. Parametros (query, path, headers, body)
4. Ejemplo de request payload
5. Ejemplo de response
