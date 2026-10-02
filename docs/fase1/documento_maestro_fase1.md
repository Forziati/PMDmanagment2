# Plataforma de Control Histórico de Contratos (PMO) · Documento Maestro de Fase 1

**Versión 1.1 (propuesta de cierre de Fase 1)** · 02-oct-2026
Alcance: aeropuerto GDL · Fuente única: Excel **Performance** (GAPINFRA-F-074)
Base de evidencia: 3 cortes de Performance (16, 23 y 30-sep-2026) y 1 informe OPR (referencia).
Documento hermano: `diccionario_datos_metricas.md` (detalle campo por campo, reglas de calidad y registro de decisiones D-01 a D-11).

> Este documento no contiene código, SQL, APIs ni componentes. Define qué se construye, por qué y en qué orden. Los puntos marcados **[Decisión pendiente]** requieren respuesta antes de la Fase 2.

---

## 0. Resumen ejecutivo

**El problema.** Cada semana la PMO consolida a mano en un Excel el desempeño de los contratos del aeropuerto. El archivo es una fotografía: al reemplazarlo se pierde la historia y con ella la capacidad de saber qué empeoró, qué mejoró y qué se repite.

**La solución.** Una plataforma que recibe ese Excel cada semana, lo valida, lo guarda sin sobrescribir nada y construye sobre él historia, comparativos, tendencias y alertas. El Excel pasa de ser herramienta de análisis a ser solo vehículo de captura.

**Lo que se aprendió de los datos reales** (cambia el diseño original):

1. El Excel **no es la fuente de los números**: PV, EV y Estimado los copia la PMO desde informes OPR por contrato. Se decidió **no usar el OPR**, así que la plataforma toma esos montos tal como llegan y los protege con reglas de consistencia entre semanas.
2. La clave de un contrato es la **Orden de Compra**, no el número de contrato (5 de 34 filas no tienen un número válido) ni la posición de la fila (cambia cada semana).
3. El **estatus se edita en ambos sentidos** (dos contratos pasaron de ACTIVO a TERMINADO y volvieron) y los KPI del Excel tratan ACTIVO como "SIN INICIAR". Los totales del portafolio se movieron por captura, no por la obra.
4. Los KPI de plazo del Excel dependen de la fecha en que se abre el archivo (`HOY()`) y marcan como "vencidos" a contratos terminados.
5. La plantilla **cambia entre semanas** (el 16-sep traía dos columnas extra que desplazaron todas las demás).
6. El archivo del miércoles reporta la semana que **cierra el jueves siguiente**.

**Alcance de la primera versión.** Cargas con validación y aprobación, historia semanal, comparativo entre cortes, resumen ejecutivo, ficha de contrato, vista por proveedor y portafolio. Quedan fuera: OPR, PPC, riesgos, causas de no cumplimiento y fianzas.

---

## 1. Comprensión del problema

### 1.1 Qué se resuelve
Se pasa de "el estado de hoy, que se sobrescribe" a "la trayectoria de cada contrato, que se acumula". Sin esa memoria la organización reporta pero no aprende.

### 1.2 Por qué Excel ya no alcanza
| Limitación | Evidencia en los archivos |
|---|---|
| No guarda historia consultable | Cada semana es un archivo aislado |
| Sin identidad estable | Número de contrato vacío, `N/A` o `-` en 5 de 34 filas; 3 filas sin OC ni proveedor |
| Fórmulas opacas y volátiles | `HOY()` altera los KPI según la fecha de apertura; vínculos externos a otro libro |
| Datos mezclados en origen | PV, EV y Estimado: fórmulas vinculadas en 20 filas y valores pegados en 11 |
| KPI con lógica equivocada | ACTIVO = "SIN INICIAR" aunque está en ejecución; el "CPI" documentado al revés |
| Sin control de cambios | Estatus que retrocede, días negativos (−177), convenios que cambian sin pasar por la hoja de ODC |
| Sin validación estructural | Columnas extra que desplazan el resto |
| Sin seguridad por filas ni auditoría | Macros, vínculos externos, archivos que circulan por correo y repositorio |

### 1.3 Beneficios esperados
1. Una sola fuente de verdad con historia completa.
2. Detección en lugar de reporte: la plataforma señala qué cambió y qué importa.
3. Menos tiempo preparando la junta semanal.
4. Conversaciones con contratistas basadas en historia, no en la foto de la semana.
5. Calidad de datos medible y con responsables.

### 1.4 Qué preguntas del negocio puede y no puede responder
| Pregunta original | ¿La responde la v1? | Con qué |
|---|---|---|
| ¿Qué contrato empeoró esta semana? | Sí | Comparador de cortes |
| ¿Qué proveedor ha venido deteriorando su desempeño? | Sí | SPI, %OENE y desviación por proveedor, con su serie histórica completa. Con 26 proveedores reales para 31 contratos, la mayoría tiene un solo contrato, por lo que cada posición muestra su nivel de confianza |
| ¿Cómo evolucionó la OENE en los últimos meses? | Sí | Serie semanal desde la carga histórica de 2026 |
| ¿Qué contratos están sistemáticamente atrasados? | Sí | SPI persistente bajo umbral y corrimientos de Fin Previsto |
| ¿Qué responsable tiene más contratos críticos? | Sí | Con asignación vigente por periodo |
| ¿Qué contratos mejoraron? | Sí | Comparador de cortes |
| ¿Qué riesgos aparecen repetidamente? | **Parcial** | Solo "alertas calculadas" (por ejemplo, SPI < 0.90 repetido). No hay matriz de riesgos ni causas: viven en el OPR |

