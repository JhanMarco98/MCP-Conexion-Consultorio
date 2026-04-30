# Skill: General

Asistente de datos para producción general de Clínica Foianini. Responde preguntas sobre ingresos, servicios, estudios, laboratorio, medicamentos, honorarios, descuentos y más.

---

## Flujo de Trabajo

1. Interpretar la pregunta del usuario
2. Usar la query base PRODUCCION_GENERAL
3. Adaptar fechas, filtros y agrupaciones según lo que piden
4. Ejecutar con `oracle_query`
5. Responder con resultados claros

---

## REGLAS CRÍTICAS

### Estructura de Cuentas

```
GESTION / INTERNACION - ENC_ATENCION_ID
   ↓          ↓              ↓
  Año      Cuenta        Subevento
```

Ejemplo: Paciente entra por emergencia y luego se hospitaliza:
- 2026/100-1 → Emergencia Adulto (subevento 1)
- 2026/100-2 → Hospitalización (subevento 2)
- = **1 visita** con **2 subeventos**

### Métricas

| Métrica | Cálculo |
|---------|---------|
| Pacientes | `COUNT(DISTINCT A.PERSONA_NUMERO)` |
| Visitas | `COUNT(DISTINCT A.GESTION \|\| A.INTERNACION)` |
| Subeventos | `COUNT(DISTINCT A.ENC_ATENCION_ID)` |
| Cantidad (unidades) | `SUM(D.CANTIDAD)` |
| Precio unitario | `D.PRECIO_BOB` |
| **Paciente MontoBruto** | `SUM(MONTO_USUARIO_BOB)` |
| **Paciente Desc.** | `SUM(DESCUENTO_USUARIO_BOB)` |
| **Paciente MontoNeto** | `Paciente MontoBruto - Paciente Desc.` |
| **Seguro MontoBruto** | `SUM(MONTO_PLAN_BOB)` |
| **Seguro Desc.** | `SUM(DESCUENTO_PLAN_BOB)` |
| **Seguro MontoNeto** | `Seguro MontoBruto - Seguro Desc.` |
| **Total MontoBruto** | `Paciente MontoBruto + Seguro MontoBruto` |
| **Total Desc.** | `Paciente Desc. + Seguro Desc.` |
| **Total MontoNeto** | `Total MontoBruto - Total Desc.` |

**IMPORTANTE:**
- Siempre filtrar por `FECHA_DESCARGO`, NO por FECHA_ATENCION
- Usar columnas en **BOB**, no USD

### Cuándo usar cada tabla

| Pregunta sobre... | Usar campos de... |
|-------------------|-------------------|
| **Diagnósticos** | CIE10_MYSQL (COD_CIE10, CIE10_DESCRIPCION), CIAP2_MYSQL (COD_CIAP, CIAP_DESCRIPCION) |
| **Seguros/Planes** | ENC_ATENCION (NOMBRE_SEGURO, GRUPO_SEGURO), SEGURO_PLAN_MYSQL (PLAN_NOMBRE) |
| **Tipo de atención** | ENC_TIPO_ATENCION (TIPO_ATENCION, GRUPO_TIPO_ATENCION, GRUPO_TIPO_ATENCION_2, TIPO_INGRESO) |
| **Lugar de atención** | ENC_TIPO_GRUPO (NOMBRE_TIPO_GRUPO, ENC_LUGAR_ATENCION_ID) |
| **NPS/Encuestas** | ENC_ENCUESTA_USUARIO (NPS_VALORACION, NPS_CATEGORIA) |
| **Montos/Facturación** | ENC_ATENCION_DETALLE (PACIENTE_*, SEGURO_*, TOTAL_*) |

---

## QUERY BASE: PRODUCCION_GENERAL

**Usar para:** "¿Cuánto facturó la clínica?", "¿Cuántos estudios de imagen?", "¿Producción por área?", "¿Descuentos por seguro?", "¿Fee de médicos?", "¿Ingresos de laboratorio?", "¿Medicamentos por sector?"

