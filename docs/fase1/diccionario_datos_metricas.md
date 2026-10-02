# Fase 1 · Diccionario de datos y de métricas

Estado: **borrador v0.3 para validación de la PMO** (incorpora las decisiones de la sección 9) · Alcance inicial: aeropuerto GDL · Base de análisis: 3 cortes de Performance (16, 23 y 30-sep-2026) y 1 OPR (contrato GDLC25-027, OC GDL-OC-0007238).

Este documento no contiene código, SQL ni diseño de pantallas. Define **qué es cada dato, de dónde viene, cómo se valida y cómo lo trata la plataforma**.

---

## 0. Convenciones

**Clases de campo** (columna "Clase"):

| Clase | Significado |
|---|---|
| **ID** | Identifica a una entidad (aeropuerto, proyecto, contrato). |
| **DIM** | Atributo descriptivo que cambia lento (responsable, proveedor, estatus). Se guarda con vigencia. |
| **CAP** | Dato capturado por una persona. Es el más propenso a error. |
| **OPR** | Dato que nace en el informe OPR y el Performance solo lo replica. |
| **CALC** | Calculado en el Excel. La plataforma lo **ignora como fuente** y lo recalcula; el valor del Excel se guarda como "reportado" para conciliar. |
| **REF** | Catálogo o referencia. |

**Reglas generales de la plataforma**

1. Nada se sobrescribe: cada carga queda íntegra y una recarga crea versión nueva.
2. Se guardan **medidas base**; los ratios se recalculan con precisión completa y siempre sobre sumas (nunca promedios de ratios).
3. Cada medida conserva su **procedencia**: fórmula vinculada, fórmula local, valor pegado o capturado.
4. Valores con |x| < 0.000001 se tratan como 0 (el OPR trae residuos como `1e-11`).
5. Texto vacío, `-`, `N/A` y nulo se normalizan a "sin dato", salvo que el campo permita 0.
6. Todo parseo es **por nombre de columna o etiqueta**, nunca por posición ni por número de fila. Los encabezados se normalizan (espacios sobrantes, `$ E stimaciones`, mayúsculas).
7. Los archivos se leen **sin ejecutar macros ni actualizar vínculos externos**.

---

## 1. Fuentes y grano

| Fuente | Formato | Grano | Frecuencia | Rol |
|---|---|---|---|---|
| **Performance · Control de Contratos** (GAPINFRA-F-074) | .xlsm | 1 contrato (u OC) × 1 corte | Semanal, límite miércoles | **Única fuente cargada en la Fase 1** (decisión D-01) |
| OPR · Avance Semanal (GAPINFRA-F-036 Rev.02) | .xlsb | 1 contrato × 1 semana (serie semanal completa desde el inicio del contrato) | Semanal, límite miércoles | **Fuera de alcance** (D-01). Existen todos los OPR de 2026 pero no se usarán. Se documenta como referencia de origen de PV, EV, E, facturado y convenios, que la PMO copia a mano al Performance |
| Hoja ODC (dentro de Performance) | .xlsm | 1 ODC | Semanal | **Degradada**: idéntica en 3 cortes, no alimenta nada (ver §5, regla Q-12) |
| Hoja Facturas (dentro de Performance) | .xlsm | 1 factura | Semanal | **Degradada**: 14 filas, idéntica en 3 cortes, sin fechas |

**Nombres de archivo**

- Performance: `AAMMDD IATA – Control de Contratos`. El archivo recibido no incluye el IATA.
- OPR: `AAMMDD <código de proyecto, 6 caracteres> <últimos 4 dígitos de la OC> – Avance Semanal`.
- Regla: el **contenido manda sobre el nombre**. Si el nombre dice una OC y el contenido otra, la carga se bloquea.
- Los últimos 4 dígitos de la OC no son únicos entre prefijos (GDL-OC-, DTI-OC-); la identidad se confirma con la OC completa leída del contenido.

**Completitud esperada de un corte (Fase 1)**: 1 Performance del aeropuerto. Cuando se incorpore el OPR, se esperará además 1 OPR por cada contrato ACTIVO.

**Consecuencia de cargar solo el Performance**: la plataforma no concilia PV, EV y Estimado contra su origen, ni tiene PPC, riesgos, fianzas, causas de no cumplimiento ni historia semanal propia. Esos números se aceptan como los captura la PMO y se protegen con reglas de consistencia entre cortes (Q-11 a Q-15 y Q-21). Con una sola fuente, la ingesta es más simple y la historia de la plataforma nace de los archivos semanales de Performance (los de 2026 están disponibles).

---

