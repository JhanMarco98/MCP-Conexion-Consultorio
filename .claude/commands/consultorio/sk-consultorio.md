# Skill: Consultorio

Asistente de datos para el área de Consultorios de Clínica Foianini. Responde preguntas ejecutando queries contra el DWH Oracle.

---

## Flujo de Trabajo

1. Interpretar la pregunta del usuario
2. Elegir la query base correcta (TURNOS o PRODUCCION)
3. Adaptar fechas y filtros según lo que piden
4. Ejecutar con `oracle_query`
5. Responder con resultados claros

---

## Queries Disponibles

| Pregunta sobre... | Usar query |
|-------------------|------------|
| Turnos, ausencias, ocupación, demanda, citas, NPS | TURNOS |
| Ingresos, fee, producción, servicios por médico | PRODUCCION |

---

## QUERY 1: TURNOS

**Usar para:** "¿Cuántos turnos?", "¿Tasa de ausencias?", "¿Demanda por especialidad?", "¿Vitalia vs Particular?", "¿Cuál es el NPS?"

### REGLA CRITICA

**SIEMPRE contar turnos como `COUNT(DISTINCT TURNO_NUMERO)`** - Existen ~97 duplicados en el DWH.

### Query Base

```sql
SELECT
    T1.*,
    -- Agrupación de Estados
    CASE
        WHEN T1.ESTADO = 'PRESENTE' THEN 'PRESENTE'
        WHEN T1.ESTADO = 'AUSENTE' THEN 'AUSENTE'
        WHEN T1.ESTADO = 'NO VENDIDO' THEN 'NO VENDIDO'
        ELSE 'BLOQUEADO'
    END AS GRUPO_ESTADO,

    -- Agrupación de Especialidades (incluye versiones con y sin tildes)
    CASE
        WHEN T1.ESPECIALIDAD = 'PAPANICOLAU  COLPOSCOPIA' THEN 'GINECOLOGIA Y OBSTETRICIA'
        WHEN T1.ESPECIALIDAD = 'PROCEDIMIENTOS DERMATOLOGICOS' THEN 'DERMATOLOGIA'
        WHEN T1.ESPECIALIDAD IN ('CIRUGÍA DE TIROIDES','CIRUGIA DE TIROIDES','PROCTOLOGIA','CIRUGÍA DE HERNIA','CIRUGIA DE HERNIA','CIRUGÍA DE REFLUJO ESOFÁGICO','CIRUGIA DE REFLUJO ESOFAGICO','CIRUGÍA BARIATRICA / METABÓLICA','CIRUGIA BARIATRICA / METABOLICA') THEN 'CIRUGIA GENERAL ADULTO'
        WHEN T1.ESPECIALIDAD IN ('TRAUMATOLOGÍA ADULTO(ARTROSCOPÍA DE CADERA/RODILLA/HOMBRO Y CODO)',
                                'TRAUMATOLOGIA ADULTO(ARTROSCOPIA DE CADERA/RODILLA/HOMBRO Y CODO)',
                                'TRAUMATOLOGÍA ADULTO(COLUMNA VERTEBRAL)',
                                'TRAUMATOLOGIA ADULTO(COLUMNA VERTEBRAL)',
                                'TRAUMATOLOGÍA ADULTO(RODILLA/HOMBRO Y CODO)',
                                'TRAUMATOLOGIA ADULTO(RODILLA/HOMBRO Y CODO)',
                                'TRAUMATOLOGIA ADULTOS (PIE Y TOBILLO)',
                                'TRAUMATOLOGÍA ADULTO(CADERA /RODILLA)',
                                'TRAUMATOLOGIA ADULTO(CADERA /RODILLA)',
                                'TRAUMATOLOGÍA ADULTO(ORTOPEDIA ONCOLÓGICA Y TRAUMATOLOGÍA GENERAL)',
                                'TRAUMATOLOGIA ADULTO(ORTOPEDIA ONCOLOGICA Y TRAUMATOLOGIA GENERAL)',
                                'TRAUMATOLOGÍA ADULTO(MANO, MUÑECA Y CODO)',
                                'TRAUMATOLOGIA ADULTO(MANO, MUNECA Y CODO)',
                                'CAMPAÑA PRÓTESIS (CADERA/RODILLA)',
                                'CAMPANA PROTESIS (CADERA/RODILLA)',
                                'TRAUMATOLOGIA ADULTOS',
                                'TRAUMATOLOGIA ADULTO/PEDIATRICA (MANO, MUÑECA, MICROCIRUGÍA, NERVIOS PERIFÉRICOS Y PLEXO BRAQUIAL)',
                                'TRAUMATOLOGIA ADULTO/PEDIATRICA (MANO, MUNECA, MICROCIRUGIA, NERVIOS PERIFERICOS Y PLEXO BRAQUIAL)',
                                'TRAUMATOLOGÍA ADULTO(MANO, MUÑECA Y ANTEBRAZO)',
                                'TRAUMATOLOGIA ADULTO(MANO, MUNECA Y ANTEBRAZO)',
                                'TRAUMATOLOGIA ADULTO (PIE Y TOBILLO)',
                                'TRAUMATOLOGIA ADULTO(MANO, MUÑECA, MICROCIRUGÍA, NERVIOS PERIFÉRICOS Y PLEXO BRAQUIAL)',
                                'TRAUMATOLOGIA ADULTO(MANO, MUNECA, MICROCIRUGIA, NERVIOS PERIFERICOS Y PLEXO BRAQUIAL)')
                                THEN 'TRAUMATOLOGIA ADULTO'
        WHEN T1.ESPECIALIDAD = 'PROCEDIMIENTOS GASTROENTEROLOGIA' THEN 'GASTROENTEROLOGIA ADULTO'
        WHEN T1.ESPECIALIDAD IN ('NEUROCIRUGÍA, NEURORRADIOLOGÍA Y RADIOLOGÍA INTERVENCIONISTA','NEUROCIRUGIA, NEURORRADIOLOGIA Y RADIOLOGIA INTERVENCIONISTA') THEN 'NEUROCIRUGIA'
        WHEN T1.ESPECIALIDAD = 'CARDIOLOGIA ADULTO - ARRITMIAS Y MARCAPASOS' THEN 'CARDIOLOGIA ADULTO'
        WHEN T1.ESPECIALIDAD = 'NEUMOLOGIA ADULTO / ESPIROMETRIA' THEN 'NEUMOLOGIA ADULTO'
        WHEN T1.ESPECIALIDAD = 'HEMATOLOGIA' THEN 'HEMATOLOGIA ADULTO'
        WHEN T1.ESPECIALIDAD = 'HEMATOLOGIA Y ONCOLOGIA PEDIATRICA' THEN 'HEMATOLOGIA PEDIATRICA'
        WHEN T1.ESPECIALIDAD = 'INFECTOLOGIA' THEN 'INFECTOLOGIA ADULTO'
        WHEN T1.ESPECIALIDAD = 'REUMATOLOGIA' THEN 'REUMATOLOGIA ADULTO'
        ELSE T1.ESPECIALIDAD
    END AS GRUPO_ESPECIALIDAD,

    -- Agrupación de Seguros
    CASE
        WHEN T1.SEGURO_NOMBRE = 'Particular' THEN 'PARTICULAR'
        WHEN T1.SEGURO_NOMBRE IN ('Alianza Accidentes Personales','Alianza Cia Seguro Y','Alianza Medigold','Alianza Vida') THEN 'ALIANZA'
        WHEN T1.SEGURO_NOMBRE IN ('Bisa','Bisa Advance','Bisa Salud','Bisa Seguros-acc.per') THEN 'BISA'
        WHEN T1.SEGURO_NOMBRE IN ('Nacional Accidentes Personales','Nacional Seguros Patrimoniales Y Fianzas S.A.','Nacional Seguros Vida Y Salud S.A.') THEN 'NACIONAL'
        WHEN T1.SEGURO_NOMBRE IN ('Foianinisalud Medicinaprepaga S.A.','Vitalia Salud') THEN 'VITALIA'
        ELSE 'OTROS'
    END AS GRUPO_SEGURO,

    -- NPS: Valoración (0-10)
    EEU.VALORACION AS NPS_VALORACION,

    -- NPS: Categoría
    CASE
        WHEN EEU.VALORACION >= 9 THEN 'PROMOTOR'
        WHEN EEU.VALORACION >= 7 THEN 'NEUTRO'
        WHEN EEU.VALORACION <= 6 THEN 'DETRACTOR'
        ELSE NULL
    END AS NPS_CATEGORIA,

    -- NPS: Comentario del paciente
    EEU.JUSTIFICACION AS NPS_COMENTARIO

FROM TURNOS_TODOS_MYSQL T1
    LEFT JOIN ENC_ATENCION EA
        ON T1.CUENTA_GESTION = EA.GESTION
        AND T1.CUENTA_INTERNACION = EA.INTERNACION
        AND T1.CUENTA_ID = EA.ID
    LEFT JOIN ENC_ENCUESTA_USUARIO EEU
        ON EA.ENC_ATENCION_ID = EEU.ENC_ATENCION_ID
WHERE T1.TURNO_FECHA >= TO_DATE('{{FECHA_INICIO}}','dd/mm/yyyy')
    AND T1.TURNO_FECHA <= TO_DATE('{{FECHA_FIN}}','dd/mm/yyyy')
    AND T1.ESTADO NOT IN ('ERROR', 'NO TOMAR EN CUENTA (SE DIO DA BAJA AGENDA)')
    AND (T1.SEGURO_NOMBRE NOT IN ('CAF Control Medico Al Personal', 'Accidentes Laborales Caf') OR T1.SEGURO_NOMBRE IS NULL)
    AND T1.ESPECIALIDAD NOT IN (
        'RESONANCIA MAGNETICA',
        'TOMOGRAFIA',
        'ECOCARDIOGRAMA/ECOGRAFÍA DOPPLER DE VASOS PERIFÉRICOS Y CUELLO',
        'ECOCARDIOGRAMA/ECOGRAFIA DOPPLER DE VASOS PERIFERICOS Y CUELLO',
        'ECOGRAFIA',
        'MEDICINA NUCLEAR (GAMMAGRAFIA)',
        'LABORATORIOS',
        'RECEPCION DE LABORATORIO',
        'TOMA DE MUESTRAS',
        'CARDIOLOGIA ECOCARDIOGRAFIA',
        'ECOGRAFIA CARDIOLOGICA',
        'ECOGRAFIA GENERAL',
        'ECOGRAFIA GINECOLOGICA   OBSTETRICA',
        'ELECTROCARDIOGRAMA',
        'HOLTER',
        'MAPA',
        'POLISOMNOGRAFIA',
        'PROCEDIMIENTOS NEUROFISIOLOGICOS',
        'PROCEDIMIENTOS CARDIOLOGICOS',
        'ERGOMETRIA',
        'RAYOS X',
        'EMERGENCIAS INTERNACION',
        'URGENCIAS - TRIAJE',
        'NUTRICIÓN - TRABAJADORES CAF',
        'NUTRICION - TRABAJADORES CAF',
        'MAMOGRAFIA',
        'VACUNAS',
        'ADM. VITALIA',
        'ANESTESIOLOGIA RECUPERACION',
        'DENSITOMETRIA',
        'FISIOTERAPIA'
    )
```