### Query Base

```sql
WITH PRIMER_INGRESO AS (
    -- Obtiene el tipo de atención del primer subevento de cada cuenta
    SELECT
        A2.GESTION,
        A2.INTERNACION,
        T2.GRUPO AS TIPO_INGRESO_GRUPO,
        CASE
            WHEN T2.GRUPO IN ('EMERGENCIAS', 'URGENCIAS') THEN 'EMERGENCIA'
            WHEN T2.GRUPO IN ('HOSPITALIZACION') THEN 'HOSPITALIZACION'
            ELSE 'AMBULATORIO'
        END AS TIPO_INGRESO
    FROM ENC_ATENCION A2
        LEFT JOIN ENC_TIPO_ATENCION T2 ON T2.ENC_TIPO_ATENCION_ID = A2.ENC_TIPO_ATENCION_ID
    WHERE A2.ENC_ATENCION_ID = (
        SELECT MIN(A3.ENC_ATENCION_ID)
        FROM ENC_ATENCION A3
        WHERE A3.GESTION = A2.GESTION AND A3.INTERNACION = A2.INTERNACION
    )
)
SELECT
    -- Identificadores
    D.ENC_ATENCION_DETALLE_ID,
    D.ENC_ATENCION_ID,
    A.GESTION,
    A.INTERNACION,

    -- Fechas
    D.FECHA_DESCARGO,
    D.ANHO_MES_DESCARGO,

    -- Tipo de Atención (desde ENC_TIPO_ATENCION)
    A.ENC_TIPO_ATENCION_ID,
    T.NOMBRE_TIPO_ATENCION AS TIPO_ATENCION,
    T.GRUPO AS GRUPO_TIPO_ATENCION,
    CASE
        WHEN T.GRUPO IN ('EMERGENCIAS', 'URGENCIAS') THEN 'EMERGENCIA'
        WHEN T.GRUPO IN ('HOSPITALIZACION') THEN 'HOSPITALIZACION'
        ELSE 'AMBULATORIO'
    END AS GRUPO_TIPO_ATENCION_2,
    T.NOMBRE_WEB_TIPO_ATENCION,
    T.PRIORIDAD AS PRIORIDAD_TIPO_ATENCION,

    -- Tipo de Ingreso (tipo del primer subevento de la cuenta)
    PI.TIPO_INGRESO,

    -- Tipo Grupo / Lugar de Atención (desde ENC_TIPO_GRUPO)
    A.ENC_TIPO_GRUPO_ID,
    TG.NOMBRE_TIPO_GRUPO,
    TG.ENC_LUGAR_ATENCION_ID,

    -- Clase (tipo de ítem facturado)
    D.ENC_CLASE_ID,
    CASE D.ENC_CLASE_ID
        WHEN 1 THEN 'HONORARIOS MEDICOS'
        WHEN 2 THEN 'SERVICIOS CLINICA'
        WHEN 3 THEN 'SERVICIOS EXTERNOS'
        WHEN 4 THEN 'LABORATORIO'
        WHEN 5 THEN 'MEDICAMENTO'
        WHEN 6 THEN 'DESCARTABLE'
        ELSE 'OTRO'
    END AS CLASE,

    -- Grupo y Subgrupo del ítem
    D.GRUPO,
    D.SUBGRUPO,

    -- Sector/Área
    D.COD_SECTOR,
    D.NOMBRE_SECTOR,

    -- Prestación
    D.CODIGO_PRESTACION,
    D.DESCRIPCION_PRESTACION,
    D.CANTIDAD,

    -- Paciente
    A.PERSONA_NUMERO,
    A.NOMBRE_PACIENTE,
    A.EDAD,
    A.GENERO_PACIENTE,

    -- Seguro
    A.COD_SEGURO,
    A.NOMBRE_SEGURO,
    A.COD_SEGURO_PLAN,
    SP.PLAN_NOMBRE,

    -- Agrupación de Seguros
    CASE
        WHEN A.NOMBRE_SEGURO = 'Particular' THEN 'PARTICULAR'
        WHEN A.NOMBRE_SEGURO IN ('Alianza Accidentes Personales','Alianza Cia Seguro Y','Alianza Medigold','Alianza Vida') THEN 'ALIANZA'
        WHEN A.NOMBRE_SEGURO IN ('Bisa','Bisa Advance','Bisa Salud','Bisa Seguros-acc.per') THEN 'BISA'
        WHEN A.NOMBRE_SEGURO IN ('Nacional Accidentes Personales','Nacional Seguros Patrimoniales Y Fianzas S.A.','Nacional Seguros Vida Y Salud S.A.') THEN 'NACIONAL'
        WHEN A.NOMBRE_SEGURO IN ('Foianinisalud Medicinaprepaga S.A.','Vitalia Salud') THEN 'VITALIA'
        ELSE 'OTROS'
    END AS GRUPO_SEGURO,

    -- Médico/Prestador
    D.NUMERO_PRESTADOR,
    D.NOMBRE_PRESTADOR,
    D.DESCRIP_ESPECIALIDAD1 AS ESPECIALIDAD_PRESTADOR,

    -- Médico Solicitante
    D.NUMERO_MED_SOL,
    D.MEDICO_SOLICITANTE,

    -- Precio Unitario (BOB)
    D.PRECIO_BOB,

    -- Montos Paciente (BOB)
    D.MONTO_USUARIO_BOB AS PACIENTE_MONTO_BRUTO,
    D.DESCUENTO_USUARIO_BOB AS PACIENTE_DESC,
    (COALESCE(D.MONTO_USUARIO_BOB, 0) - COALESCE(D.DESCUENTO_USUARIO_BOB, 0)) AS PACIENTE_MONTO_NETO,

    -- Montos Seguro (BOB)
    D.MONTO_PLAN_BOB AS SEGURO_MONTO_BRUTO,
    D.DESCUENTO_PLAN_BOB AS SEGURO_DESC,
    (COALESCE(D.MONTO_PLAN_BOB, 0) - COALESCE(D.DESCUENTO_PLAN_BOB, 0)) AS SEGURO_MONTO_NETO,

    -- Montos Totales (BOB)
    (COALESCE(D.MONTO_USUARIO_BOB, 0) + COALESCE(D.MONTO_PLAN_BOB, 0)) AS TOTAL_MONTO_BRUTO,
    (COALESCE(D.DESCUENTO_USUARIO_BOB, 0) + COALESCE(D.DESCUENTO_PLAN_BOB, 0)) AS TOTAL_DESC,
    (COALESCE(D.MONTO_USUARIO_BOB, 0) + COALESCE(D.MONTO_PLAN_BOB, 0)
     - COALESCE(D.DESCUENTO_USUARIO_BOB, 0) - COALESCE(D.DESCUENTO_PLAN_BOB, 0)) AS TOTAL_MONTO_NETO,

    -- Cobertura
    D.PORCENTAJE_COBERTURA,

    -- Fee/Honorarios (BOB)
    D.FEE_BOB,
    D.FEE_ESTIMADO_BOB,
    COALESCE(D.FEE_BOB, D.FEE_ESTIMADO_BOB) AS FEE_FINAL_BOB,

    -- Costo (BOB)
    D.COSTO_BOB,

    -- Indicadores de cuenta
    A.CON_CIRUGIA,
    A.CON_EMERGENCIAS,
    A.ES_PAQUETE,
    A.PRIMERA_ATENCION,

    -- Diagnósticos CIE/CIAP
    A.COD_CIE10,
    C10."CIE_10Descripcion" AS CIE10_DESCRIPCION,
    A.COD_CIAP,
    CIAP."CIAP2Descripcion" AS CIAP_DESCRIPCION,
    D.COD_CIE9,
    D.DESCRIPCION_CIE9,

    -- Encuesta/NPS (desde ENC_ENCUESTA_USUARIO)
    EU.ENC_ENCUESTA_USUARIO_ID,
    EU.VALORACION AS NPS_VALORACION,
    EU.JUSTIFICACION AS NPS_JUSTIFICACION,
    EU.FECHA_RESPUESTA_USUARIO AS NPS_FECHA_RESPUESTA,
    CASE
        WHEN EU.VALORACION >= 9 THEN 'PROMOTOR'
        WHEN EU.VALORACION >= 7 THEN 'NEUTRO'
        WHEN EU.VALORACION >= 0 THEN 'DETRACTOR'
        ELSE NULL
    END AS NPS_CATEGORIA,

    -- Observación/Seguimiento NPS (última observación desde ENC_ATENCION_OBSERVACION)
    OBS.OBSERVACION AS NPS_OBSERVACION,
    OBS.USUARIO AS NPS_OBSERVACION_USUARIO,
    OBS.FECHA_CREACION AS NPS_OBSERVACION_FECHA,
    OBS.TIPO_ESTADO_ENCUESTA_ID AS NPS_ESTADO_ID,
    OBS.ACCION_CORRECTIVA AS NPS_ACCION_CORRECTIVA,
    OBS.ACCION_PREVENTIVA AS NPS_ACCION_PREVENTIVA

FROM ENC_ATENCION_DETALLE D
    INNER JOIN ENC_ATENCION A ON A.ENC_ATENCION_ID = D.ENC_ATENCION_ID
    LEFT JOIN ENC_TIPO_ATENCION T ON T.ENC_TIPO_ATENCION_ID = A.ENC_TIPO_ATENCION_ID
    LEFT JOIN ENC_TIPO_GRUPO TG ON TG.ENC_TIPO_GRUPO_ID = A.ENC_TIPO_GRUPO_ID
    LEFT JOIN ENC_ENCUESTA_USUARIO EU ON EU.ENC_ATENCION_ID = A.ENC_ATENCION_ID
    LEFT JOIN (
        SELECT ENC_ENCUESTA_USUARIO_ID, OBSERVACION, USUARIO, FECHA_CREACION,
               TIPO_ESTADO_ENCUESTA_ID, ACCION_CORRECTIVA, ACCION_PREVENTIVA
        FROM ENC_ATENCION_OBSERVACION O1
        WHERE O1.FECHA_CREACION = (
            SELECT MAX(O2.FECHA_CREACION)
            FROM ENC_ATENCION_OBSERVACION O2
            WHERE O2.ENC_ENCUESTA_USUARIO_ID = O1.ENC_ENCUESTA_USUARIO_ID
        )
    ) OBS ON OBS.ENC_ENCUESTA_USUARIO_ID = EU.ENC_ENCUESTA_USUARIO_ID
    LEFT JOIN PRIMER_INGRESO PI ON PI.GESTION = A.GESTION AND PI.INTERNACION = A.INTERNACION
    LEFT JOIN SEGURO_PLAN_MYSQL SP ON SP.COD_SEGURO = A.COD_SEGURO AND SP.COD_SEGURO_PLAN = A.COD_SEGURO_PLAN
    LEFT JOIN CIE10_MYSQL C10 ON TRIM(C10."CIE_10Codigo") = TRIM(A.COD_CIE10)
    LEFT JOIN CIAP2_MYSQL CIAP ON UPPER(TRIM(CIAP."CIAP2Codigo")) = UPPER(TRIM(A.COD_CIAP))
WHERE D.FECHA_DESCARGO >= TO_DATE('{{FECHA_INICIO}}','dd/mm/yyyy')
    AND D.FECHA_DESCARGO < TO_DATE('{{FECHA_FIN}}','dd/mm/yyyy') + 1
    -- Exclusión: Cuenta 2024/144304-1 tiene honorarios con montos erróneos (Bs 35.2M por error de carga)
    AND NOT (A.GESTION = 2024 AND A.INTERNACION = 144304 AND A.ID = 1)
```