## 2. Identidad y calendario

### 2.1 Claves
| Entidad | Clave natural | Notas |
|---|---|---|
| Aeropuerto | Código IATA | Hoy solo GDL. |
| Proyecto | Código de proyecto (`29INFRA - GDL121`) | Asignado por la PMO. |
| Contrato / línea de control | **(Aeropuerto, Orden de Compra)** | La OC fue única en 31 de 31 filas y estable en los 3 cortes. `Contrato` queda como atributo: 5 de 34 filas tienen valor vacío, `N/A` o `-`. |
| Alcance sin contratar | **ID de línea** (a crear) | 3 filas sin OC ni proveedor hoy no tienen identidad. Se propone un ID asignado por la PMO. |
| Proveedor | ID interno + alias | 31 valores en 28 distintos; variantes por comas y mayúsculas. |
| Persona | ID interno + alias | Variantes de nombre y errores de captura. |

### 2.2 Tiempo: fechas y semana PMO (decisión D-03: la semana cierra en jueves)
| Fecha | Dónde aparece | Observación |
|---|---|---|
| **Fecha de corte** | Se declara al cargar | Fecha oficial de la plataforma; por defecto toma la del nombre del archivo y el usuario la confirma. |
| Fecha del nombre | `AAMMDD` | Miércoles de entrega (30-sep-2026 fue miércoles). |
| Fecha dentro del OPR | Celda "Fecha" | Este OPR trae 30-sep y 01-oct en dos celdas distintas. |
| Cierre de semana del OPR | Filas semanales | Las filas semanales de este OPR terminan en **jueves**; el instructivo dice domingo. A resolver. |
| Fecha de guardado | Metadatos del archivo | **No usar**: el archivo del 23 se guardó el 02-oct. |

**Regla acordada: la semana PMO cierra en jueves.** Se verificó con el contrato GDLC25-027: el Performance del miércoles 16-sep trae los valores de la semana que cierra el jueves 17-sep; el del 23-sep, los del jueves 24-sep; y el del 30-sep, los del jueves 01-oct. Es decir, **el archivo del miércoles reporta la semana que cierra el jueves siguiente** (parte de ella aún no ha ocurrido al entregar).

Por tanto:
- **Semana PMO** = el jueves de cierre. Es la fecha de corte oficial de la plataforma (corte 16-sep → 17-sep; 23-sep → 24-sep; 30-sep → 01-oct).
- La fecha del nombre del archivo se guarda como **fecha de entrega** y se compara con la anterior.
- Si el archivo se entrega tarde (después del jueves de cierre), la plataforma lo marca como "entrega tardía" y no lo asigna automáticamente a otra semana: la PMO confirma.
- El corte oficial se calcula, el usuario lo confirma al cargar.
- El instructivo del OPR habla de domingo; debe corregirse a jueves.

---

## 3. Diccionario · Performance (`ES | Performance`)

Tabla definida como `Performance_ES`. Encabezado en la fila 3, totales en la fila 2, filas plantilla vacías arriba (4 a 10), bloque BAC/EAC/VAC debajo (≈ filas 55 a 85) y datos reales entre medias. **Se localiza por nombre de tabla y encabezado.**

Leyenda de "Verif.": ✔ = coincide exactamente con el OPR del contrato GDLC25-027; ○ = por verificar con más OPR.

### 3.1 Identificación y clasificación
| Campo | Clase | Regla de calidad | Tratamiento |
|---|---|---|---|
| Aeropuerto | ID | Debe ser IATA válido y coincidir con el aeropuerto de la carga. | Dimensión. |
| Presupuesto (ADP / OPEX / INVERSIÓN / PMD) | DIM | Lista cerrada. Hoy solo PMD. | Dimensión de fondo. |
| Dirección Gestora | DIM | No vacío. | Dimensión. |
| Nivel I, Nivel II (WBS) | DIM | Lista cerrada; Nivel II depende de Nivel I. | Jerarquía de análisis. |
| Proyecto, Nombre de Proyecto | ID/DIM | Código asignado por la PMO; nombre sin abreviaturas. 8 proyectos en GDL. | Entidad Proyecto. |
| Responsable, Responsable Obra | DIM | Persona del catálogo; hoy hay variantes y errores. `Responsable Obra` vacío en 11 de 34. | Asignación con vigencia. |
| Alcance | CAP | Texto libre; único por fila. | Atributo del contrato. |
| Orden de Compra | **ID** | Obligatoria salvo alcances sin contratar; única por aeropuerto. | **Clave del contrato.** |
| Contrato | DIM | Formato `GDLC25-027`; rechazar `N/A` y `-` como valores válidos. | Atributo. |
| Proveedor (Nombre Comercial) | DIM | Debe resolverse al catálogo; el instructivo pide razón social completa y el encabezado dice nombre comercial (contradicción). | Entidad Proveedor + alias. |
| Estatus | DIM | Lista cerrada: **PLANEACIÓN, ACTIVO (= en ejecución), TERMINADO**. `EN EJECUCIÓN` queda como alias de ACTIVO. Todo cambio de estatus pasa por aprobación de la PMO (ver Q-04, Q-05). | Dimensión con vigencia y log de aprobación. |

