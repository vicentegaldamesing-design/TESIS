# Plan: Histórico de KPIs Acumulados (Prácticas MGO)

## 1. Diagnóstico: por qué Semana 29 y Semana 30 muestran lo mismo

### Qué está pasando

En el tablero **Detalle Cumplimiento - Prácticas MGO**, las filas grises **(Acumulado)** no cambian al filtrar `Semana`, mientras las blancas **(Semanal)** sí.

Eso **no es un bug del filtro del front**. Es el comportamiento esperado cuando:

1. La expresión del KPI **no está anclada a una fecha/semana de corte**, y
2. La fuente de datos es un **estado vivo** (snapshot actual), no un **histórico semanal**.

### Semanal vs Acumulado (hoy)

| Tipo | Comportamiento actual | ¿Responde a `Semana`? |
|------|------------------------|------------------------|
| `(Semanal)` | Mide eventos/actividad de esa semana | Sí |
| `(Acumulado)` | Cuenta el estado actual del universo (todo lo aprobado hasta “hoy”) | No (o casi no) |

Ejemplo concreto: `2. % Avance plan documental (Acumulado)` → `2.2_KPI_Avance_Doc`.

En Semana 29 y Semana 30 ves `1130/2244` en Celulosa porque ambas consultas leen **el mismo estado actual** de documentos, no “cómo estaba el viernes de esa semana”.

### Causa A (la más probable en tu caso)

Tu numerador:

```qlik
Count(
  {
    <
      Estado_DOC_Avance = {'Aprobado'},
      [Avance_.OrigenDoc] = {'Plan inicial','Modificación plan'}
    >
  }
  [Avance_.id]
)
```

Esta expresión:

- Fija estado y origen (correcto para la definición del KPI).
- **No fija** `Semana`, `Fecha`, `Fecha_Aprobacion`, ni un `FechaCorte`.
- En Set Analysis, si no mencionas un campo, **hereda la selección del usuario**… **pero solo si ese campo está asociado al modelo del documento**.

Si `Semana` del tablero **no está enlazada** a la tabla `Avance_` (o la relación es incorrecta / vía un calendario semanal distinto), entonces:

- filtrar Semana 29 o 30 **no reduce** los documentos aprobados,
- el count queda siempre igual → `1130/2244`.

Conclusión: **sí, la causa A es muy probable**. No es que el KPI “ignore el filtro a propósito por ser acumulado”; es que **no hay dimensión temporal usable** (o no hay histórico) sobre la que filtrar.

---

## 2. ¿Está bien tu fórmula del Num?

### Respuesta corta

**Sí, está bien como definición de “stock actual” (estado vivo).**  
**No, no está completa si quieres histórico semana a semana.**

### Qué hace bien

```qlik
Count({
  <
    Estado_DOC_Avance = {'Aprobado'},
    [Avance_.OrigenDoc] = {'Plan inicial','Modificación plan'}
  >
} [Avance_.id])
```

- Cuenta IDs únicos de avance documental.
- Solo documentos **Aprobados**.
- Solo orígenes del plan (`Plan inicial`, `Modificación plan`).
- Eso es coherente con un **% Avance plan documental acumulado**.

### Qué le falta para tendencia / comparación semanal

Necesitas **una de estas dos estrategias** (elige una; no mezcles sin diseño):

#### Estrategia 1 — Histórico por snapshot (recomendada para “Acumulados”)

Guardar cada viernes el valor ya calculado (Num, Den, %) por planta/SI y semana.

Front solo lee:

```qlik
Sum({<Semana={$(=SemanaSeleccionada)}>} Num_Snapshot)
/
Sum({<Semana={$(=SemanaSeleccionada)}>} Den_Snapshot)
```

Ventaja: el KPI acumulado **sí cambia** al cambiar semana, porque cada semana tiene su foto.

#### Estrategia 2 — Histórico por eventos con fecha de corte

Si cada documento tiene `Fecha_Aprobacion` (o similar), el Num de “hasta el viernes de la semana S” sería:

```qlik
Count({
  <
    Estado_DOC_Avance = {'Aprobado'},
    [Avance_.OrigenDoc] = {'Plan inicial','Modificación plan'},
    Fecha_Aprobacion = {"<=$(=FechaCierreSemana)"}
  >
} [Avance_.id])
```