---

## Campos para Filtrar y Agrupar

### Por Tipo de Atención (desde ENC_TIPO_ATENCION y ENC_TIPO_GRUPO)

| Campo | Fuente | Descripción |
|-------|--------|-------------|
| TIPO_ATENCION | ENC_TIPO_ATENCION | Nombre del tipo (ej: Consulta Externa, Hospitalización) |
| GRUPO_TIPO_ATENCION | ENC_TIPO_ATENCION | Agrupación original (7 grupos) |
| GRUPO_TIPO_ATENCION_2 | Calculado | Agrupación simplificada (3 grupos) |
| TIPO_INGRESO | CTE | Tipo del primer subevento (cómo ingresó) |
| NOMBRE_TIPO_GRUPO | ENC_TIPO_GRUPO | Tipo + Lugar específico (ej: Consulta Externa-CAF, Consultorio Virtual) |
| ENC_LUGAR_ATENCION_ID | ENC_TIPO_GRUPO | ID del lugar (1=CAF, 2=Virtual, 3=Beauty Plaza, etc.) |

**GRUPO_TIPO_ATENCION_2:**
| Grupo | Incluye |
|-------|---------|
| EMERGENCIA | Emergencias + Urgencias |
| HOSPITALIZACION | Hospitalización + Hosp. Ambulatoria |
| AMBULATORIO | Consulta Externa, Lab, Imagen, Vacunación, otros |

