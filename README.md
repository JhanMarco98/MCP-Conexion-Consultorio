# MCP Clínica Foianini - Skills

Configuración de MCP Oracle y Skills para consultas de Clínica Foianini.

## Estructura

```
.claude/commands/
├── sk-consultorio.md    ← Área Consultorios
├── sk-general.md        ← Producción general de la clínica
└── (futuras áreas...)
```

## Skills Disponibles

| Skill | Comando | Descripción |
|-------|---------|-------------|
| Consultorio | `/sk-consultorio` | Turnos, ausencias, NPS del área consultorios |
| General | `/sk-general` | Producción general: ingresos, servicios, lab, imagen, fee, descuentos |

## Instalación

1. Instalar el MCP Oracle: `docs/MCP_Oracle_Instalacion.md`
2. Clonar este repositorio
3. Usar los comandos `/sk-*` en Claude Code

## Ejemplos

```
/sk-consultorio
Dame los turnos presentes de enero a junio 2025
```