Esto **solo funciona** si:

- el estado histórico no se pierde (no se sobreescribe),
- o guardas historial de cambios de estado,
- y el Den también usa la misma lógica de corte.

Si hoy la BD solo tiene el estado actual (sin fechas de transición / sin snapshots), **la Estrategia 2 no reconstruye el pasado**. No hay magia en el front.

### Checklist rápido de validación del Num actual

1. ¿`[Avance_.id]` es único por documento/avance? Si hay duplicados, usa `Count(DISTINCT ...)`.
2. ¿El Den usa el mismo universo de origen (`Plan inicial`, `Modificación plan`)? Si no, el % miente.
3. ¿Hay campo de fecha de aprobación / última modificación de estado?
4. ¿`Semana` del filtro está asociada a esa fecha en el modelo de datos?
5. Si deseleccionas todas las semanas, ¿el número sigue siendo `1130/2244`? Si sí → es stock actual global.

---

## 3. ¿Excel manual cada viernes o automatizar?

### Respuesta directa

- **Empezar a capturar desde ahora: sí.**
- **Excel manual: solo como puente de 1–2 semanas**, no como solución definitiva.
- **Automatizar cada viernes: sí, es el camino correcto.**

No podrás recuperar con exactitud Semana 20…28 si esas bases no tenían histórico.  
Lo que sí puedes hacer es **congelar desde la próxima captura** (ej. viernes Semana 31 en adelante) y construir tendencia hacia adelante.

### ¿Sacar captura cada viernes?

Sí, pero no como pantallazo suelto. Necesitas una **tabla de hechos de snapshot** con al menos:

| Campo | Ejemplo | Para qué |
|-------|---------|----------|
| `FechaSnapshot` | 2026-07-18 | día de captura (viernes) |
| `Anio` | 2026 | filtro año |
| `Semana` | 29 | filtro semana |
| `FechaInicio` | 2026-07-13 | igual que hoy en el tablero |
| `FechaCierre` | 2026-07-19 | corte semanal |
| `Planta` / `PlantaSI` | Celulosa | columna del tablero |
| `KPI_ID` | `2.2_KPI_Avance_Doc` | traza |
| `Ciclo` | 2. Ejecución | layout |
| `Elemento` | 2. Gestión documental | layout |
| `Indicador` | 2. % Avance plan documental (Acumulado) | layout |
| `Num` | 1130 | numerador congelado |
| `Den` | 2244 | denominador congelado |
| `%` | 0.5036 | opcional (también se puede recalcular) |
| `Fuente` | SoftExpert / QVD / API | auditoría |
| `TsCarga` | timestamp ETL | control |

Esa tabla es tu **histórico**. Sin ella, el front no puede inventar tendencia.

---

## 4. ¿Se puede arreglar solo en el front?

### Lo que el front SÍ puede hacer (mejoras rápidas)

1. **Etiquetar honestidad del KPI**
   - En filas `(Acumulado)` mostrar subtítulo: `Estado actual (sin histórico semanal)` hasta que exista snapshot.
2. **Desacoplar el filtro Semana**
   - Para acumulados sin histórico: que `Semana` no pretenda filtrarlos (o mostrar candado / mensaje).
   - Evita la falsa sensación de que “estoy viendo Semana 29 acumulada”.
3. **Comparador de semanas (cuando exista histórico)**
   - Selector `Semana A` vs `Semana B`.
   - Delta: `%B - %A`, y opcionalmente semáforo.
4. **Mini-tendencia**
   - Sparkline de las últimas N semanas por KPI/planta leyendo la tabla snapshot.
5. **Toggle de modo**
   - `Vista operativa (hoy)` vs `Vista histórica (por semana)`.

### Lo que el front NO puede hacer solo

- Reconstruir valores pasados si la fuente no los guarda.
- Hacer que `Count` de estado vivo “recuerde” cómo estaba el viernes anterior.
- Comparar Semana 29 vs 30 de acumulados **sin** tabla histórica o fechas de evento confiables.

