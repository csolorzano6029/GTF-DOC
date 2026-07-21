# Analisis integral de Revision de Reposiciones (Legacy vs Migrado)

Fecha de corte: 2026-07-21

## 1) Objetivo del documento

Entregar un handoff tecnico-funcional para que otro desarrollador continúe el modulo de Revision de Reposiciones mientras el frente de Administracion sigue en paralelo.

Este documento responde:

1. Como funciona Revision en legacy, flujo por flujo.
2. Que APIs actuales del migrado se pueden reutilizar.
3. Que APIs nuevas hay que crear para paridad funcional.
4. Que logica de negocio nueva falta.
5. Si conviene nuevo contexto de rutas o usar el contexto actual.
6. Como convertir todo en epicas, historias y tareas de Jira con bajo riesgo de retrabajo.

## 2) Conclusiones ejecutivas

1. El backend migrado tiene base fuerte para Administracion, pero no tiene aun un modulo de Revision completo.
2. Legacy de Revision es amplio: incluye ajuste por linea, rechazo/activacion, retenciones (manual/electronica/XML), carga de archivo soporte y anulación.
3. Para separar responsabilidades y reducir retrabajo, conviene exponer Revision en contexto /api/v1/replenishment-reviews para acciones propias de review; sin embargo, en H1 no es obligatorio crear endpoint nuevo porque findByFilter actual ya cubre filtros dinamicos.
4. Estado revisable operativo confirmado en legacy: ENVIADA (ENV) para editar/validar/anular; otros estados pueden aparecer en busqueda, pero no habilitan esas acciones.
5. Se recomienda mantener el alcance actual de migracion (solo Caja Chica, transactionCode=1) tambien en Revision; Gastos Personales debe quedar como decision de negocio separada.
6. Con un backlog por fases (consulta -> ajuste -> validacion/anulacion -> retenciones/archivo -> hardening), el traspaso a otra persona es viable sin bloquear Administracion.

## 2.1 Ajustes aplicados en estructura (corte 2026-07-21)

1. Controllers separados al mismo nivel por contexto:
   - shared: `controller/replenishment`
   - administracion: `controller/replenishment-management`
   - revision: `controller/replenishment-reviews`
2. Colecciones Bruno separadas por contexto:
   - shared: `REPLACEMENTS`
   - administracion: `REPLACEMENTS-MANAGEMENT`
   - revision: `REPLACEMENTS-REVIEWS`
3. Requests Bruno renumerados de forma secuencial por carpeta para evitar saltos de orden.

## 3) Evidencia revisada

### Legacy (JSF + Controller)

1. SOPORTES/gtf-root/gtf-web-intranet/src/main/java/ec/com/smx/gtf/web/administracion/controller/RevisionReposicionController.java
2. SOPORTES/gtf-root/gtf-web-intranet/src/main/java/ec/com/smx/gtf/web/administracion/datamanager/RevisionReposicionDataManager.java
3. SOPORTES/gtf-root/gtf-web-intranet/src/main/webapp/pages/rembolsos/revision/adminRevisionRembolso.xhtml
4. SOPORTES/gtf-root/gtf-web-intranet/src/main/webapp/pages/rembolsos/revision/adminRevisionReposicionCenter.xhtml
5. SOPORTES/gtf-root/gtf-web-intranet/src/main/webapp/pages/rembolsos/revision/cabeceraReposicion.xhtml
6. SOPORTES/gtf-root/gtf-web-intranet/src/main/webapp/pages/rembolsos/revision/detalleReposicionCenter.xhtml
7. SOPORTES/gtf-root/gtf-web-intranet/src/main/webapp/pages/rembolsos/revision/revisionReposicionGasto.xhtml
8. SOPORTES/gtf-root/gtf-web-intranet/src/main/webapp/pages/rembolsos/revision/popupAjusteReposicion.xhtml
9. SOPORTES/gtf-root/gtf-web-intranet/src/main/webapp/pages/rembolsos/revision/popupValidarReposicion.xhtml
10. SOPORTES/gtf-root/gtf-web-intranet/src/main/webapp/pages/rembolsos/revision/popupAnularReposicion.xhtml
11. SOPORTES/gtf-root/gtf-web-intranet/src/main/webapp/pages/rembolsos/revision/popupValidarRetencionR.xhtml
12. SOPORTES/gtf-root/gtf-web-intranet/src/main/webapp/pages/rembolsos/revision/popupArchivoRembolso.xhtml
13. SOPORTES/gtf-root/gtf-web-intranet/src/main/webapp/pages/rembolsos/revision/popupConceptoAreTraFin.xhtml