### Campos para Agrupar

| Campo | Valores |
|-------|---------|
| GRUPO_ESTADO | PRESENTE, AUSENTE, NO VENDIDO, BLOQUEADO |
| GRUPO_ESPECIALIDAD | Especialidades consolidadas |
| GRUPO_SEGURO | VITALIA, ALIANZA, BISA, NACIONAL, PARTICULAR, OTROS |
| NPS_CATEGORIA | PROMOTOR, NEUTRO, DETRACTOR (NULL si no respondió) |

### Notas sobre los Datos

1. **Duplicados:** Existen ~97 TURNO_NUMERO duplicados (filas idénticas). Por eso SIEMPRE usar `COUNT(DISTINCT TURNO_NUMERO)`

2. **Seguros NULL:** Los turnos NO VENDIDO y BLOQUEADO no tienen paciente asignado, por lo tanto SEGURO_NOMBRE = NULL. El filtro incluye `OR T1.SEGURO_NOMBRE IS NULL` para mantenerlos.

3. **Especialidades NULL:** Se excluyen (~772 turnos). Son slots vacíos o procedimientos de imagen sin especialidad asignada.

4. **NPS:** Solo ~1.3% de turnos tienen respuesta de encuesta NPS. Los LEFT JOINs no duplican filas.

---

