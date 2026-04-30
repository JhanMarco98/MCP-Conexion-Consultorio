# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Propósito

Repositorio de Skills y configuración MCP para consultas de Clínica Foianini contra el DWH Oracle.

## Skills Disponibles

| Skill | Comando | Uso |
|-------|---------|-----|
| Consultorio | `/sk-consultorio` | Turnos, ausencias, ocupación, NPS del área consultorios |
| General | `/sk-general` | Producción general: ingresos, servicios, lab, imagen, fee, descuentos |

## Arquitectura

```
.claude/commands/
├── sk-consultorio.md    ← Turnos y NPS de consultorios
├── sk-general.md        ← Producción general (9 vistas + CTE PRIMER_INGRESO)
docs/
└── MCP_Oracle_Instalacion.md  ← Guía de instalación del MCP server
```

Los skills funcionan como asistentes de datos que:
1. Interpretan preguntas del usuario
2. Eligen la query base correcta según el skill
3. Adaptan fechas y filtros según el contexto
4. Ejecutan contra Oracle vía `oracle_query`
5. Responden con resultados claros

## Herramientas MCP Oracle

- `oracle_query` - Ejecutar SELECTs (máx 1000 filas por defecto)
- `oracle_execute` - DDL/DML (CREATE, INSERT, UPDATE, DELETE)
- `oracle_list_tables` - Listar tablas
- `oracle_describe` - Ver estructura de tablas
- `oracle_test_connection` - Probar conexión

## Reglas Críticas de Datos

1. **Turnos:** SIEMPRE contar como `COUNT(DISTINCT TURNO_NUMERO)` - Existen ~97 duplicados
2. **Atenciones:** Contar como `COUNT(DISTINCT ENC_ATENCION_ID)` para evitar duplicados por línea de detalle
3. Fechas en formato Oracle: `TO_DATE('dd/mm/yyyy','dd/mm/yyyy')`
4. Para NPS usar `NULLIF(COUNT(NPS_VALORACION), 0)` para evitar división por cero
5. Solo ~1.3% de turnos tienen respuesta NPS

## Tablas y Vistas Principales

| Objeto | Tipo | Skill | Uso |
|--------|------|-------|-----|
| TURNOS_TODOS_MYSQL | Vista | sk-consultorio | Turnos, citas, demanda |
| TURNOS_MYSQL | Vista | sk-consultorio | Turnos con filtros (CTE MEDICOS_CON_PRESENCIA) |
| ENC_ATENCION | Vista | ambos | Cabecera de atención (paciente, seguro, tipo) |
| ENC_ATENCION_DETALLE | Vista | sk-general | Detalle: prestaciones, montos, fee, clases |
| ENC_ENCUESTA_USUARIO | Vista | ambos | Respuestas NPS (LEFT JOIN en sk-general) |
| ENC_ATENCION_OBSERVACION | Vista | sk-general | Observaciones/seguimiento NPS (última por encuesta) |
| ENC_TIPO_ATENCION | Vista | sk-general | Catálogo tipos de atención (LEFT JOIN) |
| ENC_TIPO_GRUPO | Vista | sk-general | Tipo + Lugar de atención (CAF, Virtual, Beauty Plaza, etc.) |
| ENC_CLASE | Vista | sk-general | Catálogo clases de ítem |
| SEGURO_PLAN_MYSQL | Vista | sk-general | Planes por seguro (JOIN por COD_SEGURO + COD_SEGURO_PLAN) |
| CIE10_MYSQL | Vista | sk-general | Catálogo diagnósticos CIE-10 |
| CIAP2_MYSQL | Vista | sk-general | Catálogo motivos CIAP-2 |
| ESPECIALIDADES_MYSQL | Vista | sk-consultorio | Catálogo de especialidades |

## Agrupaciones de Datos

### sk-consultorio (Turnos)
- **GRUPO_ESTADO**: PRESENTE, AUSENTE, NO VENDIDO, BLOQUEADO
- **GRUPO_ESPECIALIDAD**: Especialidades consolidadas (normaliza subespecialidades)
- **NPS_CATEGORIA**: PROMOTOR (9-10), NEUTRO (7-8), DETRACTOR (0-6)

### sk-general (Producción)
- **GRUPO_TIPO_ATENCION**: CONSULTA EXTERNA, HOSPITALIZACION, EMERGENCIAS, URGENCIAS, LABORATORIOS, IMAGENOLOGIA, VACUNACION
- **CLASE (ENC_CLASE_ID)**: HONORARIOS MEDICOS (1), SERVICIOS CLINICA (2), SERVICIOS EXTERNOS (3), LABORATORIO (4), MEDICAMENTO (5), DESCARTABLE (6)

### Compartido
- **GRUPO_SEGURO**: VITALIA, ALIANZA, BISA, NACIONAL, PARTICULAR, OTROS

## Exclusiones Estándar

**Estados:** ERROR, NO TOMAR EN CUENTA (SE DIO DA BAJA AGENDA)

**Seguros:** CAF Control Medico Al Personal, Accidentes Laborales Caf (mantener NULL)

**Especialidades no-consulta:** Imágenes (resonancia, tomografía, rayos X), laboratorio, procedimientos diagnósticos (ECG, holter, MAPA), vacunas, fisioterapia, emergencias