### Migrado (Spring Boot)

1. MIGRACION/gtf-replacements-root/gtf-replacements-services/src/main/java/ec/com/smx/gtf/replacements/controller/replenishment/ReplenishmentController.java
2. MIGRACION/gtf-replacements-root/gtf-replacements-services/src/main/java/ec/com/smx/gtf/replacements/controller/replenishment-management/ReplenishmentManagementController.java
3. MIGRACION/gtf-replacements-root/gtf-replacements-services/src/main/java/ec/com/smx/gtf/replacements/controller/replenishment-reviews/ReplenishmentReviewController.java
4. MIGRACION/gtf-replacements-root/gtf-replacements-client/src/main/java/ec/com/smx/gtf/replacements/replenishment/service/IReplenishmentService.java
5. MIGRACION/gtf-replacements-root/gtf-replacements-core/src/main/java/ec/com/smx/gtf/replacements/replenishment/service/ReplenishmentService.java
6. MIGRACION/gtf-replacements-root/gtf-replacements-vo/src/main/java/ec/com/smx/gtf/replacements/vo/replenishment/ReplenishmentDetailVo.java

### Documentacion actual

1. GTF-DOC/docs/jira/JIRA-Backend-Reposiciones-Revision-Plan-Implementacion.md
2. GTF-DOC/docs/apis/Backend-Reposiciones-Endpoints-Implementados.md
3. GTF-DOC/docs/apis/BACK-FRONT.md

## 4) Flujo legacy de Revision (end to end)

## 4.1 Busqueda y listado de reposiciones

Acciones:

1. Buscar por codigo local, nombre de area o todos.
2. Filtrar por estado y rango de fechas.
3. Cargar grid paginado.

Reglas:

1. Filtro por estado activo.
2. Filtros de fecha desde/hasta.
3. Seleccion de fila abre cabecera de revision.

## 4.2 Apertura y visualizacion de la solicitud

Acciones:

1. Abrir solicitud desde grilla.
2. Ver cabecera (monto solicitado, estado, area, personas).
3. Ver detalle por linea (documento, concepto, IVA, total, observaciones).

Reglas:

1. Carga de datos de fondos/pendientes para contexto contable.
2. Diferenciacion de pantalla por tipo transaccion (Caja Chica vs Gastos Personales en legacy).

## 4.3 Edicion en modo revision

Acciones:

1. Entrar a Editar cuando estado sea ENVIADA.
2. Ajustar valor aprobado e IVA aprobado por linea.
3. Guardar ajuste global desde popup de ajuste.

Reglas:

1. Si valor aprobado cambia respecto al solicitado, observacion es obligatoria.
2. Recalculo de totales cada vez que cambia una linea.

## 4.4 Rechazar y activar linea

Acciones:

1. Rechazar linea: estado inactivo y valor aprobado en 0.
2. Activar linea rechazada: restituye valores originales.

Reglas:

1. Rechazo exige observacion.
2. Siempre recalcula totales luego de rechazar/activar.

## 4.5 Cambio de concepto por linea

Acciones:

1. Abrir popup de conceptos por area/transaccion/tipo documento.
2. Seleccionar concepto y actualizar linea.

Reglas:

1. Si concepto exige huella de carbono, se habilitan campos de medida/archivo.
2. Si no exige huella, se limpian campos de medida/archivo.

## 4.6 Retenciones (manual, electronica, XML)

Acciones:

1. Retencion manual: RUC + documento + comprobante + conceptos retenidos.
2. Retencion electronica: validacion por clave de acceso SRI.
3. XML: carga de archivo XML para extraer datos.

Reglas:

1. Validacion de RUC.
2. Validacion de sumatoria de conceptos.
3. Validacion de documento y no duplicidad en contexto.

## 4.7 Archivo soporte

Acciones:

1. Carga archivo fisico para soporte (cuando aplica).
2. Carga desde clave de acceso (documento electronico).

Reglas:

1. Archivo obligatorio cuando concepto exige huella.
2. Control de reemplazo/limpieza de archivo previo.

## 4.8 Validar reposicion

Acciones:

1. Confirmar validacion final desde popup.
2. Cambiar estado a VALIDADA si pasa validaciones.

Reglas:

1. Precondiciones por tipo de transaccion.
2. Integracion con validacion de facturacion legacy.

## 4.9 Anular reposicion

Acciones:

1. Abrir popup de anulacion.
2. Registrar observacion y confirmar.

Reglas:

1. Observacion obligatoria.
2. Ajuste de saldos en flujo Caja Chica.

## 5) Estado actual del migrado (que ya existe)

Contextos actuales:

1. Shared: contexto de responsabilidades compartidas de reposiciones
2. Administracion: `/api/v1/replenishment-management`
3. Revision: `/api/v1/replenishment-reviews` (controller ancla, aun sin endpoints funcionales de negocio)

Endpoints disponibles relevantes por contexto:

Shared (contexto de responsabilidades compartidas):

1. POST /findByFilter
2. POST /validate-vat
3. POST /validate-duplicate
4. GET /billing-concepts
5. GET /responsibles
6. POST /{replenishmentId}/details/validate-add
7. POST /details/validate-add
8. GET /{replenishmentId}
9. GET /{replenishmentId}/details

Administracion (`/api/v1/replenishment-management`):

1. GET /current-fund
2. POST /validate-new
3. POST / (create)
4. POST /{replenishmentId} (update)
5. POST /{replenishmentId}/details
6. POST /{replenishmentId}/details/{detailId}
7. POST /{replenishmentId}/details/{detailId}/delete
8. POST /{replenishmentId}/details/delete
9. POST /{replenishmentId}/send

Observacion clave de alcance vigente:

1. La logica actual del servicio restringe fuertemente a transactionCode=1 para los flujos principales.

## 6) Matriz de reuso para Revision

## 6.1 Reuso interno de logica (sin exponer rutas shared a Front)

1. Validacion de IVA de detalle.
2. Validacion de duplicidad documental.
3. Consulta de conceptos de facturacion.
4. Lectura de cabecera por `replenishmentId`.
5. Lectura de detalle por `replenishmentId`.
6. Prevalidacion de detalle (con y sin cabecera persistida).

Uso en Revision:

1. Reusar endpoints shared en capacidades sin diferencia funcional y reservar `replenishment-reviews` para capacidades propias de Revision o brechas funcionales comprobadas.

## 6.2 Consumo actual para Revision (reuso directo)

1. POST /api/v1/replenishments/findByFilter
2. GET /api/v1/replenishments/{replenishmentId}
3. GET /api/v1/replenishments/{replenishmentId}/details
4. GET /api/v1/replenishments/billing-concepts
5. POST /api/v1/replenishments/{replenishmentId}/details/validate-add
6. POST /api/v1/replenishments/details/validate-add

Nota tecnica H1:

1. El endpoint oficial de H1, con implementacion actual, es `POST /api/v1/replenishments/findByFilter`.
2. Si se detecta brecha funcional para Revision, se crea wrapper en `replenishment-reviews` delegando sin duplicar query/reglas.