**TIPO_INGRESO:** Es el tipo del **primer subevento** (ENC_ATENCION_ID más bajo) de la cuenta. Se propaga a todos los registros de la misma cuenta.

Ejemplo:
```
Cuenta 2026/100:
  - ID 1 → Emergencia      → TIPO_INGRESO = "EMERGENCIA"
  - ID 2 → Hospitalización → TIPO_INGRESO = "EMERGENCIA"
```
Ambos tienen TIPO_INGRESO = "EMERGENCIA" porque así ingresó el paciente.

### Por Clase (tipo de ítem)

| ENC_CLASE_ID | CLASE | Descripción |
|--------------|-------|-------------|
| 1 | HONORARIOS MEDICOS | Fee de médicos |
| 2 | SERVICIOS CLINICA | Servicios propios de la clínica |
| 3 | SERVICIOS EXTERNOS | Servicios tercerizados |
| 4 | LABORATORIO | Exámenes de laboratorio |
| 5 | MEDICAMENTO | Fármacos |
| 6 | DESCARTABLE | Insumos descartables |

### Por Seguro

| Campo | Descripción |
|-------|-------------|
| COD_SEGURO | Código del seguro |
| NOMBRE_SEGURO | Nombre del seguro |
| COD_SEGURO_PLAN | Código del plan |
| PLAN_NOMBRE | Nombre del plan (desde SEGURO_PLAN_MYSQL) |
| GRUPO_SEGURO | Agrupación de seguros |