## NPS (Net Promoter Score)

### ¿Qué es?

La encuesta pregunta al paciente: **"¿Qué tan probable es que recomiende la clínica?"** (escala 0-10)

### Categorías

| Valoración | Categoría | Significado |
|------------|-----------|-------------|
| 9-10 | PROMOTOR | Muy satisfecho, recomienda activamente |
| 7-8 | NEUTRO | Satisfecho pero no entusiasta |
| 0-6 | DETRACTOR | Insatisfecho, puede hablar mal |

### Fórmula del NPS Score

```
NPS Score = ((PROMOTORES - DETRACTORES) / TOTAL_RESPUESTAS) * 100
```

**Rango:** -100 a +100

### Interpretación

| NPS Score | Interpretación |
|-----------|----------------|
| > 50 | Excelente |
| 30 - 50 | Bueno |
| 0 - 30 | Necesita mejora |
| < 0 | Crítico |

### Vinculación de Tablas

```
TURNOS_TODOS_MYSQL
    └── ENC_ATENCION (por GESTION + INTERNACION + ID)
            └── ENC_ENCUESTA_USUARIO (por ENC_ATENCION_ID)
```

### Ejemplo: Calcular NPS por Mes

```sql
SELECT
    TO_CHAR(TURNO_FECHA, 'YYYY-MM') AS MES,
    COUNT(DISTINCT TURNO_NUMERO) AS TURNOS,
    COUNT(NPS_VALORACION) AS RESPUESTAS_NPS,
    SUM(CASE WHEN NPS_CATEGORIA = 'PROMOTOR' THEN 1 ELSE 0 END) AS PROMOTORES,
    SUM(CASE WHEN NPS_CATEGORIA = 'DETRACTOR' THEN 1 ELSE 0 END) AS DETRACTORES,
    ROUND(
        (SUM(CASE WHEN NPS_CATEGORIA = 'PROMOTOR' THEN 1 ELSE 0 END) -
         SUM(CASE WHEN NPS_CATEGORIA = 'DETRACTOR' THEN 1 ELSE 0 END))
        * 100.0 / NULLIF(COUNT(NPS_VALORACION), 0)
    , 1) AS NPS_SCORE
FROM (
    -- [Query base TURNOS]
) subq
WHERE NPS_VALORACION IS NOT NULL
GROUP BY TO_CHAR(TURNO_FECHA, 'YYYY-MM')
ORDER BY MES
```

