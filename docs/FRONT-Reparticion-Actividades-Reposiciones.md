# Repartición de Actividades — Frontend Proyecto Reposiciones

> División de tareas del **frontend** para el proyecto de **Reposiciones**, asignado a **dos desarrolladores**, de forma que sus actividades **no se choquen**.

- **Proyecto (repo backend):** `gtf-replacements-root` — expone las APIs bajo **`/gtfReplacementsServices/api/v1/...`**.
- **Alcance:** este proyecto contiene **dos sub-módulos**:
  1. **Administración de Reposiciones** (lo que ingresa el **local**) — `replenishments`.
  2. **Revisión de Reposiciones** (lo que revisa **contabilidad**) — `replenishment-reviews`.
- **Orden (según el video):** primero *administración*, luego *revisión* `[00:48:59–00:50:10]`. Ambos van en este mismo proyecto.
- **Fecha:** 2026-07-06.

> **Supuesto a confirmar:** el framework de frontend (Angular / React) no altera la repartición (dividida por sub-módulos/componentes), solo la terminología técnica.

---

## 1. Estrategia de división (cómo evitar choques)

Como hay **2 desarrolladores** y **2 sub-módulos**, la frontera natural es **un dev por sub-módulo**. Para equilibrar carga (administración es más pesada por los formularios y validaciones), **DEV B** (revisión, más liviano) también construye los **componentes compartidos** que ambos reutilizan.

```
┌───────────────────────────────────────────────────────────────┐
│  FASE 0 — BASE COMPARTIDA (juntos, antes de dividir)            │
│  Modelos, cliente HTTP, envelope, auth Keycloak, ruteo, UI base │
└───────────────────────────────────────────────────────────────┘
        │                                          │
        ▼                                          ▼
┌────────────────────────────┐        ┌────────────────────────────────┐
│ DEV A — Administración de   │        │ DEV B — Revisión de            │
│ Reposiciones (local)        │◄─────► │ Reposiciones (contabilidad)    │
│ crear/editar, docs,         │ COMPARTE│ + COMPONENTES COMPARTIDOS      │
│ validaciones, enviar        │  (§4)  │ (cabecera fondo, visor detalle,│
│                             │        │  catálogos)                    │
└────────────────────────────┘        └────────────────────────────────┘
```

**Regla de oro anti-choques:** cada dev es **dueño exclusivo** de su carpeta de sub-módulo. Los **componentes compartidos** (los construye DEV B) tienen su propia carpeta `shared/` y una **API estable congelada en Fase 0**; DEV A los **consume** pero no los edita.

---

## 2. FASE 0 — Base compartida (bloqueante, hacer juntos)

Duración objetivo: **1–2 días**. Se congela antes de repartir.

| ID | Tarea | Detalle | Responsable |
|----|-------|---------|-------------|
| F0-1 | Estructura del proyecto | `reposiciones/` con `administracion/` (DEV A), `revision/` (DEV B), `shared/`, `models/`, `services/`. | Ambos |
| F0-2 | Modelos / interfaces | Tipos espejo de los VOs: `ReplenishmentVo`, `ReplenishmentDetailVo`, `FundHeaderVo`, `ResponsibleVo`, `ReviewDetailVo` (solicitado/aprobado); enums `ReplenishmentStatus`, `DocumentType`. | Ambos |
| F0-3 | Cliente HTTP + envelope | Interceptor token Keycloak (Bearer); manejo de `BaseResponseVo {code, message, data}`. Base URL `/gtfReplacementsServices/api/v1`. | DEV B |
| F0-4 | UI base | Badge de estado, formato moneda USD, toasts, spinner. | DEV B |
| F0-5 | Ruteo | Navegación de ambos sub-módulos. | DEV A |
| F0-6 | Contrato de componentes compartidos | Congelar API de cabecera de fondo, visor de detalle y catálogos (§4). | Ambos |

---

## 3. Componentes compartidos (los construye DEV B, los usa también DEV A)

Van en `shared/` con API estable:

- **Cabecera de fondo** — muestra fondo asignado, saldo efectivo, pendientes. Input: `workAreaCode`. (`GET /replenishments/current-fund`).
- **Visor de detalle de reposición** — tabla de líneas de documento en modo **lectura** (usado por administración para revisar lo ingresado y por revisión para comparar). Input: `details`, `readonly`.
- **Catálogos** — tipos de documento y conceptos de facturación (`GET /replenishments/document-types`, `/billing-concepts`).

---

## 4. Contrato de integración (congelar en Fase 0)

- **Cabecera de fondo (compartido):** `@Input workAreaCode`; se refresca ante `refresh()`.
- **Visor de detalle (compartido):** `@Input details: ReplenishmentDetailVo[]`, `@Input readonly: boolean`.
- **Administración (DEV A)** incrusta el visor en modo edición envolviéndolo con sus formularios; **no** edita el componente compartido.
- **Revisión (DEV B)** usa el visor en modo comparación (columnas solicitado/aprobado).
- DEV A **no** implementa pantallas de revisión; DEV B **no** implementa formularios de ingreso de documentos.

---

## 5. DEV A — Administración de Reposiciones (local)

**Objetivo:** que el local cree, edite, envíe, anule e imprima sus reposiciones, con todas las validaciones.