**GRUPO_SEGURO:**
| Grupo | Incluye |
|-------|---------|
| VITALIA | Foianinisalud, Vitalia Salud |
| ALIANZA | Alianza (todas las variantes) |
| BISA | Bisa (todas las variantes) |
| NACIONAL | Nacional (todas las variantes) |
| PARTICULAR | Sin seguro |
| OTROS | Otros seguros |

**Planes más comunes:** Plan 10, Vitalia Salud, Vitalia Active, Plan Nuevo USD 10, Infinity Green, Red Max 10$, Único, Familiar, etc.

### Por Sector/Área

Principales sectores: Emergencia, Quirofano, Consultorios, UCI Adultos, Urgencias, Pisos (3ro B, 4to C, 5to C, 6to C), Ecografía, Rayos X, etc.

---

## Campos Económicos (todos en BOB)

| Campo | Descripción |
|-------|-------------|
| PRECIO_BOB | Precio unitario |
| **PACIENTE_MONTO_BRUTO** | Lo que paga el paciente (bruto) |
| **PACIENTE_DESC** | Descuento al paciente |
| **PACIENTE_MONTO_NETO** | Paciente bruto - descuento |
| **SEGURO_MONTO_BRUTO** | Lo que paga el seguro (bruto) |
| **SEGURO_DESC** | Descuento al seguro |
| **SEGURO_MONTO_NETO** | Seguro bruto - descuento |
| **TOTAL_MONTO_BRUTO** | Paciente + Seguro (bruto) |
| **TOTAL_DESC** | Paciente desc. + Seguro desc. |
| **TOTAL_MONTO_NETO** | Total bruto - Total desc. |
| FEE_FINAL_BOB | Fee médico (FEE_BOB o FEE_ESTIMADO_BOB) |
| COSTO_BOB | Costo del ítem |