---

## 2. Alcance

| Dentro (Fase 1 → MVP) | Fuera (decisiones D-01, D-11) |
|---|---|
| Excel Performance de GDL, una carga semanal | OPR (existen los de 2026, no se usan) |
| Contratos, proyectos, proveedores y responsables | PPC |
| Montos, plazos y KPI recalculados por la plataforma | Riesgos, actividades en riesgo, fianzas |
| Curva mensual (real y programado) con etiqueta "no conciliado" | Causas de no cumplimiento y ranking causal |
| Historia 2026 (carga retroactiva) | Conciliación contra el origen de los datos |
| Aprobación de cambios de estatus por la PMO | Hojas ODC y Facturas del Excel (no alimentan nada) |
| Diseño listo para más aeropuertos | Otros aeropuertos (hoy solo GDL) |

---

## 3. Usuarios y casos de uso

| Rol | Preguntas clave | Qué consulta |
|---|---|---|
| **Director** | ¿El programa está sano? ¿Qué requiere mi decisión? ¿Qué cambió esta semana? | Resumen ejecutivo de 2 minutos: salud, KPI clave, contratos críticos, cambios de la semana, tendencia de 12 a 26 semanas |
| **Dirección de Infraestructura** | ¿Qué proyecto está peor? ¿Dónde se concentra el riesgo financiero? ¿Qué proveedores sostienen el programa? | Portafolio por proyecto y nivel WBS, concentración por proveedor, valor contratado en contratos críticos |
| **PMO** (usuario más intensivo) | ¿Qué cambió entre cortes? ¿La carga está limpia? ¿Qué contratos son crónicos? ¿Qué estatus pido aprobar? | Cargas y calidad, comparador, tendencias, maestros, aprobaciones |
| **Responsable de proyecto** | ¿Cómo van mis contratos? ¿Qué se me desvió? | Sus contratos, su historia y sus alertas |
| **Supervisor** | ¿Cuál es el avance real contra lo programado? ¿Qué fecha de término es realista? | Detalle por contrato: PV, EV, SPI, fechas |
| **Contratos** | ¿Cuánto han crecido los contratos por convenios? ¿Cuánto hay por regularizar? ¿Qué vence pronto? | Convenios acumulados, OENE contratada y por regularizar, fechas, vigencias |

---

## 4. Arquitectura conceptual

```
┌──────────────┐  ┌────────────────┐  ┌───────────────────────┐  ┌────────────────┐  ┌──────────────┐
│ FUENTE       │  │ INGESTA        │  │ ALMACÉN HISTÓRICO     │  │ CAPA SEMÁNTICA │  │ EXPERIENCIA  │
│              │  │                │  │                       │  │ Y DE MÉTRICAS  │  │              │
│ Excel        │─▶│ Lectura segura │─▶│ 1. Crudo (inmutable)  │─▶│ KPI oficiales  │─▶│ Resumen      │
│ Performance  │  │ por encabezado │  │ 2. Conformado         │  │ Motor de       │  │ Qué cambió   │
│ semanal      │  │ Validación     │  │    (maestros+hechos)  │  │ cambios        │  │ Contrato 360 │
│ (humano)     │  │ Resolución de  │  │ 3. Derivados          │  │ Umbrales y     │  │ Proveedores  │
│              │  │ identidad      │  │    (cambios, series)  │  │ alertas        │  │ Portafolio   │
│              │  │ Aprobación PMO │  │                       │  │                │  │ Cargas/Cal.  │
└──────────────┘  └────────────────┘  └───────────────────────┘  └────────────────┘  └──────────────┘
                              Gobierno transversal: seguridad por filas · auditoría · catálogo
```

### 4.1 Responsabilidades
| Capa | Responsabilidad | Regla de oro |
|---|---|---|
| **Fuente** | El Excel Performance, con huella de plantilla | La plataforma no corrige el Excel: lo rechaza, lo observa o lo acepta con justificación |
| **Ingesta** | Leer sin ejecutar macros ni actualizar vínculos; localizar la tabla por nombre; mapear por encabezado; derivar el corte (jueves); validar; resolver identidades; pedir aprobación | Es **idempotente**: cargar dos veces el mismo archivo no cambia nada |
| **Almacén histórico** | Crudo inmutable (permite reprocesar años después), conformado (maestros y hechos limpios), derivados (cambios entre cortes y series) | Nunca se sobrescribe; solo se añade |
| **Capa semántica** | Define una sola vez cada KPI, umbral y alerta; recalcula todo desde medidas base | **La lógica de negocio no vive en las pantallas** |
| **Experiencia** | Muestra y navega; no calcula | Cualquier número en pantalla tiene su definición a un clic |

### 4.2 Volumen: el crecimiento no es el problema
GDL tiene 34 filas por semana y ≈ 130 columnas. Un año completo son ≈ 1,800 filas de contrato. **El desempeño técnico no es un riesgo en esta etapa**; los riesgos son de calidad de datos, identidad y adopción. Esto justifica no sobredimensionar la infraestructura y priorizar el modelo y las reglas.

### 4.3 Decisión de construcción **[Resuelta: D-12 = opción B, híbrido]**
| Opción | Qué implica | Valoración |
|---|---|---|
| A. Todo propio | Máximo control, máximo mantenimiento | No recomendada: el volumen no la justifica |
| **B. Híbrido** | Almacén y capa semántica propios. Módulos a medida donde está el valor (Cargas y Calidad, Qué Cambió, Contrato 360). Herramienta BI existente para exploración libre, sobre la misma capa semántica | **Recomendada** |
| C. Todo en herramienta BI | Arranque rápido | El motor de cambios, la resolución de identidad y la aprobación de estatus no caben bien y la lógica volvería a dispersarse |