## 6.3 No reutilizar como endpoint de Revision (solo referencia interna)

1. POST /api/v1/replenishment-management/{replenishmentId}/send
2. POST /api/v1/replenishment-management/{replenishmentId}/details/{detailId}/delete

Motivo:

1. Son acciones de Administracion y no representan semanticamente el flujo contable de Revision.

## 7) APIs nuevas requeridas para paridad de Revision

Contexto propuesto:

/api/v1/replenishment-reviews

## 7.1 Consulta y apertura (condicional)

1. Por defecto, H1/H2 reutilizan shared y no requieren API nueva.
2. Solo si hay brecha funcional comprobada en H1/H2 se crean wrappers en `/api/v1/replenishment-reviews`.

## 7.2 Ajuste contable de detalle

1. POST /{replenishmentId}/details/{detailId}/adjust
2. POST /{replenishmentId}/details/{detailId}/reject
3. POST /{replenishmentId}/details/{detailId}/reactivate
4. POST /{replenishmentId}/save-adjustments

## 7.3 Retenciones

1. POST /{replenishmentId}/details/{detailId}/withholding/validate-manual
2. POST /{replenishmentId}/details/{detailId}/withholding/validate-electronic
3. POST /{replenishmentId}/details/{detailId}/withholding/validate-access-key
4. POST /{replenishmentId}/details/{detailId}/withholding/validate-xml

## 7.4 Conceptos y archivo soporte

1. POST /{replenishmentId}/details/{detailId}/adjust-concept
2. POST /{replenishmentId}/details/{detailId}/support-file
3. POST /{replenishmentId}/details/{detailId}/support-file-from-access-key
4. DELETE /{replenishmentId}/details/{detailId}/support-file

## 7.5 Cierre de proceso

1. POST /{replenishmentId}/validate
2. POST /{replenishmentId}/cancel

## 8) Logica nueva que falta en migrado para Revision

1. Modelo dual solicitado vs aprobado por detalle y cabecera.
2. Regla de observacion obligatoria al diferir aprobado vs solicitado.
3. Rechazo/activacion de linea como accion de Revision (no delete).
4. Recalculo de totales aprobados y tipo de ajuste (ingreso/egreso/NA).
5. Validacion de monto maximo por documento en contexto de ajuste.
6. Validacion de no sobregiro de saldo en flujo Caja Chica.
7. Gestion completa de retenciones (manual/electronica/XML).
8. Integracion de archivo soporte con regla de huella de carbono.
9. Estado y auditoria de validacion final contable.
10. Estado y auditoria de anulacion con observacion obligatoria.
11. Guardado transaccional de lote de ajustes para evitar inconsistencias.

## 9) Decision de arquitectura de rutas

Pregunta: usar contexto actual o nuevo contexto para Revision.

Comparativo:

1. Usar contexto shared actual para exponer Revision
   Pros: menos controladores nuevos.
   Contras: mezcla Admin y Revision, permisos mas complejos, contratos mas ambiguos, mayor riesgo de regresion.

2. Usar contexto nuevo /api/v1/replenishment-reviews
   Pros: separacion de responsabilidades, contratos claros, permisos por rol mas limpios, mejor trazabilidad para Jira y QA.
   Contras: mas endpoints a exponer (aunque mucho reuso interno).

Recomendacion final:

1. Mantener contexto nuevo /api/v1/replenishment-reviews para acciones propias de Revision (ajuste, validar, anular, retenciones y soporte).
2. Para H1 y H2, reutilizar por defecto endpoints shared (`/api/v1/replenishments/...`) mientras no exista brecha funcional.
3. Si aparece brecha funcional en H1/H2 (filtros, payload, permisos, flags), crear wrapper de Revision delegando a la logica existente.
4. Regla funcional fija para Jira y QA: acciones de Revision (editar/validar/anular) solo con estado ENVIADA (ENV).

## 10) Backlog Jira listo para ejecutar

## 10.1 Epica principal