### 3.2 Contractual y financiero
| Campo | Clase | Verif. | Regla / tratamiento |
|---|---|---|---|
| $ Ppto. Base | CAP | ○ | Sin IVA. No debe cambiar entre cortes; un cambio dispara alerta. |
| $ Ppto. Contratado | CAP | ○ | Sin IVA. Estable; ídem. |
| $ Convenios Autorizados | OPR | ✔ | Suma de convenios del OPR (A1 a A15: aditivas + deductivas + OC + reclamos), copiada por la PMO. El instructivo dice que viene de la hoja ODC, pero esta no cambia. **Los valores negativos son legítimos** (deductivas reales, confirmado por Contratos, D-09) y no generan alerta; solo el cambio entre cortes se presenta como evento de convenio. |
| $ Ppto. Total Contratado | CALC | ✔ | Contratado + convenios. Se recalcula. |
| % Cambios | CALC | ○ | Convenios / contratado original. Límite de 20 % según instructivo (regla Q-10). |
| $ ODC Potencial | OPR | ✔ | "ODC por presentar" del OPR. Hoy cambia entre positivo, 0 y nulo; se normaliza el nulo a 0. |
| $ Ppto. Mejor Esperado | CALC | ✔ | Total contratado + ODC potencial. |
| % Anticipo | CAP | ○ | 0 a 100 %. |
| $ Anticipo Facturado | CALC | ○ | % anticipo × contratado. En el OPR se llama "Anticipo otorgado" (GDLC25-027: $432.8 M, 40 %). |
| $ Valor Programado (PV) | **OPR** | ✔ | Acumulado semanal incluida la semana actual. Entre cortes pasó de fórmula vinculada a valor pegado: se guarda procedencia. **Se toma del OPR.** |
| $ Valor Ganado (EV) | **OPR** | ✔ | Ídem. |
| $ Estimado (E) | **OPR** | ✔ | Estimaciones acumuladas. |
| $ Facturado | **OPR** | ✔ | **Es bruto: incluye el anticipo.** GDLC25-027: $432.8 M anticipo + $310.4 M facturado neto = $743.2 M. Por eso supera al EV sin ser anomalía. La plataforma lo separa en anticipo, amortizado y facturado neto. |
| $ Facturado Programado | OPR | ○ | Acumulado programado de facturación. |
| $ Por Facturar | CALC | ○ | Se recalcula. |
| $ Estimaciones en Revisión | OPR | ○ | Del OPR. |
| $ OENE | CALC | ✔ | **EV − E**; coincide en 100 % de las filas de los 3 cortes. |
| $ OENE Contratada | CAP/OPR | ✔ | Parte de la OENE ya respaldada por convenio. |
| $ OENE por Regularizar | CAP/OPR | ○ | Parte sin respaldo contractual. Regla Q-09: OENE contratada + por regularizar ≤ OENE. |
| $ OENE Por Facturar, $ Desviación Por Facturar | CALC | ○ | Se recalculan. |
| % OENE, % Financ. | CALC | ○ | OENE / total contratado. Los dos son idénticos en los datos. |
| $ Desviación, % Desviación | CALC | ○ | EV − PV y su proporción sobre PV. |
| % PV, % EV, % E, % F, % FP | CALC | ○ | Cada medida sobre el total contratado. |

### 3.3 Desempeño
| Campo | Clase | Verif. | Definición oficial en la plataforma |
|---|---|---|---|
| SPI | CALC | ✔ | **EV / PV**, con precisión completa. El Excel lo redondea a 2 decimales. |
| CPI | CALC | ✔ | **Se renombra "Índice de Estimación" = E / EV** (confirmado por la PMO). Mide qué proporción de lo ejecutado ya está estimado; menor a 1 significa OENE. El instructivo lo documenta al revés (EV/E) y debe corregirse. |
| PPC | OPR | ○ | Vacío en las 34 filas. Existe en el OPR, pero el OPR no se carga en la Fase 1: **no habrá PPC hasta que alguien lo capture en Performance o se incorpore el OPR**. No se promete en las pantallas iniciales. |
| Holgura | CAP | ○ | Origen: MS Project; **la captura la hará una persona en Performance** (D-08). Hoy vacío en las 34 filas. Días enteros; positivo = atraso, negativo = adelanto. Un valor vacío en contrato ACTIVO es advertencia (Q-22). |