**Decidido (D-12):** plataforma propia con herramienta BI complementaria (opción B). Queda por elegir **cuál** herramienta BI (D-12b); la capa semántica propia debe poder exponerse a la herramienta elegida sin duplicar cálculos.

---

## 5. Módulos

| # | Módulo | Función | Versión |
|---|---|---|---|
| 1 | **Cargas y Calidad** | Subir el Excel, ver resultado de validación, justificar advertencias, aprobar cambios de estatus, publicar el corte | MVP |
| 2 | **Resumen Ejecutivo** | Salud del programa, KPI clave, contratos críticos, "qué cambió", tendencia | MVP |
| 3 | **Qué Cambió** (comparador de cortes) | Núcleo del sistema: clasifica y explica cada cambio entre dos cortes | MVP |
| 4 | **Contrato 360** | Ficha con financiero, plazo, desempeño, historia semana a semana y eventos | MVP |
| 5 | **Portafolio** | Por proyecto y nivel WBS, con mapa de calor y semáforos | MVP |
| 6 | **Proveedores** | Cartera, tendencia y concentración. Ranking **descriptivo** (sección 9) | MVP |
| 7 | **Tendencias** | Series de tiempo libres y comparación de periodos | MVP (básico) |
| 8 | **Maestros** | Contratos, proveedores, responsables, alias, fusiones y vigencias | MVP |
| 9 | **Administración** | Usuarios, roles, seguridad por filas, umbrales, auditoría | MVP |
| 10 | **Flujo mensual** | Curva real y programada por mes, con etiqueta "no conciliado" | Posterior |
| 11 | **Responsables** | Carga y condición de contratos por persona (uso de gestión, no disciplinario) | Posterior |
| 12 | **Reportes** | Paquete semanal para la junta (PDF o diapositivas) | Posterior |
| — | Riesgos y alertas del OPR, PPC, fianzas | Dependen de datos que no se cargarán | Fuera de alcance |

---

## 6. Modelo de datos (conceptual)

### 6.1 Principios
1. **Solo se añade**: nada se sobrescribe; una recarga crea una nueva versión.
2. **Medidas base, no ratios**: SPI, Índice de Estimación y porcentajes se recalculan con precisión completa y siempre como ratio de sumas.
3. **Identidad interna estable**, independiente de lo que diga el Excel.
4. **Tres tiempos**: corte (semana PMO, jueves de cierre), entrega (fecha del archivo) y versión de carga.
5. **Procedencia por valor**: fórmula vinculada, fórmula local, valor pegado o capturado.
6. **Los valores del Excel se guardan como "reportado"**; los oficiales son los recalculados.

### 6.2 Entidades
**Maestros (cambian despacio, con vigencia):**
- **Aeropuerto** (IATA).
- **Proyecto** (código `29INFRA - GDL121`, nombre) bajo aeropuerto; 8 en GDL.
- **Clasificación**: Presupuesto (ADP/OPEX/INVERSIÓN/PMD), Dirección Gestora, Nivel I y Nivel II (WBS).
- **Contrato** (también "línea de control"): clave interna; Orden de Compra como identificador natural junto con el aeropuerto; número de contrato como atributo; alcance; proveedor; proyecto.
- **Alcance sin contratar**: línea sin OC (estatus PLANEACIÓN o filas hoy sin proveedor); necesita un **ID de línea** asignado por la PMO.
- **Equivalencia de contrato**: alias, divisiones y fusiones.
- **Proveedor**: ID interno y tabla de alias (28 escrituras para 26 proveedores reales).
- **Persona**: responsable de proyecto y responsable de obra, con asignación vigente por periodo.
- **Calendario**: semana PMO (cierre jueves), mes, trimestre, año.
- **Catálogos**: estatus (PLANEACIÓN, ACTIVO = en ejecución, TERMINADO), tipo de presupuesto, niveles WBS.

**Hechos (crecen sin parar):**
- **Carga**: archivo, huella, versión de plantilla, quién, cuándo, resultado, estado.
- **Corte**: semana PMO; estado (abierto, cerrado, reabierto); apunta a la carga vigente.
- **Foto semanal del contrato**: una fila por contrato y corte con medidas base (presupuesto base y contratado, convenios, ODC potencial, PV, EV, Estimado, Facturado, OENE contratada y por regularizar, fechas, estatus, responsables).
- **Curva mensual**: monto por contrato, mes y corte, con tipo (real, parcial, programado).
- **Aprobación de estatus**: cambio solicitado, aprobador, motivo y fecha.
- **Resultado de calidad**: regla, severidad, contrato afectado y decisión tomada.
- **Nota**: comentario de la PMO ligado a contrato y corte.

**Derivados (se recalculan y se guardan):**
- **Cambio entre cortes** por contrato: clasificación, magnitud y causa.
- **Series de tiempo** por KPI y contrato.
- **Agregados** por proyecto, proveedor y programa.

**Reservados para el futuro** (no se construyen): convenios como eventos, fianzas, riesgos, PPC, causas, serie semanal del OPR.

