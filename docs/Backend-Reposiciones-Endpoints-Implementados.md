# Backend Reposiciones - Endpoints Implementados

## Proposito

Documentar los endpoints ya implementados del modulo de reposiciones con un formato estandar:

- Metodo HTTP
- URL
- Parametros
- Ejemplo de request
- Ejemplo de response

Este archivo se actualiza cada vez que se crea o modifica un endpoint.

## Alcance

- Modulo: replenishments
- Base URL aplicacion: /gtfReplacementsServices
- Base path controller: /api/v1/replenishments

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
- URL: /gtfReplacementsServices/api/v1/replenishments/current-fund

### Parametros

| Tipo  | Nombre       | Requerido | Tipo dato | Descripcion               |
| ----- | ------------ | --------- | --------- | ------------------------- |
| Query | workAreaCode | Si        | Integer   | Codigo de area de trabajo |

### Ejemplo de request

```http
GET /gtfReplacementsServices/api/v1/replenishments/current-fund?workAreaCode=186
```

### Ejemplo de response con data

```json
{
  "code": 200,
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
        "replenishmentStatus": "PENDING",
        "workAreaCode": 186
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

### Request payload

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

## 4) Obtener reposicion por id

### Resumen

- Nombre: Obtener detalle de reposicion por identificador
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
    "requestedSubtotal": 173.6,
    "requestedTotal": 178.24,
    "responsiblePersonId": 72643,
    "transactionCode": "1",
    "vatTotal": 4.82,
    "workAreaCode": 156
  }
}
```

### Response cuando no existe la reposicion (200)

```json
{
  "code": 200,
  "message": "OK"
}
```

---

## 5) Crear reposicion (cabecera)

### Resumen

- Nombre: Crear reposicion en estado pendiente
- Metodo: POST
- URL: /gtfReplacementsServices/api/v1/replenishments
- URL completa sugerida: {{host}}/gtfReplacementsServices/api/v1/replenishments

### Request payload

```json
{
  "workAreaCode": 186,
  "responsiblePersonId": 72643,
  "observation": "Reposicion inicial"
}
```

Notas:

- `companyCode` se toma del token Keycloak.
- El backend fuerza `replenishmentStatus = "PENDING"` al crear.
- `responsiblePersonId` es obligatorio (mapea a `CODIGOPERSONARESPONSABLE` en DB2).

### Response exitosa (200)

```json
{
  "code": 200,
  "message": "Created",
  "data": {
    "workAreaCode": 186,
    "responsiblePersonId": 72643,
    "observation": "Reposicion inicial",
    "replenishmentStatus": "PENDING"
  }
}
```

### Response cuando ya existe pendiente en el local (200)

```json
{
  "code": 200,
  "message": "Ya existe una reposicion pendiente para el local."
}
```

### Response cuando falta responsiblePersonId (200)

```json
{
  "code": 200,
  "message": "El campo responsiblePersonId es obligatorio para crear la reposicion."
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
- Metodo: PUT
- URL: /gtfReplacementsServices/api/v1/replenishments/{replenishmentId}
- URL completa sugerida: {{host}}/gtfReplacementsServices/api/v1/replenishments/1

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
- Si la reposicion existe y su estado no es `PENDING`, retorna `code: 200` con mensaje `Solo se puede editar una reposicion en estado PENDING.`.
- Si aplica la actualizacion, retorna `code: 200` con mensaje `OK` y se conserva estado `PENDING`.

### Response exitosa (200)

```json
{
  "code": 200,
  "message": "OK",
  "data": {
    "replenishmentId": 1,
    "workAreaCode": 186,
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
  "message": "Solo se puede editar una reposicion en estado PENDING."
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

## Regla de actualizacion

Por cada endpoint nuevo se debe agregar en este archivo:

1. Metodo HTTP
2. URL completa
3. Parametros (query, path, headers, body)
4. Ejemplo de request payload
5. Ejemplo de response