EPIC REV-01 - Revision de Reposiciones Caja Chica (Contabilidad)

Objetivo:

1. Cubrir busqueda, apertura, ajuste, validacion y anulacion de reposiciones ENVIADAS con paridad funcional legacy para Caja Chica.

## 10.2 Historias y tareas

### REV-01-H1 Consulta de reposiciones para Revision (filtros y paginacion)

Endpoints:

1. POST /api/v1/replenishments/findByFilter

Tareas:

1. Validar que el endpoint shared actual cubre filtros de Revision (area, estado y fechas).
2. Verificar normalizacion de alias/comparadores y paginacion sin duplicar query.
3. Confirmar forzado de contexto de compania y area por sesion.
4. Solo si hay brecha funcional, crear endpoint de Revision que delegue a la logica existente.
5. Pruebas unitarias y de contrato.

Criterios de aceptacion:

1. Devuelve paginado.
2. Soporta filtros dinamicos esperados por UI de Revision.
3. Regla funcional de acciones: editar/validar/anular solo en estado ENVIADA (ENV).
4. No rompe busqueda de Administracion.

### REV-01-H2 Apertura de reposicion para Revision (cabecera y detalle)

Endpoints:

1. GET /api/v1/replenishments/{replenishmentId}
2. GET /api/v1/replenishments/{replenishmentId}/details

Tareas:

1. Validar que lectura shared de cabecera y detalle cubre informacion requerida por Revision.
2. Confirmar requested vs approved y flags operativos sin romper contrato actual.
3. Solo si hay brecha funcional, crear wrapper de lectura en Revision.
4. Tests de contrato.

Criterios de aceptacion:

1. Se visualiza toda la informacion necesaria para revisar.
2. Incluye flags operativos de accion (editable/validable/anulable).

### REV-01-H3 Ajuste contable por linea con recalculo

Endpoints:

1. POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/adjust
2. POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/reject
3. POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/reactivate
4. POST /api/v1/replenishment-reviews/{replenishmentId}/save-adjustments

Tareas:

1. Definir VO de ajuste.
2. Implementar validaciones de observacion, maximo y sobregiro.
3. Implementar rechazo/activacion sin borrar.
4. Recalcular totales y tipo de ajuste.
5. Pruebas de casos OK/error.

Criterios de aceptacion:

1. Observacion obligatoria cuando cambia el valor aprobado.
2. Rechazar y activar funcionan con recálculo correcto.
3. Persistencia transaccional sin inconsistencia.

### REV-01-H4 Validacion contable de reposicion revisada

Endpoint:

1. POST /api/v1/replenishment-reviews/{replenishmentId}/validate

Tareas:

1. Implementar servicio de validacion final.
2. Verificar precondiciones funcionales.
3. Cambiar estado a VALIDATED cuando corresponda.
4. Integrar respuesta funcional clara para front.

Criterios de aceptacion:

1. Solo valida estados permitidos.
2. Reporta errores funcionales de forma controlada.

### REV-01-H5 Anulacion de reposicion en flujo de Revision

Endpoint:

1. POST /api/v1/replenishment-reviews/{replenishmentId}/cancel

Tareas:

1. Definir VO de anulacion con observacion obligatoria.
2. Implementar cambio de estado y auditoria.
3. Ajustar saldos de Caja Chica cuando aplique.
4. Pruebas.

Criterios de aceptacion:

1. Sin observacion no permite anular.
2. Registra motivo de anulacion.

### REV-01-H6 Gestion de conceptos y regla de huella

Endpoints:

1. GET /api/v1/replenishment-reviews/billing-concepts
2. POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/adjust-concept

Tareas:

1. Reusar consulta de conceptos.
2. Persistir cambio de concepto por linea.
3. Aplicar regla de huella (archivo/medida obligatoria cuando corresponda).
4. Pruebas.

Criterios de aceptacion:

1. El popup de conceptos funciona con datos consistentes.
2. El cambio impacta correctamente en detalle y validaciones.