### 6.3 Relaciones
```
Aeropuerto 1─N Proyecto 1─N Contrato N─1 Proveedor
Contrato 1─N FotoSemanal N─1 Corte N─1 Carga(vigente)
Contrato 1─N CurvaMensual (por corte)
Contrato N─N Persona (asignación con vigencia)
Contrato 1─N Equivalencia
Corte A ─ Corte B  ⇒  Cambio por contrato
```

### 6.4 Cómo se conserva la historia
- El **crudo** es inmutable y permite reconstruir todo si cambia una regla.
- Una recarga de un corte marca la nueva versión como vigente; las anteriores siguen consultables (modo auditoría).
- Los atributos que cambian (responsable, proveedor, proyecto) se guardan con vigencia, de modo que "¿qué responsable tuvo más críticos en marzo?" no cambie por una reasignación posterior.
- Cada carga registra la versión de plantilla y de reglas con que se interpretó.

### 6.5 Cómo se evitan los duplicados
1. **Archivo duplicado**: huella; si ya existe, se informa y no se reprocesa.
2. **Mismo corte cargado dos veces**: solo una versión vigente; reemplazarla exige decisión explícita y queda registrada.
3. **Fila duplicada dentro del corte**: la pareja (aeropuerto, OC) debe ser única; si no, la carga se bloquea.
4. **Duplicado de identidad** (mismo contrato con dos nombres o números): lo resuelve el maestro con alias y revisión humana; nunca se fusiona por coincidencia parcial de nombre.

### 6.6 Cómo se comparan las semanas
- Siempre sobre la foto vigente de cada corte, por clave interna.
- Se comparan **medidas base**; los ratios se recalculan en cada corte.
- Se puede comparar cualquier par de cortes (contra la semana anterior, contra hace cuatro, contra el cierre del mes, contra el mismo corte del año pasado).
- Cada variación se **separa por causa** (sección 10).

---

## 7. Diseño funcional

### 7.1 Flujo de carga
1. **Recepción**: la PMO sube el archivo. Se registra quién, cuándo y la huella.
2. **Lectura segura**: sin ejecutar macros ni actualizar vínculos externos; la tabla se localiza por su nombre y las columnas por encabezado normalizado. Las filas plantilla, los totales de la fila 2 y el bloque de cálculos inferior se ignoran.
3. **Detección de plantilla**: se compara el conjunto de encabezados con las versiones conocidas. Columnas desconocidas van a un área "no mapeada" y generan advertencia; no bloquean.
4. **Corte**: se deriva el jueves de cierre a partir de la fecha de entrega (el miércoles reporta la semana que cierra el jueves siguiente) y el usuario lo confirma. Entrega después del jueves de cierre: "entrega tardía", confirmación manual.
5. **Validación por niveles** (detalle en el diccionario): estructural, integridad, reglas de negocio, consistencia contra el corte anterior y recálculo contra los valores reportados.
6. **Revisión**: la PMO ve hallazgos clasificados en **bloqueantes**, **advertencias** (se aceptan con justificación escrita) e **informativos**.
7. **Aprobaciones**: todo cambio de estatus se somete a aprobación. Pasar de TERMINADO a ACTIVO exige además un motivo.
8. **Resolución de identidad**: contratos, proveedores o personas nuevos o desconocidos pasan a una cola de revisión.
9. **Publicación**: se escriben los hechos conformados, se calculan los derivados y el corte se marca como cerrado. Un corte en revisión es invisible para los usuarios de negocio.
10. **Notificación**: aviso a responsables con los cambios que afectan a sus contratos.

### 7.2 Políticas
| Situación | Política propuesta |
|---|---|
| Semana sin carga | Se marca "sin dato"; no se copia ni se interpola |
| Reapertura de un corte cerrado | Requiere rol, motivo y notificación; se recalcula todo lo posterior |
| Carga retroactiva 2026 | Se carga como histórico, etiquetada "reconstruido", con mapa por versión de plantilla |
| Contrato que desaparece | No se interpreta como terminado; genera alerta de calidad |
| Entrega tardía | Marcada; la PMO decide a qué corte pertenece |
| Valor pegado vs fórmula | Se guarda la procedencia; un cambio de procedencia entre cortes genera aviso informativo |

### 7.3 Consulta y análisis
Los usuarios solo ven cortes publicados. Todo número muestra tendencia y variación contra el corte anterior, y su definición está a un clic.

---

## 8. Diseño UX/UI

### 8.1 Principios
1. Responder preguntas, no mostrar tablas.
2. Del resumen al detalle en un clic (drill down) y del detalle al contexto (drill through a Contrato 360).
3. El tiempo siempre está presente: sin números sin tendencia.
4. **El estado del dato es visible**: corte activo, calidad y etiquetas honestas ("no conciliado", "parcial", "reconstruido").
5. Pocos números bien elegidos para la Dirección; densidad para la PMO.

### 8.2 Navegación y filtros globales
```
┌─ Corte: [Sem 01-oct ▾]   Comparar con: [Sem 24-sep ▾]   Calidad: ● 2 advertencias ──┐
├─ Proyecto ▾  Nivel I ▾  Nivel II ▾  Proveedor ▾  Responsable ▾  Estatus ▾  Presupuesto ▾ ─┤
├────────────┬─────────────────────────────────────────────────────────────────────┤
│ Resumen    │                                                                     │
│ Qué cambió │        Área de contenido (filtros persistentes y cruzados)          │
│ Portafolio │                                                                     │
│ Contratos  │                                                                     │
│ Proveedores│                                                                     │
│ Tendencias │                                                                     │
│ Calidad    │                                                                     │
│ Admin      │                                                                     │
└────────────┴─────────────────────────────────────────────────────────────────────┘
```
Comportamiento tipo Power BI: filtros globales persistentes, filtrado cruzado, drill down (Programa → Proyecto → Contrato), drill through, vistas guardadas y compartibles, exportación respetando seguridad por filas.