### 3.4 Plazo
| Campo | Clase | Regla / tratamiento |
|---|---|---|
| Plazo de Ejecución | CAP | Días naturales; entero positivo. |
| Acta de Inicio Replanteo (AIR) | CAP | Obligatoria si el contrato está ACTIVO. Fecha válida. |
| Fin de ejecución Original | CALC | AIR + plazo. |
| Incremento en Días | CALC/OPR | Días de convenios autorizados. |
| Incremento en Días (En Proceso) | CALC/OPR | **Debe ser ≥ 0**. Se observó −177 y −4 durante 2 semanas (regla Q-06). |
| Plazo total contratado | CALC | Plazo original + incrementos autorizados. |
| Fin Conveniado | CALC | Fin original + incrementos autorizados. |
| Fin Previsto | CALC | Fin conveniado + días en proceso. Con días negativos se adelanta sin sentido. |
| Fin Proyectado | CALC | Proyección con SPI. Se recalcula a la fecha de corte. |
| Acta de Entrega Recepción | CAP | Vacía en los 15 TERMINADO. Obligatoria al terminar (regla Q-07). |
| % Documental, % Físico, Comisionamiento | CAP | 0 a 100 %; la fecha de comisionamiento no puede ser anterior al AIR. |
| % Tiempo transcurrido | CALC | A la **fecha de corte**, no a la fecha de apertura. |
| Días para Fin | CALC | Fin previsto − fecha de corte. Hoy vacío o `-`. |
| % Incremento | CALC | Incremento / plazo original. |

### 3.5 KPI del Excel (se guardan como "reportado", no son oficiales)
| Campo | Lógica del Excel | Problema | Tratamiento |
|---|---|---|---|
| KPI - OENE | Alto ≥ 10 %; moderado > 5 % y < 10 %; controlado ≤ 5 % | Trata ACTIVO como "SIN INICIAR" | Se recalcula con la tabla de umbrales (§6). |
| KPI - Desviación | Alta ≤ −10 %; moderada entre −10 % y −5 %; esperada ≥ −5 %; positiva > 5 %; alto desempeño > 10 % | Mismo problema; los rangos se solapan | Se recalcula con rangos exclusivos. |
| KPI - Tiempo transcurrido | Fuera de plazo, por terminar (≥ 90 %), en curso | Usa la fecha actual | Se recalcula a la fecha de corte. |
| KPI - Incremento | Adelanto, sin cambio, menor ≤ 20 %, moderado ≤ 50 %, mayor > 50 % | Depende de AIR | Se recalcula. |
| KPI - Incumplimiento en Plazo | Cascada de mensajes (vencido, sin avance, rezagado…) | Usa `HOY()`; no distingue TERMINADO; 16 de 34 filas en "vencido" | Se recalcula. Un contrato TERMINADO nunca es "vencido en ejecución". |

### 3.6 Curva mensual (`ene-25` a `dic-28`, 48 columnas + 4 subtotales anuales)
- Origen: Performance. Las 48 columnas **no cambiaron en 3 cortes**.
- Definición de la PMO: **facturación real del mes**.
- **Aclarado por la PMO (D-02)**: los meses ya cerrados son **facturación real**; los **meses futuros son montos programados**. La plataforma los distingue por la fecha de corte: mes anterior al del corte = real, mes posterior = programado; el mes que contiene el corte se marca "parcial".
- Origen del dato real: no se conoce el sistema. Para GDLC25-027 **no coincide con las series mensuales del OPR** (sep-26: Performance $37.3 M; OPR facturado real $55.7 M). Se acepta como lo captura la PMO y se etiqueta "no conciliado".
- Se contrastó con todas las series mensuales del OPR (PV, EV, estimado, facturado neto, amortizado, facturación programada) y ninguna coincide: jul-26 $21.3 M contra estimado $7.0 M; ago-26 $33.6 M contra $58.5 M.
- **Tratamiento**: se almacena por mes con su tipo (real, parcial, programado). Puede usarse en la vista de flujo mensual, con la etiqueta "origen Performance, no conciliado". No entra en KPI ejecutivos hasta que se identifique el sistema de origen. Los subtotales anuales se descartan (se recalculan).
- **Regla de estabilidad (Q-21)**: un mes ya cerrado no debe cambiar entre cortes; si cambia, es reescritura de la historia y se señala.