### REV-01-H7 Validacion de retenciones

Endpoints:

1. POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/withholding/validate-manual
2. POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/withholding/validate-electronic
3. POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/withholding/validate-access-key
4. POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/withholding/validate-xml

Tareas:

1. Contratos de retencion manual/electronica.
2. Integracion de validacion documental.
3. Persistencia de conceptos retenidos.
4. Pruebas de escenarios de error y exito.

Criterios de aceptacion:

1. Soporta flujo manual y electronico.
2. Mensajeria funcional clara ante errores.

### REV-01-H8 Gestion de archivo de soporte

Endpoints:

1. POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/support-file
2. POST /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/support-file-from-access-key
3. DELETE /api/v1/replenishment-reviews/{replenishmentId}/details/{detailId}/support-file

Tareas:

1. Carga y reemplazo de archivo.
2. Integracion de almacenamiento.
3. Regla de obligatoriedad por huella.
4. Pruebas.

Criterios de aceptacion:

1. Archivo obligatorio cuando aplica huella.
2. Soporta reemplazo sin residuos inconsistentes.

### REV-01-H9 Cierre de alcance de Gastos Personales

Sin endpoint nuevo inmediato.

Tareas:

1. Confirmar decision de negocio (incluir o excluir).
2. Si se excluye: registrar exclusion explicita en Jira y validaciones.
3. Si se incluye: crear epica separada para reglas y datos de Gastos Personales.

Criterios de aceptacion:

1. No queda ambiguedad de alcance.

## 11) Plan de ejecucion sugerido para delegacion

1. Sprint 1: H1 + H2 (consultas y lectura).
2. Sprint 2: H3 (ajustes y recálculo).
3. Sprint 3: H4 + H5 (validar y anular).
4. Sprint 4: H6 + H7 + H8 (conceptos, retenciones, archivo).
5. Sprint 5: hardening, pruebas integrales, Sonar/PMD y documentacion final.

## 12) Riesgos y mitigaciones

1. Riesgo: mezclar Admin y Review en el mismo contexto.
   Mitigacion: contexto dedicado replenishment-reviews.
2. Riesgo: divergencia de reglas al comparar con legacy.
   Mitigacion: pruebas de paridad por historia con casos legacy.
3. Riesgo: validaciones externas inestables (documental/facturacion).
   Mitigacion: manejo de error controlado y tests de fallback.
4. Riesgo: inconsistencias por guardado parcial.
   Mitigacion: save-adjustments transaccional.

## 13) Checklist anti-retrabajo antes de construir

1. Alcance confirmado: solo Caja Chica o tambien Gastos Personales.
2. Contratos de endpoint revisados por Front y QA antes de codificar.
3. Criterios de aceptacion por historia cerrados en Jira.
4. Casos de prueba de paridad legacy definidos desde el inicio.
5. Ruta de documentacion Bruno/docs/apis acordada por historia.

## 14) Entregables de documentacion esperados durante implementacion

1. Bruno:
   - requests shared reutilizables en `REPLACEMENTS`
   - requests de administracion en `REPLACEMENTS-MANAGEMENT`
   - requests nuevos de Revision en `REPLACEMENTS-REVIEWS` (una request por endpoint)
2. docs/apis: archivo de endpoints de Revision actualizado por cada entrega.
3. BITACORA-PROMPTS.md: registro de avance por historia.

## 15) Recomendacion final para tu asignacion

Para delegar ya:

1. Entregar al nuevo responsable H1 y H2 de inmediato (lectura/consulta).
2. Mantener en tu frente actual Administracion sin cambios de contrato.
3. Acordar desde ahora la regla ENVIADA (ENV) para acciones de Revision y mantener contratos de Front solo en `/api/v1/replenishment-reviews`.

Con esta separacion, ambos frentes pueden avanzar en paralelo con bajo acoplamiento y menor riesgo de retrabajo.