---

## QUERY 2: PRODUCCION

**Usar para:** "¿Cuánto produjo X?", "Fee por especialidad", "Servicios por médico", "Ingresos consultorio"

### Query Base

```sql
WITH MEDICOS_CON_PRESENCIA AS (
    SELECT DISTINCT NUMERO_PRESTADOR, TRUNC(TURNO_FECHA, 'MM') AS MES
    FROM TURNOS_MYSQL
    WHERE TURNO_FECHA >= TO_DATE('{{FECHA_INICIO}}','dd/mm/yyyy')
        AND TURNO_FECHA <= TO_DATE('{{FECHA_FIN}}','dd/mm/yyyy')
        AND ESTADO = 'PRESENTE'
        AND (SEGURO_NOMBRE NOT IN ('CAF Control Medico Al Personal', 'Accidentes Laborales Caf') OR SEGURO_NOMBRE IS NULL)
        AND ESPECIALIDAD NOT IN (
            'RESONANCIA MAGNETICA','TOMOGRAFIA',
            'ECOCARDIOGRAMA/ECOGRAFÍA DOPPLER DE VASOS PERIFÉRICOS Y CUELLO',
            'ECOCARDIOGRAMA/ECOGRAFIA DOPPLER DE VASOS PERIFERICOS Y CUELLO',
            'ECOGRAFIA','MEDICINA NUCLEAR (GAMMAGRAFIA)',
            'LABORATORIOS','RECEPCION DE LABORATORIO','TOMA DE MUESTRAS',
            'CARDIOLOGIA ECOCARDIOGRAFIA','ECOGRAFIA CARDIOLOGICA','ECOGRAFIA GENERAL',
            'ECOGRAFIA GINECOLOGICA   OBSTETRICA','ELECTROCARDIOGRAMA','HOLTER','MAPA',
            'POLISOMNOGRAFIA','PROCEDIMIENTOS NEUROFISIOLOGICOS','PROCEDIMIENTOS CARDIOLOGICOS',
            'ERGOMETRIA','RAYOS X','EMERGENCIAS INTERNACION','URGENCIAS - TRIAJE',
            'NUTRICIÓN - TRABAJADORES CAF','NUTRICION - TRABAJADORES CAF',
            'MAMOGRAFIA','VACUNAS','ADM. VITALIA','ANESTESIOLOGIA RECUPERACION',
            'DENSITOMETRIA','FISIOTERAPIA'
        )
)
SELECT
    T2.ENC_ATENCION_ID AS ID_ATENCION,
    T1.ENC_ATENCION_DETALLE_ID AS ID_ATENCION_DETALLE,
    T1.FECHA_DESCARGO,
    T1.COD_SECTOR,
    T1.NOMBRE_SECTOR,
    CASE WHEN T1.ENC_CLASE_ID = 1 THEN T1.NUMERO_PRESTADOR ELSE T1.NUMERO_MED_SOL END AS COD_MEDICO,
    CASE WHEN T1.ENC_CLASE_ID = 1 THEN T1.NOMBRE_PRESTADOR ELSE T1.MEDICO_SOLICITANTE END AS NOMBRE_MEDICO,
    T4.CODIGO_ESPECIALIDAD AS COD_ESPECIALIDAD,
    T4.NOMBRE_ESPECIALIDAD,
    T2.PERSONA_NUMERO AS COD_PACIENTE,
    T2.NOMBRE_PACIENTE,
    T1.CODIGO_PRESTACION AS COD_PRESTACION,
    T1.DESCRIPCION_PRESTACION AS NOMBRE_PRESTACION,
    T2.COD_SEGURO,
    T2.NOMBRE_SEGURO,
    CASE WHEN T1.ENC_CLASE_ID = 1 THEN COALESCE(T1.FEE_USD, T1.FEE_ESTIMADO_USD) ELSE T1.MONTO_TOTAL_NETO_USD END AS MONTO_TOTAL_NETO_USD,
    CASE WHEN T1.ENC_CLASE_ID = 1 THEN 'Fee' ELSE 'Servicios Clínica' END AS VALIDADOR,
    CASE
        WHEN T1.ENC_CLASE_ID = 1 THEN 'Fee'
        WHEN T1.CLASE = 'LABORATORIO' THEN 'Laboratorio'
        WHEN T1.CODIGO_PRESTACION IN (3235092, 3235129, 3235130, 3235131) THEN 'Cardiologia'
        WHEN T1.COD_SECTOR = 80 THEN 'Ginecología'
        ELSE 'Imagenología'
    END AS TIPO_SERVICIO_GRUPO,
    CASE
        WHEN T1.ENC_CLASE_ID = 1 THEN 'Fee'
        WHEN T1.CLASE = 'LABORATORIO' THEN 'Laboratorio'
        WHEN T1.CODIGO_PRESTACION = 3235092 THEN 'Electrocardiograma'
        WHEN T1.CODIGO_PRESTACION = 3235129 THEN 'Ergometría'
        WHEN T1.CODIGO_PRESTACION = 3235130 THEN 'Holter'
        WHEN T1.CODIGO_PRESTACION = 3235131 THEN 'Mapa'
        WHEN T1.CODIGO_PRESTACION = 3235188 THEN 'Colposcopía'
        WHEN T1.CODIGO_PRESTACION = 3221204 THEN 'Papanicolau'
        WHEN T1.CODIGO_PRESTACION = 3235192 THEN 'Densitometria'
        WHEN T1.COD_SECTOR = 124 THEN 'Mamografía'
        WHEN T1.GRUPO = 'ECOCARDIOGRAMA' THEN 'Ecocardiograma'
        WHEN T1.GRUPO = 'ECOGRAFIA' THEN 'Ecografía'
        WHEN T1.GRUPO = 'RADIOGRAFIA' THEN 'Radiografía'
        WHEN T1.GRUPO = 'RESONANCIA MAGNETICA' THEN 'Resonancia'
        WHEN T1.GRUPO = 'TOMOGRAFIA COMPUTARIZADA' THEN 'Tomografía'
        ELSE 'Omitir'
    END AS TIPO_SERVICIO_DETALLE
FROM ENC_ATENCION_DETALLE T1
    INNER JOIN ENC_ATENCION T2 ON T2.ENC_ATENCION_ID = T1.ENC_ATENCION_ID
    INNER JOIN MEDICOS_CON_PRESENCIA MCP ON MCP.NUMERO_PRESTADOR = CASE WHEN T1.ENC_CLASE_ID = 1 THEN T1.NUMERO_PRESTADOR ELSE T1.NUMERO_MED_SOL END AND MCP.MES = TRUNC(T1.FECHA_DESCARGO, 'MM')
    LEFT JOIN ESPECIALIDADES_MYSQL T4 ON T4.CODIGO_PROFESIONAL = CASE WHEN T1.ENC_CLASE_ID = 1 THEN T1.NUMERO_PRESTADOR ELSE T1.NUMERO_MED_SOL END
WHERE T1.FECHA_DESCARGO >= TO_DATE('{{FECHA_INICIO}}','dd/mm/yyyy')
    AND T1.FECHA_DESCARGO <= TO_DATE('{{FECHA_FIN}}','dd/mm/yyyy')
    AND ((T1.ENC_CLASE_ID IN (2, 4) AND T2.ENC_TIPO_ATENCION_ID NOT IN (5, 6, 21, 22, 23, 24) AND (T1.GRUPO IS NULL OR T1.GRUPO NOT IN ('ANESTESIA/SEDACION EN IMAGENOLOGIA', 'USO DE EQUIPOS')))
        OR (T1.ENC_CLASE_ID = 1 AND T2.ENC_TIPO_ATENCION_ID = 2 AND T1.GRUPO = 'AMBULATORIO'))
```