---

## Campos de Encuesta/NPS (desde ENC_ENCUESTA_USUARIO y ENC_ATENCION_OBSERVACION)

| Campo | Fuente | Descripción |
|-------|--------|-------------|
| NPS_VALORACION | ENC_ENCUESTA_USUARIO | Puntuación 0-10 del paciente |
| NPS_JUSTIFICACION | ENC_ENCUESTA_USUARIO | Comentario del paciente |
| NPS_FECHA_RESPUESTA | ENC_ENCUESTA_USUARIO | Fecha en que respondió |
| NPS_CATEGORIA | Calculado | PROMOTOR (9-10), NEUTRO (7-8), DETRACTOR (0-6) |
| NPS_OBSERVACION | ENC_ATENCION_OBSERVACION | Última observación/seguimiento |
| NPS_OBSERVACION_USUARIO | ENC_ATENCION_OBSERVACION | Quién registró la observación |
| NPS_OBSERVACION_FECHA | ENC_ATENCION_OBSERVACION | Fecha de la observación |
| NPS_ESTADO_ID | ENC_ATENCION_OBSERVACION | Estado de la encuesta |
| NPS_ACCION_CORRECTIVA | ENC_ATENCION_OBSERVACION | Acción correctiva tomada |
| NPS_ACCION_PREVENTIVA | ENC_ATENCION_OBSERVACION | Acción preventiva tomada |

**Nota:** Solo ~3% de las atenciones tienen respuesta NPS. Las observaciones son aún más raras (~0.15%), usadas para casos especiales (detractores, reclamos).

---

## Campos de Diagnóstico (CIE/CIAP)

| Campo | Fuente | Descripción |
|-------|--------|-------------|
| COD_CIE10 | ENC_ATENCION | Código CIE-10 del diagnóstico |
| CIE10_DESCRIPCION | CIE10_MYSQL | Descripción del diagnóstico CIE-10 |
| COD_CIAP | ENC_ATENCION | Código CIAP-2 (motivo de consulta) |
| CIAP_DESCRIPCION | CIAP2_MYSQL | Descripción del motivo CIAP-2 |
| COD_CIE9 | ENC_ATENCION_DETALLE | Código CIE-9 del procedimiento |
| DESCRIPCION_CIE9 | ENC_ATENCION_DETALLE | Descripción del procedimiento |

**Cobertura (Enero 2026):**
- ~21% de registros tienen CIE10
- ~20% de registros tienen CIAP
- Los JOINs usan TRIM y UPPER para normalizar códigos

### REGLA IMPORTANTE PARA NPS

**Siempre preguntar al usuario:** ¿Quieres el NPS filtrado por:
1. **Fecha Respuesta** (FECHA_RESPUESTA_USUARIO) — cuándo respondió la encuesta
2. **Fecha Atención** (FECHA_ATENCION de ENC_ATENCION) — cuándo fue atendido

**NO usar FECHA_DESCARGO para NPS.**

---

## Ejemplos de Uso

### Ejemplo 1: Facturación total por mes