### 8.3 Wireframes conceptuales

> Los wireframes son conceptuales. Las cifras de los bocetos A, B y D son **ilustrativas**; el boceto C usa valores reales del contrato GDLC25-027 (OPR del 01-oct-2026).

**A. Resumen Ejecutivo**
```
┌───────────────────────────────────────────────────────────────────────────┐
│ PROGRAMA GDL · Salud ● Ámbar                    Corte 01-oct vs 24-sep    │
├────────────┬────────────┬────────────┬────────────┬────────────┬──────────┤
│ SPI        │ Índice de  │ Avance     │ OENE       │ Contratos  │ Por      │
│ 0.94 ▲     │ Estimación │ ganado     │ $ 149 M ▼  │ críticos   │ vencer   │
│ ▁▂▃▃▂▃     │ 0.91 ▲     │ 55% ▲      │ ▃▃▂▂▂▁     │ 7 ▼ (−1)   │ 4        │
├────────────┴────────────┴────────────┴────────────┴────────────┴──────────┤
│ LO QUE CAMBIÓ  ▼ 6 empeoraron  ▲ 4 mejoraron  ⇄ 2 estructurales  ⚠ 3 alertas│
├─────────────────────────────────────┬─────────────────────────────────────┤
│ TOP 10 QUE REQUIEREN ATENCIÓN       │ MAPA DE CALOR  Proyecto × Estado    │
│ 1. GDLC26-016  SPI 0.77  ▼          │                                     │
│ 2. GDLC25-034  corrimiento 3ª vez   │                                     │
├─────────────────────────────────────┴─────────────────────────────────────┤
│ TENDENCIA 26 SEMANAS: SPI / OENE / contratos críticos                     │
└───────────────────────────────────────────────────────────────────────────┘
```

**B. Qué Cambió**
```
┌ Corte A: [24-sep ▾]   Corte B: [01-oct ▾]   Sensibilidad: [Estándar ▾] ─────┐
│ ＋Nuevos 0  －Salieron 0  ▼Deterioro 9  ▲Mejora 5  ⇄Estructural 3  ＝Sin cambio 14 │
├──────────────────────────────────────────────────────────────────────────────┤
│ Contrato     │ Cambio principal       │ Causa                │ Severidad │ ✓  │
│ GDLC25-053   │ Estatus TERMINADO→ACTIVO│ Reversión sin motivo │ ●●●       │ ⏳ │
│ GDLC26-012   │ Fin previsto −9 meses  │ Días en proceso −177 │ ●●●       │    │
│ GDLC25-073   │ Presupuesto −$10.8 M   │ Deductiva (convenio) │ ●         │    │
├──────────────────────────────────────────────────────────────────────────────┤
│ ▸ Seleccionar fila → panel lateral: desglose causal, gráfica A→B y notas     │
└──────────────────────────────────────────────────────────────────────────────┘
```

**C. Contrato 360**
```
┌ GDLC25-027 · OC GDL-OC-0007238 · Proyecto GDL121 · Proveedor X ● En ejecución ┐
├ Financiero ─────────────┬ Plazo ─────────────────┬ Desempeño ───────────────┤
│ Base      $1,082 M      │ AIR        04-ago-2025 │ SPI 0.94 ▼               │
│ Contratado $1,082 M     │ Fin orig.  16-nov-2027 │ Índice estimación 0.96   │
│ Convenios  −$0.23       │ Fin prev.  16-nov-2027 │ OENE $23.9 M (2.2 %)     │
│ ODC potenc. $52 M       │ Corrimientos: 0        │ Desv. −$31.8 M           │
├ HISTORIA (selector de métrica) ───────────────────────────────────────────────┤
│  [gráfica 40 semanas con marcas: convenio, cambio de estatus, cambio de resp.]│
├ Notas │ Calidad de datos de este contrato │ Procedencia de cada valor ────────┤
└───────────────────────────────────────────────────────────────────────────────┘
```

