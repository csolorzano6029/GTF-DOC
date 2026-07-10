# BACK-FRONT - Mapeo API vs Frontend

Fecha de corte: 2026-07-10

## 1. Objetivo

Relacionar cada API del backend con su uso en frontend:

- menu u opcion
- pantalla o componente
- boton o accion
- estado actual de implementacion

## 2. Convencion de nombres (ajuste solicitado)

Regla aplicada desde este punto:

- En nombres de metodos Java no se usara prefijo `get`; se usara `find` cuando corresponda.
- En endpoints HTTP se mantiene la semantica REST actual (verbos HTTP como `GET`, `POST`, `PUT`), porque eso es parte del protocolo.
- En rutas URL no se usa `get` en el path; se mantiene estilo actual por recurso.

Ejemplos aplicados:

- `getCurrentFundHeader` -> `findCurrentFundHeader`
- `getCurrentFund` -> `findCurrentFund`
- `getByCode` -> `findByCode`
- `getNextReplenishmentId` -> `findNextReplenishmentId`

## 3. Estado de endpoints existentes (antes de P0)

Decision:

- Los endpoints actuales se mantienen y NO se reemplazan.
- Se completan por evolucion incremental para no romper front ni pruebas ya validadas.

Brecha puntual de los endpoints existentes:

| Endpoint actual | Estado | Que falta para quedar completo contra legacy |
| --- | --- | --- |
| GET /replenishments/current-fund | Parcial | Incluir validaciones operativas de precheck (bloqueo/alerta por porcentaje, solicitudes pendientes y cheque emitido) |
| POST /replenishments/findByFilter | Parcial | Forzar filtro por area de trabajo del usuario y mejorar soporte de filtros de fecha/estado como en legacy |
| GET /replenishments/{id} | Parcial | Incluir detalle completo de documentos y contexto operativo para edicion/revision |
| POST /replenishments | Parcial | Aplicar validaciones de detalle legacy (IVA, duplicados, monto maximo, reglas por transaccion, disponibilidad de fondo) |
| PUT /replenishments/{id} | Parcial | Actualizar detalle de forma transaccional y recalcular totales de cabecera |

Campos de salida para front en listados/consulta por id:

- `transactionCode`: codigo tecnico del tipo de reposicion (`1`, `2`).
- `replenishmentType`: descripcion legible (`Caja Chica`, `Gastos Personales`).
- `createdDate` y `replenishmentDate`: equivalentes para fecha de reposicion (legacy `fechaRegistro`).

## 4. Matriz BACK-FRONT (Administracion)

Base URL: `/gtfReplacementsServices/api/v1`

| Modulo front | Menu/Opcion | Pantalla/Componente | Boton/Accion | Endpoint backend | Estado backend | Uso esperado |
| --- | --- | --- | --- | --- | --- | --- |
| Administracion de Reposiciones | Reposiciones > Administracion | Lista de reposiciones | Buscar | POST /replenishments/findByFilter | Implementado (parcial) | Cargar grilla con filtros y paginacion |
| Administracion de Reposiciones | Reposiciones > Administracion | Cabecera de fondo | Refrescar cabecera | GET /replenishments/current-fund?workAreaCode={id} | Implementado (parcial) | Mostrar fondo, saldo, alertas y estado actual |
| Administracion de Reposiciones | Reposiciones > Administracion | Formulario cabecera | Nueva reposicion | POST /replenishments | Implementado (parcial) | Crear solicitud en estado pendiente |
| Administracion de Reposiciones | Reposiciones > Administracion | Formulario cabecera | Guardar cambios | PUT /replenishments/{replenishmentId} | Implementado (parcial) | Editar cabecera en estado pendiente |
| Administracion de Reposiciones | Reposiciones > Administracion | Vista detalle solicitud | Abrir reposicion | GET /replenishments/{replenishmentId} | Implementado (parcial) | Consultar reposicion por id |
| Administracion de Reposiciones | Reposiciones > Administracion | Tabla de detalle | Agregar fila | POST /replenishments/{id}/details | Pendiente P0 | Crear linea de detalle |
| Administracion de Reposiciones | Reposiciones > Administracion | Tabla de detalle | Editar fila | PUT /replenishments/{id}/details/{detailId} | Pendiente P0 | Actualizar linea de detalle |
| Administracion de Reposiciones | Reposiciones > Administracion | Tabla de detalle | Eliminar fila | DELETE /replenishments/{id}/details/{detailId} | Pendiente P0 | Eliminar linea y recalcular totales |
| Administracion de Reposiciones | Reposiciones > Administracion | Formulario de documento | Validar IVA | POST /replenishments/validate-vat | Implementado (P0 - corte 1) | Validar reglas IVA legacy |
| Administracion de Reposiciones | Reposiciones > Administracion | Formulario de documento | Validar duplicado | POST /replenishments/validate-duplicate | Implementado (P0 - corte 2) | Validar no duplicidad por RUC+documento |
| Administracion de Reposiciones | Reposiciones > Administracion | Flujo de envio | Enviar reposicion | POST /replenishments/{id}/send | Pendiente P0 | Cambiar estado y ejecutar validaciones de envio |
| Administracion de Reposiciones | Reposiciones > Administracion | Flujo de envio | Seleccionar responsable | GET /replenishments/responsibles | Pendiente P1 | Cargar responsables para pago por ventanilla |
| Administracion de Reposiciones | Reposiciones > Administracion | Acciones de solicitud | Anular | POST /replenishments/{id}/cancel | Pendiente (definir Admin vs Revision) | Anulacion con observacion |
| Administracion de Reposiciones | Reposiciones > Administracion | Acciones de solicitud | Imprimir | GET /replenishments/{id}/print | Pendiente P1 | Generar comprobante |

## 5. Matriz BACK-FRONT (Revision)

Base URL: `/gtfReplacementsServices/api/v1`

| Modulo front | Menu/Opcion | Pantalla/Componente | Boton/Accion | Endpoint backend | Estado backend | Uso esperado |
| --- | --- | --- | --- | --- | --- | --- |
| Revision de Reposiciones | Reposiciones > Revision | Lista de solicitudes enviadas | Buscar | POST /replenishment-reviews/findByFilter | Pendiente P0 | Listar solicitudes para contabilidad |
| Revision de Reposiciones | Reposiciones > Revision | Pantalla de revision | Abrir solicitud | GET /replenishment-reviews/{id} | Pendiente P0 | Ver solicitado vs aprobado |
| Revision de Reposiciones | Reposiciones > Revision | Tabla de revision detalle | Aprobar/ajustar detalle | PUT /replenishment-reviews/{id}/details/{detailId}/approve | Pendiente P0 | Ajustar valor aprobado con observacion |
| Revision de Reposiciones | Reposiciones > Revision | Acciones de revision | Validar solicitud | POST /replenishment-reviews/{id}/validate | Pendiente P0 | Validar contablemente y pasar estado |
| Revision de Reposiciones | Reposiciones > Revision | Acciones de revision | Anular solicitud | POST /replenishment-reviews/{id}/cancel | Pendiente P0 | Anulacion contable con observacion |

## 6. Criterio para arrancar P0

Para iniciar P0 sin romper lo ya estable:

1. Mantener endpoints implementados y agregar capacidades faltantes por capas.
2. Implementar primero CRUD de detalle y validaciones de detalle (IVA/duplicado).
3. Luego implementar flujo de envio en Administracion.
4. Finalmente habilitar modulo base de Revision (listado, consulta, aprobar, validar, anular).