### 3.7 Columnas que aparecieron o desaparecieron
- **Corte 16-sep**: 2 columnas extra (`Required Effort`, `Effort Increse2`) entre SPI y CPI; el resto se desplazó 2 posiciones. No existen en los cortes 23 y 30.
- Tratamiento: columnas desconocidas se guardan en un área "no mapeada" y generan advertencia; no bloquean.

---

## 4. Diccionario · OPR (`GAPINFRA-F-036`) — fase posterior

> **No se carga en la Fase 1 (D-01).** Se documenta porque el modelo de datos debe reservar sitio para estas entidades (convenios, fianzas, riesgos, PPC, causas, serie semanal) y porque es el origen real de los números de Performance.

Hojas: `Instrucciones`, `ES | Seguimiento de Ejecución`, `ES | Lean`, `ES | OPR` (la que se presenta), `Flecha` (colores, se ignora). Contiene imágenes y gráficas (el archivo pesa ≈ 6 MB): se conservan como adjuntos y no se interpretan.

### 4.1 Cabecera del contrato (hoja `ES | OPR`)
| Campo | Clase | Regla / tratamiento |
|---|---|---|
| Aeropuerto, Código de proyecto, Nombre del proyecto | ID | Debe coincidir con Performance. |
| Contrato, Orden de Compra, Proveedor, Serie PMD | ID/DIM | Debe coincidir con Performance (conciliación de identidad). |
| STE / PMO (supervisión) | DIM | Empresa supervisora (en este OPR: una consultora de gestión). Entidad nueva. |
| Responsable (SIAP) | DIM | Debe resolver al mismo responsable que Performance. |
| Fecha, semana actual del contrato | CAP | Ver §2.2. |
| $ Presupuesto Base, Contratado, Total Contratado, Mejor Esperado, Convenios autorizados / en trámite, ODC por presentar | OPR | Conciliar con Performance. |

### 4.2 Plazos y garantías
| Campo | Clase | Regla / tratamiento |
|---|---|---|
| Plazo de ejecución original (inicio/fin), convenios en plazo (días), terminación final esperada, terminación de comisionamiento, cierre administrativo | OPR | Complementa y reemplaza el cálculo de fechas del Excel. |
| **Fianzas**: anticipo, cumplimiento, seguro de responsabilidad civil (fecha de vencimiento y días restantes) | OPR | Entidad nueva. Alerta por vencimiento próximo (umbral a definir). En GDLC25-027: 17-oct-2027, 17-oct-2027, 23-jul-2028. |
| Anticipo otorgado, % y pendiente por amortizar | OPR | Base para separar facturado bruto y neto. |

### 4.3 Convenios (A1 a A15)
Por cada convenio: **monto inicial, aditivas, deductivas, OC, reclamos, aumento, acumulado, monto total, fecha final inicial y fecha final previsto.**

- Se modelan como **eventos** (un convenio es un hecho con identidad), no como foto.
- El −$0.23 del GDLC25-027 es una deductiva real del convenio A1 (ajuste de redondeo en el alta), no ruido.
- Regla Q-10: el acumulado de convenios (en positivo) no debe superar 20 % del contratado original sin alerta. Las deductivas son válidas (D-09).

### 4.4 Avance semanal y acumulado (`ES | Seguimiento de Ejecución`)
Serie semanal desde el inicio del contrato. Por semana:

| Campo | Notas |
|---|---|
| Año (semana del año), # (semana del contrato), Mes, Fecha | Cierre de semana en jueves. |
| Semana programada, acumulado programado, % | PV semanal y acumulado. |
| Semana real, acumulado real, % | EV semanal y acumulado. |
| Estimado semanal, acumulado, % | E. |
| Facturado programado / real / acumulado, % | Facturación neta. |
| Amortizado, acumulado, % | Amortización del anticipo. |
| Facturado mensual programado vs real, diferencia | Agregación mensual. |

Tratamiento:
- Cada OPR trae **toda la historia semanal**; se carga completa en cada corte y se compara con lo que el mismo contrato reportó antes (para detectar reescritura de la historia).
- Las filas futuras traen programado pero no real: se distinguen por la "semana actual del contrato".
- Se detectaron valores redondos sospechosos (EV semanal = $5,000,000 exactos en la semana 59 del GDLC25-027): regla de valores redondos (Q-13).