| ID | Tarea | Endpoints (`/gtfReplacementsServices/api/v1`) | Criterio de aceptación |
|----|-------|----------------------------------------|------------------------|
| A-1 | Listado + búsqueda de reposiciones | `POST /replenishments/findByFilter` | Tabla paginada por estado/fecha; badge; abrir. |
| A-2 | Crear reposición | `POST /replenishments`, `GET /replenishments/{id}` | Bloquea si hay una `PENDING`; si hay `PAID/ISSUED`, modal para elegir qué reponer. |
| A-3 | Editar cabecera + incrustar detalle | `PUT /replenishments/{id}` | Solo editable en `PENDING`; usa cabecera de fondo y visor de detalle compartidos. |
| A-4 | Formularios por tipo de documento | `POST/PUT/DELETE /replenishments/{id}/details` | Factura, factura electrónica, nota de venta, recibo manual (oculta IVA donde no aplica). |
| A-5 | Validaciones de ingreso | `POST /replenishments/validate-vat`, `/validate-duplicate` | IVA ≤ `total − total/1.15`; duplicado por RUC+doc; valor máximo con excepciones. |
| A-6 | Retenciones (ingreso) | `GET/POST /withholdings/electronic*`, `POST /withholdings/manual` | Retención electrónica (validar + subir XML) y manual (IVA + renta). |
| A-7 | Adjuntar archivo | file upload | Para conceptos que lo requieren (gas / estudios de mercado). |
| A-8 | Enviar | `GET /replenishments/responsibles`, `POST /replenishments/{id}/send` | Selección de responsable; si no hay → mensaje (422). Pasa a `SENT`. |
| A-9 | Anular + imprimir | `POST /replenishments/{id}/cancel`, `GET /replenishments/{id}/print` | Anula con observación; genera PDF. |

---

## 6. DEV B — Revisión de Reposiciones (contabilidad) + Compartidos

**Objetivo:** que contabilidad revise lo enviado, ajuste, valide o anule; y construir los componentes compartidos (§3).

| ID | Tarea | Endpoints (`/gtfReplacementsServices/api/v1`) | Criterio de aceptación |
|----|-------|----------------------------------------|------------------------|
| B-0 | **Componentes compartidos** (§3) | `/replenishments/current-fund`, `/document-types`, `/billing-concepts` | Cabecera de fondo, visor de detalle y catálogos con API estable (contrato §4). |
| B-1 | Listado de reposiciones enviadas | `POST /replenishment-reviews/findByFilter` | Lista `SENT` por local/fecha; abrir. |
| B-2 | Pantalla de revisión (solicitado vs aprobado) | `GET /replenishment-reviews/{id}` | Muestra dos columnas por línea; usa el visor de detalle compartido en modo comparación. |
| B-3 | Ajustar valor aprobado | `PUT /replenishment-reviews/{id}/details/{detailId}/approve` | Edita aprobado + observación; avisa que se enviará mail al local. |
| B-4 | Validar | `POST /replenishment-reviews/{id}/validate` | Pasa a `VALIDATED`; maneja el mensaje si falta punto de emisión. |
| B-5 | Anular | `POST /replenishment-reviews/{id}/cancel` | Anula con observación. |

---

## 7. Reglas de negocio por dev

**DEV A (ingreso):** IVA `total/1.15`; documento duplicado; valor máximo con excepciones (estudios de mercado, gas); IVA solo en facturas; una sola `PENDING`; envío requiere responsable; solo `PENDING` editable.

**DEV B (revisión):** columnas solicitado/aprobado; ajuste dispara notificación al local; validación requiere punto de emisión (usuario de pruebas Cintia/Cristina Escobar); tras `VALIDATED` el backend (job) pasa a `PAID/ISSUED`.

---

## 8. Secuencia e hitos

| Hito | Contenido | Depende de |
|------|-----------|------------|
| **H0** | Fase 0 completa + contrato congelado + componentes compartidos (B-0) base. | — |
| **H1** | DEV A: listado + crear + editar (A-1..A-3). DEV B: listado revisión + pantalla revisión (B-1, B-2). | H0 |
| **H2** | DEV A: formularios + validaciones (A-4, A-5). DEV B: ajustar + validar (B-3, B-4). | H1 |
| **H3** | DEV A: retenciones + adjuntos + enviar + anular/imprimir (A-6..A-9). DEV B: anular + pulido (B-5). | H2 |
| **H4** | Integración end-to-end de ambos sub-módulos. | H3 |

> **Paralelismo real:** tras H0, cada dev trabaja en su carpeta de sub-módulo sin tocar la del otro. Único punto de contacto: los componentes compartidos de `shared/` (API congelada).

---

## 9. Checklist de arranque

- [ ] Confirmar framework de frontend.
- [ ] Confirmar disponibilidad de endpoints del backend (`replenishments` y `replenishment-reviews` aún por codificar en `gtf-replacements-root`).
- [ ] Fase 0 completa y **componentes compartidos + contrato congelados**.
- [ ] Branches por dev (`feature/front-administracion` / `feature/front-revision`).
- [ ] Retención electrónica (A-6): coordinar con backend por el bloqueo del SRI (queda al final).