**D. Cargas y Calidad**
```
┌ Carga 01-oct · v1 · 30-sep 11:02 · Estado: EN REVISIÓN ────────────────────────┐
│ ✖ Bloqueantes 1   ⚠ Advertencias 4   ℹ Informativos 6                          │
│ ✖ GDLC25-053 pasa de TERMINADO a ACTIVO       [Aprobar con motivo][Rechazar]   │
│ ⚠ GDLC26-012 días en proceso = −177           [Justificar]                     │
│ ⚠ 15 contratos TERMINADO sin acta de entrega  [Justificar todos]               │
│ [Publicar corte]  (deshabilitado mientras haya bloqueantes)                    │
└────────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Métricas

Definiciones oficiales y umbrales: ver `diccionario_datos_metricas.md` §6.

| Nivel | Métricas |
|---|---|
| **Base (del Excel)** | Presupuesto base y contratado, convenios, ODC potencial, PV, EV, Estimado, Facturado, OENE contratada y por regularizar, estimaciones en revisión, fechas (AIR, fin original, conveniado, previsto), anticipo |
| **Derivadas (las recalcula la plataforma)** | SPI (ΣEV/ΣPV), **Índice de Estimación** (ΣE/ΣEV), % de avance programado, ganado y estimado, OENE $ y %, desviación $ y %, % de cambios por convenios, días para fin, % de tiempo transcurrido, % de incremento de plazo, corrimientos de Fin Previsto |
| **Tendencia** | Variación semanal, media móvil, semanas consecutivas de deterioro, aceleración |
| **Ejecutivas** | Salud del programa (semáforo según la regla D-13, abajo), SPI del programa, valor contratado en contratos críticos, número de contratos críticos y su cambio, OENE y su tendencia, crecimiento del presupuesto contra el base, contratos por vencer sin cierre, concentración por proveedor |

**Contrato crítico (D-05):** SPI < 0.90, o % OENE ≥ 10 %, o KPI de plazo en crítico.

**Salud del programa (D-13):** se calcula solo sobre contratos **ACTIVO** (los terminados diluyen el resultado) y toma el **peor color** de dos componentes, con los umbrales ya establecidos en el instructivo:

| Componente | Verde | Ámbar | Rojo |
|---|---|---|---|
| Desviación del programa = (ΣEV − ΣPV) / ΣPV | ≥ −5 % | entre −10 % y −5 % | ≤ −10 % |
| % OENE del programa = ΣOENE / Σtotal contratado | ≤ 5 % | > 5 % y < 10 % | ≥ 10 % |

Además se muestran, sin colorear, el número de contratos críticos y el valor contratado que representan. Aplicada a los tres cortes de septiembre: 16-sep desviación −8.1 % y OENE 5.27 % → **ámbar**; 23-sep −8.9 % y 4.65 % → **ámbar**; 30-sep −5.7 % y 3.66 % → **ámbar**, con mejora sostenida. El 23-sep tenía 17 contratos activos en lugar de 19 por el cambio de estatus de dos contratos: el cambio de composición se muestra aparte.

**Cuidados de interpretación**
- El SPI de un contrato recién iniciado es ruido; se señala cuando hay pocas semanas de avance.
- El "CPI" del Excel es **Estimado/EV** (confirmado por la PMO) y se muestra como Índice de Estimación, nunca como desempeño de costo.
- `$ Facturado` incluye el anticipo. En el contrato analizado (GDLC25-027), $743.2 M = $432.8 M de anticipo + $310.4 M de facturado neto. Restar el anticipo teórico (% anticipo × contratado) a `$ Facturado` da una aproximación al facturado neto, **validada solo en un contrato**.
- La semana más reciente de cada archivo incluye datos del jueves aún no ocurrido al entregar el miércoles; los valores de esa semana son provisionales hasta el siguiente corte.

**Se destaca en pantalla:** lo que cambió, lo crónico, lo acelerado, las fechas que se corren y el estado de calidad del dato.

---

## 10. Ranking de contratistas (conceptual)

**Decisión de la PMO (D-14): aparecen todos los contratistas y todo ranking muestra su serie histórica.** No hay mínimo de contratos o semanas para figurar.

**Realidad de los datos:** en GDL hay 31 contratos con OC y 26 proveedores reales; 22 tienen un solo contrato y solo uno tiene tres. Un proveedor con un solo contrato joven puede subir o bajar mucho por azar, así que la plataforma compensa **mostrando**, no ocultando: cada posición lleva un indicador de confianza y su tendencia histórica. Como el OPR queda fuera, tampoco hay causas de no cumplimiento para separar lo atribuible al contratista de lo externo.

**Propuesta:**
1. **V1: ranking a nivel de contrato**, por segmento comparable (Nivel I × Nivel II: construcción frente a diseño, edificación frente a campo de vuelo), con SPI, % OENE, desviación, corrimientos de fecha y crecimiento por convenios.
2. **Vista por proveedor**: **todos los contratistas, con posición**, cartera, valor contratado, número de contratos críticos y **serie histórica** de SPI, % OENE y desviación. Los de poca base llevan la etiqueta "confianza baja" (menos de 3 contratos o menos de 12 semanas de historia) y se pueden ordenar o filtrar, pero **nunca se excluyen**.
3. **Sin score único opaco**: cuatro o cinco dimensiones visibles; un compuesto opcional, ajustable y desglosable.
4. **Evitar sesgos sin esconder a nadie**: indicador de confianza en cada posición, ajuste por madurez (los contratos jóvenes pesan menos), indicadores sobre ventanas de varias semanas y no sobre una sola, suavizado estadístico para que pocos datos no den extremos, y doble vista ponderada por valor y por conteo para que un contrato gigante no domine.
5. **Tendencias**: trayectoria de las últimas N semanas y alerta por deterioro sostenido, no por una mala semana.
6. **Gobierno**: pesos y reglas son parámetros versionados y revisados en comité, no decisión de desarrollo.
7. Cuando exista el campo de causa (hoy solo en el OPR), se excluirá del puntaje únicamente la causa LA (liberación de áreas por el aeropuerto), según la decisión D-04.

---

## 11. Comparación de cortes (módulo núcleo)

### 11.1 Qué hace
Compara dos cortes cualesquiera (por defecto el último contra el anterior), **clasifica** cada contrato y **explica** cada variación, ordenando por importancia (severidad × monto) y no alfabéticamente.

### 11.2 Clasificación
| Categoría | Detección | Cuidado |
|---|---|---|
| **Nuevo** | Existe en B, no en A, y no es alias de otro | Distinguir alta real de reaparición |
| **Salió** | Existe en A, no en B | **No asumir cierre**: cerrado formalmente, renombrado, o desaparecido sin explicación (alerta de calidad) |
| **Cambio estructural** | Alias, división, fusión; cambio de proveedor, proyecto o responsable | Evita que un cambio de identidad parezca mejora o deterioro |
| **Reversión de estatus** | TERMINADO → ACTIVO, o PLANEACIÓN → TERMINADO | Va a aprobación de la PMO antes de entrar al análisis |
| **Sin cambio material** | Dentro de las tolerancias | Se resume, no se lista |
| **Mejora** | Movimiento favorable material | Distinguir sostenida de puntual |
| **Deterioro** | Movimiento desfavorable material | Severidad por magnitud, velocidad y exposición |
| **Corrección de dato** | Cambió un valor ya cerrado de un corte anterior | Se muestra aparte; nunca como mejora o deterioro |

### 11.3 Explicación de la variación
- **Δ Presupuesto total contratado** = convenios nuevos (los negativos son deductivas reales) + corrección de datos.
- **Δ SPI** = efecto de lo ejecutado (ΔEV) frente al efecto de lo programado (ΔPV).
- **Δ Fin Previsto** = Δ Fin Conveniado + Δ días en proceso. Si los días en proceso son negativos, se marca como dato inválido y no como adelanto.
- **Δ OENE** = ΔEV − ΔEstimado.
- **Δ Estatus** = estatus anterior, estatus nuevo, quién lo aprobó y por qué.

### 11.4 Riesgos emergentes (señales con los datos disponibles)
- **Cruce de umbral**: SPI < 0.90 o % OENE ≥ 10 %.
- **Deterioro sostenido**: N cortes consecutivos de empeoramiento.
- **Aceleración**: el deterioro se acelera.
- **Corrimiento en cadena**: Fin Previsto se mueve semana tras semana.
- **Reversión de estatus**.
- **Silencio**: EV y Estimado idénticos durante N semanas en un contrato ACTIVO.
- **Contagio**: varios contratos del mismo proveedor, proyecto o responsable empeoran a la vez.
- **Crecimiento contractual**: % de cambios acercándose al límite del 20 %.

### 11.5 Evidencia real en 3 cortes
| Observación | Qué debería detectar el módulo |
|---|---|
| 2 contratos ACTIVO → TERMINADO → ACTIVO en dos semanas | Reversión de estatus y bloqueo hasta aprobación |
| Días en proceso = −177 durante 2 semanas | Dato inválido; Fin Previsto saltó de jun-2027 a ago-2026 |
| Convenios autorizados negativos (−$10.8 M, −$5.6 M) | Evento de convenio, no alerta |
| 13 a 19 de 31 contratos cambian montos cada semana | Resumen por categoría para no listar 19 filas |
| Totales de portafolio que se mueven al cambiar el estatus de 2 contratos | Separar efecto de composición de efecto de desempeño |

### 11.6 Entregables del módulo
Panel de conteos, lista priorizada con causa, vista agrupada por proveedor, proyecto o responsable, detalle causal por contrato, exportable a "minuta de cambios" y registro de cada comparación estándar para auditar qué se señaló.

---

## 12. Riesgos técnicos y de adopción

| Área | Riesgo | Mitigación |
|---|---|---|
| **Calidad del Excel** | La plantilla cambia entre semanas | Lectura por encabezado, huella de plantilla, área de columnas no mapeadas |
| | Captura manual: estatus, días negativos, datos pegados | Reglas bloqueantes, aprobación de la PMO, procedencia por valor |
| | Valores calculados con `HOY()` y vínculos externos | Todo se recalcula a la fecha de corte; los vínculos no se actualizan |
| | La PMO consolida a mano desde los OPR (punto único de fallo) | Reglas de consistencia entre cortes; recuperar el OPR queda como opción futura |
| | Identidad frágil de contratos | Clave OC, alias y revisión humana; ID de línea para alcances sin contratar |
| **Mantenimiento** | Lógica de negocio dispersa | Capa semántica única y diccionario con dueño |
| | Reglas de validación frágiles | Reglas declarativas, versionadas, con propietario de negocio |
| | Dependencia de una persona | Documentación y pruebas con cortes reales |
| **Crecimiento** | Más aeropuertos o más fuentes | Modelo por aeropuerto desde el inicio; capa de ingesta con contrato por fuente |
| | Cambios de definición de KPI | Métricas versionadas; marcar series no comparables |
| | Reorganización de proyectos | Jerarquías con vigencia: "como era" y "como es hoy" |
| **Desempeño** | Bajo en esta etapa (≈1,800 filas por año) | Derivados precalculados por buena práctica, no por necesidad |
| **Seguridad** | Datos contractuales y financieros sensibles | Seguridad por filas, cifrado, mínimo privilegio, auditoría |
| | Archivos `.xlsm` con macros y vínculos externos | Lectura aislada sin ejecutar nada; descartar metadatos internos y rutas |
| | Fuga por exportación | Exportaciones controladas y bitácora |
| | Cambios al histórico | Crudo inmutable; auditoría de reaperturas y justificaciones |
| | Nombres de personas evaluadas | Política explícita de uso; vista de gestión, no disciplinaria |
| **Adopción** (el mayor) | La PMO sigue trabajando en su Excel paralelo | Operar en paralelo con criterios de salida; el reporte de la junta debe salir de la plataforma |
| | Pérdida de confianza si el primer dato es erróneo | Calidad de datos visible; no publicar cortes con bloqueantes |
| | La plantilla sigue teniendo errores de origen (KPI de ACTIVO, CPI documentado al revés) | Corregir la plantilla en paralelo (sección 14) |

---

## 13. Roadmap

| Fase | Contenido | Criterio de salida |
|---|---|---|
| **1. Arquitectura** (esta) | Documento maestro, diccionario, decisiones D-01 a D-12 y prueba de la regla del jueves y la deriva de plantilla con 3 a 4 cortes más | Documentos firmados por la PMO y la Dirección; D-12b y D-15 a D-17 resueltas |
| **2. Datos** | Modelo conformado definitivo, maestros y alias, catálogo de reglas, mapa de versiones de plantilla, **carga retroactiva de los Performance de 2026** (≈ 39 cortes, etiquetada "reconstruida") | Los cortes históricos cargados concilian con lo que la PMO reportó en su momento; cifras de control firmadas |
| **3. Backend** | Pipeline de carga con validación, aprobación e idempotencia; capa semántica; motor de cambios; seguridad y auditoría | Una carga completa de punta a punta; pruebas de regresión con cortes reales |
| **4. Frontend** | Orden: Cargas y Calidad → Resumen → Qué Cambió → Contrato 360 → Portafolio → Proveedores → Tendencias | La PMO prepara la junta semanal desde la plataforma sin abrir el Excel para analizar |
| **5. Analytics** | Ranking descriptivo de contratos, alertas de riesgo emergente, vistas por rol, notificaciones, flujo mensual | Alertas validadas contra casos históricos ya conocidos ("¿habría detectado lo que ya sabíamos?") |
| **6. Producción** | Operación en paralelo con el Excel (4 a 8 semanas), endurecimiento de seguridad, respaldo y recuperación, capacitación, soporte | Aceptación formal; el Excel deja de usarse para análisis |

**Olas futuras** (no comprometidas): incorporar el OPR (conciliación, PPC, riesgos, fianzas, causas), otros aeropuertos, conciliación con el sistema de facturación.

---

## 14. Cambios recomendados a la plantilla Performance (en paralelo, bajo control de la PMO)

1. Número de versión visible de plantilla.
2. ID de línea estable para alcances sin contratar.
3. Estatus como lista cerrada (PLANEACIÓN, ACTIVO = en ejecución, TERMINADO); retirar `EN EJECUCIÓN`.
4. Corregir las fórmulas de KPI que tratan ACTIVO como "SIN INICIAR".
5. Corregir el instructivo del "CPI" (es Estimado/EV) y renombrarlo "Índice de Estimación".
6. Quitar `HOY()` de los KPI o sustituirlo por una fecha de corte capturada.
7. Hacer obligatorio `Acta de Entrega Recepción` al terminar.
8. Catálogos desplegables para proveedor y responsable.
9. Impedir días en proceso negativos.
10. Retirar o reparar las hojas ODC y Facturas, que no alimentan nada.

---

## 15. Decisiones

### 15.1 Tomadas (resumen; detalle en el diccionario, §9)
| ID | Decisión |
|---|---|
| D-01 | Solo Performance; el OPR queda fuera de alcance |
| D-02 | Curva mensual: meses cerrados reales, futuros programados; no concilia con el OPR |
| D-03 | La semana PMO cierra en jueves; el archivo del miércoles reporta la semana que cierra el jueves siguiente |
| D-04 | La única causa externa es LA (liberación de áreas por el aeropuerto) |
| D-05 | Contrato crítico = SPI < 0.90 o % OENE ≥ 10 % o KPI de plazo crítico |
| D-06 | ACTIVO = en ejecución; "CPI" = Índice de Estimación = Estimado/EV |
| D-07 | La PMO aprueba los cambios de estatus |
| D-08 | La holgura (MS Project) la captura una persona |
| D-09 | Las deductivas negativas en convenios son legítimas |
| D-10 | Los umbrales de semáforo son los del instructivo |
| D-11 | El PPC queda fuera de alcance |
| D-12 | Plataforma propia con herramienta BI complementaria (híbrido) |
| D-13 | Semáforo de salud del programa: peor color entre desviación y % OENE de contratos ACTIVO |
| D-14 | Aparecen todos los contratistas y todo ranking muestra su serie histórica |

### 15.2 Resueltas el 02-oct-2026 y pendientes para cerrar la Fase 1

| ID | Estado | Resolución |
|---|---|---|
| D-12 | Resuelta | Plataforma propia con herramienta BI complementaria (opción B) |
| D-13 | Resuelta | Regla de salud del programa de la sección 9 |
| D-14 | Resuelta | Aparecen todos los contratistas y todo ranking muestra su serie histórica |

Pendientes:
| ID | Decisión | Quién |
|---|---|---|
| D-12b | Qué herramienta BI se usa como complemento y quién la licencia | Dirección + TI |
| D-15 | Dueño del diccionario de métricas y quién aprueba cambios de definición | Dirección |
| D-16 | Calendario de cambios a la plantilla (sección 14) | PMO |
| D-17 | Quién consume la plataforma y con qué roles y alcance por fila | Dirección + PMO |

### 15.3 Datos que mejorarían la Fase 1
- 3 a 4 Performance más de 2026, repartidos entre enero y agosto, para probar la regla del jueves, el parseo por encabezado y la deriva de plantilla en un rango más largo.
- Un Performance de un aeropuerto distinto (cuando exista) para confirmar la generalización del modelo.

---

## 16. Criterios de aceptación de la Fase 1

1. La PMO valida el diccionario y los umbrales.
2. D-12b y D-15 a D-17 tienen respuesta o fecha.
3. La regla del jueves se confirma con cortes adicionales.
4. Se acuerda el plan de corrección de la plantilla y quién lo ejecuta.
5. Se confirma la lista de usuarios y roles de la primera versión.