### 4.5 Lean construction (`ES | Lean`)
| Campo | Notas |
|---|---|
| Holgura (días) | Resuelve el campo vacío de Performance. |
| PPC histórico semanal (% y fecha) | Serie semanal. |
| **Causas de no cumplimiento (CNC)**: catálogo de **18 claves** (SM suministro de materiales, FE equipo, FT fuerza de trabajo, TO/TP trabajos previos de otros o propios, CD calidad deficiente, EP error de programación, ET estimación incorrecta de tiempo, RT retrabajo, TT trámite a destiempo, DI diseño, RP requerimientos fuera de proyecto, AD contrato/ODC/convenios, CI condiciones inseguras, SO seguridad operacional, CC clima, LA liberación de áreas por el aeropuerto, LT logística) y total por causa | Clasificación acordada (D-04): **la única causa externa es LA (liberación de áreas por el aeropuerto)**; las otras 17 se consideran atribuibles al contratista. Como el OPR queda fuera de alcance, **esta información no estará en la plataforma**; el ranking de contratistas de la Fase 1 será solo descriptivo (sin causas) y así deberá presentarse. |

### 4.6 Riesgos
| Campo | Notas |
|---|---|
| **Matriz de riesgos**: riesgo, probabilidad, impacto, clasificación, costo, tiempo (días), estrategia (transferir, mitigar, escalar, evitar) | Identidad del riesgo = texto libre, sin ID. Se propone un **ID de riesgo** en la plantilla. Mientras tanto, identidad provisional por contrato + texto normalizado. |
| **Actividades en riesgo**: solicita, fecha requerida, fecha promesa, días abierto | Eventos de seguimiento; el "días abierto" se recalcula a fecha de corte. |
| Línea de tiempo, avance gráfico, reporte fotográfico | Adjuntos. |

### 4.7 Índices del OPR
| Campo | Definición | Observación |
|---|---|---|
| SPI | EV / PV | Coincide con Performance. |
| CPI | **E / EV** | Misma definición que usa Performance; confirmado por la PMO. |
| Índice de Facturación | Facturación real acumulada / EV (GDLC25-027: 1.37) | **Incluye el anticipo**, por lo que engaña como indicador de salud. Se reemplaza por facturado neto / EV. |
| Desviación semanal y acumulada | EV − PV | |
| PPC, holgura | Ver §4.5. | |

---

## 5. Catálogo de reglas de calidad

Severidad: **B** = bloquea la publicación; **A** = advertencia que exige justificación; **I** = informativa.

| ID | Regla | Sev. | Evidencia observada |
|---|---|---|---|
| Q-01 | Estructura reconocida (hojas, tabla y encabezados por nombre) | B | Columnas extra el 16-sep |
| Q-02 | Una sola fila por (aeropuerto, OC) en el corte | B | — |
| Q-03 | OC presente en contratos ACTIVO y TERMINADO | B | 3 filas sin OC |
| Q-04 | Estatus dentro de la lista cerrada; el cambio exige aprobación de la PMO | B | 2 contratos ACTIVO → TERMINADO → ACTIVO |
| Q-05 | Un contrato TERMINADO no puede volver a ACTIVO sin aprobación y motivo | B | Ídem |
| Q-06 | Días en proceso ≥ 0 | B | −177 y −4 |
| Q-07 | TERMINADO exige acta de entrega-recepción | A | 15/15 sin acta |
| Q-08 | ACTIVO exige AIR y EV > 0 | A | — |
| Q-09 | OENE contratada + por regularizar ≤ OENE | A | ○ |
| Q-10 | Convenios acumulados ≤ 20 % del contratado original | A | ○ |
| Q-11 | EV ≤ total contratado × (1 + tolerancia) | A | 2 a 3 contratos por semana |
| Q-12 | Performance vs OPR: PV, EV, E, facturado, convenios y fechas deben coincidir | A | **Diferida** hasta que se cargue el OPR. Mientras tanto: los convenios de Performance no coinciden con la hoja ODC en 7/31 contratos |
| Q-13 | Valores redondos exactos o repetidos en EV/PV/E semanal | A | EV = $5,000,000 en la semana 59 |
| Q-14 | Acumulados no decrecientes (EV, E, facturado, amortizado) | A | ○ |
| Q-15 | Campos maestros estables (base, contratado, proveedor, responsable) | A | Estables en 3 cortes |
| Q-16 | Proveedor y responsable resueltos al catálogo | A | 31 valores, 28 distintos |
| Q-17 | OPR recibido para cada contrato ACTIVO | A | **Diferida** (no aplica en Fase 1) |
| Q-18 | Fecha de entrega coherente con el jueves de cierre derivado (§2.2); entrega tardía marcada | A | Tres fechas distintas |
| Q-19 | Vencimiento de fianzas dentro del umbral | I | **Fuera de alcance** (depende del OPR) |
| Q-20 | Archivos con macros o vínculos externos: se registran y se leen en modo aislado | I | 3 vínculos externos en el Performance |
| Q-21 | Los meses cerrados de la curva mensual no cambian entre cortes | A | Estables en 3 cortes |
| Q-22 | Holgura presente en contratos ACTIVO | A | Vacía en 34 de 34 |
| Q-23 | PPC presente en contratos ACTIVO (si la PMO decide capturarlo) | I | Vacío en 34 de 34 |