**Veredicto:** el front mejora UX y tendencia **después** de tener datos históricos.  
El trabajo crítico es **modelo de datos + ETL semanal**, no solo Set Analysis cosmético.

---

## 5. Arquitectura recomendada (paso a paso)

### Paso 0 — Inventario de KPIs acumulados

Lista cerrada (los de tus imágenes + los que falten):

1. `1. Cumplimiento plan Real / Planificado (Acumulado)`
2. `2. % Avance plan documental (Acumulado)` → `2.2_KPI_Avance_Doc`
3. `2. %Cierre Acciones de Mejora (Acumulado)`
4. `1. %Investigaciones cerradas (Nivel 3 o más) (Acumulado)`
5. `2. %Acciones cerradas en plazo (Nivel 3 o más) (Acumulado)`
6. `4. %Investigaciones cerradas (Nivel 1 y 2) (Acumulado)`
7. `5. %Acciones cerradas en plazo (Nivel 1 y 2) (Acumulado)`
8. `1. % de mejoras cerradas de la matriz en softexpert (Acumulado)`

Para cada uno documenta:

- `KPI_ID`
- expresión Num actual
- expresión Den actual
- fuente (tabla/QVD/API)
- ¿tiene fecha de evento? (sí/no)
- owner funcional

### Paso 1 — Decidir modo de cálculo por KPI

Para cada KPI acumulado elige:

- **SNAPSHOT_VALUE**: congelar Num/Den cada viernes (default recomendado).
- **EVENT_CUTOFF**: recalcular con `FechaEvento <= FechaCierreSemana` (solo si hay historial confiable).

Regla práctica:

> Si la fuente es “estado actual editable”, usa **SNAPSHOT_VALUE**.  
> Si es “log de eventos inmutables”, puedes usar **EVENT_CUTOFF**.

Para `2.2_KPI_Avance_Doc`, por tu fórmula y síntoma, parte con **SNAPSHOT_VALUE**.

### Paso 2 — Crear capa de snapshot (QVD / tabla)

Ejemplo de carga Qlik (conceptual):

```qlik
// 1) Calcular KPIs acumulados “de hoy” (mismas reglas de negocio)
TMP_KPI_HOY:
LOAD
  Planta,
  '2.2_KPI_Avance_Doc' as KPI_ID,
  Num,
  Den
Resident ...;

// 2) Enriquecer con calendario de la semana en curso
LEFT JOIN (TMP_KPI_HOY)
LOAD
  Week(Today()) as Semana,
  Year(Today()) as Anio,
  Date(WeekStart(Today())) as FechaInicio,
  Date(WeekStart(Today())+6) as FechaCierre,
  Date(Today()) as FechaSnapshot
Autogenerate 1;

// 3) Append al histórico
CONCATENATE (HIST_KPI_ACUMULADO)
LOAD * Resident TMP_KPI_HOY;

STORE HIST_KPI_ACUMULADO INTO [lib://Data/HIST_KPI_ACUMULADO.qvd] (qvd);
```

Importante: el job debe ser **idempotente**.

Clave natural sugerida:

`Anio + Semana + Planta + KPI_ID`

Si se re-ejecuta el viernes, **upsert** (borrar esa clave y reinsertar), no duplicar.

### Paso 3 — Automatizar “cada viernes”

Opciones (de más robusta a más frágil):

#### Opción A — Qlik Sense / Qlik Cloud Reload Task (recomendada si ya viven en Qlik)

1. App o script ETL dedicado: `ETL_MGO_Snapshot_Viernes`.
2. Task programada: **viernes 18:00** (o después del cierre operativo).
3. Condición: solo escribe snapshot si `WeekDay(Today())=Fri` **o** si el scheduler ya garantiza viernes.
4. Al terminar, recarga el dashboard de cumplimiento.
5. Alertas: si Num/Den vienen nulos o Den=0, falla el task y notifica.

#### Opción B — Power Automate / script + Excel/SharePoint (puente corto)

1. Viernes 17:30: exporta tabla del tablero (o API) a Excel/SharePoint.
2. Append a hoja `HIST_KPI_ACUMULADO`.
3. Qlik lee ese Excel como fuente histórica.