```sql
SELECT
    ANHO_MES_DESCARGO AS MES,
    COUNT(DISTINCT PERSONA_NUMERO) AS PACIENTES,
    COUNT(DISTINCT GESTION || INTERNACION) AS VISITAS,
    COUNT(DISTINCT ENC_ATENCION_ID) AS SUBEVENTOS,
    SUM(TOTAL_MONTO_NETO) AS FACTURACION_BOB
FROM (
    -- [Query base con fechas]
) subq
GROUP BY ANHO_MES_DESCARGO
ORDER BY MES
```

### Ejemplo 2: Producción por tipo de atención

```sql
SELECT
    GRUPO_TIPO_ATENCION,
    COUNT(DISTINCT PERSONA_NUMERO) AS PACIENTES,
    COUNT(DISTINCT GESTION || INTERNACION) AS VISITAS,
    COUNT(DISTINCT ENC_ATENCION_ID) AS SUBEVENTOS,
    SUM(CANTIDAD) AS UNIDADES,
    SUM(TOTAL_MONTO_NETO) AS INGRESOS_BOB
FROM (
    -- [Query base]
) subq
GROUP BY GRUPO_TIPO_ATENCION
ORDER BY INGRESOS_BOB DESC
```

### Ejemplo 3: Ingresos por clase de servicio

```sql
SELECT
    CLASE,
    SUM(CANTIDAD) AS UNIDADES,
    SUM(TOTAL_MONTO_BRUTO) AS BRUTO_BOB,
    SUM(TOTAL_DESC) AS DESCUENTO_BOB,
    SUM(TOTAL_MONTO_NETO) AS NETO_BOB
FROM (
    -- [Query base]
) subq
GROUP BY CLASE
ORDER BY NETO_BOB DESC
```

### Ejemplo 4: Top 10 médicos por fee

```sql
SELECT
    NOMBRE_PRESTADOR,
    ESPECIALIDAD_PRESTADOR,
    COUNT(DISTINCT PERSONA_NUMERO) AS PACIENTES,
    COUNT(DISTINCT GESTION || INTERNACION) AS VISITAS,
    SUM(FEE_FINAL_BOB) AS FEE_TOTAL_BOB
FROM (
    -- [Query base]
) subq
WHERE ENC_CLASE_ID = 1  -- Solo honorarios médicos
GROUP BY NOMBRE_PRESTADOR, ESPECIALIDAD_PRESTADOR
ORDER BY FEE_TOTAL_BOB DESC
FETCH FIRST 10 ROWS ONLY
```

### Ejemplo 5: Producción de laboratorio por seguro

```sql
SELECT
    GRUPO_SEGURO,
    COUNT(DISTINCT PERSONA_NUMERO) AS PACIENTES,
    SUM(CANTIDAD) AS EXAMENES,
    SUM(TOTAL_MONTO_NETO) AS INGRESOS_BOB
FROM (
    -- [Query base]
) subq
WHERE ENC_CLASE_ID = 4  -- Solo laboratorio
GROUP BY GRUPO_SEGURO
ORDER BY INGRESOS_BOB DESC
```

### Ejemplo 6: Descuentos por seguro

```sql
SELECT
    GRUPO_SEGURO,
    SUM(TOTAL_MONTO_BRUTO) AS BRUTO_BOB,
    SUM(TOTAL_DESC) AS DESCUENTO_BOB,
    ROUND(SUM(TOTAL_DESC) * 100 / NULLIF(SUM(TOTAL_MONTO_BRUTO), 0), 1) AS PCT_DESCUENTO
FROM (
    -- [Query base]
) subq
GROUP BY GRUPO_SEGURO
ORDER BY DESCUENTO_BOB DESC
```

### Ejemplo 7: Estudios de imagen por tipo

```sql
SELECT
    TIPO_ATENCION,
    COUNT(DISTINCT GESTION || INTERNACION) AS ESTUDIOS,
    COUNT(DISTINCT PERSONA_NUMERO) AS PACIENTES,
    SUM(TOTAL_MONTO_NETO) AS INGRESOS_BOB
FROM (
    -- [Query base]
) subq
WHERE GRUPO_TIPO_ATENCION = 'IMAGENOLOGIA'
GROUP BY TIPO_ATENCION
ORDER BY ESTUDIOS DESC
```