---

## 6. Diccionario de métricas oficiales

Las fórmulas son **definiciones conceptuales** para acordar con la PMO; aún no son implementación.

| Métrica | Definición oficial | Agregación | Umbrales / lectura |
|---|---|---|---|
| **BAC / Total contratado** | Contratado + convenios autorizados | Suma | — |
| **Mejor esperado** | Total contratado + ODC potencial | Suma | — |
| **PV, EV, E** | Acumulados al corte, del OPR | Suma | — |
| **% Avance programado / ganado / estimado** | PV, EV o E ÷ total contratado | Ratio de sumas | — |
| **SPI** | EV ÷ PV | **ΣEV ÷ ΣPV** | Crítico < 0.90 (instructivo) |
| **Índice de Estimación** (antes "CPI") | E ÷ EV | **ΣE ÷ ΣEV** | Crítico < 0.90 |
| **Desviación $ / %** | EV − PV; ÷ PV | Suma / ratio de sumas | Alta ≤ −10 %; moderada −10 % a −5 %; esperada ≥ −5 %; positiva > 5 %; alto desempeño > 10 % |
| **OENE $** | EV − E | Suma | — |
| **% OENE** | OENE ÷ total contratado | Ratio de sumas | Controlado ≤ 5 %; moderado > 5 % y < 10 %; alto ≥ 10 % |
| **OENE regularizada** | OENE contratada ÷ OENE | Ratio de sumas | Seguimiento |
| **Facturado bruto** | Anticipo + facturado neto | Suma | No equivale a avance |
| **Facturado neto** | Facturado − amortizado | Suma | — |
| **Anticipo pendiente de amortizar** | Anticipo − amortizado acumulado | Suma | — |
| **Índice de facturación neta** | Facturado neto ÷ EV | Ratio de sumas | Reemplaza al "Índice de Facturación" del OPR |
| **% tiempo transcurrido** | (Fecha de corte − AIR) ÷ plazo total | Por contrato | Por terminar ≥ 90 % |
| **Días para fin** | Fin previsto − fecha de corte | Por contrato | Negativo = vencido |
| **Corrimientos de fecha** | Número de veces que cambió Fin Previsto entre cortes y días acumulados | Contrato | Señal de atraso crónico |
| **% Incremento de plazo** | (Fin previsto − fin original) ÷ plazo original | Por contrato | Menor ≤ 20 %; moderado ≤ 50 %; mayor > 50 % |
| **PPC** | Del OPR (Lean), semanal | Promedio ponderado por actividades cuando exista el dato | — |
| **Causas de no cumplimiento** | Conteo por clave CNC y por periodo (excluye LA del puntaje) | Suma | Solo cuando se cargue el OPR |
| **Contratos críticos** | Contratos con SPI < 0.90 **o** % OENE ≥ 10 % **o** KPI de plazo en crítico | Conteo y valor contratado | **Aprobado (D-05)** |
| **Salud del programa** | Se aplican los **umbrales ya establecidos por la PMO** (instructivo del formato, §6 y apartado "Indicadores críticos") sobre las sumas del programa: ΣEV/ΣPV, %OENE total, % de contratos críticos y valor contratado en riesgo | Programa | Definidos (D-10); falta solo fijar cómo se agregan a un único color |

---

## 7. Matriz de versiones de plantilla

| Plantilla | Versión observada | Diferencias detectadas | Estrategia |
|---|---|---|---|
| Performance (F-074) | Sin número de versión; cortes 16, 23, 30 | 16-sep tiene 2 columnas extra | Firma por conjunto de encabezados; registrar una "huella de plantilla" por carga |
| OPR (F-036) | Rev.02 | Una sola muestra | Mismo mecanismo; pedir 3 a 5 OPR de contratos y semanas distintas |

**Cambios recomendados a las plantillas** (los acordados hasta ahora): número de versión visible, ID de línea, ID de riesgo, estatus con lista cerrada y corregida (sin `EN EJECUCIÓN`), validación en `Facturas`, catálogos para proveedor y responsable, y corrección de las fórmulas de KPI que tratan ACTIVO como "SIN INICIAR".