Sirve para empezar **esta semana**, pero migra a QVD/DB apenas puedas.

#### Opción C — Base de datos + job (mejor largo plazo)

1. Job SQL/Python calcula Num/Den con las mismas reglas.
2. Inserta en `fact_kpi_acumulado_semanal`.
3. Qlik solo consume esa tabla para filas grises.
4. Auditoría y backfill más fáciles.

### Paso 4 — Cambiar el front del tablero

Para filas `(Acumulado)`:

**Antes (estado vivo):**

```qlik
Count({<Estado_DOC_Avance={'Aprobado'}, [Avance_.OrigenDoc]={'Plan inicial','Modificación plan'}>} [Avance_.id])
```

**Después (histórico por semana seleccionada):**

```qlik
Sum({
  <
    KPI_ID = {'2.2_KPI_Avance_Doc'},
    TipoKPI = {'Acumulado'}
  >
} Num)
/
Sum({
  <
    KPI_ID = {'2.2_KPI_Avance_Doc'},
    TipoKPI = {'Acumulado'}
  >
} Den)
```

Notas:

- Aquí `Semana`/`Año` del filtro del usuario **sí deben aplicar** (herencia de selección).
- Si quieres forzar la semana explícitamente:

```qlik
Sum({<KPI_ID={'2.2_KPI_Avance_Doc'}, Semana=P(Semana), Anio=P(Anio)>} Num)
/
Sum({<KPI_ID={'2.2_KPI_Avance_Doc'}, Semana=P(Semana), Anio=P(Anio)>} Den)
```

Y el formato de celda sigue siendo `Num & '/' & Den` como hoy.

### Paso 5 — Comparar dos semanas (lo que pediste)

Cuando el histórico exista:

1. Crear variables:
   - `vSemanaBase`
   - `vSemanaComp`
2. En un objeto de comparación:

```qlik
// % Semana base
Sum({<Semana={$(vSemanaBase)}, KPI_ID={'2.2_KPI_Avance_Doc'}>} Num)
/
Sum({<Semana={$(vSemanaBase)}, KPI_ID={'2.2_KPI_Avance_Doc'}>} Den)

// % Semana comparación
Sum({<Semana={$(vSemanaComp)}, KPI_ID={'2.2_KPI_Avance_Doc'}>} Num)
/
Sum({<Semana={$(vSemanaComp)}, KPI_ID={'2.2_KPI_Avance_Doc'}>} Den)

// Delta pp
( ...%Comp... ) - ( ...%Base... )
```

Así sí puedes decir: “entre S29 y S30 el avance documental en Celulosa subió/bajó X puntos”.

### Paso 6 — Validación funcional (UAT)

Checklist por KPI:

1. Viernes N: correr snapshot → verificar fila insertada.
2. Cambiar filtro a esa semana → coincide con la captura.
3. Semana sin snapshot → mostrar `-` o `Sin dato histórico`, no el valor de hoy.
4. KPI semanal (blanco) sigue comportándose igual (no romper lo que ya funciona).
5. Reproceso del mismo viernes no duplica.
6. Plantas todas presentes; no “desaparece” una planta por join malo.

---

## 6. Plan de implementación sugerido (sin inventar pasado)

### Semana 0 (ahora)

1. Congelar definición Num/Den de cada acumulado (documento de reglas).
2. Crear estructura `HIST_KPI_ACUMULADO` (Excel temporal o QVD vacío).
3. Tomar **primera captura manual** del viernes en curso (baseline).
4. En el front, marcar acumulados como `Estado al corte más reciente` si no hay histórico por semana.

### Semana 1

1. Automatizar task viernes.
2. Conectar filas grises a la tabla histórica.
3. Probar Semana actual vs Semana anterior (ya deberían diferir si hubo movimiento).

### Semana 2–4

1. Agregar comparador S vs S-1.
2. Sparklines de tendencia.
3. Migrar Excel temporal → QVD/DB si partiste en Excel.
4. Monitoreo: alerta si un viernes no corre el job.

### Sobre el pasado

- **No intentes reconstruir** semanas antiguas con el estado de hoy.
- Si negocio exige backfill, solo con evidencias (exports viejos, tickets, SoftExpert logs). Si no existen, declara punto de inicio del histórico.