### Ejemplo 8: Producción por sector

```sql
SELECT
    NOMBRE_SECTOR,
    COUNT(DISTINCT PERSONA_NUMERO) AS PACIENTES,
    COUNT(DISTINCT GESTION || INTERNACION) AS VISITAS,
    SUM(TOTAL_MONTO_NETO) AS INGRESOS_BOB
FROM (
    -- [Query base]
) subq
GROUP BY NOMBRE_SECTOR
ORDER BY INGRESOS_BOB DESC
FETCH FIRST 15 ROWS ONLY
```

---

## Exclusiones de Datos

| GESTION | INTERNACION | ID | Motivo |
|---------|-------------|-----|--------|
| 2024 | 144304 | 1 | Honorarios con montos erróneos (Bs 35.2M por error de carga) |

**Estas exclusiones están incluidas en la query base y se aplican siempre.**

---

## Reglas de Operación

1. Siempre filtrar por `FECHA_DESCARGO` (NO FECHA_ATENCION)
2. Reemplazar `{{FECHA_INICIO}}` y `{{FECHA_FIN}}` con fechas en formato `dd/mm/yyyy`
3. **Pacientes:** `COUNT(DISTINCT PERSONA_NUMERO)`
4. **Visitas:** `COUNT(DISTINCT GESTION || INTERNACION)` — cuentas únicas
5. **Subeventos:** `COUNT(DISTINCT ENC_ATENCION_ID)` — atenciones dentro de cuentas
6. **Cantidad/Unidades:** `SUM(CANTIDAD)`
7. Si piden datos de muchas filas, agregar agregaciones
8. Indicar siempre el período consultado en la respuesta
9. Si hay más de 20 filas, mostrar top relevante y mencionar que hay más
10. Usar `NULLIF(..., 0)` para evitar división por cero en porcentajes

---

## Catálogos de Referencia

### ENC_TIPO_ATENCION (vista - tipos de atención)

| ID | Nombre | Grupo | Prioridad |
|----|--------|-------|-----------|
| 5 | Hospitalización | HOSPITALIZACION | 10 |
| 6 | Hospitalización Ambulatoria | HOSPITALIZACION | 20 |
| 21 | Emergencias - Adultos | EMERGENCIAS | 30 |
| 22 | Emergencias - Pediátrico | EMERGENCIAS | 30 |
| 23 | Urgencias - Adultos | URGENCIAS | 35 |
| 24 | Urgencias - Pediátrico | URGENCIAS | 35 |
| 9 | Resonancia Magnética | IMAGENOLOGIA | 40 |
| 46 | Neurofisiología | IMAGENOLOGIA | 45 |
| 10 | Tomografía | IMAGENOLOGIA | 50 |
| 7 | Ecografía | IMAGENOLOGIA | 60 |
| 45 | MAPA | IMAGENOLOGIA | 61 |
| 44 | Holter 24 Horas | IMAGENOLOGIA | 62 |
| 43 | Ergometría (Test de Esfuerzo) | IMAGENOLOGIA | 63 |
| 42 | Electrocardiograma | IMAGENOLOGIA | 64 |
| 41 | Ecocardiograma | IMAGENOLOGIA | 65 |
| 47 | Colposcopía/Papanicoláu | IMAGENOLOGIA | 66 |
| 8 | Rayos-X | IMAGENOLOGIA | 70 |
| 11 | Laboratorios | LABORATORIOS | 80 |
| 2 | Consulta Externa | CONSULTA EXTERNA | 90 |
| 14 | Vacunación | VACUNACION | 130 |

### ENC_CLASE (tipos de ítem)

| ID | Nombre |
|----|--------|
| 1 | HONORARIOS MEDICOS |
| 2 | SERVICIOS CLINICA |
| 3 | SERVICIOS EXTERNOS |
| 4 | LABORATORIO |
| 5 | MEDICAMENTO |
| 6 | DESCARTABLE |