---

## 8. Estado de preguntas

Respuestas de la PMO del 02-oct-2026:

| ID | Estado | Resolución |
|---|---|---|
| P-01 | Cerrada | Solo Performance |
| P-02 | Cerrada con reserva | Meses pasados = real, futuros = programado. Sistema de origen desconocido, no concilia con el OPR |
| P-03 | Cerrada | Los OPR de 2026 existen pero no se usarán |
| P-04 | Cerrada | La semana cierra en jueves |
| P-05 | Cerrada | La mayoría de las causas es del contratista; solo LA es externa |
| P-06 | Cerrada | Reglas ya establecidas en el instructivo del formato |
| P-07 | Cerrada | Holgura viene de MS Project y la captura una persona en Performance |
| P-08 | Cerrada | Todos los OPR usan la misma plantilla (irrelevante por D-01) |
| P-09 | Cerrada | No aplica |
| P-10 | Cerrada | Las deductivas negativas son convenios reales |

Queda abierta **P-11**: ¿alguien capturará PPC en Performance? Hoy está vacío y su origen es el OPR, que no se usará.

## 8.1 Preguntas originales (referencia histórica)

| ID | Pregunta | Quién |
|---|---|---|
| P-01 | ¿Se cargará **cada OPR** (≈ 31 por semana) o solo el Performance? Recomendación: ambos, con el OPR como fuente. | PMO |
| P-02 | La curva mensual de Performance no coincide con el OPR. ¿De qué sistema sale (SAP, facturas con IVA, otro)? ¿Por qué hay montos en meses futuros? | PMO |
| P-03 | ¿Existen los OPR de todo 2026 para todos los contratos? Con ~39 semanas × 31 contratos serían cerca de 1,200 archivos de ≈ 6 MB. | PMO |
| P-04 | ¿Cierra la semana el jueves, el domingo o el miércoles? Definir la **semana PMO**. | PMO |
| P-05 | Clasificación de causas CNC en atribuibles al contratista o externas. | PMO + Dirección |
| P-06 | Reglas del semáforo de salud del programa y de "contrato crítico". | Dirección |
| P-07 | Holgura: ¿viene de MS Project? ¿Quién lo captura en el OPR? | PMO |
| P-08 | ¿Los OPR de otros aeropuertos usarán la misma plantilla? | PMO |
| P-09 | Política de retención de archivos originales y adjuntos (fotos). | TI / PMO |
| P-10 | ¿Las deductivas negativas en convenios (−$10.8 M, −$5.6 M) corresponden a convenios reales? | Contratos |

---

## 9. Decisiones registradas

| ID | Decisión | Fecha | Impacto |
|---|---|---|---|
| D-01 | Solo se carga el Performance; el OPR queda **fuera de alcance** (existen los de 2026 pero no se usarán) | 02-oct-2026 | Sin conciliación contra el origen ni PPC, riesgos, fianzas o causas. El modelo puede reservar sitio por si se retoma |
| D-02 | Curva mensual: meses cerrados = facturación real, meses futuros = programado; se distingue por la fecha de corte | 02-oct-2026 | Usable en flujo mensual con etiqueta "no conciliado"; fuera de KPI ejecutivos |
| D-03 | La semana PMO cierra en jueves; el archivo del miércoles reporta la semana que cierra el jueves siguiente | 02-oct-2026 | Fecha de corte = jueves derivado; el nombre del archivo es fecha de entrega |
| D-04 | La única causa externa es LA (liberación de áreas por el aeropuerto) | 02-oct-2026 | Aplica al ranking cuando exista el dato |
| D-05 | Contrato crítico = SPI < 0.90 o % OENE ≥ 10 % o KPI de plazo en crítico | 02-oct-2026 | Base de KPI ejecutivos |
| D-06 | ACTIVO equivale a en ejecución; el CPI se redefine como Índice de Estimación = E/EV | previas | Ver §3.1 y §3.3 |
| D-07 | Los cambios de estatus los aprueba la PMO | previas | Reglas Q-04 y Q-05 |
| D-08 | La holgura (MS Project) la captura una persona en Performance | 02-oct-2026 | Campo CAP con regla Q-22 |
| D-09 | Las deductivas negativas en convenios son legítimas | 02-oct-2026 | Sin alerta por signo negativo |
| D-10 | Los umbrales de semáforo son los ya establecidos en el instructivo del formato | 02-oct-2026 | Se incorporan al catálogo de reglas |