---

## 7. Respuestas directas a tus preguntas

### “¿Debería sacar cada viernes una captura o crear un Excel desde ahora?”

Sí: **empieza ya** con captura semanal.  
Excel sirve para arrancar; automatiza a Qlik Task/QVD/DB en paralelo.

### “¿Hay cambio en el front para mejorar la tendencia de acumulados?”

Sí, pero **después** de tener histórico:

- leer Num/Den desde snapshot por `Semana`,
- comparador de semanas,
- sparkline,
- mensaje claro cuando no hay dato histórico.

Sin histórico, el front solo puede dejar de mentir (no fingir filtro semanal).

### “¿Cómo automatizar sacar datos cada viernes y actualizar acumulados?”

Reload Task (o job DB) cada viernes que:

1. recalcula Num/Den con las mismas reglas de hoy,
2. guarda una fila por `Planta + KPI + Semana`,
3. recarga el dashboard,
4. las expresiones de filas grises leen ese histórico.

### “¿La fórmula del Num de `2.2_KPI_Avance_Doc` está bien?”

**Sí para medir el stock actual de documentos aprobados del plan.**  
**No alcanza sola para histórico semanal.**  
Para tendencia necesitas snapshot semanal (recomendado) o fecha de corte sobre historial de eventos.

---

## 8. Mini-diseño solo para `2.2_KPI_Avance_Doc`

### Definición de negocio (propuesta)

- **Num**: documentos del plan (`Plan inicial` / `Modificación plan`) en estado `Aprobado`.
- **Den**: documentos del plan en el universo acordado (todos los del plan vigente, o todos los planificados; confirmar con negocio).
- **%**: `Num / Den`.
- **Corte semanal**: valor al cierre del viernes (Fecha cierre de la semana ISO).

### Expresión operativa “hoy” (para generar snapshot)

```qlik
// Num
Count({
  <
    Estado_DOC_Avance = {'Aprobado'},
    [Avance_.OrigenDoc] = {'Plan inicial','Modificación plan'}
  >
} [Avance_.id])

// Den (ejemplo; validar con tu Den actual)
Count({
  <
    [Avance_.OrigenDoc] = {'Plan inicial','Modificación plan'}
  >
} [Avance_.id])
```

### Expresión del tablero histórico

```qlik
Num(
  Sum({<KPI_ID={'2.2_KPI_Avance_Doc'}>} Num)
  /
  Sum({<KPI_ID={'2.2_KPI_Avance_Doc'}>} Den)
, '#,##0%')
```

Y en celda tipo fracción:

```qlik
Sum({<KPI_ID={'2.2_KPI_Avance_Doc'}>} Num)
& '/' &
Sum({<KPI_ID={'2.2_KPI_Avance_Doc'}>} Den)
```

Con el filtro `Semana`/`Año` del usuario activo, Semana 29 y Semana 30 **dejarán de coincidir** cuando los snapshots difieran.

---

## 9. Riesgos y cuidados

1. **Cambiar la regla de Num/Den a mitad de año** rompe comparabilidad → versiona `KPI_Version`.
2. **Correr el job un jueves por error** ensucia la serie → valida día/fecha cierre.
3. **SoftExpert puede recalcular estados** → el snapshot debe guardarse fuera, no depender del vivo.
4. **Denominador que crece** (entran docs nuevos al plan) hace que el % baje aunque Num suba; eso es correcto, pero hay que explicarlo al usuario.
5. No mezclar en la misma expresión “acumulado vivo” + “filtro semana” sin snapshot: produce confusión exacta a la de tus imágenes.

---

## 10. Decisión recomendada (ejecutiva)

1. Declarar los KPIs grises como **serie histórica por snapshot semanal**.
2. Empezar captura **este viernes** (Excel temporal aceptable).
3. Automatizar con **task Qlik cada viernes**.
4. Cambiar el front de acumulados para leer `HIST_KPI_ACUMULADO`.
5. Mantener tu fórmula actual de Num como **motor del snapshot**, no como expresión final del tablero histórico.
6. No gastar esfuerzo intentando “arreglar solo el Set Analysis” para inventar pasado.