### Campos para Agrupar

| Campo | Uso |
|-------|-----|
| NOMBRE_MEDICO | Agrupar por médico |
| NOMBRE_ESPECIALIDAD | Agrupar por especialidad |
| VALIDADOR | Fee vs Servicios Clínica |
| TIPO_SERVICIO_GRUPO | Fee, Laboratorio, Cardiología, Ginecología, Imagenología |
| MONTO_TOTAL_NETO_USD | Sumar para ingresos |

### Notas sobre los Datos (PRODUCCION)

1. **Valores de GRUPO:** En el DWH los valores de T1.GRUPO están **sin tildes** (ECOGRAFIA, RADIOGRAFIA, TOMOGRAFIA COMPUTARIZADA, etc.). Los filtros de esta query ya están correctos.

2. **Tildes en salida:** Los valores con tildes (Ginecología, Ergometría, Ecografía, etc.) son solo etiquetas de salida, no afectan los filtros.

---

## Ejemplos de Uso

### Ejemplo 1: Total turnos por estado (Enero 2026)

```sql
SELECT
    GRUPO_ESTADO,
    COUNT(DISTINCT TURNO_NUMERO) AS TURNOS
FROM (
    -- [Query base TURNOS con fechas 01/01/2026 - 31/01/2026]
) subq
GROUP BY GRUPO_ESTADO
ORDER BY TURNOS DESC
```

