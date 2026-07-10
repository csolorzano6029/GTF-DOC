# Reunión GTF — Transferencia de conocimiento y plan de migración

- **Fecha del video:** 2026-07-02 (09:35:21)
- **Archivo original:** `2026-07-02 09-35-21.mkv`
- **Duración:** 56 min 09 s (~3369 s)
- **Contexto:** Corporación Favorita — Sistema **GTF** (gestión de **caja chica / fondos** de locales)
- **Objetivo de la reunión:** Explicar el funcionamiento del sistema *legacy* GTF al equipo y planificar su **migración a tecnología nueva (Java 25)**.

> **Nota:** Este documento reconstruye todo lo hablado en el video a partir de la transcripción de audio (Whisper `large-v3`, español). Se incluyen marcas de tiempo `[hh:mm:ss]` como referencia. El tramo final del audio (~[00:55:13] en adelante) contiene sonido de fondo no relacionado con la reunión (música / conversación personal) y se documenta como tal.

---

## Índice

1. [Estructura de la reunión](#estructura-de-la-reunión)
2. [Parte 1 — Demo funcional del sistema GTF](#parte-1--demo-funcional-del-sistema-gtf)
3. [Parte 2 — Recorrido del código fuente y plan de migración](#parte-2--recorrido-del-código-fuente-y-plan-de-migración)
4. [Parte 3 — Cierre y temas administrativos](#parte-3--cierre-y-temas-administrativos)
5. [Problemas y pendientes técnicos (consolidado)](#problemas-y-pendientes-técnicos-consolidado)
6. [Plan de migración (consolidado)](#plan-de-migración-consolidado)
7. [Acciones y responsables](#acciones-y-responsables)
8. [Glosario](#glosario)

---

## Estructura de la reunión

| Bloque | Tiempo | Contenido |
|--------|--------|-----------|
| Parte 1 | `[00:00:00] – [00:39:30]` | Demo funcional del sistema actual (GTF) con usuarios reales. |
| Parte 2 | `[00:39:30] – [00:53:20]` | Recorrido del código fuente y planificación de la migración. |
| Parte 3 | `[00:53:20] – [00:56:09]` | Cierre, reunión opcional de las 11:00 y despliegue del FCM. Al final, audio de fondo ajeno a la reunión. |

---

## Parte 1 — Demo funcional del sistema GTF

### 1.1 Tipos de usuario `[00:00:00]`

Existen **dos tipos de usuario**:

- **Usuario de local / caja** — código tipo `SECT` (secretario) + turno (`M` mañana, `T` tarde) + código de local. Ejemplos usados en la demo: `SECT186`, `SECT` con locales `153`, `101`. Contraseña de prueba: `Password01`.
- **Usuario de contabilidad** — revisa y valida lo que envían los locales.

> Para pruebas se recomienda el usuario **Cintia / Cristina Escobar**, porque tiene configurado el **punto de emisión** necesario en el CIF.

### 1.2 Qué puede hacer el usuario de local `[00:01:24]`

1. **Ingresar una reposición de caja** (reembolso por compras del local: esferos, insumos, fiestas de Quito, etc.).
2. **Consultar retenciones electrónicas.**
3. **Solicitar un aumento de fondos** del local.

### 1.3 Configuración del fondo del local `[00:02:09] – [00:05:01]`

Cuando se crea un local nuevo, **contabilidad** le asigna un fondo con los siguientes parámetros:

| Parámetro | Descripción | Ejemplo |
|-----------|-------------|---------|
| **Fondo asignado** | Monto total del fondo de caja chica. | $200 |
| **% de uso** | Porcentaje del fondo que puede consumir antes de tener que **enviar la reposición**. No se le deja llegar al 100%. | 60% |
| **% de alerta** | Al alcanzar este porcentaje ingresado, el sistema muestra una **alerta** de que debe enviar la reposición pronto. | 50% |
| **Valor máximo de factura** | Monto máximo permitido por factura individual. | $50 |
| **Forma de depósito** | Depósito, ventanilla, pago, cheque. Es "transparente" para el proceso (cómo cobran/depositan). | — |

### 1.4 Ajuste de fondo (aumentar / disminuir) `[00:03:56] – [00:05:01]`

- En el **historial de reposiciones** hay un botón **"Ajuste"** que permite aumentar o disminuir el fondo.
- Uso típico: **temporada navideña** (más gastos: almuerzos, etc.). Ejemplo: se agregan $500 → el fondo pasa de $200 a $700, con la observación "temporada navideña".
- Al terminar la temporada se **disminuye** nuevamente ($-500) y se vuelve a $200.
- Todo queda registrado en el historial.

### 1.5 Estados de una reposición `[00:05:16]`

```
pendiente → enviado → validado → pago → emitido → cobrado
                                                    (+ anulado)
```

- **Pendiente:** el local puede seguir editando y agregando documentos.
- **Enviado:** ya se envió a contabilidad; no se puede editar.
- **Validado:** contabilidad la revisó y aprobó.
- **Pago / Emitido:** una tarea programada cambia el estado tras la validación.
- **Cobrado:** el local aceptó/recibió la reposición.
- **Anulado:** contabilidad puede anular en cualquier momento con una observación.

> **Regla:** no se puede crear una reposición nueva si existe una **pendiente**. El local debe buscar la pendiente y trabajar sobre ella. `[00:23:07]`

### 1.6 Ingreso de una nueva reposición `[00:04:56] – [00:11:17]`

Al crear una reposición se ingresa: número de cédula, descripción y luego cada documento. Campos generales: **fecha del documento, RUC, número de documento, código de concepto, observación, IVA y total.**

**Conceptos** `[00:05:47] – [00:07:18]`
- Los conceptos se **configuran desde el CIF** (sistema de facturación); GTF solo consume lo configurado.
- Dos familias con el mismo nombre/descripción, diferenciadas por el tipo de cuenta:
  - **LC** = Liquidación de Compra → se envían y son **validados por el SRI** (aplican a factura de ley).
  - **LO** = Liquidación Otros.

### 1.7 Tipos de documento `[00:05:41] – [00:22:13]`

| Documento | Cómo llega | Campos | ¿Tiene IVA? |
|-----------|-----------|--------|-------------|
| **Factura (manual)** | Física; el local digita los datos. Pide fecha de factura. | Fecha, RUC, N° doc, concepto, observación, IVA, total. | Sí |
| **Factura electrónica** | Impresa | Igual que la factura manual. | Sí |
| **Nota de venta** | Física | Fecha, RUC, N° documento, concepto. | No |
| **Recibo manual** | Física | Solo fecha y concepto. | No |
| **Retención electrónica** | Impresa / por correo | RUC, documentación, valor. Se **valida** contra base de datos / SRI. | (retención) |
| **Retención manual** | Libretín físico | RUC/cédula, total, N° doc, impuesto al IVA e impuesto a la renta (en centavos). | (retención) |

### 1.8 Control del cálculo del IVA `[00:07:36] – [00:10:09]` ⚠️

- **Método antiguo (locales):** regla de tres. Ej. si el total es $2 y el IVA es 15% → `15 × 2 / 100 = 0.30`.
- **Método nuevo (SIF):** el SIF **ignora el IVA ingresado** y solo toma el total: `2 / 1.15 = 1.739` → IVA `= 2 − 1.739 = 0.26`.
- **Problema:** si el local ingresa `0.30` y el SIF calcula `0.26`, sale **error** al validar.
- **Control requerido:** al ingresar, el sistema debe validar que el IVA ingresado **no supere** el valor calculado (`total − total/1.15`). Se permite un IVA **menor** (facturas con productos gravados y no gravados).

### 1.9 Control de documento duplicado `[00:10:21] – [00:11:00]` ⚠️

- El sistema valida que la combinación **RUC + N° de documento** no haya sido **cancelada** en otra fecha / otro local.
- Mensaje de ejemplo: *"La factura ingresada se encuentra cancelada en la fecha 29 de junio del 2026 en el local tal."*
- Objetivo: evitar que se cobren copias del mismo documento. El mensaje indica código de reposición, local y fecha.

### 1.10 Valor máximo y excepciones `[00:11:57] – [00:15:27]`

- Si se supera el valor máximo (ej. $50): *"El valor ingresado del documento sobrepasa el valor máximo permitido, 50 dólares."*
- También controla el % del fondo: *"La presente reposición sobrepasó el 60% del fondo asignado."*
- **Excepciones** (sí permiten superar el máximo):
  - **Estudios de mercado** (gastos altos).
  - **Gas / gasolina** (por el problema de cortes de luz — se disparó el gasto).
- Estas excepciones se controlan en una **tabla de parámetros** en el módulo **Finance**: `GTF_PARAMETRO` → *"códigos de conceptos de facturación que no validan valor máximo"*. Se agregan/quitan códigos de concepto.
- **Idea de mejora:** crear una pantalla para administrar estos parámetros (no urgente). `[00:14:40] – [00:15:03]`

### 1.11 Retención electrónica y bloqueo del SRI `[00:16:32] – [00:21:20]` ⚠️

- La retención llega impresa; el cliente podría **editar los valores** (riesgo de fraude). Por eso se **valida**.
- Flujo: se ingresa RUC + documentación + valor y se pulsa **Validar**.
- Existe una **tarea programada** que recibe (por correo electrónico) todas las retenciones que llegan a Corporación Favorita y las guarda en base de datos. Al validar, GTF busca la retención en esa base.
- **Problema 1:** si la tarea programada falla, no hay datos guardados.
- **Problema 2 (SRI):** antes existía un **servicio web del SRI** que, con la clave de acceso, devolvía toda la información de la retención. **El SRI bloqueó ese servicio.** Ahora la consulta solo devuelve el **estado** (autorizado / no autorizado / por procesar / anulado), **sin el valor**.
- **Consecuencia actual:** las retenciones del mes en curso (junio) sí salen; las de meses anteriores no. Como *workaround*, el cliente entra al SRI, descarga el **XML** y lo sube al sistema para tomar los datos.
- **Pendiente:** coordinar con **Frank** un método nuevo: si el estado es *autorizado*, permitir el ingreso directamente. **Queda para el final.**

### 1.12 Retención manual `[00:21:21] – [00:22:13]`

- Cuando llega en libretín físico. Se ingresa RUC/cédula, total, N° de documento.
- Dos impuestos: **impuesto al IVA** e **impuesto a la renta** (se ingresan los centavos de cada uno). Sin mayor complejidad.

### 1.13 Envío de la reposición y responsables `[00:22:19] – [00:24:54]`

- Al terminar, el estado inicial es **pendiente** (editable). Luego el local hace el **envío**.
- Al enviar aparece el **listado de responsables** de la caja chica (se administra en *Administración de responsables*).
- Reglas de responsables:
  - Todo local **debe tener un responsable**. Si el local es nuevo o el responsable fue reasignado, sale: *"No existe un responsable asignado"* → se coordina con contabilidad.
  - Hay **un principal** (normalmente el administrador del local) y varios **secundarios** (subjefes, por si el principal está de vacaciones). **Solo puede haber un principal.**
- Tras enviar, la reposición pasa a **enviado** y llega a **contabilidad**.

### 1.14 Flujo de contabilidad `[00:24:56] – [00:31:11]`

- Contabilidad revisa las reposiciones recibidas contra los documentos físicos/impresos.
- Dos columnas: **"solicitado"** (lo que ingresó el local) vs **"aprobado"** (lo que aprueba contabilidad).
- Puede **corregir/ajustar** montos (ej. IVA mal ingresado, o descontar gastos personales de una factura). Al pulsar **Ajustar** se confirma: *"¿Está seguro que desea realizar el ajuste? Un mail de alerta será enviado al administrador del local."*
- Acciones: **Validar** (aprobar) o **Anular** (con observación).
- Nota de la demo: al validar dio un problema, posiblemente por falta de configuración del **punto de emisión** del usuario de pruebas (Cristina Escobar). Pendiente de revisar. `[00:28:46] – [00:30:56]`

### 1.15 Paso a "pago emitido" y "cobrado" `[00:31:11] – [00:36:00]` ⚠️

- Cuando contabilidad valida, la reposición pasa a **validado**. Debería correr una **tarea programada** que la cambie a **pago emitido**.
- **La tarea programada no está funcionando**, así que actualmente se ejecuta **un script manual** para transformar los estados de `validado → pago emitido`.
- Cabecera del fondo (visible en todas las pantallas): **fondo asignado**, **saldo efectivo** y **reposiciones pendientes**. Ej.: asignado $250, saldo $142.60, pendientes $107.40.
- Al pulsar **Nuevo** con un pago emitido, aparece un modal para reponer (ej. $107.29). Al aceptar, el saldo efectivo se **suma** y el estado pasa a **cobrado**.

### 1.16 Desfase de centavos `[00:36:00] – [00:36:48]` ⚠️

- El desfase ocurre porque el local puede ingresar cualquier cantidad de centavos, pero el **SIF valida solo el total** (saca el IVA del total). Hay que **ajustar los centavos manualmente** en la parte de validación. Pendiente de arreglar.

### 1.17 Administración desde contabilidad `[00:36:53] – [00:39:29]`

- **Editar fondo del local:** % de uso, % de alerta, valor máximo, forma de depósito. También cargan toda la info cuando el local es nuevo.
- **Administración de responsables** (principal único, se puede borrar/reasignar).
- **Consulta de documento** — módulos que *no son del equipo* ("esto no es nuestro").
- **Consulta electrónica / reporte** — asociado a las retenciones; depende de los servicios del SRI que fueron cortados. **Queda para el final.**

---

## Parte 2 — Recorrido del código fuente y plan de migración

### 2.1 Gestión del proyecto `[00:39:48] – [00:40:30]`

- Crear una **historia** en Jira / **Comprendo** que liste los **módulos a migrar**, para tenerla presente.
- Ya existe un proyecto llamado **GTF2** (creado antes con "Hombre"/el equipo), con pocas tareas.
- Usar ese mismo espacio también para **reportar errores**.

### 2.2 Proyectos del código actual `[00:40:31] – [00:41:04]`

Existen tres proyectos:

1. **GTF caja chica (ejb)** — corresponde a la **consulta de retenciones** (se deja para después; "no le paramos el vuelo ahorita").
2. **GTF Root** — ⭐ **foco principal**: *"aquí está todo, todo, todo"*.
3. **GTF intranet web**.

### 2.3 Estructura de `GTF Root` `[00:41:08] – [00:47:40]`

- **Administración** → controles + data managers.
  - **Depósito** (controller) — se ve después.
  - **Administración de fondos** (`admin fondos`, `fondos controller`, `ajuste fondo`) — configuración de saldo, valor máximo, etc. que ingresa el local y administra Cintia Escobar. Tiene también su **data manager**.
- **Funcionario** → administración de **responsables** de caja chica.
- **Reembolso**:
  - `admin reembolso` (controller) — **lo que ingresa el local**.
  - `revisión / reposición` (controller) — **lo que hace contabilidad**.
- **Retenciones**:
  - Al hacer clic en *retención electrónica* → módulo separado (`retención control`), porque su lógica es **compleja** (va al SRI, etc.). Por eso se hizo aparte.
- **Otros paquetes:** `utils` (llamados desde los controllers — sirven para entender la funcionalidad), `constantes`, `resources`. Parte **web**: `pages`.

### 2.4 Código que NO se migra `[00:44:56] – [00:47:15]` ⚠️

- En reembolso hay **"nueva reposición gasto"** / **gastos personales** / **viáticos**: *se desarrolló pero nunca se utilizó*.
- En los controllers hay **muchos `if` de gasto personal** (`si es gasto personal haga esto…`) que **ya no interesan** — probablemente se movieron a otro proyecto. **Ignorar todo lo que diga "gastos".**

### 2.5 Estrategia de migración `[00:47:56] – [00:50:10]`

- Migrar **por módulos**, con **pantallas nuevas**.
- **No dar de baja** las pantallas actuales: agregar un botón nuevo al lado (ej. *"administración versión 2"*).
- Usar un **generador** para crear la plantilla base en **Java 25**.
- Hacer **todo con integración continua** desde el inicio ("todo lo nuevo de una vez").

### 2.6 Riesgo técnico: servicios antiguos `[00:29:34] – [00:30:22]`

- Preocupación del equipo: al generar la plantilla nueva (Java 25) y conectarse con **servicios muy antiguos** podría fallar.
- Se menciona **"AINAC"**: hay consultas ya hechas contra servicios viejos que **habrá que portar a lo nuevo** ("coger esas consultas y pasarlas a lo nuevo").

### 2.7 Orden de migración propuesto `[00:48:59] – [00:50:10]`

1. **Administración de reposición** (lo que ingresa el local).
2. **Revisión de reposición** (lo que revisa contabilidad).
3. Los demás módulos.
4. **Depósitos** — al final ("no ha dado problemas", pero se explicará cómo funciona).

### 2.8 Logística y entorno `[00:50:50] – [00:53:20]`

- La líder empezará a **documentar las reglas de negocio**.
- El equipo debe **descargar los proyectos base** mientras tanto (empezar a diseñar pantallas y servicios).
- Necesitan el **JBoss** que tiene **Carlos** para levantar el entorno localmente.
- **Carlos** debe crear una tarea en **Jira** ("levantar JBoss") y **compartir el archivo JBoss** (debe quedar disponible permanentemente para todo el equipo).
- Se compartió un archivo **`GTF29`** (~4 GB) con los proyectos base.

---

## Parte 3 — Cierre y temas administrativos `[00:53:20] – [00:56:09]`

- **Reunión opcional a las 11:00** — *"Presentación de pantallas"*: el equipo de **soporte** desarrolló pantallas para **administrar scripts** (recibían muchos scripts). Se recomienda asistir **al menos como oyentes** para tener conocimiento del tema (no hay que hacer nada).
- **Despliegue del FCM** — conversación aparte con "Carlitos":
  - Ya se probó en **pre**, está bien → se puede subir a **prod**.
  - Generar la tarea de paso a producción (por **Fizz** / Jira).
  - Los cambios quedaron en una **feature branch**; falta **compactarlos a master**, generar el **release** y avisar para poner en producción.
- **`[00:55:13]` en adelante:** el audio captura **sonido de fondo no relacionado con la reunión** (música / diálogo tipo novela — *"La culpa de Johnny"*, *"Mija…"* — y una conversación personal). **No forma parte del contenido técnico.**

---

## Problemas y pendientes técnicos (consolidado)

| # | Problema / pendiente | Acción sugerida | Ref. |
|---|----------------------|-----------------|------|
| 1 | **Cálculo del IVA**: el SIF usa `total/1.15`, no regla de 3. | Validar que el IVA ingresado ≤ IVA calculado. | `[07:36]` |
| 2 | **Documento duplicado** (RUC + N° doc). | Mantener la validación de no-duplicidad. | `[10:21]` |
| 3 | **Excepciones de valor máximo** (estudios de mercado, gas/gasolina) en `GTF_PARAMETRO`. | Crear pantalla para administrar la tabla de parámetros. | `[12:56]` |
| 4 | **Retención electrónica**: el SRI bloqueó el servicio web (solo devuelve estado). | Rediseñar con Frank: si "autorizado" → permitir ingreso; alternativa vía XML. **Al final.** | `[17:06]` |
| 5 | **Tarea programada** (validado → pago emitido) **no funciona**; hoy se corre script manual. | Reimplementar la tarea programada. | `[32:34]` |
| 6 | **Desfase de centavos** en validación (SIF valida solo el total). | Automatizar el ajuste de centavos. | `[36:00]` |
| 7 | **Punto de emisión** debe existir en el CIF. | Probar solo con usuario Cintia/Cristina Escobar. | `[26:24]` |
| 8 | **Validación de responsables** débil ("medias turlas") por tecnología antigua. | Rehacer validaciones al migrar. | `[38:25]` |
| 9 | **Servicios antiguos (AINAC)** y consultas legacy. | Portar consultas al stack nuevo. | `[29:34]` |
| 10 | **Consulta electrónica / reporte** depende de servicios SRI cortados. | Buscar forma alternativa. **Al final.** | `[39:01]` |

---

## Plan de migración (consolidado)

- **Destino tecnológico:** Java 25, plantilla generada, integración continua.
- **Enfoque:** por módulos, con pantallas nuevas conviviendo con las actuales (botón "versión 2").
- **Foco de código:** proyecto **GTF Root**.
- **Ignorar:** código de gastos personales / viáticos / "nueva reposición gasto".
- **Orden:** (1) Administración de reposición → (2) Revisión de reposición → (3) resto → (4) Depósitos.
- **Gestión:** historia en Jira/Comprendo (proyecto **GTF2**) con módulos a migrar y registro de errores.

---

## Acciones y responsables

| Acción | Responsable | Ref. |
|--------|-------------|------|
| Documentar las reglas de negocio. | Líder (Inés/"Inesita") | `[50:50]` |
| Crear la historia en Jira con los módulos a migrar. | Líder / equipo | `[39:48]` |
| Descargar los proyectos base (`GTF29`, ~4 GB). | Todo el equipo | `[52:39]` |
| Crear tarea "levantar JBoss" y compartir el JBoss (permanente). | Carlos | `[51:46]` |
| Coordinar el rediseño de retenciones electrónicas (SRI). | Con Frank | `[21:05]` |
| Desplegar el **FCM** a prod (compactar a master, release, paso a prod). | Carlitos | `[54:34]` |
| Asistir (como oyentes) a la reunión de las 11:00 sobre pantallas de soporte. | Todo el equipo | `[53:20]` |

---

## Glosario

| Término | Significado |
|---------|-------------|
| **GTF** | Sistema de gestión de caja chica / fondos de locales de Corporación Favorita. |
| **Caja chica** | Fondo de dinero asignado a cada local para gastos menores. |
| **Reposición** | Solicitud de reembolso de gastos del local para reponer el fondo. |
| **SECT** | Tipo de usuario "secretario" de local (con turno M/T). |
| **SIF / CIF** | Sistema de facturación del que GTF consume conceptos y validaciones. |
| **SRI** | Servicio de Rentas Internas (Ecuador). |
| **LC / LO** | Liquidación de Compra / Liquidación Otros (tipos de concepto). |
| **Retención** | Comprobante de retención de impuestos (IVA / renta), electrónica o manual. |
| **Punto de emisión** | Configuración requerida en el CIF para emitir/validar documentos de ley. |
| **JBoss** | Servidor de aplicaciones donde corre el sistema legacy. |
| **Comprendo / Jira / Fizz** | Herramientas de gestión de tareas y despliegue. |
| **FCM** | Otro componente en despliegue (conversación aparte del cierre). |
| **AINAC** | Servicios antiguos referenciados por consultas legacy a portar. |

---

*Documento generado a partir de la transcripción automática del video. Las marcas de tiempo son aproximadas; los nombres propios pueden tener variaciones de la transcripción (p. ej., "Inés/Inesita", "Cintia/Cristina Escobar").*