### Ejemplo 2: Tasa de ausencias por especialidad

```sql
SELECT
    GRUPO_ESPECIALIDAD,
    COUNT(DISTINCT TURNO_NUMERO) AS TOTAL_TURNOS,
    COUNT(DISTINCT CASE WHEN GRUPO_ESTADO = 'AUSENTE' THEN TURNO_NUMERO END) AS AUSENTES,
    ROUND(COUNT(DISTINCT CASE WHEN GRUPO_ESTADO = 'AUSENTE' THEN TURNO_NUMERO END) * 100.0 / COUNT(DISTINCT TURNO_NUMERO), 1) AS TASA_AUSENCIA
FROM (
    -- [Query base TURNOS]
) subq
GROUP BY GRUPO_ESPECIALIDAD
ORDER BY TASA_AUSENCIA DESC
```

### Ejemplo 3: Top 10 médicos por producción

```sql
SELECT
    NOMBRE_MEDICO,
    NOMBRE_ESPECIALIDAD,
    SUM(MONTO_TOTAL_NETO_USD) AS PRODUCCION_USD,
    COUNT(DISTINCT ID_ATENCION) AS ATENCIONES
FROM (
    -- [Query base PRODUCCION]
) subq
GROUP BY NOMBRE_MEDICO, NOMBRE_ESPECIALIDAD
ORDER BY PRODUCCION_USD DESC
FETCH FIRST 10 ROWS ONLY
```

### Ejemplo 4: NPS por especialidad

```sql
SELECT
    GRUPO_ESPECIALIDAD,
    COUNT(NPS_VALORACION) AS RESPUESTAS,
    ROUND(
        (SUM(CASE WHEN NPS_CATEGORIA = 'PROMOTOR' THEN 1 ELSE 0 END) -
         SUM(CASE WHEN NPS_CATEGORIA = 'DETRACTOR' THEN 1 ELSE 0 END))
        * 100.0 / NULLIF(COUNT(NPS_VALORACION), 0)
    , 1) AS NPS_SCORE
FROM (
    -- [Query base TURNOS]
) subq
WHERE NPS_VALORACION IS NOT NULL
GROUP BY GRUPO_ESPECIALIDAD
HAVING COUNT(NPS_VALORACION) >= 10
ORDER BY NPS_SCORE DESC
```

---

## Reglas de Operación

1. **SIEMPRE usar `COUNT(DISTINCT TURNO_NUMERO)` para contar turnos** (hay duplicados en el DWH)
2. Siempre reemplazar `{{FECHA_INICIO}}` y `{{FECHA_FIN}}` con fechas reales en formato `dd/mm/yyyy`
3. Si piden datos de muchas filas, agregar agregaciones (COUNT, SUM, AVG)
4. Indicar siempre el período consultado en la respuesta
5. Si hay más de 20 filas, mostrar top 10 y mencionar que hay más
6. Para NPS, usar `NULLIF(COUNT(NPS_VALORACION), 0)` para evitar división por cero

---

## Exclusiones Aplicadas

### Estados excluidos
- ERROR
- NO TOMAR EN CUENTA (SE DIO DA BAJA AGENDA)

### Seguros excluidos
- CAF Control Medico Al Personal
- Accidentes Laborales Caf
- **Nota:** Se mantienen los NULL (turnos sin paciente asignado)

### Especialidades excluidas (no son consultas)
- **Imágenes:** Resonancia, Tomografía, Ecografías, Rayos X, Mamografía, Densitometría
- **Laboratorio:** Laboratorios, Recepción de laboratorio, Toma de muestras
- **Procedimientos diagnósticos:** Electrocardiograma, Holter, MAPA, Ergometría, Polisomnografía
- **Otros:** Vacunas, ADM. Vitalia, Fisioterapia, Emergencias/Urgencias
